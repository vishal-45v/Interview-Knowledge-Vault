# Chapter 09 — Replication & Sharding: Diagram Explanations

---

## Diagram 1: Replica Set Write and Replication Flow

```
REPLICA SET: WRITE AND REPLICATION FLOW
══════════════════════════════════════════════════════════════════════

APPLICATION WRITE (w: "majority"):
──────────────────────────────────────────────────────────────────────

Application
    │
    │  db.orders.insertOne({ customerId: "c1", amount: 99.99 })
    ▼
┌──────────────────────────────────────────────────────────────────┐
│  mongos (if sharded) or direct connection                        │
│  → Routes to PRIMARY node                                        │
└──────────────────────────────────────────────────────────────────┘
    │
    ▼
┌──────────────────────────────────────────────────────────────────┐
│  NODE 1: PRIMARY (node1:27017)                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 1. Write to WiredTiger storage engine                    │   │
│  │ 2. Append to oplog:                                      │   │
│  │    { ts: Timestamp(T,1), op:"i", ns:"db.orders",        │   │
│  │      o: { _id: ObjectId, customerId: "c1", ... } }      │   │
│  │ 3. Acknowledge receipt                                   │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
    │                           │
    │ heartbeat + oplog sync    │ heartbeat + oplog sync
    │ (async, continuous)       │ (async, continuous)
    ▼                           ▼
┌────────────────────────┐  ┌────────────────────────┐
│  NODE 2: SECONDARY     │  │  NODE 3: SECONDARY     │
│  ┌──────────────────┐  │  │  ┌──────────────────┐  │
│  │ Tails primary    │  │  │  │ Tails primary    │  │
│  │ oplog            │  │  │  │ oplog            │  │
│  │ Applies entries  │  │  │  │ Applies entries  │  │
│  │ in order         │  │  │  │ in order         │  │
│  └──────────────────┘  │  │  └──────────────────┘  │
│  "Ack to primary"      │  │  "Ack to primary"       │
└────────────────────────┘  └────────────────────────┘
    │
    ▼ Primary receives acks from 1 secondary (majority of 3 = 2 nodes)
Application receives SUCCESS (w: "majority" satisfied)

WRITE CONCERN ACKNOWLEDGMENT TIMING:
──────────────────────────────────────────────────────────────────────
w:0           → Fire and forget, no ack
w:1           → Primary writes → immediate ack (~1ms)
w:majority    → Primary + 1 secondary ack → slightly later (~5-50ms)
w:3           → All 3 nodes ack → latest secondary determines latency
j:true        → Add: journal flush before ack (adds ~1-5ms)
```

---

## Diagram 2: Election Process State Machine

```
REPLICA SET ELECTION PROCESS
══════════════════════════════════════════════════════════════════════

NORMAL STATE:
┌─────────────────────────────────────────────────────────────────┐
│  node1: PRIMARY  ──heartbeat──▶  node2: SECONDARY               │
│        ▲                               │                        │
│        └───────────heartbeat───────────┘                        │
│                                                                  │
│  node1: PRIMARY  ──heartbeat──▶  node3: SECONDARY               │
│        ▲                               │                        │
│        └───────────heartbeat───────────┘                        │
│                                                                  │
│  Heartbeats every 2 seconds, timeout = 10 seconds               │
└─────────────────────────────────────────────────────────────────┘

TRIGGER EVENT: node1 crashes (T=0)
──────────────────────────────────────────────────────────────────────

T=0:   node1 crashes
T=2s:  node2 and node3 send heartbeat to node1 → no response
T=4s:  heartbeat again → no response
T=6s:  heartbeat again → no response
T=8s:  heartbeat again → no response
T=10s: electionTimeoutMS exceeded → node2 and node3 independently decide:
       "Primary is gone. I should call an election."

ELECTION (T=10s → T=12s):
──────────────────────────────────────────────────────────────────────

node2 (priority:1, optime:T_100) calls election:
    │
    ├──▶ Votes for ITSELF (term 5 → 6)
    ├──▶ Sends voteRequest to node3:
    │      { candidateId: "node2", term: 6, optime: T_100 }
    │
node3 evaluates voteRequest:
    │  Is node2's optime >= my optime? YES (T_100 >= T_100)
    │  Is node2's priority >= my priority? YES (1 >= 1)
    │  Have I already voted in term 6? NO
    └──▶ Grants vote to node2

node2 has 2 votes (itself + node3) = majority of 3
node2 transitions to PRIMARY

APPLICATION (driver) detects new topology:
    │  serverSelectionTimeoutMS: 30000 gives 30s to find new primary
    └──▶ Routes subsequent writes to node2

T=60s: node1 comes back online
    │  Discovers node2 is primary with higher term
    │  Rolls back any writes that weren't majority-replicated
    └──▶ Transitions to SECONDARY, starts oplog catch-up

ELECTION RULES:
──────────────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────┐
│ Rule 1: Candidate must have optime >= voter's optime         │
│ Rule 2: Candidate must have priority > 0                     │
│ Rule 3: Each node grants max 1 vote per term                 │
│ Rule 4: Need majority of all votes (not just available)      │
│ Rule 5: Step-down inhibit period prevents immediate re-elect │
└─────────────────────────────────────────────────────────────┘
```

