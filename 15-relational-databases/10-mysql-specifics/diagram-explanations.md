# MySQL Specifics — Diagram Explanations

ASCII diagrams for InnoDB architecture, replication, index structures, and MySQL vs PostgreSQL comparisons.

---

## Diagram 1: InnoDB Architecture Overview

```
  ┌────────────────────────────────────────────────────────────────────────┐
  │                         INNODB MEMORY STRUCTURES                       │
  │                                                                        │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │                    BUFFER POOL (e.g., 96GB)                      │  │
  │  │                                                                  │  │
  │  │  ┌─────────────────────┐    ┌─────────────────────────────────┐ │  │
  │  │  │  Young Sublist      │    │  Old Sublist                    │ │  │
  │  │  │  (recently accessed)│    │  (candidates for eviction)      │ │  │
  │  │  │  ~63% of pool       │    │  ~37% of pool                   │ │  │
  │  │  └─────────────────────┘    └─────────────────────────────────┘ │  │
  │  │                                                                  │  │
  │  │  New pages inserted at 3/8 point → move to young on re-access   │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  │                                                                        │
  │  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────────────┐ │
  │  │  Log Buffer     │  │  Change Buffer   │  │  Adaptive Hash Index  │ │
  │  │  (wal_buffers)  │  │  (secondary idx  │  │  (hot page cache)     │ │
  │  │  → redo logs    │  │  buffered writes)│  │  auto-managed         │ │
  │  └─────────────────┘  └──────────────────┘  └───────────────────────┘ │
  └────────────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────────────┐
  │                         INNODB ON-DISK STRUCTURES                      │
  │                                                                        │
  │  ┌─────────────────────────────────────────────────────────────────┐   │
  │  │  Tablespace Files (.ibd per table in file_per_table mode)       │   │
  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │   │
  │  │  │  Clustered  │  │  Secondary  │  │  Insert Buffer Bitmap   │ │   │
  │  │  │  Index      │  │  Indexes    │  │  (for change buffer)    │ │   │
  │  │  │  (PK data)  │  │             │  │                         │ │   │
  │  │  └─────────────┘  └─────────────┘  └─────────────────────────┘ │   │
  │  └─────────────────────────────────────────────────────────────────┘   │
  │                                                                        │
  │  ┌──────────────────────────────┐   ┌──────────────────────────────┐  │
  │  │  Redo Log Files              │   │  Undo Tablespaces            │  │
  │  │  (#ib_redo0, #ib_redo1, ...) │   │  (undo_001, undo_002, ...)   │  │
  │  │  Crash recovery              │   │  MVCC old versions           │  │
  │  │  innodb_redo_log_capacity    │   │  Transaction rollback        │  │
  │  └──────────────────────────────┘   └──────────────────────────────┘  │
  │                                                                        │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  Doublewrite Buffer (ibdata1 or separate file)                   │  │
  │  │  Torn-page protection: write here first, then to actual location │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 2: InnoDB Clustered Index vs Secondary Index Structure

```
  Table: employees (id INT PK, dept_id INT, last_name VARCHAR, salary DECIMAL)
  Index: idx_dept ON (dept_id)  → actually stores (dept_id, id) in InnoDB

  CLUSTERED INDEX (PRIMARY KEY — physical row storage):
  ────────────────────────────────────────────────────
                    ┌────────────────────────────────┐
                    │  Internal B-tree node           │
                    │  [id=50] [id=100] [id=150]      │
                    └────┬──────────┬────────────┬───┘
               ┌─────────┘          │            └───────────┐
               ▼                    ▼                        ▼
  ┌────────────────────────┐ ┌─────────────────────────┐ ...
  │ Leaf page (data page): │ │ Leaf page:               │
  │ id=51, dept=3, 'Smith' │ │ id=101, dept=1, 'Jones'  │
  │ id=52, dept=1, 'Brown' │ │ id=102, dept=3, 'Davis'  │
  │ id=53, dept=2, 'Wilson'│ │ id=103, dept=2, 'Taylor' │
  └────────────────────────┘ └─────────────────────────┘
  ↑ Actual row data stored in leaf pages (clustered by PK)

  SECONDARY INDEX (idx_dept):
  ────────────────────────────
                    ┌──────────────────────────────┐
                    │  Internal B-tree node         │
                    │  [dept=1] [dept=2] [dept=3]   │
                    └────┬────────────┬─────────────┘
               ┌─────────┘            └──────────────┐
               ▼                                     ▼
  ┌────────────────────────────┐    ┌────────────────────────────┐
  │ Leaf page (secondary idx): │    │ Leaf page:                 │
  │ (dept=1, id=52)            │    │ (dept=3, id=51)            │
  │ (dept=1, id=101)           │    │ (dept=3, id=102)           │
  └────────────────────────────┘    └────────────────────────────┘
  ↑ Secondary index leaf nodes contain: (index key, PRIMARY KEY)
  ↑ To get full row: take id → lookup in clustered index (two-step)

  COVERING INDEX (no second lookup needed):
  SELECT dept_id, id FROM employees WHERE dept_id = 1;
  → Fully satisfied by idx_dept leaf pages (dept_id + id already there)
  → EXPLAIN shows: "Using index"
