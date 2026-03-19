# Chapter 03 — Cassandra: Diagram Explanations

ASCII diagrams for Cassandra architecture, data flow, and storage internals.

---

## Diagram 1: Cassandra Ring Topology with Replication Factor 3

```
  6-Node Cassandra Cluster, RF=3, NetworkTopologyStrategy
  Tokens evenly distributed (simplified — real clusters use vnodes)

                     Node A
                   (token: 0)
                       │
              ┌────────┴────────┐
         Node F              Node B
       (token:               (token:
        -2.7T)                +2.7T)
           │                     │
      ┌────┘                     └────┐
  Node E                            Node C
 (token:                           (token:
  -5.5T)                            +5.5T)
      └────────────────────────────────┘
                   Node D
                (token: ±8.2T)

  DATA OWNERSHIP (RF=3):
  Key "user:alice" hashes to token +3.2T
  ┌────────────────────────────────────────────────────────┐
  │  Primary replica:  Node C (first node clockwise)      │
  │  Replica 2:        Node D (second node clockwise)     │
  │  Replica 3:        Node E (third node clockwise)      │
  └────────────────────────────────────────────────────────┘

  CLIENT WRITE at QUORUM (W=2):
  Client ──► Coordinator (any node, e.g., Node A)
  Node A forwards write to Node C, D, E
  Waits for ACK from any 2 of {C, D, E}
  Returns success to client
  Node not responding: hint stored at coordinator
```

**Explanation:** Any node can be a coordinator — there is no dedicated master. The
coordinator hashes the partition key to determine the primary replica and RF-1 additional
clockwise replicas. Writes are sent to all RF replicas; the coordinator waits for W
acknowledgements before responding to the client. This peer-to-peer model means losing
any single node (or even multiple nodes, depending on RF and consistency level) does not
cause an outage.

---

## Diagram 2: Cassandra Read Path — From Request to Data

```
  READ: SELECT * FROM orders WHERE user_id = 'alice' AND order_date = '2024-01-15'

  Client ──────────────────────────────────────────────────────────►
                                                                    │
  Coordinator                                                       │
      │                                                             │
      ├── Contacts R replicas (e.g., LOCAL_QUORUM: 2 of 3)        │
      │                                                             │
  Each Replica:                                                     │
      │                                                             │
      │   ┌──────────────────────────────────────────────────────┐ │
      │   │ Step 1: Bloom Filter (in memory, per SSTable)       │ │
      │   │  hash(partition_key) → "maybe here" or "not here"  │ │
      │   │  Skip SSTables that say "not here" → save disk I/O │ │
      │   └──────────────────────────────────────────────────────┘ │
      │         │ "maybe here" for SSTable-3 and SSTable-7         │
      │         ▼                                                   │
      │   ┌──────────────────────────────────────────────────────┐ │
      │   │ Step 2: Key Cache (in memory)                       │ │
      │   │  partition_key → exact byte offset in Data.db       │ │
      │   │  HIT: jump directly to data                         │ │
      │   │  MISS: continue to Step 3                           │ │
      │   └──────────────────────────────────────────────────────┘ │
      │         │ cache miss                                        │
      │         ▼                                                   │
      │   ┌──────────────────────────────────────────────────────┐ │
      │   │ Step 3: Partition Summary (in memory)               │ │
      │   │  Sparse index of Index.db                           │ │
      │   │  Narrows range: "look between byte 45000-46000      │ │
      │   │   in Index.db"                                      │ │
      │   └──────────────────────────────────────────────────────┘ │
      │         │                                                   │
      │         ▼                                                   │
      │   ┌──────────────────────────────────────────────────────┐ │
      │   │ Step 4: Partition Index (Index.db on disk)          │ │
      │   │  1 disk read to find: partition_key → offset 45312  │ │
      │   └──────────────────────────────────────────────────────┘ │
      │         │                                                   │
      │         ▼                                                   │
      │   ┌──────────────────────────────────────────────────────┐ │
      │   │ Step 5: Data File (Data.db on disk)                 │ │
      │   │  1 disk read at offset 45312                        │ │
      │   │  Returns rows for partition (all clustering cols)   │ │
      │   └──────────────────────────────────────────────────────┘ │
      │                                                             │
      └── Response ──────────────────────────────────────────────► │
                                                                    │
  Coordinator merges R responses, picks latest by timestamp        │
  Triggers background READ REPAIR on stale replicas               │
  Returns to client ◄──────────────────────────────────────────── ┘
```

**Explanation:** The read path is designed to minimise disk I/O. The Bloom filter skips
most SSTables. The key cache skips index reads for hot keys. The partition summary minimises
the index file scan. Only cold reads (not cached, not bloom-filtered) touch 2 disk files:
the index file and the data file. Multiple SSTables (not yet compacted) multiply this cost.

---

## Diagram 3: SSTable Components

