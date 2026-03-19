# Chapter 08: Wide-Column Stores — Diagram Explanations

## Diagram 1: LSM Tree Full Write and Read Path

```
LSM-TREE: COMPLETE WRITE AND READ PATH
═══════════════════════════════════════════════════════════════════════════

WRITE PATH:
Client PUT(key="K1", value="V1")
          │
          ▼
    ┌─────────────┐  sequential   ┌──────────────────────────────────────┐
    │   WAL/HLog  │ ──────────► │ HDFS/Disk (append-only log)           │
    │  (commit    │   write      │ [seq=1001,K1=V1][seq=1002,K2=V2]...  │
    │   log)      │              └──────────────────────────────────────┘
    └─────────────┘
          │ ACK to client (WAL write complete = durable)
          ▼
    ┌─────────────┐
    │  MemTable   │  Red-Black Tree or Skip List (sorted)
    │  (in-mem)   │  K1→V1, K3→V3, K7→V7 ...
    └─────────────┘
          │ (async) when full (e.g., 128MB)
          ▼
    ┌─────────────┐
    │   L0 HFile  │  Immutable SSTable on disk
    │  SST-001    │  K1→V1, K3→V3, K7→V7  (sorted, may overlap with other L0 files)
    └─────────────┘

COMPACTION (L0 → L1 when L0 has 4+ files):
┌──────────────────────────────────────────────────────────────────────┐
│ L0: [SST-001: K1-K9][SST-002: K2-K8][SST-003: K1-K5][SST-004: K3-K7]│
│      (overlapping key ranges)                                         │
│           │ merge all overlapping ranges                              │
│           ▼                                                           │
│ L1: [SST-010: K1-K3][SST-011: K4-K6][SST-012: K7-K9]                │
│      (no overlap, each key in exactly ONE file)                       │
└──────────────────────────────────────────────────────────────────────┘

READ PATH for key K5:
          │
          ├── 1. Check MemTable → NOT FOUND
          │
          ├── 2. Check L0 files (ALL of them, may overlap):
          │       Bloom(SST-001) → "possibly" → Read → NOT FOUND (false positive)
          │       Bloom(SST-002) → "not present" → SKIP
          │       Bloom(SST-003) → "possibly" → Read → NOT FOUND
          │       Bloom(SST-004) → "possibly" → Read → FOUND K5=V5 (old version)
          │
          └── 3. Check L1 (only SST-011 covers K4-K6):
                  Bloom(SST-011) → "possibly" → Read → FOUND K5=V5_new (newest)

                  Return newest version across all found: K5=V5_new
```

**Explanation:**
The read path must potentially check all L0 files (they may have overlapping key ranges)
plus at most one file per subsequent level (because L1+ files have non-overlapping ranges).
Bloom filters skip files that definitely don't contain the key. For a write-heavy workload
with many L0 files accumulating faster than compaction can consume them, read latency
spikes because more L0 files must be checked. This is the "L0 compaction bottleneck"
that requires tuning L0 file count thresholds.

---

## Diagram 2: HBase Architecture — Component Relationships

```
HBASE CLUSTER ARCHITECTURE
═══════════════════════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT                                       │
│  (Java API / REST / Thrift)                                         │
│  HBase client caches:                                               │
│  • ZooKeeper quorum address (configured)                            │
│  • hbase:meta region location (from ZK)                             │
│  • Row-key → RegionServer mapping (from meta)                       │
└──────────────┬──────────────────────────────────────┬──────────────┘
               │ ① bootstrap (one-time)               │ ③ direct RPC
               ▼                                       ▼
┌──────────────────────┐       ┌──────────────────────────────────────┐
│      ZooKeeper       │       │         RegionServer-1               │
│  (3 or 5 nodes)      │       │                                      │
│  Stores:             │       │  ┌──────────┐   ┌──────────────────┐ │
│  • /hbase/meta-rs    │       │  │MemStore  │   │     WAL (HLog)   │ │
│    (which RS hosts   │       │  │(in heap) │   │  (HDFS, append)  │ │
│     hbase:meta)      │       │  └──────────┘   └──────────────────┘ │
│  • /hbase/master     │       │         │                            │
│    (active HMaster)  │       │         │ flush                      │
│  • /hbase/rs/        │       │         ▼                            │
│    (live RS nodes)   │       │  ┌───────────────────────────────┐  │
└──────────────────────┘       │  │  HFiles in HDFS (SSTables)    │  │
               │               │  │  /hbase/mytable/region-id/cf/ │  │
               │ ② meta read   │  └───────────────────────────────┘  │
               ▼               └──────────────────────────────────────┘
┌──────────────────────┐
│       HMaster        │  Admin operations only (NOT in read/write path)
│  Responsibilities:   │  • Region assignment to RegionServers
│  • Table DDL         │  • RegionServer failure detection (via ZK)
│  • Region balancing  │  • Orchestrates WAL replay during RS recovery
│  • Schema changes    │  • Triggered by ZK session expiry of RS node
└──────────────────────┘

hbase:meta TABLE (special system table on a RegionServer):
┌─────────────────────────────────────────────────────────────────────┐
│ Row Key                    │ info:server           │ info:startkey  │
├─────────────────────────────────────────────────────────────────────┤
│ mytable,,1234567890001.    │ rs1.example.com:16020 │ ""             │
│ mytable,m,1234567890002.   │ rs2.example.com:16020 │ "m"            │
│ mytable,s,1234567890003.   │ rs1.example.com:16020 │ "s"            │
└─────────────────────────────────────────────────────────────────────┘
                    Row key encodes table + startkey + region timestamp
                    Client caches this to route requests directly to RS
```

