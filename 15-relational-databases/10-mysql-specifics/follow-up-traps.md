# MySQL Specifics — Follow-Up Traps

Tricky follow-up questions exposing gaps in MySQL internals knowledge.

---

## Trap 1: "InnoDB and MyISAM both support transactions — InnoDB just has more features."

**What most people say:** Both engines support some form of transaction-like behavior.

**Correct answer:** MyISAM does NOT support transactions at all. MyISAM uses table-level locking, has no ACID guarantees, and any statement that fails mid-way leaves the table in a partially modified state with no rollback mechanism. It also does not support foreign keys or row-level locking. InnoDB is the MySQL default since MySQL 5.5 and provides full ACID transactions with row-level locking, crash recovery via redo logs, MVCC for consistent reads, and foreign key constraints. MyISAM's only legitimate modern use case is for very read-heavy tables that never update (like lookup tables), where table-level locking and the lack of MVCC overhead provide a marginal read performance edge — and even then, InnoDB's covering indexes typically make this irrelevant.

```sql
-- Demonstrate MyISAM's lack of transactions:
-- In MySQL with a MyISAM table:
START TRANSACTION;
INSERT INTO myisam_table VALUES (1, 'test');
ROLLBACK;  -- Has NO effect! Row is committed immediately in MyISAM.

-- Confirm the row was inserted despite ROLLBACK:
SELECT * FROM myisam_table;  -- Row is there!

-- InnoDB correctly rolls back:
START TRANSACTION;
INSERT INTO innodb_table VALUES (1, 'test');
ROLLBACK;
SELECT * FROM innodb_table;  -- Row is gone.
```

---

## Trap 2: "UUID primary keys are fine as the clustered index in MySQL InnoDB."

**What most people say:** UUIDs are unique and convenient for distributed systems — perfect for primary keys.

**Correct answer:** Random UUID (UUIDv4) primary keys in InnoDB cause severe performance problems because InnoDB's clustered index is physically ordered by primary key. With random UUIDs, every INSERT goes to a random location in the B-tree, causing: (1) frequent page splits, (2) highly fragmented B-tree with poor space utilization, (3) secondary indexes that include the 16-byte UUID as a suffix (bloating all index sizes), and (4) poor buffer pool efficiency (random page access pattern). For high-write workloads, sequential UUIDs (UUIDv7, ULID, or MySQL's UUID_TO_BIN(UUID(), 1) which reorders bytes to be time-sequential) mitigate most of these issues while preserving uniqueness.

```sql
-- BAD: Random UUID4 as primary key (causes page splits)
CREATE TABLE events (
  id CHAR(36) DEFAULT (UUID()) PRIMARY KEY,  -- random UUID4
  data TEXT
);

-- BETTER: Sequential UUID using MySQL 8.0 bit manipulation
CREATE TABLE events (
  id BINARY(16) DEFAULT (UUID_TO_BIN(UUID(), 1)) PRIMARY KEY,  -- time-ordered
  data TEXT
);

-- BEST for most cases: BIGINT AUTO_INCREMENT (sequential, compact, 8 bytes)
CREATE TABLE events (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  data TEXT
);
-- Secondary index entry size: 8 bytes (BIGINT) vs 16 bytes (UUID BINARY)
-- All secondary indexes implicitly include the PK suffix
```

---

## Trap 3: "EXPLAIN showing 'Using index' means an index is being used for the query."

**What most people say:** 'Using index' just means the query uses an index.

**Correct answer:** In MySQL's EXPLAIN output, "Using index" specifically means a covering index is being used — all columns needed to resolve the query are available in the index itself, and the table data file (the clustered index) does NOT need to be accessed. This is more informative than just "an index is used." A plain index scan (without "Using index" in Extra) still accesses the index AND the primary key (clustered index) to retrieve the row data. The distinction matters for performance: a covering index scan reads only the index pages; a non-covering index scan reads index pages PLUS the corresponding data pages (two reads per row).

