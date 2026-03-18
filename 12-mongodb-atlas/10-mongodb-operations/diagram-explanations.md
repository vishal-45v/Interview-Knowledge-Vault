# Chapter 10 — MongoDB Operations: Diagram Explanations

---

## Diagram 1: WiredTiger Write Path and Crash Recovery

```
WIREDTIGER: WRITE PATH AND CRASH RECOVERY
══════════════════════════════════════════════════════════════════════

WRITE PATH:
──────────────────────────────────────────────────────────────────────

Application Write
    │
    │  db.orders.insertOne({ customerId: "c1", amount: 99.99 })
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  WiredTiger Cache (in RAM)                                      │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Dirty Pages:                                              │ │
│  │ page_0x5a3f: { _id: ObjectId, customerId: "c1", ... }   │ │
│  │ (DIRTY = modified but not yet on disk)                   │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
    │
    │ Simultaneously: Write to Journal (Write-Ahead Log)
    │ (every 100ms, or immediately with j:true)
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Journal (WAL) on Disk                                          │
│  /data/db/journal/                                              │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Entry: { ts: T1, op: "insert", doc: { ... } }            │ │
│  │ Entry: { ts: T2, op: "update", ... }                     │ │
│  │ (sequential append-only log — fast writes)               │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
    │
    │ Every 60 seconds: CHECKPOINT (flush dirty pages to data files)
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Data Files on Disk (durable)                                   │
│  /data/db/collection-0-...wt                                    │
│  /data/db/index-0-...wt                                         │
│  (after checkpoint: dirty pages = clean pages)                  │
└─────────────────────────────────────────────────────────────────┘

CRASH RECOVERY SCENARIO:
──────────────────────────────────────────────────────────────────────

T=0:   Checkpoint completed (data files consistent to T=0)
T=1s:  Write A: Insert order_001  → Journal entry written
T=30s: Write B: Update order_002  → Journal entry written
T=45s: Write C: Insert order_003  → Journal entry written
T=55s: CRASH (10 seconds before next checkpoint!)

Data files on disk: consistent to T=0 (missing writes A, B, C)
Journal on disk: contains entries for A, B, C

RECOVERY (on restart):
    │
    │ 1. MongoDB reads last checkpoint timestamp from data files
    │ 2. Scans journal for entries AFTER last checkpoint
    │ 3. Replays journal entries A, B, C in order
    │ 4. Data files now consistent to T=45s
    └──▶ MongoDB starts normally, no data loss!

LATENCY IMPLICATIONS:
──────────────────────────────────────────────────────────────────────
writeConcern: { j: false }  → Ack after cache write (~0.1ms)
                               Risk: up to 100ms of data loss on crash
writeConcern: { j: true }   → Ack after journal flush (~1-5ms)
                               Risk: zero data loss on single-node crash
writeConcern: { w: "majority", j: true }
                             → Ack after journal flush on majority
                               Risk: zero data loss even on primary failure
```

---

## Diagram 2: Query Execution with Index vs COLLSCAN