```
  SSTable (one per flush/compaction output) — immutable once written

  ┌──────────────────────────────────────────────────────────────────┐
  │  File                Size      Location  Purpose                 │
  ├──────────────────────────────────────────────────────────────────┤
  │  Data.db             large     disk      Actual row data         │
  │  Index.db            medium    disk      partition key→offset    │
  │  Summary.db          small     memory    sparse index of Index   │
  │  Filter.db           small     memory    Bloom filter            │
  │  CompressionInfo.db  tiny      disk      compression chunk map   │
  │  Statistics.db       tiny      disk      min/max ts, tombstones  │
  │  TOC.txt             tiny      disk      table of contents       │
  │  Digest.crc32        tiny      disk      checksum for Data.db    │
  └──────────────────────────────────────────────────────────────────┘

  DATA.DB INTERNAL STRUCTURE:
  ┌──────────────────────────────────────────────────────────────────┐
  │  [Partition 1 header: partition key, deletion info]             │
  │  [Row 1: clustering cols + regular cols + timestamps]           │
  │  [Row 2: clustering cols + regular cols + timestamps]           │
  │  ...                                                            │
  │  [Partition 2 header]                                           │
  │  [Row 1] [Row 2] ...                                            │
  │  ...                                                            │
  │  Partitions are sorted by TOKEN(partition_key) ascending        │
  │  Within a partition, rows are sorted by clustering columns      │
  └──────────────────────────────────────────────────────────────────┘

  COMPACTION MERGES MULTIPLE SSTables:
  SSTable-1: {alice: [order1, order2],  bob: [orderA]}
  SSTable-2: {alice: [order2-updated],  carol: [orderX]}
                     │
                     ▼ Compaction (merge by partition, resolve by timestamp)
  SSTable-merged: {alice: [order1, order2-updated], bob: [orderA], carol: [orderX]}
```

**Explanation:** SSTables are immutable — once written, they are never modified. Updates
and deletes create new SSTables with newer timestamps for the same keys. Compaction is
the process of merging SSTables, resolving conflicting values by keeping the latest
timestamp, and purging expired tombstones (those older than gc_grace_seconds).

---

## Diagram 4: Cassandra Consistency Level Quorum Math

```
  CLUSTER: 6 nodes, RF=3 (3 replicas per partition)
  2 datacenters: DC1 (nodes A,B,C) and DC2 (nodes D,E,F)

  ┌──────────────────────────────────────────────────────────────────┐
  │  DC1                             DC2                            │
  │  ┌──────┐ ┌──────┐ ┌──────┐    ┌──────┐ ┌──────┐ ┌──────┐   │
  │  │  A   │ │  B   │ │  C   │    │  D   │ │  E   │ │  F   │   │
  │  └──────┘ └──────┘ └──────┘    └──────┘ └──────┘ └──────┘   │
  └──────────────────────────────────────────────────────────────────┘

  Partition key "user:alice" replicates to: A, B, D (RF=3, cross-DC)

  CONSISTENCY LEVEL COMPARISON:
  ┌─────────────────┬───────┬───────┬──────────────────────────────────┐
  │  Level          │  W    │  R    │  Notes                           │
  ├─────────────────┼───────┼───────┼──────────────────────────────────┤
  │  ONE            │   1   │   1   │  Fastest, weakest guarantee      │
  │  TWO            │   2   │   2   │  Rare                            │
  │  THREE          │   3   │   3   │  Full replication                │
  │  QUORUM         │  ⌈N/2⌉+1     │  N=RF across ALL DCs (risky!)   │
  │  LOCAL_QUORUM   │  ⌈N_dc/2⌉+1  │  Quorum in LOCAL DC only        │
  │  EACH_QUORUM    │  per DC      │  Quorum in EACH DC (slowest)    │
  │  ALL            │  RF   │  RF   │  Max consistency, min avail     │
  │  ANY            │   1   │  N/A  │  Even hint counts (not durable) │
  └─────────────────┴───────┴───────┴──────────────────────────────────┘

  QUORUM MATH EXAMPLE (RF=3):
  ┌──────────────────────────────────────────────────────────────────┐
  │  QUORUM  = ⌈3/2⌉ + 1 = 2                                       │
  │  W=2, R=2, N=3 → W+R=4 > 3 → STRONG CONSISTENCY               │
  │  Tolerates 1 node failure (3-1=2 ≥ quorum=2)                  │
  │                                                                  │
  │  LOCAL_QUORUM in DC1 (3 nodes):                                │
  │  W_dc1=2, R_dc1=2 → Strong consistency WITHIN DC1             │
  │  DC2 replication is async → DC2 reads may be stale            │
  │                                                                  │
  │  EACH_QUORUM:                                                  │
  │  Must get quorum from DC1 AND DC2 simultaneously              │
  │  If DC2 is slow/partitioned → write fails                     │
  └──────────────────────────────────────────────────────────────────┘
```

**Explanation:** The most operationally safe configuration for a multi-DC cluster is
`LOCAL_QUORUM` for both reads and writes within each application's local DC. This provides
strong consistency within a DC while allowing async cross-DC replication. `QUORUM` across
DCs is risky because it couples the write latency to the cross-DC network speed and can cause
writes to fail if either DC is partitioned.