```sql
-- Table: orders(id PK, customer_id, status, total, notes)
-- Index: idx_customer_status ON (customer_id, status)

-- EXPLAIN Extra: "Using index" (covering index — no table access)
EXPLAIN SELECT customer_id, status
FROM orders WHERE customer_id = 42;
-- idx_customer_status covers all needed columns

-- EXPLAIN Extra: (empty or "Using index condition") — NOT covering
EXPLAIN SELECT customer_id, status, total
FROM orders WHERE customer_id = 42;
-- 'total' not in idx_customer_status → must access clustered index
-- Add total to index to make it covering:
ALTER TABLE orders ADD INDEX idx_customer_status_total (customer_id, status, total);
```

---

## Trap 4: "MySQL's MVCC works the same way as PostgreSQL's MVCC."

**What most people say:** Both databases implement MVCC for consistent reads — the implementation is essentially the same.

**Correct answer:** They are architecturally very different. PostgreSQL stores ALL versions of a row in the heap file — old versions remain until VACUUM removes them. MySQL InnoDB stores only the current version in the clustered index; old versions are stored in separate undo log segments (rollback segments). When a read needs an older version, InnoDB reconstructs it by applying undo records backward. This has opposite trade-offs: PostgreSQL's approach means reads never need to access undo logs (only VACUUM is needed for cleanup), but causes table bloat under heavy UPDATEs. InnoDB's undo log approach keeps the primary data compact but requires I/O to undo segments for long-running reads of frequently-updated rows. Under heavy UPDATE workloads, InnoDB's undo segments can grow very large (and cannot shrink in older versions).

```
PostgreSQL MVCC:
  Heap: [xmin=100, 'Alice'] [xmin=150, 'Alicia']  ← both versions in heap
  VACUUM removes old version when safe

  Read needs old version: already in heap, no extra I/O

InnoDB MVCC:
  Clustered index: ['Alicia']  ← only current version
  Undo log segment: [revert to: 'Alice']

  Read needs old version: read clustered index → apply undo → reconstruct 'Alice'
  Long-running read + frequent updates = deep undo chain to traverse
```

---

## Trap 5: "Setting innodb_flush_log_at_trx_commit=0 is the same as innodb_flush_log_at_trx_commit=2."

**What most people say:** Both values turn off per-transaction fsync, so they are equivalent.

**Correct answer:** They have critically different crash recovery behaviors:
- `= 1` (default): Log written to OS AND fsynced to disk on every commit. Full ACID durability.
- `= 2`: Log written to OS buffer on commit, fsynced to disk once per second. If MySQL crashes, committed transactions in the OS buffer are safe (OS flushes on a clean shutdown or OS handles the cache). If the entire SERVER crashes (power failure), up to 1 second of committed transactions can be lost.
- `= 0`: Log written to InnoDB's in-memory buffer only, flushed to OS AND fsynced once per second. If MySQL process crashes, up to 1 second of committed transactions in the InnoDB buffer are lost even without a power failure.

The practical consequence: setting 0 loses data on any crash (even a MySQL process crash); setting 2 only loses data on an OS/hardware crash. In cloud environments with reliable power, 2 is often acceptable for a 5-10x write throughput improvement; 0 is almost never appropriate for production.

---

## Trap 6: "GTID replication makes it safe to use any replica as a new source immediately after failover."

**What most people say:** GTID tracks all transactions globally, so any replica can become source without data loss.

**Correct answer:** GTID ensures consistency (executed transactions are tracked by ID and not re-executed) but does NOT guarantee that all replicas are caught up with the source at failover time. With asynchronous replication, a replica could be 100,000 transactions behind the source when it fails. After promoting that replica, those 100,000 transactions are permanently lost — GTID cannot recover them. GTID's benefit is that the new source can safely accept replicas that have different binary log positions (GTID auto-positioning finds the right starting point) and prevents accidentally re-executing transactions. Zero data loss requires: (1) semi-synchronous replication with at least one replica, OR (2) choosing the most up-to-date replica (highest GTID executed set) for promotion.

