# MySQL Specifics — Structured Answers

Complete interview-ready answers covering InnoDB internals, replication, query optimization, and MySQL vs PostgreSQL comparison.

---

## Q1: Explain the InnoDB architecture — buffer pool, redo log, undo log, and doublewrite buffer.

**Answer:**

InnoDB is a transactional storage engine with a complex multi-component architecture designed for ACID compliance and crash recovery.

**Buffer Pool:**
```ini
# my.cnf
innodb_buffer_pool_size = 12G              # 70-75% of RAM on dedicated server
innodb_buffer_pool_instances = 8           # Reduce contention; 1 per ~1-2GB
innodb_buffer_pool_chunk_size = 128M       # Resize granularity
```

The buffer pool caches data and index pages using a modified LRU list. The midpoint insertion strategy places new pages in the middle (not the head) to prevent large full-table scans from evicting frequently accessed hot data. Two sublists: the "young" (recently accessed) and "old" (candidates for eviction) lists.

```sql
-- Monitor buffer pool state:
SELECT
  pool_id,
  pool_size,
  free_buffers,
  database_pages,
  old_database_pages,
  pages_made_young,
  pages_made_not_young,
  pages_read,
  pages_written,
  hit_rate / 1000.0 AS hit_rate_pct
FROM information_schema.INNODB_BUFFER_POOL_STATS;
```

**Redo Log (for crash recovery):**
- Stores physical changes (page-level modifications) to be replayed after crash
- Written to disk before data pages (WAL principle)
- MySQL 8.0.30+: uses dedicated redo log directory, configured with innodb_redo_log_capacity
```ini
innodb_redo_log_capacity = 8G     # MySQL 8.0.30+ (replaces innodb_log_file_size)
```

**Undo Log (for MVCC and rollback):**
- Stores before-images of modified rows
- Used to reconstruct old row versions for consistent reads
- Used to roll back transactions on ROLLBACK or failure
- Stored in undo tablespaces (separate files in MySQL 8.0)
```ini
innodb_undo_tablespaces = 4       # Separate undo tablespace files
innodb_max_undo_log_size = 1G     # Per-tablespace size before truncation
innodb_undo_log_truncate = ON     # Auto-truncate large undo tablespaces
```

**Doublewrite Buffer (for torn page protection):**
Before writing modified pages to their actual location, InnoDB first writes them to the doublewrite buffer (a sequential area). If a crash occurs during the actual write, recovery can use the complete page from the doublewrite buffer. This prevents partial page writes (torn pages) corrupting the data files.
```ini
innodb_doublewrite = ON           # Default ON; disable only on ZFS/atomic-write storage
```

**Change Buffer:**
Buffers secondary index changes when the index pages are not in the buffer pool. Merged when pages are read into buffer pool. Reduces random I/O for writes.
```ini
innodb_change_buffer_max_size = 25  # % of buffer pool for change buffer (default 25%)
innodb_change_buffering = all       # Buffer inserts, deletes, changes
```

---

## Q2: Configure MySQL replication with GTID and explain the failover procedure.

**Answer:**

**Source configuration (my.cnf):**
```ini
[mysqld]
server_id = 1
gtid_mode = ON
enforce_gtid_consistency = ON
binlog_format = ROW                    # Required for GTID, best for safety
binlog_row_image = FULL                # Log full row (before + after images)
log_bin = /var/log/mysql/mysql-bin
log_replica_updates = ON               # Replicas also write to binlog (for chaining)
sync_binlog = 1                        # Flush binlog to disk per transaction (durability)
innodb_flush_log_at_trx_commit = 1    # Redo log flushed per commit (full ACID)
binlog_expire_logs_seconds = 604800    # Retain binlogs for 7 days
```

