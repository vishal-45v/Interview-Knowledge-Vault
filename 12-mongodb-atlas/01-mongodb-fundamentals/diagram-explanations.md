# Chapter 01 — MongoDB Fundamentals: Diagram Explanations

---

## Diagram 1: MongoDB Document Structure

```
MongoDB Document (BSON)
═══════════════════════════════════════════════════════════════════
{
  _id:        ObjectId("64f1a2b3c4d5e6f7a8b9c0d1")   ← Primary Key (auto)
  │
  ├─ name:    "Alice Johnson"                          ← String (BSON type 2)
  ├─ age:     32                                       ← Int32  (BSON type 16)
  ├─ balance: NumberDecimal("1500.75")                 ← Decimal128 (BSON type 19)
  ├─ active:  true                                     ← Boolean (BSON type 8)
  ├─ created: ISODate("2024-01-15T10:30:00Z")          ← Date (BSON type 9)
  │
  ├─ address: {                        ← Embedded Document (BSON type 3)
  │    street: "123 Main St"
  │    city:   "Austin"
  │    zip:    "78701"
  │  }
  │
  └─ tags: [                           ← Array (BSON type 4)
       "nodejs"
       "mongodb"
       "typescript"
     ]
}

Max document size: 16 MB
_id field: required, immutable, uniquely indexed
```

---

## Diagram 2: RDBMS vs MongoDB Data Model

```
RELATIONAL DATABASE (PostgreSQL)
════════════════════════════════════════════════════════════════════

TABLE: users
┌────────┬───────────┬──────────────────────┬─────┐
│ id     │ name      │ email                │ age │
├────────┼───────────┼──────────────────────┼─────┤
│ 1      │ Alice     │ alice@example.com    │ 32  │
│ 2      │ Bob       │ bob@example.com      │ 28  │
└────────┴───────────┴──────────────────────┴─────┘

TABLE: addresses
┌────────┬─────────┬─────────────────┬────────┬───────┐
│ id     │ user_id │ street          │ city   │ zip   │
├────────┼─────────┼─────────────────┼────────┼───────┤
│ 1      │ 1       │ 123 Main St     │ Austin │ 78701 │
│ 2      │ 2       │ 456 Oak Ave     │ Dallas │ 75201 │
└────────┴─────────┴─────────────────┴────────┴───────┘

TABLE: phones
┌────────┬─────────┬──────────────────┬──────────┐
│ id     │ user_id │ number           │ type     │
├────────┼─────────┼──────────────────┼──────────┤
│ 1      │ 1       │ +1-512-555-0100  │ mobile   │
│ 2      │ 1       │ +1-512-555-0200  │ work     │
└────────┴─────────┴──────────────────┴──────────┘

To get Alice with address + phones:
SELECT u.*, a.street, a.city, p.number
FROM users u
JOIN addresses a ON u.id = a.user_id
JOIN phones p ON u.id = p.user_id
WHERE u.id = 1;

3 tables, 2 JOINs, schema changes require ALTER TABLE


MONGODB (Document Model)
════════════════════════════════════════════════════════════════════

COLLECTION: users
{
  _id: ObjectId("..."),
  name: "Alice",
  email: "alice@example.com",
  age: 32,
  address: {                    ← Embedded (co-located data)
    street: "123 Main St",
    city: "Austin",
    zip: "78701"
  },
  phones: [                     ← Embedded array (bounded)
    { number: "+1-512-555-0100", type: "mobile" },
    { number: "+1-512-555-0200", type: "work" }
  ]
}

To get Alice with everything:
db.users.findOne({ _id: ObjectId("...") })

1 collection, 0 JOINs, add new fields to any document instantly

Trade-off: Data duplication if address/phone appears in multiple collections
```

---

## Diagram 3: Replica Set Architecture