```

---

## Diagram 3: MySQL GTID Replication Architecture

```
  NORMAL OPERATION:
  ─────────────────────────────────────────────────────────────────────
  Source MySQL                              Replica MySQL
  (server_uuid = A)                         (server_uuid = B)

  Transaction commits:                       Binary log relay:
  ┌────────────────────────────────┐        ┌────────────────────────┐
  │ binlog.000001                  │        │ relay-bin.000001        │
  │ GTID: A:1 → INSERT orders      │───────►│ GTID: A:1              │
  │ GTID: A:2 → UPDATE inventory   │        │ GTID: A:2              │
  │ GTID: A:3 → INSERT order_items │        │ GTID: A:3              │
  └────────────────────────────────┘        └──────────┬─────────────┘
                                                       │ SQL thread applies
                                                       ▼
                                            Executed_Gtid_Set = A:1-3

  FAILOVER SEQUENCE:
  ─────────────────────────────────────────────────────────────────────
  1. Source crashes → promoted replica has Executed_Gtid_Set = A:1-3
  2. Other replicas may have different sets:
     Replica 2: Executed_Gtid_Set = A:1-2 (behind by 1 transaction)
     Replica 3: Executed_Gtid_Set = A:1-3 (fully caught up)

  3. Promote Replica 3 (most complete) as new source
  4. Replica 2 reconnects with GTID auto-positioning:
     "I have A:1-2, send me everything else starting from A:3"
  5. New source sends A:3 to Replica 2 → Replica 2 catches up

  MULTI-THREADED REPLICATION (parallel replay):
  ─────────────────────────────────────────────────────────────────────
  Single SQL thread (old way):
  A:1 → A:2 → A:3 → A:4 → A:5  (sequential, can't keep up with fast source)

  LOGICAL_CLOCK parallel (slave_parallel_type = LOGICAL_CLOCK):
  A:1 ──────────────────────────────► Worker 1
  A:2 (same logical clock) ─────────► Worker 2  (applied in parallel!)
  A:3 (same logical clock) ─────────► Worker 3
  A:4 (next clock cycle) ───────────► Worker 1 (after A:1-3 committed)

  slave_parallel_workers = 8  → 8x throughput improvement possible
```

---

## Diagram 4: InnoDB Lock Types and Deadlock Scenario

```
  INNODB LOCK TYPES:
  ──────────────────────────────────────────────────────────────────
  Gap Lock:    Locks the gap BETWEEN rows (prevents phantom inserts)
  Record Lock: Locks a specific row (the index entry itself)
  Next-Key Lock = Record Lock + Gap Lock (default in REPEATABLE READ)
  Insert Intention Lock: Pre-insert signal (multiple can coexist)

  Index: id = [10, 20, 30, 40, 50]

  WHERE id = 30 with S-lock:
  ├── Record Lock on id=30 only

  WHERE id BETWEEN 20 AND 40 with X-lock (range lock):
  ├── Gap Lock: (10, 20]
  ├── Record Lock: id=20
  ├── Gap Lock: (20, 30]
  ├── Record Lock: id=30
  ├── Gap Lock: (30, 40]
  └── Record Lock: id=40
  (prevents inserts of id=21, 22, ..., 39 while lock held)

  DEADLOCK SCENARIO:
  ──────────────────────────────────────────────────────────────────

  Time → →  t=1         t=2         t=3         t=4
  TX A:     LOCK row100  —           WAIT row200  ← (blocked by TX B)
  TX B:     —           LOCK row200  —            WAIT row100
            ^                        ^
            TX A holds row100        TX B wants row100 (held by TX A)
            TX B holds row200        TX A wants row200 (held by TX B)
                                              ↓
                                     DEADLOCK DETECTED by InnoDB
                                     Victim: TX with fewer undo records
                                     Victim TX rolled back with ERROR 1213

  PREVENTION (consistent lock ordering):
  ──────────────────────────────────────
  BEFORE (inconsistent order):
  TX A: lock row100 → lock row200
  TX B: lock row200 → lock row100  ← cycle possible

  AFTER (consistent order, always row100 first):
  TX A: lock row100 → lock row200
  TX B: lock row100 → lock row200  ← TX B waits for row100, no cycle

  SQL to diagnose current locks:
  ┌──────────────────────────────────────────────────────────────────┐
  │ SELECT * FROM performance_schema.data_lock_waits\G               │
  │ SELECT * FROM sys.innodb_lock_waits\G                            │
  │ SHOW ENGINE INNODB STATUS\G  -- see LATEST DETECTED DEADLOCK     │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Diagram 5: MySQL vs PostgreSQL MVCC Architecture Comparison

```
  POSTGRESQL MVCC (Heap-based old versions):
  ──────────────────────────────────────────────────────────────────

  Table heap file:
  ┌──────────────────────────────────────────────────────────────┐
  │  Page 1                                                       │
  │  ┌───────────────────────────┐  ← dead tuple (UPDATE victim) │
  │  │ xmin=100 xmax=150 id=1    │    visible to TXN 100-149     │
  │  │ name='Alice'              │                               │
  │  └───────────────────────────┘                               │
  │  ┌───────────────────────────┐  ← live tuple                 │
  │  │ xmin=150 xmax=0   id=1    │    visible to TXN >= 150      │
  │  │ name='Alicia'             │                               │
  │  └───────────────────────────┘                               │
  └──────────────────────────────────────────────────────────────┘
  Read old version: already in heap — no extra I/O needed
  Consequence: VACUUM needed to remove dead tuples → table bloat

  MYSQL INNODB MVCC (Undo log based):
  ──────────────────────────────────────────────────────────────────

  Clustered index (always current):
  ┌──────────────────────────────────────────────────────────────┐
  │  id=1, name='Alicia'  ← current version                     │
  │  rollback_ptr ──────────────────────────────────────────────►┐
  └──────────────────────────────────────────────────────────────┘│
                                                                  │
  Undo log segment:                                               │
  ┌──────────────────────────────────────────────────────────────┐│
  │  old_version: id=1, name='Alice', trx_id=100  ◄──────────────┘
  │  prev_rollback_ptr ──────────────────────────────────────────►┐
  └──────────────────────────────────────────────────────────────┘│
                                                                  │
  │  (older versions further back in undo chain...)               │
  Read old version: follow rollback pointer → reconstruct from undo
  Consequence: undo log purge needed → but clustered index stays compact

  Performance comparison:
  ┌───────────────────────────────┬──────────────────────────────┐
  │  PostgreSQL                   │  MySQL InnoDB                 │
  ├───────────────────────────────┼──────────────────────────────┤
  │  Read old version: O(1)       │  Read old version: O(depth   │
  │  (already in heap)            │  of undo chain)              │
  ├───────────────────────────────┼──────────────────────────────┤
  │  Table bloat under heavy UPD  │  Undo log bloat under long   │
  │  (dead tuples in heap)        │  transactions                │
  ├───────────────────────────────┼──────────────────────────────┤
  │  VACUUM needed (maintenance)  │  Undo purge needed           │
  │  Can't reclaim without VACUUM │  Undo cannot shrink ibdata1  │
  │                               │  in older versions           │
  └───────────────────────────────┴──────────────────────────────┘
```

---

## Diagram 6: MySQL EXPLAIN Output Interpretation

```
  Query: SELECT c.name, COUNT(o.id) FROM customers c
         JOIN orders o ON c.id = o.customer_id
         WHERE c.region = 'US' GROUP BY c.id ORDER BY COUNT(o.id) DESC LIMIT 10;

  EXPLAIN output (traditional format):
  ┌────┬─────────────┬────────────┬─────────────┬────────────┬──────────────────────────────┐
  │ id │ select_type │ table      │ type        │ key        │ Extra                        │
  ├────┼─────────────┼────────────┼─────────────┼────────────┼──────────────────────────────┤
  │  1 │ SIMPLE      │ c          │ ref         │ idx_region │ Using index condition;       │
  │    │             │            │             │            │ Using temporary;             │
  │    │             │            │             │            │ Using filesort               │
  ├────┼─────────────┼────────────┼─────────────┼────────────┼──────────────────────────────┤
  │  1 │ SIMPLE      │ o          │ ref         │ idx_cust   │ NULL                         │
  └────┴─────────────┴────────────┴─────────────┴────────────┴──────────────────────────────┘

  Key indicators:
  ┌──────────────────────────┬──────────────────────────────────────────────────────────────┐
  │ Signal                   │ Meaning                                                      │
  ├──────────────────────────┼──────────────────────────────────────────────────────────────┤
  │ type = ref               │ Index lookup on non-unique key (good)                        │
  │ type = ALL               │ Full table scan (usually bad)                                │
  │ type = range             │ Index range scan (good if selective)                         │
  │ type = eq_ref            │ Unique key lookup (best join type)                           │
  ├──────────────────────────┼──────────────────────────────────────────────────────────────┤
  │ Extra: Using filesort    │ No usable index for ORDER BY → sort on disk (slow!)          │
  │ Extra: Using temporary   │ Temp table needed for GROUP BY/DISTINCT (slow!)              │
  │ Extra: Using index       │ Covering index (no table lookup needed — fast!)              │
  │ Extra: Using where       │ WHERE filter applied after index scan                        │
  │ Extra: Using index cond  │ Index condition pushdown (ICP) — filter in storage engine   │
  ├──────────────────────────┼──────────────────────────────────────────────────────────────┤
  │ rows column              │ Estimated rows examined (not returned!)                      │
  │                          │ High rows with few returned = inefficient                    │
  └──────────────────────────┴──────────────────────────────────────────────────────────────┘

  EXPLAIN ANALYZE (MySQL 8.0.18+) — actual execution:
  ──────────────────────────────────────────────────────────────────────────────────────────
  -> Sort: count(o.id) DESC  (actual time=234.5..234.5 rows=10 loops=1)
      -> Table scan on <temporary>  (actual time=0.001..1.23 rows=18432 loops=1)
          -> Aggregate using temporary table  (actual time=45.6..45.6 rows=18432 loops=1)
              -> Nested loop inner join  (actual time=0.15..38.2 rows=145230 loops=1)
                  -> Index lookup on c using idx_region (region='US')
                     (actual time=0.08..4.23 rows=18432 loops=1)
                  -> Index lookup on o using idx_cust (customer_id=c.id)
                     (actual time=0.001..0.002 rows=7.88 loops=18432)

  Diagnosis: "Using temporary; Using filesort" → add composite index for GROUP BY + ORDER BY
```