**Replica configuration (my.cnf):**
```ini
[mysqld]
server_id = 2                          # Must be unique across all servers
gtid_mode = ON
enforce_gtid_consistency = ON
log_bin = /var/log/mysql/mysql-bin
log_replica_updates = ON
relay_log_recovery = ON               # Recover relay log on crash (important!)
replica_parallel_type = LOGICAL_CLOCK # Multi-threaded replication
replica_parallel_workers = 8          # Match source's CPU count
replica_preserve_commit_order = ON    # Maintain commit order (required with LOGICAL_CLOCK)
```

**Set up replication:**
```sql
-- On replica:
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'source-server',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'replicator',
  SOURCE_PASSWORD = 'secure_password',
  SOURCE_AUTO_POSITION = 1;            -- GTID auto-positioning

START REPLICA;
SHOW REPLICA STATUS\G
-- Key fields: Replica_IO_Running, Replica_SQL_Running → both must be 'Yes'
-- Seconds_Behind_Source should be near 0
```

**Create replication user (on source):**
```sql
CREATE USER 'replicator'@'10.0.0.%' IDENTIFIED BY 'secure_password';
GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'10.0.0.%';
```

**Monitor replication lag:**
```sql
-- On replica:
SHOW REPLICA STATUS\G
-- Seconds_Behind_Source: seconds of replication lag

-- More precise lag from performance_schema:
SELECT
  CHANNEL_NAME,
  SERVICE_STATE,
  COUNT_RECEIVED_HEARTBEATS,
  LAST_HEARTBEAT_TIMESTAMP,
  LAST_QUEUED_TRANSACTION,
  LAST_APPLIED_TRANSACTION,
  APPLYING_TRANSACTION
FROM performance_schema.replication_connection_status;
```

**GTID failover procedure:**
```bash
# 1. Identify the replica with the most complete GTID set
# On each replica:
mysql -e "SHOW REPLICA STATUS\G" | grep Executed_Gtid_Set

# 2. Choose the replica with the most GTIDs applied
# 3. Promote it:
mysql -h replica1 -e "STOP REPLICA; RESET REPLICA ALL;"
# Now accepts writes

# 4. Point other replicas to the new source:
mysql -h replica2 -e "
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = 'replica1',
  SOURCE_AUTO_POSITION = 1;
START REPLICA;"

# 5. Update application connection string to point to new source
```

---

## Q3: How does InnoDB locking work and how do you diagnose deadlocks?

**Answer:**

InnoDB uses row-level locking with multiple lock types. Understanding the locking model is essential for diagnosing and preventing deadlocks.

**InnoDB lock types:**
- **Shared (S) lock**: Allows concurrent reads, blocks writes
- **Exclusive (X) lock**: Blocks all concurrent access to the row
- **Intention locks** (table-level): Signal intent to lock rows (IS, IX)
- **Gap locks**: Lock the gap between index values (prevents phantom reads in REPEATABLE READ)
- **Next-key locks**: Row lock + gap lock (default locking mode in REPEATABLE READ)
- **Insert intention locks**: Before inserting, signal intent (multiple inserts can proceed if no conflict)

**Classic deadlock scenario:**
```sql
-- Transaction A (session 1):
BEGIN;
UPDATE orders SET status = 'processing' WHERE id = 100;  -- X lock on row 100
-- ... some delay ...
UPDATE inventory SET qty = qty - 1 WHERE product_id = 50; -- waits for TX B

-- Transaction B (session 2, concurrent):
BEGIN;
UPDATE inventory SET qty = qty - 1 WHERE product_id = 50;  -- X lock on inventory 50
-- ... some delay ...
UPDATE orders SET status = 'processing' WHERE id = 100;    -- waits for TX A
-- DEADLOCK! Each transaction holds one lock and waits for the other's lock.
```

**InnoDB deadlock detection:**
InnoDB automatically detects deadlocks using a lock dependency graph. When a cycle is detected, InnoDB selects the transaction with fewer undo log records (approximately "less work done") as the victim and rolls it back with error 1213.