```
MONGODB REPLICA SET (3-member PSS)
════════════════════════════════════════════════════════════════════

                    ┌──────────────────────┐
   APPLICATION      │      Application     │
     (Driver)       │  MongoClient(srv://  │
                    │  host1,host2,host3)  │
                    └──────┬───────────────┘
                           │  Writes → PRIMARY
                           │  Reads → configured by readPref
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
  │   PRIMARY   │   │  SECONDARY  │   │  SECONDARY  │
  │ mongo1:27017│   │ mongo2:27017│   │ mongo3:27017│
  │             │   │             │   │             │
  │ Accepts:    │   │ Accepts:    │   │ Accepts:    │
  │ • Reads     │   │ • Reads     │   │ • Reads     │
  │ • Writes    │   │   (if configured)            │
  │             │   │             │   │             │
  │ local.      │   │ local.      │   │ local.      │
  │ oplog.rs    │──►│ oplog.rs    │   │ oplog.rs    │
  │ (writes log)│   │ (replicated)│   │ (replicated)│
  └─────────────┘   └─────────────┘   └─────────────┘
         │                 ▲                 ▲
         │    OPLOG        │                 │
         │  REPLICATION    │                 │
         └─────────────────┴─────────────────┘
           Secondaries tail primary's oplog
           (async — usually <1 second lag)

ELECTION PROCESS (when primary unavailable):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
t=0:    Primary becomes unreachable
t=10s:  Secondary1 detects missed heartbeats
        Secondary1 → CANDIDATE state, increments term
t=10s:  Secondary1 requests vote from Secondary2
        Secondary2 votes YES (S1 has up-to-date oplog)
t=10s:  Secondary1 receives majority (2/3 votes: self + S2)
t=10s:  Secondary1 → PRIMARY state
t=10s:  Writes resume
t+?:    Old primary rejoins as SECONDARY, syncs oplog

KEY NUMBERS:
• Heartbeat interval: 2 seconds
• Election timeout: 10 seconds
• Minimum members: 3 (or 2 data + 1 arbiter)
• Voting members: up to 7 (only 7 vote, rest are non-voting)
```

---

## Diagram 4: ObjectId Byte Structure

```
OBJECTID: 64f1a2b3 c4d5e6 f7a8b9
══════════════════════════════════════════════

Hex:    6  4  f  1  a  2  b  3  c  4  d  5  e  6  f  7  a  8  b  9  c  0  d  1
Bytes:  [  byte 0  ] [  byte 1  ] [  byte 2  ] [  byte 3  ]  ← UNIX TIMESTAMP (seconds)
        [  byte 4  ] [  byte 5  ] [  byte 6  ] [  byte 7  ] [  byte 8  ]  ← RANDOM VALUE
        [  byte 9  ] [  byte 10 ] [  byte 11 ]  ← INCREMENTING COUNTER

Breakdown:
┌──────────────────┬──────────────────────────┬──────────────────────┐
│  4 bytes         │  5 bytes                 │  3 bytes             │
│  Unix timestamp  │  Random (per process)    │  Counter (per sec)   │
│  (seconds)       │  Machine + PID entropy   │  0-16,777,215        │
├──────────────────┼──────────────────────────┼──────────────────────┤
│ "64f1a2b3"       │ "c4d5e6f7a8"             │ "b9c0d1"             │
│ = 1693526707     │                          │                      │
│ = 2023-09-01     │                          │                      │
│   00:05:07 UTC   │                          │                      │
└──────────────────┴──────────────────────────┴──────────────────────┘

Properties:
• 12 bytes = 24 hex characters
• Globally unique (timestamp + random + counter)
• Sortable: newer ObjectIds > older ObjectIds
• Timestamp precision: SECONDS (not milliseconds)
• Generated CLIENT-SIDE by the driver
• Can be decoded: ObjectId("...").getTimestamp()
```

---

## Diagram 5: Write Concern Acknowledgment Flow