```sql
-- Check GTID executed sets on all replicas before failover:
SHOW MASTER STATUS;  -- on source (or SHOW REPLICA STATUS on replica)
-- gtid_executed shows all GTIDs that have been applied

-- On replica, check how far behind:
SHOW REPLICA STATUS\G
-- Executed_Gtid_Set vs Retrieved_Gtid_Set shows if all received GTIDs are applied
-- Auto_Position: 1 confirms GTID auto-positioning is active
```

---

## Trap 7: "MySQL's query cache improves performance for read-heavy workloads."

**What most people say:** The query cache caches query results so repeated identical queries are instant.

**Correct answer:** MySQL's query cache was REMOVED in MySQL 8.0 because it was found to be a net performance NEGATIVE in most production workloads. The problems: (1) It uses a single global mutex, so high-concurrency read workloads caused severe mutex contention. (2) Any write to a table invalidates ALL cached queries for that table — in OLTP systems with mixed reads/writes, the cache hit rate was typically near zero. (3) The cache is byte-exact match — `SELECT * FROM t WHERE id = 1` and `select * from t where id = 1` are different cache entries. The correct solution for query result caching is an application-level cache (Redis, Memcached) which has higher hit rates, better eviction policies, and no database-side mutex contention.

---

## Trap 8: "Adding more indexes to a MySQL table only affects write performance, not read performance."

**What most people say:** More indexes = slower writes, but always faster reads (can only help, never hurt reads).

**Correct answer:** Too many indexes can hurt read performance in MySQL through: (1) The optimizer spending more time evaluating access paths (especially with 10+ indexes). (2) Index merge sometimes choosing a worse plan than a good composite index. (3) Index statistics becoming less accurate when the optimizer must choose between many overlapping indexes. (4) Buffer pool pollution — unused indexes consume buffer pool space that could cache actual data. The most common case where a read is SLOWER with an extra index: when the optimizer incorrectly estimates that a low-selectivity index is highly selective (due to stale statistics) and chooses an index scan with high "rows_examined" when a full table scan would have been faster. Always run `ANALYZE TABLE` after bulk changes.

```sql
-- Find unused indexes in MySQL (performance_schema must be enabled):
SELECT
  object_schema,
  object_name AS table_name,
  index_name,
  count_star AS times_used,
  count_read AS reads_served
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema NOT IN ('mysql', 'information_schema', 'performance_schema')
  AND count_star = 0                  -- Never used since last restart
  AND index_name IS NOT NULL
  AND index_name != 'PRIMARY'
ORDER BY object_schema, object_name;
```

---

## Trap 9: "MySQL 8.0 window functions work the same as in PostgreSQL."

**What most people say:** Window functions are ANSI SQL and work identically across databases.

**Correct answer:** The syntax and semantics of window functions are largely the same (ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, SUM OVER, etc.) but there are important behavioral differences: (1) MySQL's NTILE uses a different tie-breaking rule for uneven bucket sizes than PostgreSQL in some edge cases. (2) MySQL does not support the FILTER clause in window aggregates (`SUM(x) FILTER (WHERE ...)`) while PostgreSQL does. (3) MySQL 8.0 introduced window functions in 8.0.2 — earlier 5.7 and 8.0.0/8.0.1 have NO window function support at all. (4) MySQL's GROUP BY desugaring of `SELECT id, name FROM t GROUP BY id` (without id in aggregate or all columns in GROUP BY) produces unpredictable results and is controlled by ONLY_FULL_GROUP_BY sql_mode — PostgreSQL requires explicit GROUP BY or aggregate for all non-aggregate columns.

---

## Trap 10: "FOREIGN KEY constraints always improve data integrity in MySQL."

**What most people say:** Foreign keys are always good practice and improve data integrity.