---

## Diagram 3: Sharded Cluster Architecture

```
SHARDED CLUSTER: FULL ARCHITECTURE
══════════════════════════════════════════════════════════════════════

APPLICATION TIER
──────────────────────────────────────────────────────────────────────
┌────────────────┐  ┌────────────────┐  ┌────────────────┐
│  App Server 1  │  │  App Server 2  │  │  App Server 3  │
└────────┬───────┘  └────────┬───────┘  └────────┬───────┘
         │                   │                   │
         └───────────────────┴───────────────────┘
                             │
                             ▼
ROUTING TIER (mongos — stateless, can run multiple)
──────────────────────────────────────────────────────────────────────
┌──────────────────────┐  ┌──────────────────────┐
│  mongos 1            │  │  mongos 2            │
│  (query router)      │  │  (query router)      │
│  Caches chunk map    │  │  Caches chunk map    │
│  from config servers │  │  from config servers │
└──────────┬───────────┘  └──────────────────────┘
           │ (read chunk routing table from config)
           ▼
CONFIG SERVER TIER (3-node replica set)
──────────────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────────────────────┐
│  Config Server Replica Set (csRS)                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │ Config Primary  │  │ Config Secondary│  │ Config Secondary│ │
│  │                 │──│                 │──│                 │ │
│  │ Stores:         │  │                 │  │                 │ │
│  │ - Chunk ranges  │  │                 │  │                 │ │
│  │ - Shard map     │  │                 │  │                 │ │
│  │ - DB metadata   │  │                 │  │                 │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
           │
           ▼
SHARD TIER (each shard is a replica set)
──────────────────────────────────────────────────────────────────────
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  SHARD 1 (rs1)  │  │  SHARD 2 (rs2)  │  │  SHARD 3 (rs3)  │
│  P + S + S      │  │  P + S + S      │  │  P + S + S      │
│                 │  │                 │  │                 │
│ Chunks:         │  │ Chunks:         │  │ Chunks:         │
│ customerId:     │  │ customerId:     │  │ customerId:     │
│  A → G          │  │  H → N          │  │  O → Z          │
└─────────────────┘  └─────────────────┘  └─────────────────┘

QUERY FLOW:
──────────────────────────────────────────────────────────────────────
db.orders.find({ customerId: "johnson_123" })
    │
    ▼ mongos checks chunk routing table in config servers
    │ "johnson_123" starts with J → between H-N → SHARD 2
    │
    ▼ mongos sends query to SHARD 2 only (TARGETED QUERY)
    │
    ▼ Returns results

db.orders.find({ amount: { $gt: 1000 } })  ← no shard key!
    │
    ▼ mongos cannot determine which shard
    │
    ▼ Sends query to SHARD 1, SHARD 2, SHARD 3 (SCATTER-GATHER)
    │
    ▼ mongos merges and returns combined results
```