```
WRITE CONCERN LEVELS
════════════════════════════════════════════════════════════════════

APPLICATION                  PRIMARY         SECONDARY 1    SECONDARY 2
    │                           │                 │               │
    │──── insertOne() ─────────►│                 │               │
    │                           │                 │               │
    │  w:0 (fire and forget)    │                 │               │
    │◄── immediate return ──────│                 │               │
    │    (before write!)        │                 │               │
    │                           │                 │               │
    │  w:1 (primary ack)        │                 │               │
    │                           │── writes to ───►│               │
    │                           │   memory+journal│               │
    │◄──────── ack ─────────────│                 │               │
    │    (primary confirmed)    │                 │               │
    │                           │                 │               │
    │  w:majority               │                 │               │
    │                           │── replicate ───►│               │
    │                           │── replicate ───────────────────►│
    │                           │                 │               │
    │                           │◄── ack ─────────│               │
    │◄──────── ack ─────────────│  (majority = 2  │               │
    │    (majority confirmed)   │   out of 3      │               │
    │                           │   acknowledged) │               │
    │                           │                 │               │

DURABILITY GUARANTEE:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
w:0        → No guarantee. Data may be lost.
w:1        → Survives if primary stays up. Lost if primary crashes before replication.
w:majority → Survives primary failure. Majority of nodes have the data.
w:majority
+ j:true   → Maximum: data on disk (journal) on majority of nodes.

LATENCY IMPACT (relative):
w:0    ──► [  ] nearly 0ms
w:1    ──► [███] ~1ms local
w:maj  ──► [██████] ~5-20ms (depends on network to secondaries)
w:all  ──► [████████████] slowest (waits for EVERY member including slow ones)
```

---

## Diagram 6: MongoDB vs RDBMS — Scaling Model

```
RDBMS SCALING (Vertical)
════════════════════════════════════════════════════════════════════

Small DB               Medium DB              Large DB
┌───────────┐         ┌───────────┐         ┌───────────┐
│  4 CPU    │  →→→→  │  32 CPU   │  →→→→  │ 128 CPU   │
│  16 GB RAM│         │ 128 GB RAM│         │ 512 GB RAM│
│  1 TB SSD │         │  8 TB SSD │         │  32 TB SSD│
└───────────┘         └───────────┘         └───────────┘
    $500/mo               $5,000/mo              Hits hardware ceiling
                                                 $50,000/mo+
                                                 Eventually: NO BIGGER BOX EXISTS

MONGODB SCALING (Horizontal Sharding)
════════════════════════════════════════════════════════════════════

┌──────────────────────────────────────────────┐
│                 mongos router                │  ← Clients connect here
└──────────────────────────────────────────────┘
         │              │              │
         ▼              ▼              ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│   SHARD 1    │ │   SHARD 2    │ │   SHARD 3    │  ← Add more shards
│  (Replica    │ │  (Replica    │ │  (Replica    │     as data grows
│   Set)       │ │   Set)       │ │   Set)       │
│  userId A-F  │ │  userId G-N  │ │  userId O-Z  │  ← Ranged sharding
│              │ │              │ │              │
│ $1,000/mo    │ │ $1,000/mo    │ │ $1,000/mo    │
└──────────────┘ └──────────────┘ └──────────────┘

As data grows: add SHARD 4, SHARD 5... linearly scale cost and capacity

Config Servers (3-node replica set): stores shard metadata, chunk ranges
```

---

## Diagram 7: CAP Theorem Positioning

```
                    CONSISTENCY
                    (All nodes see same data)
                          ▲
                          │
                          │
           MongoDB ──────►●  (w:majority, r:majority)
           (default)      │  Strong consistency,
                          │  May sacrifice availability
                          │  during partitions
                    ──────┼──────
                   /      │      \
                  /       │       \
         CP /           │           \ CA
           /             │             \
          ▼              │              ▼
PARTITION              (RDBMS single
TOLERANCE             node, no partition)
(Network split
 resilience)
          \              │              /
           \             │             /
         AP \           │           /
             \           │          /
              ▼          │         ▼
               Cassandra ●  AVAILABILITY
               DynamoDB     (Always responds)
               (relaxed
                consistency)

MongoDB tuning:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
More CP (strict): w:majority + r:majority + r:linearizable
Balanced:         w:1 + r:local (default)
More AP (relaxed): w:1 + readPref:secondary + r:available

Note: No system is purely one corner — these are trade-off dials.
```
