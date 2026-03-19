# Chapter 07: Key-Value Stores — Diagram Explanations

## Diagram 1: Cache Patterns Comparison — Request Flow

```
FIVE CACHE PATTERNS — REQUEST FLOW COMPARISON
═══════════════════════════════════════════════════════════════════════════

① CACHE-ASIDE (Lazy Loading)
   App ──► Cache  (HIT)  ──► App
   App ──► Cache  (MISS) ──► App ──► DB ──► App ──► Cache (populate)

② READ-THROUGH
   App ──► Cache  (HIT)  ──► App
   App ──► Cache  (MISS) ──► Cache ──► DB ──► Cache ──► App
              (cache handles miss internally)

③ WRITE-THROUGH
   App ──► Cache ──► DB  (synchronous write to both)
           │
           └── App waits for BOTH cache and DB to confirm

④ WRITE-BEHIND (Write-Back)
   App ──► Cache ──► App  (immediate return)
                 Queue ──► (async) ──► DB
            [FAST but DATA LOSS risk if queue not durable]

⑤ REFRESH-AHEAD (Proactive)
   Background: Timer ──► Cache TTL check ──► DB ──► Cache (before expiry)
   App ──► Cache  (always HIT if refresh is timely)

FAILURE MODE MATRIX:
┌──────────────┬─────────────────────┬──────────────────────┬──────────────┐
│ Pattern      │ Cache Down          │ DB Down              │ Data Loss?   │
├──────────────┼─────────────────────┼──────────────────────┼──────────────┤
│ Cache-Aside  │ Reads bypass cache  │ Cache misses fail    │ No           │
│              │ → DB direct (slow   │ → errors             │              │
│              │   but functional)   │                      │              │
├──────────────┼─────────────────────┼──────────────────────┼──────────────┤
│ Read-Through │ ALL reads fail      │ Cache misses fail    │ No           │
│              │ (cache in critical  │ → errors             │              │
│              │  path)              │                      │              │
├──────────────┼─────────────────────┼──────────────────────┼──────────────┤
│ Write-Through│ ALL writes fail     │ ALL writes fail      │ No           │
│              │                     │                      │              │
├──────────────┼─────────────────────┼──────────────────────┼──────────────┤
│ Write-Behind │ Queued writes lost  │ Queue builds up      │ YES (if      │
│              │ (if queue in-mem)   │ until DB recovers    │ queue not    │
│              │                     │                      │  durable)    │
├──────────────┼─────────────────────┼──────────────────────┼──────────────┤
│ Refresh-Ahead│ No fresh data;      │ Refresh fails;       │ No           │
│              │ old TTLs expire,    │ stale data served    │              │
│              │ then cache is empty │ until DB recovers    │              │
└──────────────┴─────────────────────┴──────────────────────┴──────────────┘
```

**Explanation:**
Cache-aside is the most resilient pattern — cache is purely an optimization. Read-through
is the simplest for application code but creates a hard dependency. Write-behind gives
the best write performance but introduces the only pattern with real data loss risk.
Choose based on your failure tolerance: financial systems should never use write-behind
for critical transaction data; social media feed caches are fine candidates for it.

---

## Diagram 2: Consistent Hashing Ring with Virtual Nodes

```
CONSISTENT HASHING: Physical Nodes vs Virtual Nodes
═══════════════════════════════════════════════════════════════════════════

WITHOUT VIRTUAL NODES (uneven distribution):

        0
      ┌───┐
   315│   │45
      │   │
   270│   │90   ← Server A at 90°
      │   │        Server B at 180°
   225│   │135      Server C at 270°
      └───┘
       180
                    Distribution:
                    A handles: 90° → 180° = 25% of ring
                    B handles: 180° → 270° = 25% of ring
                    C handles: 270° → 90° = 50% of ring ← IMBALANCED

WITH VIRTUAL NODES (150 per physical server):

     Server A's virtual positions: 12°, 37°, 89°, 134°, 198°, 267°, 312°, ...
     Server B's virtual positions:  8°, 45°, 91°, 156°, 203°, 278°, 319°, ...
     Server C's virtual positions: 23°, 67°, 112°, 167°, 221°, 289°, 355°, ...

     ← With 150 virtual nodes each, each server handles ~33% of key space ─►
     (±5% variance at 150 vnodes; ±2% at 300 vnodes)

NODE ADDITION — What moves:
                         ┌──────────────────────────────────────────┐
                         │ Add Server D at 135°                      │
                         │                                           │
                         │ Old: Keys 90°→180° → Server A            │
                         │ New: Keys 90°→135° → Server A (unchanged)│
                         │      Keys 135°→180° → Server D (MOVED)   │
                         │                                           │
                         │ ≈ 25% of Server A's keys move to D       │
                         │ ≈ 0% of Server B's or C's keys move      │
                         └──────────────────────────────────────────┘
                              [Only 1/N of keys move on node change]
```