**Monitoring deadlocks:**
```sql
-- Last deadlock information:
SHOW ENGINE INNODB STATUS\G
-- Look for the "LATEST DETECTED DEADLOCK" section

-- Detailed deadlock info from performance_schema:
SELECT
  r.trx_id AS waiting_trx,
  r.trx_mysql_thread_id AS waiting_thread,
  r.trx_query AS waiting_query,
  b.trx_id AS blocking_trx,
  b.trx_mysql_thread_id AS blocking_thread,
  b.trx_query AS blocking_query
FROM information_schema.innodb_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_trx_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_trx_id;

-- Current lock waits (MySQL 8.0):
SELECT *
FROM performance_schema.data_lock_waits
LIMIT 10;

-- sys schema view (more readable):
SELECT * FROM sys.innodb_lock_waits\G
```

**Eliminating the deadlock:**
```sql
-- Fix: Always acquire locks in the SAME order across all transactions
-- Transaction A and B should both lock in order: orders → inventory

-- Transaction A (fixed):
BEGIN;
UPDATE orders SET status = 'processing' WHERE id = 100;
UPDATE inventory SET qty = qty - 1 WHERE product_id = 50;
COMMIT;

-- Transaction B (fixed — same lock order):
BEGIN;
UPDATE orders SET status = 'processing' WHERE id = 100;  -- Now they queue, not deadlock
UPDATE inventory SET qty = qty - 1 WHERE product_id = 50;
COMMIT;
```

---

## Q4: Compare MySQL vs PostgreSQL — key differences for an architect decision.

**Answer:**

```
CATEGORY              MYSQL (InnoDB)                    POSTGRESQL
──────────────────────────────────────────────────────────────────────────────
MVCC Architecture     Undo logs (rollback segments)     Tuple versioning in heap
                      Old versions reconstructed         Old versions in table
                      InnoDB keeps data compact          PostgreSQL needs VACUUM
                      Undo bloat under long transactions Table bloat under heavy UPDATE

Storage Engine        Pluggable (InnoDB default,         Single storage engine
                      MyISAM, Archive, etc.)             (no plugin system)

Replication           Binlog-based (logical)             WAL-based (physical) primary
                      GTID-based, row/statement/mixed    Also: logical replication
                      Semi-synchronous plugin            Synchronous commit built-in
                      Group Replication (Galera-like)    Patroni for HA orchestration

JSON Support          JSON data type (MySQL 5.7+)        JSONB data type
                      JSON_TABLE for shredding           @> operator, GIN indexes
                      Functional indexes on JSON paths   More complete JSON functions
                      Binary format, but less indexable  JSONB is heavily indexed

Full-Text Search      FULLTEXT index (InnoDB 5.6+)       tsvector/tsquery, GIN index
                      Limited language support           Better language support
                      No GIN equivalent for partial      pg_trgm for LIKE/ILIKE

Partitioning          RANGE, LIST, HASH, KEY             RANGE, LIST, HASH (declarative)
                      Older syntax, more limitations     PG10+: better pruning, indexes
                      No sub-partitioning inheritance    Foreign keys across partitions

Extensions            Plugin system (less flexible)      Rich extension ecosystem
                      No equivalent to PostGIS quality   PostGIS, pg_trgm, TimescaleDB
                      No pg_stat_statements equivalent   pgaudit, many contrib extensions

Window Functions      Added in MySQL 8.0                 Available since PostgreSQL 8.4
                      No FILTER clause in aggregates     Full FILTER support
                      Missing some frame specifications  Complete frame specification

CTEs                  MySQL 8.0+, always materialized    PG12+: inline by default
                      RECURSIVE CTEs supported           RECURSIVE CTEs supported

Licensing             GPL (Community), Commercial        PostgreSQL License (permissive)
                      Oracle-owned, commercial support   Community-governed, truly open

Cloud Options         AWS RDS MySQL, Aurora MySQL        AWS RDS PostgreSQL, Aurora PG
                      Google Cloud SQL MySQL             Google Cloud SQL PG, AlloyDB
                      Azure Database for MySQL           Azure Database for PG
                                                        Neon, Supabase (PG-native)

Scaling               Group Replication, Vitess          Citus for horizontal sharding
                      ProxySQL for routing               No native horizontal sharding
                      Aurora auto-scaling                Aurora PG auto-scaling
```