**Explanation:**
The HMaster is intentionally not in the read/write path — it is an admin coordinator,
not a data router. This design means HMaster failure does not affect ongoing reads and
writes (though new region assignments, splits, and DDL are blocked). ZooKeeper is the
source of truth for cluster membership — each RegionServer maintains a ZooKeeper session,
and if that session expires, HMaster detects the failure and initiates recovery.

---

## Diagram 3: HBase Row Key Hotspot vs Pre-Split Regions

```
HOTSPOT: Sequential timestamp row keys
═══════════════════════════════════════════════════════════════════════════

Write traffic (1000 writes/sec, all at current timestamp):

Region-1:  [2022-01-01 to 2022-06-30]  ████░░░░░░  10 writes/sec (OLD DATA)
Region-2:  [2022-07-01 to 2022-12-31]  ██░░░░░░░░   5 writes/sec
Region-3:  [2023-01-01 to 2023-06-30]  ██░░░░░░░░   5 writes/sec
Region-4:  [2023-07-01 to 2024-01-14]  █░░░░░░░░░   0 writes/sec
Region-5:  [2024-01-15 to ∞]           ██████████ 980 writes/sec ← HOTSPOT

RegionServer hosting Region-5: CPU 95%, disk I/O 100%
All other RegionServers: CPU 5%, disk I/O 5% ← WASTED CAPACITY

────────────────────────────────────────────────────────────────────────────

SOLUTION 1: Salted row key with pre-split
Row key: salt(1B) + original_key

Pre-split into 4 regions (salt values 0-3):
Region-1: salt=0 → AAAA... to BBBB...    handles ~25% of writes
Region-2: salt=1 → BBBB... to CCCC...    handles ~25% of writes
Region-3: salt=2 → CCCC... to DDDD...    handles ~25% of writes
Region-4: salt=3 → DDDD... to FFFF...    handles ~25% of writes

Write distribution:
Region-1:  ████████░░  250 writes/sec
Region-2:  ████████░░  250 writes/sec
Region-3:  ████████░░  250 writes/sec
Region-4:  ████████░░  250 writes/sec  ← BALANCED!

COST: Range scan needs 4 separate Scan objects + client-side merge

SOLUTION 2: Reverse the row key
Original:    "2024011512000000_event_001"
Reversed:    "100000021_51041402"
Effect: different timestamps hash differently → natural distribution
BUT: lexicographic ordering is lost → range scans require full table scan

BEST PRACTICE for IoT/Events:
Row key: hash(device_id, 3 bytes) + device_id(10B) + invertedTimestamp(8B)
Writes: distributed by device → each device's data in one region
Reads: latest N events for device = one Scan on one region (fast!)
Cross-device: scatter-gather across all regions (acceptable for analytics)
```

**Explanation:**
Pre-splitting is the preventive measure against hotspotting. Without pre-splitting, HBase
creates one region for a new table. All writes hit that one region until it splits at 10GB
(by default), then two regions exist, and so on. By pre-splitting into N regions matching
your expected salt distribution, you start with even write distribution from the first insert.
The `HBaseAdmin.createTable(tableDesc, splitKeys)` API accepts an array of split points.

---

## Diagram 4: Wide-Column Store Model vs Relational Table

```
RELATIONAL TABLE vs WIDE-COLUMN STORE
═══════════════════════════════════════════════════════════════════════════

SQL Table: users (FIXED SCHEMA)
┌──────────┬────────────┬──────────┬─────────┬──────────┬─────────────┐
│ user_id  │ name       │ email    │ age     │ country  │ preferences │
├──────────┼────────────┼──────────┼─────────┼──────────┼─────────────┤
│ u001     │ Alice      │ a@ex.com │ 32      │ US       │ NULL        │
│ u002     │ Bob        │ b@ex.com │ NULL    │ NULL     │ {"theme":"d"}│
│ u003     │ Carol      │ c@ex.com │ 28      │ UK       │ NULL        │
└──────────┴────────────┴──────────┴─────────┴──────────┴─────────────┘
NULL values stored for every missing column (waste); schema changes require ALTER TABLE

HBase Table: users (SPARSE, MULTI-VERSION, SCHEMA-FLEXIBLE)
Row Key      Column Family:Qualifier                    Timestamp  Value
──────────────────────────────────────────────────────────────────────────
"u001"       profile:name                              T3         "Alice"
"u001"       profile:email                             T3         "a@ex.com"
"u001"       profile:age                               T3         "32"
"u001"       profile:country                           T3         "US"
             ← 'preferences' column doesn't exist for u001 → NO STORAGE

"u002"       profile:name                              T5         "Bob"
"u002"       profile:email                             T5         "b@ex.com"
             ← age, country don't exist for u002 → NO STORAGE
"u002"       settings:theme                            T5         "dark"
             ← 'settings' CF only exists for rows that have settings

"u001"       profile:email                             T2         "old@ex.com"
"u001"       profile:email                             T1         "oldest@ex.com"
             ← Multi-version: last 3 versions retained, history preserved

PHYSICAL STORAGE (per column family):
profile CF HFile:     settings CF HFile:
u001/name → Alice     u002/theme → dark
u001/email → a@ex.com u003/lang → en
u001/age → 32
u002/name → Bob
u002/email → b@ex.com
                      ← Completely separate files on disk!
                      ← Query profile CF = never read settings CF file
```

**Explanation:**
The wide-column model's sparse storage means that rows 10 trillion and row 10 trillion and
one can have completely different columns with zero storage overhead for missing values.
This is fundamentally different from SQL where every column must exist for every row (NULL
takes storage and must be processed in queries). The physical separation of column families
into different HFiles enables dramatic I/O optimization: a query for user profiles never
reads the settings files, regardless of how large the settings data is.