```
QUERY EXECUTION: IXSCAN vs COLLSCAN
══════════════════════════════════════════════════════════════════════

COLLECTION: orders (10M documents)
QUERY: db.orders.find({ customerId: "cust_abc", status: "pending" })

WITHOUT INDEX (COLLSCAN):
──────────────────────────────────────────────────────────────────────

MongoDB must check EVERY document:

Doc 1: { customerId: "cust_xyz", status: "completed" }  ← SKIP (no match)
Doc 2: { customerId: "cust_abc", status: "completed" }  ← SKIP (status wrong)
Doc 3: { customerId: "cust_abc", status: "pending" }    ← MATCH! Return this
Doc 4: { customerId: "cust_123", status: "pending" }    ← SKIP (customerId wrong)
...
Doc 10,000,000: { customerId: "cust_abc", status: "pending" } ← MATCH!

Result: examined 10,000,000 docs, returned ~50 docs
Execution time: 4,500ms (4.5 seconds)
explain output: "stage": "COLLSCAN"

WITH COMPOUND INDEX { customerId: 1, status: 1 }:
──────────────────────────────────────────────────────────────────────

B-tree index structure:
"cust_aaa" → (pending) → [doc_0001]
"cust_aaa" → (shipped) → [doc_0145]
             ...
"cust_abc" → (cancelled) → [doc_4521]
"cust_abc" → (pending) → [doc_5821] ← SEEK TO HERE
"cust_abc" → (pending) → [doc_5822] ← READ FORWARD
"cust_abc" → (pending) → [doc_5823] ← UNTIL
"cust_abc" → (processing) → [doc_6100] ← STOP (past status range)
             ...

Result: examined 50 index entries, fetched 50 docs
Execution time: 2ms
explain output: "stage": "IXSCAN"

IMPROVEMENT: 2250x faster (4500ms → 2ms)

COVERED QUERY (no document fetch):
──────────────────────────────────────────────────────────────────────

INDEX: { customerId: 1, status: 1, amount: 1 }

QUERY:
db.orders.find(
  { customerId: "cust_abc", status: "pending" },
  { customerId: 1, status: 1, amount: 1, _id: 0 }  // projection in index
)

MongoDB reads index entries only:
"cust_abc" → "pending" → amount: 99.99  ← all needed data is IN the index
"cust_abc" → "pending" → amount: 149.99

Never touches actual collection documents!
explain: "stage": "IXSCAN" with NO "FETCH" stage
→ ZERO document reads → fastest possible query
```

---

## Diagram 3: Aggregation Pipeline Data Flow

```
AGGREGATION PIPELINE: STAGE-BY-STAGE DATA FLOW
══════════════════════════════════════════════════════════════════════

INPUT: orders collection, 10,000,000 documents

PIPELINE:
[ $match, $group, $sort, $limit, $project ]

Stage 1: $match { status: "completed", year: 2026 }
──────────────────────────────────────────────────────────────────────
Input:   10,000,000 documents
Process: Applies filter (uses index if available)
Output:  500,000 documents (5% match criteria)

                    ↓ 500,000 docs pass to next stage

Stage 2: $group { _id: "$customerId", total: { $sum: "$amount" } }
──────────────────────────────────────────────────────────────────────
Input:   500,000 documents
Process: Groups by customerId, sums amount per group
         (may use disk if > 100MB: allowDiskUse: true)
Output:  50,000 documents (one per unique customer)

                    ↓ 50,000 docs pass to next stage

Stage 3: $sort { total: -1 }
──────────────────────────────────────────────────────────────────────
Input:   50,000 documents
Process: Sort all 50k docs by total descending
Output:  50,000 documents (sorted)

                    ↓ 50,000 docs pass to next stage

Stage 4: $limit 10
──────────────────────────────────────────────────────────────────────
Input:   50,000 documents
Process: Keep only first 10
Output:  10 documents

                    ↓ 10 docs pass to next stage

Stage 5: $project { customerId: 1, total: 1, _id: 0 }
──────────────────────────────────────────────────────────────────────
Input:   10 documents
Process: Reshape documents
Output:  10 documents (final shape)

RESULT TO APPLICATION: 10 documents

OPTIMIZATION — $limit before $sort (when possible):
──────────────────────────────────────────────────────────────────────
If you only need top 10, $sort + $limit can be optimized together:
MongoDB detects $sort immediately followed by $limit:
→ Uses a "top-K sort" that only keeps the best K documents in memory
→ Instead of sorting 50,000 docs: tracks top 10 in a heap (much less memory)
→ Automatically applied: no code change needed
```

---

## Diagram 4: Connection Pool Lifecycle