**When to choose MySQL:**
- Migrating from legacy MySQL systems (lowest friction)
- Need Group Replication / Galera cluster (synchronous multi-master)
- Aurora MySQL ecosystem already in use
- Team has deep MySQL expertise
- Simple OLTP with well-understood access patterns

**When to choose PostgreSQL:**
- Complex queries, analytics, JOINs on many tables
- Need PostGIS for geographic data
- Time-series data (TimescaleDB extension)
- Complex JSON querying with GIN indexes
- Full-text search with language support
- Need extensibility (custom types, operators, index access methods)
- Strict ANSI SQL compliance requirements

---

## Q5: How do you tune MySQL for high-write OLTP performance?

**Answer:**

```ini
# my.cnf — Tuned for high-write OLTP (128GB RAM, NVMe storage, 32 cores)
[mysqld]

# ─── Buffer Pool ───────────────────────────────────────────────────────────
innodb_buffer_pool_size = 96G          # 75% of RAM
innodb_buffer_pool_instances = 16      # One per 6GB of pool
innodb_buffer_pool_chunk_size = 128M

# ─── I/O and Redo Log ──────────────────────────────────────────────────────
innodb_redo_log_capacity = 16G         # Large redo log → fewer I/O spikes (8.0.30+)
innodb_flush_method = O_DIRECT         # Bypass OS buffer cache for data files
innodb_io_capacity = 4000              # IOPS for background tasks (NVMe)
innodb_io_capacity_max = 8000          # Max IOPS during flushing spikes
innodb_read_io_threads = 8
innodb_write_io_threads = 8

# ─── Durability Trade-offs ─────────────────────────────────────────────────
innodb_flush_log_at_trx_commit = 1     # Full ACID (1); use 2 for 5-10x write boost
sync_binlog = 1                        # Flush binlog per commit (durability)

# ─── Undo and Purge ────────────────────────────────────────────────────────
innodb_undo_tablespaces = 4
innodb_max_undo_log_size = 2G
innodb_undo_log_truncate = ON
innodb_purge_threads = 4               # Parallel purge of undo records

# ─── Connection and Thread ─────────────────────────────────────────────────
max_connections = 500                  # Use ProxySQL for connection pooling
thread_stack = 512K
thread_cache_size = 64                 # Reuse OS threads for new connections
innodb_thread_concurrency = 0          # Let InnoDB manage (0 = no limit)

# ─── Query and Sort Buffers (per-session) ─────────────────────────────────
sort_buffer_size = 256K                # Per-thread, keep SMALL
join_buffer_size = 256K                # Per-thread, keep SMALL
tmp_table_size = 64M
max_heap_table_size = 64M             # Must match tmp_table_size
read_buffer_size = 256K
read_rnd_buffer_size = 512K

# ─── Logging ───────────────────────────────────────────────────────────────
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow-query.log
long_query_time = 0.1                  # Log queries > 100ms
log_queries_not_using_indexes = ON
min_examined_row_limit = 1000         # Only log if examining >1000 rows
```

**Monitor buffer pool efficiency:**
```sql
-- Buffer pool hit rate should be > 99%
SELECT
  ROUND((1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100, 2)
    AS buffer_pool_hit_rate
FROM (
  SELECT
    VARIABLE_VALUE AS Innodb_buffer_pool_reads
  FROM performance_schema.global_status
  WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) r
CROSS JOIN (
  SELECT VARIABLE_VALUE AS Innodb_buffer_pool_read_requests
  FROM performance_schema.global_status
  WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) rr;
```