**Explanation:**
The ring diagram shows why consistent hashing is superior to modulo hashing for distributed
caches. With modulo (key % N servers), adding one server from 3 to 4 causes 75% of all keys
to move (new modulo assignments). With consistent hashing, adding one server causes only
1/N ≈ 25% of keys to move, and only from the adjacent server. Virtual nodes (many positions
per physical server) are the critical addition that ensures even load distribution despite
the pseudo-random placement of physical nodes on the ring.

---

## Diagram 3: etcd Raft Consensus — Log Replication Flow

```
RAFT LOG REPLICATION IN etcd
═══════════════════════════════════════════════════════════════════════════

CLIENT              LEADER (Core-1)     FOLLOWER (Core-2)  FOLLOWER (Core-3)
  │                       │                    │                   │
  │  PUT /key "val"        │                    │                   │
  ├──────────────────────►│                    │                   │
  │                       │                    │                   │
  │                       │── AppendEntries ──►│                   │
  │                       │   [index=42,        │                   │
  │                       │    term=3,          │                   │
  │                       │    entry=PUT /key]  │                   │
  │                       │── AppendEntries ───────────────────────►│
  │                       │                    │                   │
  │                       │◄─── ACK ───────────│                   │
  │                       │                    │── ACK ────────────►│
  │                       │◄──────────────────────────────── ACK ──│
  │                       │                    │                   │
  │                       │  QUORUM REACHED    │                   │
  │                       │  (2 of 3 ACKs)     │                   │
  │                       │  COMMIT index=42   │                   │
  │                       │                    │                   │
  │◄── SUCCESS + bookmark─│                    │                   │
  │    (revision=42)       │                    │                   │
  │                       │── CommitIndex=42 ─►│                   │
  │                       │── CommitIndex=42 ──────────────────────►│

RAFT LOG STATE:
Core-1 (Leader):  [1: SET a=1][2: SET b=2]...[42: PUT /key=val] COMMITTED
Core-2 (Follower):[1: SET a=1][2: SET b=2]...[42: PUT /key=val] COMMITTED
Core-3 (Follower):[1: SET a=1][2: SET b=2]...[42: PUT /key=val] COMMITTED

FAILURE SCENARIO (Core-3 network partition):
Core-1 (Leader):  ...[42: PUT /key][43: PUT /x][44: PUT /y] COMMITTED
Core-2 (Follower):...[42: PUT /key][43: PUT /x][44: PUT /y] COMMITTED
Core-3 (Isolated):...[42: PUT /key]  ← Missing entries 43 and 44

When Core-3 reconnects:
Core-1 sends entries 43, 44 to Core-3 → Core-3 applies them → back in sync
(Raft leader tracks nextIndex per follower for exactly this reconciliation)
```

**Explanation:**
The Raft log replication diagram shows why etcd guarantees no data loss as long as a quorum
is available. The leader does not commit an entry until a majority acknowledges it. The
bookmark (revision) returned to the client is the Raft log index of the committed entry.
When a client presents this bookmark for a subsequent read, the Read Replica (or Follower)
must apply all log entries up to that index before serving the read — this is the mechanical
basis for etcd's causal consistency guarantee.

---

## Diagram 4: Redis Cluster Architecture — Hash Slot Distribution

```
REDIS CLUSTER: 16,384 HASH SLOTS ACROSS 3 MASTER SHARDS
═══════════════════════════════════════════════════════════════════════════

Total: 16,384 hash slots (CRC16(key) % 16384)

Master-1 ─┬─ Slots 0–5460       Replica-1a  Replica-1b
  │         │  (33% of keyspace)  (async copy) (async copy)
  │         │
Master-2 ─┬─ Slots 5461–10922   Replica-2a  Replica-2b
  │         │  (33% of keyspace)
  │         │
Master-3 ─┬─ Slots 10923–16383  Replica-3a  Replica-3b
            │  (34% of keyspace)

HASH TAG ROUTING:
  Key: "user:1001:profile"       → CRC16("user:1001:profile") % 16384 = 7834
                                  → Routes to Master-2 (slots 5461-10922)

  Key: "{user:1001}:profile"     → CRC16("user:1001") % 16384 = 5649
  Key: "{user:1001}:settings"    → CRC16("user:1001") % 16384 = 5649
  Key: "{user:1001}:cart"        → CRC16("user:1001") % 16384 = 5649
  All three → Master-2 (same slot) → MGET works! Lua scripts work!

CROSS-SLOT ERROR (no hash tags):
┌─────────────────────────────────────────────────────────────────────────┐
│  redis-cli -c MGET user:1001 user:1002 user:1003                        │
│  > (error) CROSSSLOT Keys in request don't hash to the same slot        │
│                                                                         │
│  Fix: MGET {user:1001} {user:1002} {user:1003}                         │
│  But: These hash to DIFFERENT slots (each user hashes independently)   │
│  True fix: Pipeline individual GETs across slots                        │
└─────────────────────────────────────────────────────────────────────────┘

RESHARDING (adding Master-4):
Before: Master-1: 0-5460, Master-2: 5461-10922, Master-3: 10923-16383
After:  Master-1: 0-4095, Master-2: 5461-9215,  Master-3: 10923-15359
        Master-4: 4096-5460 + 9216-10922 + 15360-16383  (migrated slots)

During migration: CLUSTER SETSLOT + MIGRATE commands move keys
Client sees: MOVED 4097 master-4:6379  (transparent redirect)
```