```
CONNECTION POOL: LIFECYCLE DIAGRAM
══════════════════════════════════════════════════════════════════════

MongoClient Created (app startup):
┌─────────────────────────────────────────────────────────────────┐
│  Connection Pool (maxPoolSize: 100, minPoolSize: 5)             │
│                                                                  │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                           │
│  │ C1 │ │ C2 │ │ C3 │ │ C4 │ │ C5 │  ← 5 connections created  │
│  │idle│ │idle│ │idle│ │idle│ │idle│    (minPoolSize = 5)        │
│  └────┘ └────┘ └────┘ └────┘ └────┘                           │
│                                                                  │
│  [empty slots for up to 95 more connections]                    │
└─────────────────────────────────────────────────────────────────┘

REQUEST 1 arrives: db.orders.find(...)
    │
    ▼ Dispatcher leases C1 from pool
┌─────────────────────────────────────────────────────────────────┐
│  Pool:                                                          │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐                           │
│  │ C1 │ │ C2 │ │ C3 │ │ C4 │ │ C5 │                           │
│  │BUSY│ │idle│ │idle│ │idle│ │idle│                            │
│  └────┘ └────┘ └────┘ └────┘ └────┘                           │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼ Query executes on MongoDB (2ms)
    │
    ▼ C1 returned to pool (idle again)

100 SIMULTANEOUS REQUESTS arrive:
──────────────────────────────────────────────────────────────────────
    │
    ▼ 5 idle connections assigned immediately
    │ Pool creates 95 more connections (up to maxPoolSize: 100)
    │
┌─────────────────────────────────────────────────────────────────┐
│  Pool at maxPoolSize:                                           │
│  ┌────┐┌────┐┌────┐...┌────┐ (100 connections, all BUSY)       │
│  │ C1 ││ C2 ││ C3 │...│C100│                                   │
│  │BUSY││BUSY││BUSY│...│BUSY│                                   │
│  └────┘└────┘└────┘...└────┘                                   │
│                                                                  │
│  Request 101: WAIT (waitQueueTimeoutMS: 5000)                   │
│  If no connection becomes available in 5s → TIMEOUT ERROR       │
└─────────────────────────────────────────────────────────────────┘

ANTI-PATTERN: Creating new MongoClient per request:
──────────────────────────────────────────────────────────────────────

Request 1: new MongoClient() → connect() → query() → close()
           ┌── 50ms TCP handshake + TLS + auth ──┐
Request 2: new MongoClient() → connect() → query() → close()
           └── 50ms overhead for every request! ──┘
           vs 2ms with connection pool reuse
```

---

## Diagram 5: Schema Migration — Lazy Migration Pattern

```
SCHEMA MIGRATION: LAZY MIGRATION WITH SCHEMA VERSION
══════════════════════════════════════════════════════════════════════

TIMELINE: Migrating user schema from v1 to v2

OLD FORMAT (v1):                    NEW FORMAT (v2):
{ firstName: "Jane",                { name: { first: "Jane",
  lastName: "Doe" }                             last: "Doe" },
                                      schemaVersion: 2 }

PHASE 1 (Day 0): Deploy new code that handles BOTH formats
──────────────────────────────────────────────────────────────────────

Collection state at Day 0:
┌─────────────────────────────────────────────────────────────────┐
│ user_001: { firstName: "Alice", lastName: "Smith" }    ← v1     │
│ user_002: { firstName: "Bob",   lastName: "Jones" }    ← v1     │
│ user_003: { firstName: "Carol", lastName: "White" }    ← v1     │
│ ...100,000 users all in v1 format...                            │
└─────────────────────────────────────────────────────────────────┘

Application code: reads v1 AND v2 (backward compatible)
                  writes ONLY v2 format

PHASE 2 (Day 1-30): Live traffic migrates documents on write
──────────────────────────────────────────────────────────────────────

user_001 logs in → profile updated → rewritten as v2:
┌─────────────────────────────────────────────────────────────────┐
│ user_001: { name:{first:"Alice",last:"Smith"}, schemaVersion:2 }│ ← MIGRATED
│ user_002: { firstName: "Bob",   lastName: "Jones" }             │ ← still v1
│ user_003: { firstName: "Carol", lastName: "White" }             │ ← still v1
│ ...                                                             │
└─────────────────────────────────────────────────────────────────┘

PHASE 3 (Day 7-30): Background batch migrates remaining v1 docs
──────────────────────────────────────────────────────────────────────

Backfill script processes 10,000 docs/batch (throttled):
Progress: ████████████████████░░░░░░░░░░ 67% complete

Day 30: Collection fully migrated
┌─────────────────────────────────────────────────────────────────┐
│ ALL documents:                                                  │
│ { name: { first: "...", last: "..." }, schemaVersion: 2 }      │
└─────────────────────────────────────────────────────────────────┘

PHASE 4 (Day 31): Remove backward compatibility code + add validation
──────────────────────────────────────────────────────────────────────

db.runCommand({ collMod: "users", validator: { $jsonSchema: {
  required: ["name", "schemaVersion"],
  properties: {
    name: { bsonType: "object", required: ["first", "last"] },
    schemaVersion: { bsonType: "int", enum: [2] }
  }
}}})

RESULT: Zero downtime, gradual migration, old code removed cleanly
```