**Correct answer:** Foreign keys in InnoDB provide referential integrity but at a real performance cost: every INSERT/UPDATE/DELETE on the referencing table requires a lock on the referenced table to verify the constraint. In high-throughput OLTP systems with deep referential relationships, FK checking can cause significant lock contention and reduce write throughput. Additionally, foreign keys make certain operations much harder: loading data in bulk (must load in dependency order or disable FK checks), dropping tables (must drop in reverse dependency order), and schema changes (ALTER TABLE on a parent table can block indefinitely while waiting for FK locks). Many high-scale systems (Facebook, Airbnb) deliberately choose to NOT use FK constraints, enforcing referential integrity in the application layer instead. The decision is a trade-off, not a universal best practice.

```sql
-- Disable FK checks for bulk loads (must re-enable and verify afterward):
SET FOREIGN_KEY_CHECKS = 0;
LOAD DATA INFILE '/data/orders.csv' INTO TABLE orders;
SET FOREIGN_KEY_CHECKS = 1;

-- Verify referential integrity after re-enabling:
SELECT COUNT(*) FROM orders o
LEFT JOIN customers c ON o.customer_id = c.id
WHERE c.id IS NULL;
-- Should be 0; if not, you have orphaned rows that FOREIGN_KEY_CHECKS was masking.
```

---

## Trap 11: "The InnoDB buffer pool should be set to 80% of available RAM."

**What most people say:** More buffer pool = more data cached = better performance. Use 80%.

**Correct answer:** Setting innodb_buffer_pool_size too high causes the OS to run out of memory for file system caching, process memory, and other operations — leading to OOM kills of the mysqld process itself. The generally safe maximum is 70-75% of available RAM for a dedicated MySQL server. However, unlike PostgreSQL which also benefits from the OS page cache, InnoDB generally bypasses OS caching for its data files (using direct I/O in some configurations) — so setting the buffer pool as large as safely possible is more important in MySQL than in PostgreSQL. The remaining 25-30% should cover: operating system needs (~1-2GB), per-thread memory (sort_buffer_size × max_connections, tmp_table_size, join_buffer_size), and InnoDB's own operational buffers outside the pool.

```ini
# my.cnf for a 128GB dedicated MySQL server
innodb_buffer_pool_size = 96G        # 75% of 128GB
innodb_buffer_pool_instances = 8     # One per 1-2GB (reduces contention)

# Per-thread memory (not from buffer pool):
sort_buffer_size = 256K              # Keep small! Allocated per sort per thread
join_buffer_size = 256K              # Keep small! One per join without index
tmp_table_size = 64M                 # Max in-memory temp table size
max_heap_table_size = 64M            # Must match tmp_table_size
```

---

## Trap 12: "MySQL's EXPLAIN shows you exactly what query plan will be executed."

**What most people say:** EXPLAIN shows the actual execution plan.

**Correct answer:** MySQL's standard EXPLAIN shows the PLANNED execution plan based on statistics at the time of the EXPLAIN call. The actual execution can differ because: (1) Statistics may be stale and get refreshed between EXPLAIN and execution. (2) MySQL can change plans at runtime based on actual row counts (Adaptive Query Execution — available in MySQL 8.0.26+ as a limited form). (3) For prepared statements, the plan may be re-evaluated on each execution with different bind values. EXPLAIN ANALYZE (MySQL 8.0.18+) actually executes the query and shows both estimated and actual values — the correct tool for diagnosis:

```sql
-- Standard EXPLAIN: estimates only (fast, safe for production)
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- EXPLAIN ANALYZE: actually executes (use in dev/staging, wraps in transaction if needed)
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;
-- Output includes: actual_rows=N, actual_time=Nms, loops=N

-- EXPLAIN FORMAT=JSON: more detailed output including cost breakdown
EXPLAIN FORMAT=JSON SELECT * FROM orders WHERE customer_id = 42;
-- Includes cost estimate per operation, index condition pushdown details

-- For mutating statements (DELETE/UPDATE), wrap EXPLAIN ANALYZE safely:
BEGIN;
EXPLAIN ANALYZE DELETE FROM sessions WHERE expires_at < NOW() LIMIT 1000;
ROLLBACK;
```