**Explanation:**
Redis Cluster's 16,384 hash slots are the atomic unit of redistribution. Each key maps
to a specific slot, and each slot is owned by exactly one master (during normal operation).
Hash tags ({}) let you force related keys to the same slot, enabling multi-key operations.
The tradeoff: if you put too many keys in one hash tag, you create a hot slot that
defeats the purpose of sharding. The MOVED redirect mechanism allows clients to cache
slot-to-master mappings and route directly, minimizing latency from redirects.

---

## Diagram 5: Cache Stampede Prevention — Three Strategies Compared

```
CACHE STAMPEDE: Scenario Setup
═══════════════════════════════════════════════════════════════════════════

Time T=0: Cache key "product:hot-item" expires
T=0 to T+0.1s: 10,000 simultaneous requests for "product:hot-item"

WITHOUT PROTECTION:
Request  ──► Cache (MISS) ──► DB query starts
Request  ──► Cache (MISS) ──► DB query starts
Request  ──► Cache (MISS) ──► DB query starts
... × 10,000
                              DB receives 10,000 concurrent queries
                              DB CPU → 100%, response time → 30s
                              RESULT: Cascading failure

STRATEGY 1: MUTEX LOCK (Thundering Herd Serialization)
Request-1 ──► Cache MISS ──► Acquire lock (SUCCESS) ──► DB query ──► Cache fill
Request-2 ──► Cache MISS ──► Acquire lock (FAIL)    ──► Wait 500ms ──► Cache hit
Request-3 ──► Cache MISS ──► Acquire lock (FAIL)    ──► Wait 500ms ──► Cache hit
...× 10,000                                              Wait + retry
RESULT: 1 DB query. All others wait. Adds 500ms latency to 9,999 requests.

STRATEGY 2: STALE-WHILE-REVALIDATE (Serve Stale, Refresh Background)
T=-1s: Set stale flag before TTL expires: "product:hot-item:stale" = old value
T=0: Cache expires
Request-1 ──► Fresh cache MISS ──► Stale cache HIT ──► Return stale + trigger async refresh
Request-2 ──► Fresh cache MISS ──► Stale cache HIT ──► Return stale (no extra DB query)
...× 10,000  → All return stale immediately; 1 background DB query
T=+0.5s: Background refresh completes; all subsequent requests get fresh value
RESULT: 1 DB query. Zero added latency. 0.5s of stale data (acceptable for most cases).

STRATEGY 3: PROBABILISTIC EARLY EXPIRATION (XFetch)
T=-30s: Random subset of requests detect "should_early_fetch = True"
        (probability increases as expiry approaches)
Request-A ──► Cache HIT but triggers early refresh
Background DB query runs → Cache refreshed
T=0: Key was already refreshed 30s ago — NO EXPIRY EVENT
RESULT: 0 added latency. 0 stale data. Cache never actually expires during traffic.
Key expires only during LOW traffic windows when probability is low.
```

**Explanation:**
The three strategies represent a spectrum of complexity vs. benefit. Mutex lock is simple
to implement but introduces latency spikes for all waiting requests — acceptable for low
read volume on expensive-to-rebuild items (e.g., machine learning model output). Stale-
while-revalidate is more complex but eliminates the wait penalty at the cost of briefly
serving stale data — ideal for product catalogs, news feeds, or any data where 1-2 seconds
of staleness is acceptable. Probabilistic early expiration (XFetch) is the most
sophisticated: it prevents expiry events from occurring during peak traffic by proactively
refreshing before expiry, weighted by how expensive the item is to rebuild.