---

## Diagram 4: Shard Key Impact on Write Distribution

```
SHARD KEY COMPARISON: WRITE DISTRIBUTION
══════════════════════════════════════════════════════════════════════

SCENARIO: 1000 inserts, collection sharded across 4 shards

BAD SHARD KEY: { _id: 1 } with ObjectId (monotonically increasing)
──────────────────────────────────────────────────────────────────────

ObjectId values: 6615a1e1... 6615a1e2... 6615a1e3... (always increasing)

Chunk distribution at T=0:
Shard1: [MinKey → 60000000]   ← EMPTY
Shard2: [60000000 → 65000000] ← EMPTY
Shard3: [65000000 → 6a000000] ← EMPTY
Shard4: [6a000000 → MaxKey]   ← ALL NEW WRITES GO HERE! ← HOT SHARD

Result after 1000 inserts:
Shard1: ░░░░░░░░░░ (0 new writes)
Shard2: ░░░░░░░░░░ (0 new writes)
Shard3: ░░░░░░░░░░ (0 new writes)
Shard4: ██████████ (1000 new writes) ← HOTSPOT

GOOD SHARD KEY: { customerId: "hashed" }
──────────────────────────────────────────────────────────────────────

customerId values: "alice", "bob", "carol", "dave", "emma", "frank"...
After hashing: random-looking numeric values distributed evenly

Chunk distribution at T=0:
Shard1: [MinKey → -4611686...]   ← 25% range
Shard2: [-4611686... → 0]        ← 25% range
Shard3: [0 → 4611686...]         ← 25% range
Shard4: [4611686... → MaxKey]    ← 25% range

Result after 1000 inserts:
Shard1: ████████   (~245 writes)
Shard2: ████████   (~255 writes)
Shard3: ████████   (~252 writes)
Shard4: ████████   (~248 writes)

TRADEOFF TABLE:
──────────────────────────────────────────────────────────────────────
Shard Key            │ Write Dist │ Range Queries │ Point Lookups
─────────────────────┼────────────┼───────────────┼──────────────────
{ _id: 1 } (ObjId)  │ HOTSPOT    │ Fast          │ Fast (targeted)
{ timestamp: 1 }    │ HOTSPOT    │ Fast          │ Scatter-gather
{ field: "hashed" } │ EVEN ✓     │ Scatter-gather│ Fast (targeted)
{ tenantId: 1, ts } │ GOOD*      │ Fast within   │ Fast (targeted)
                    │            │ tenant        │
*good if tenantId has high enough cardinality
```

---

## Diagram 5: Chunk Migration Process

```
CHUNK MIGRATION: HOW BALANCER MOVES DATA
══════════════════════════════════════════════════════════════════════

BEFORE MIGRATION:
Shard1: 50 chunks  ← over capacity (threshold: imbalance > 8)
Shard2: 42 chunks
Shard3: 10 chunks  ← under capacity
Shard4: 8 chunks

Balancer decides: move chunk from Shard1 to Shard3

MIGRATION PHASES:
──────────────────────────────────────────────────────────────────────

Phase 1: CLONE
┌─────────────────────────────────────────────────────────────────┐
│  Shard1 (SOURCE)          │  Shard3 (DESTINATION)              │
│  chunk: {a → g}           │                                    │
│                           │                                    │
│  Copy all documents       │                                    │
│  in range a→g ──────────▶│  Receive and store documents       │
│  (Shard1 still serves     │  (in parallel, not yet active)    │
│   reads/writes during     │                                    │
│   this phase!)            │                                    │
└─────────────────────────────────────────────────────────────────┘

Phase 2: CATCH-UP (apply oplog delta)
┌─────────────────────────────────────────────────────────────────┐
│  Shard1 oplog entries     │  Shard3                            │
│  during clone period:     │                                    │
│  { op: "u", doc: "ada" }──▶ Apply update to "ada"             │
│  { op: "i", doc: "ben" }──▶ Apply insert of "ben"             │
│  { op: "d", doc: "ann" }──▶ Apply delete of "ann"             │
│                           │  Shard3 now identical to Shard1   │
└─────────────────────────────────────────────────────────────────┘

Phase 3: COMMIT (update routing table)
┌─────────────────────────────────────────────────────────────────┐
│  Config Server                                                  │
│  OLD: chunk {a→g} → Shard1                                     │
│  NEW: chunk {a→g} → Shard3   ← atomic update                   │
│                                                                 │
│  mongos caches updated immediately                              │
│  Writes to {a→g} now go to Shard3                              │
└─────────────────────────────────────────────────────────────────┘

Phase 4: CLEANUP
┌─────────────────────────────────────────────────────────────────┐
│  Shard1: deletes the now-migrated chunk data                   │
│  (documents in a→g range removed from Shard1)                  │
│                                                                 │
│  If Phase 4 fails: orphaned documents exist on Shard1          │
│  MongoDB 4.4+ auto-cleans orphans in background                │
│  Manual: db.runCommand({ cleanupOrphaned: "db.coll", ... })    │
└─────────────────────────────────────────────────────────────────┘

AFTER MIGRATION:
Shard1: 49 chunks
Shard2: 42 chunks
Shard3: 11 chunks  ← one chunk added
Shard4: 8 chunks
```