---

## Diagram 5: STCS vs LCS vs TWCS Compaction Patterns

```
  STCS (Size-Tiered Compaction Strategy):
  Time ──────────────────────────────────────────────────────────────►
  Small SSTables accumulate:    [1][1][1][1]
  Merge when 4 of same size:    [1][1][1][1] → [4]
  More accumulate:              [4][4][4][4]
  Merge again:                  [4][4][4][4] → [16]
  Result: "tiered" structure, multiple overlapping ranges
  READ COST: Must check all levels for a given key

  LCS (Leveled Compaction Strategy):
  L0 (unleveled):    [a-z][a-z][a-z][a-z]    ← new SSTables arrive here
  L1 (size-limited): [a-c][d-f][g-j][k-m]... ← non-overlapping ranges
  L2 (10x larger):   [a-b][c][d]...          ← non-overlapping ranges
  READ COST: Check at most 1 SSTable per level → O(log N) with N levels

  TWCS (Time Window Compaction Strategy):
  9am-10am window:  [write][write][write] → [merged-9am]  ← sealed
  10am-11am window: [write][write][write] → [merged-10am] ← sealed
  11am-12pm window: [write][write][write][write] ← active, compacting
  12pm-1pm window:  (future)

  OLD WINDOWS EXPIRE via TTL → deleted entirely (no tombstones for old data)
  Current window compacts writes into 1 SSTable per window

  DISK SPACE AMPLIFICATION:
  ┌─────────────────────────────────────────────────────────────────┐
  │  STCS during compaction:  needs 2x data size temporarily       │
  │  LCS steady state:        ~10x write amplification             │
  │  TWCS:                    1x (no recompaction of old windows)  │
  └─────────────────────────────────────────────────────────────────┘
```

**Explanation:** Choosing the wrong compaction strategy is a common source of production
problems. STCS for time-series data leads to unbounded read amplification as old SSTables
accumulate overlapping ranges. TWCS for mutable data fails because updates to closed windows
force the window to reopen and recompact, destroying TWCS's main benefit. LCS for write-heavy
workloads causes excessive I/O as every new SSTable triggers cascading compactions up the levels.

---

## Diagram 6: Partition Key Design — Hot Partition vs Sharded Partition

```
  ANTI-PATTERN: Single sensor_id partition key
  ┌──────────────────────────────────────────────────────────────────┐
  │  Partition: sensor_id="gateway-001"                            │
  │  ┌─────────────────────────────────────────────────────────┐  │
  │  │ ts=10:00:00.001  value=22.5                             │  │
  │  │ ts=10:00:00.002  value=22.6                             │  │
  │  │ ...                                                     │  │
  │  │ ts=10:00:01.000  value=22.4  (10,000 rows per second!) │  │
  │  └─────────────────────────────────────────────────────────┘  │
  │  → ALL writes go to ONE node → HOT PARTITION                 │
  └──────────────────────────────────────────────────────────────────┘

  FIXED PATTERN: Bucket-based partition key
  Partition key = (sensor_id, bucket)
  bucket = YYYY-MM-DD-HH (1-hour time bucket)

  ┌───────────────────────────────────────────────────────────────────┐
  │  Partition: ("gateway-001", "2024-01-15-10")                    │
  │  Node A:  3,600 rows (10:00 to 10:59)                          │
  │                                                                   │
  │  Partition: ("gateway-001", "2024-01-15-11")                    │
  │  Node C:  3,600 rows (11:00 to 11:59)  ← DIFFERENT NODE        │
  │                                                                   │
  │  Partition: ("gateway-001", "2024-01-15-12")                    │
  │  Node E:  3,600 rows (12:00 to 12:59)  ← YET ANOTHER NODE      │
  └───────────────────────────────────────────────────────────────────┘

  ✓ Writes spread across multiple nodes (hourly partition change)
  ✓ Partition size bounded: max 36,000 rows/partition at 10 writes/sec
  ✓ Range queries within an hour: single partition (fast)
  ✗ Range queries across hours: multiple partition reads (application must merge)

  APPLICATION QUERY:
  buckets = get_buckets("gateway-001", start="10:00", end="12:00")
  # → ["2024-01-15-10", "2024-01-15-11", "2024-01-15-12"]
  rows = []
  for bucket in buckets:
      rows += session.execute(
          "SELECT * FROM readings WHERE sensor_id=? AND bucket=? AND ts>=? AND ts<=?",
          ("gateway-001", bucket, start, end)
      )
  return sorted(rows, key=lambda r: r.ts)
```

**Explanation:** The bucket pattern is the canonical solution for time-series data in Cassandra.
The bucket size should be chosen so that: (1) the partition size stays under ~100MB, and
(2) the most common query range fits within a single bucket (minimising cross-partition reads).
If reads typically span 1 hour, use hourly buckets. If reads typically span 1 day, use daily
buckets but verify the partition size stays bounded with the expected write rate.