---

## Diagram 6: Backup and Recovery Options Comparison

```
MONGODB BACKUP OPTIONS: RPO / RTO COMPARISON
══════════════════════════════════════════════════════════════════════

BACKUP METHOD COMPARISON:
──────────────────────────────────────────────────────────────────────

                        RPO              RTO         Complexity
──────────────────────────────────────────────────────────────────────
mongodump               Hours            Hours       Low
(manual/scheduled)      (time since      (large DB
                         last backup)    takes hours)

Filesystem Snapshot     Seconds          Minutes     Medium
(LVM/EBS + journal)     (journal         (snapshot
                         flush time)     restore)

Atlas Cloud Backup      1 hour           30-120 min  Low (managed)
(scheduled snapshots)   (snapshot        (depends on
                         frequency)      DB size)

Atlas PITR              1 second         30-120 min  Low (managed)
(oplog streaming)       (oplog lag)      (depends on
                                         DB size)

Replica Set Secondary   0 (always        Minutes     Low
(always-on copy)        in sync)         (promote
                                         secondary)

──────────────────────────────────────────────────────────────────────

ATLAS CLOUD BACKUP FLOW:
──────────────────────────────────────────────────────────────────────

Atlas Cluster (Primary + 2 Secondaries)
    │
    │ Continuous oplog shipping → Atlas Backup Storage (S3)
    │ Scheduled snapshots → S3 (every 6h on M10, every 1h on M30+)
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Atlas Backup Storage (S3)                                      │
│                                                                 │
│  Snapshots:          Oplog:                                     │
│  2026-03-17T00:00Z   T00:00 → T23:59 (continuous)              │
│  2026-03-17T06:00Z   oplog entries for every write              │
│  2026-03-17T12:00Z                                              │
│  2026-03-17T18:00Z                                              │
└─────────────────────────────────────────────────────────────────┘

RESTORE TO 2026-03-17T14:32:17Z (point-in-time):
    │
    │ 1. Restore nearest snapshot before target: 12:00Z snapshot
    │ 2. Apply oplog entries from 12:00Z to 14:32:17Z
    │ 3. Cluster ready at state of 14:32:17Z
    │ 4. Effectively: RPO = 1 second, RTO = ~60 min (for 500GB DB)
    ▼
New Cluster in restored state (original cluster untouched)
Atlas restores to a NEW cluster — no risk to live data

RETENTION POLICY:
──────────────────────────────────────────────────────────────────────
Tier    │ Snapshot Interval │ Retention     │ PITR Window
────────┼───────────────────┼───────────────┼──────────────────
M10     │ 6 hours           │ 2 days        │ 2 days
M30     │ 6 hours           │ 7 days        │ 7 days
M50+    │ 1 hour            │ 14 days       │ 14 days
Custom  │ configurable      │ up to 1 year  │ up to 35 days PITR
```