---

## Diagram 6: Zone Sharding for Geographic Data Placement

```
ZONE SHARDING: GEOGRAPHIC DATA ROUTING
══════════════════════════════════════════════════════════════════════

CLUSTER TOPOLOGY:
──────────────────────────────────────────────────────────────────────

                    mongos (global, any region)
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
    ┌───────────┐  ┌───────────┐  ┌───────────┐
    │  ZONE:    │  │  ZONE:    │  │  ZONE:    │
    │ "americas"│  │ "europe"  │  │  "apac"   │
    ├───────────┤  ├───────────┤  ├───────────┤
    │ shard-us1 │  │ shard-eu1 │  │ shard-ap1 │
    │ us-east-1 │  │ eu-west-1 │  │ap-southeast│
    ├───────────┤  ├───────────┤  ├───────────┤
    │ shard-us2 │  │ shard-eu2 │  │ shard-ap2 │
    │ us-west-2 │  │eu-central-│  │ap-northeast│
    └───────────┘  └───────────┘  └───────────┘

SHARD KEY: { region: 1, _id: 1 }
TAG RANGES:
    { region: "US", _id: MinKey } → { region: "US", _id: MaxKey } → "americas"
    { region: "EU", _id: MinKey } → { region: "EU", _id: MaxKey } → "europe"
    { region: "APAC", _id: MinKey } → { region: "APAC", _id: MaxKey } → "apac"

WRITE ROUTING (EU user creating an account):
──────────────────────────────────────────────────────────────────────

EU Application Server
    │
    │  db.users.insertOne({ region: "EU", userId: "user_123", ... })
    ▼
mongos (in EU region)
    │  Check zone: region="EU" → "europe" zone
    │
    ▼  Route to EU shard (shard-eu1 or shard-eu2)
shard-eu1 (eu-west-1)
    │  Write committed in EU data center
    │  Replication to shard-eu2 (eu-central-1) → ~20ms
    │  NO WRITE to Americas or APAC (data stays in EU!)
    ▼  Acknowledge to application: ~5ms total

READ ROUTING (EU user reading their profile):
──────────────────────────────────────────────────────────────────────

EU Application Server
    │  readPreference: "nearest"
    │  db.users.findOne({ region: "EU", userId: "user_123" })
    ▼
mongos → EU shard (nearest) → Returns in ~3ms

COMPLIANCE BENEFIT:
──────────────────────────────────────────────────────────────────────
GDPR requirement: EU user data must not leave EU
Zone sharding PROVABLY enforces this:
- shard-us1, shard-us2 NEVER receive EU data (zone tags prevent it)
- Config server routing table is the compliance record
- Atlas audit logs show all writes go only to EU-tagged shards
```
