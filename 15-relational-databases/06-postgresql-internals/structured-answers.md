# PostgreSQL Internals — Structured Answers

Complete interview-ready answers for key topics. Each answer includes code, configuration examples, and nuanced details expected at the senior/architect level.

---

## Q1: Explain PostgreSQL's MVCC implementation using xmin/xmax/cmin/cmax.

**Answer:**

PostgreSQL implements Multi-Version Concurrency Control (MVCC) by storing multiple versions of each row directly in the heap. Unlike Oracle or MySQL InnoDB (which use undo segments), PostgreSQL keeps old row versions in the table itself.

Every heap tuple has four hidden system columns:

- **xmin**: Transaction ID (XID) of the transaction that INSERT-ed this tuple version into existence.
- **xmax**: XID of the transaction that DELETE-d or UPDATE-d this tuple (making it dead). Zero if still live.
- **cmin**: Command ID within the creating transaction (for intra-transaction visibility).
- **cmax**: Command ID within the deleting transaction.

**Visibility Rule (simplified):** A tuple is visible to a snapshot S if:
1. xmin is committed AND xmin < S.xmin, OR xmin is in S.xip (in-progress but committed before snapshot was taken)
2. AND (xmax = 0, OR xmax is NOT committed, OR xmax > S.xmax)

**UPDATE mechanics:**
```sql
-- Logical: UPDATE orders SET status = 'shipped' WHERE id = 42;

-- Physical storage result:
-- Old tuple: xmin=100, xmax=150, status='pending'  ← dead tuple, not reclaimed yet
-- New tuple: xmin=150, xmax=0,   status='shipped'  ← live tuple
```

The old tuple remains in the heap until VACUUM removes it. This is why PostgreSQL tables can bloat under high-UPDATE workloads.

**Inspecting system columns:**
```sql
-- See xmin/xmax on rows (requires no table alias ambiguity)
SELECT xmin, xmax, cmin, cmax, id, status
FROM orders
WHERE id = 42;
```

**Snapshot structure:**
```
SnapshotData {
  xmin:  100   -- ignore tuples with xmin < 100 (already committed before me)
  xmax:  155   -- transactions >= 155 are in the future, not visible
  xip:   [101, 103, 107]  -- in-progress XIDs at snapshot time, not visible
}
```

**Interview insight:** The key architectural difference from undo-log MVCC is that old tuple versions in PostgreSQL accumulate in the heap, requiring VACUUM as a mandatory maintenance process. In Oracle, old versions are in undo tablespace and automatically overwritten. PostgreSQL's approach simplifies crash recovery but requires active dead-tuple management.

---

## Q2: Describe the WAL pipeline: from dirty buffer to durable storage.

**Answer:**

When a transaction modifies data in PostgreSQL, the path to durable storage is:

```
1. Transaction modifies tuple in shared_buffers (in memory)
2. WAL record constructed describing the change (not the new data page)
3. WAL record written to WAL buffers (wal_buffers, default 1/32 of shared_buffers)
4. At COMMIT: WAL buffers flushed to WAL segment files via fsync() [if synchronous_commit=on]
5. Dirty shared_buffers page written to heap file later (by bgwriter or checkpoint)
```

**Key parameters:**

```ini
# postgresql.conf

# WAL durability level
wal_level = replica              # minimal | replica | logical

# Flush WAL to disk at each commit (durability guarantee)
fsync = on                       # NEVER set to off in production

# Delay ACK until WAL is flushed (set off for async commits)
synchronous_commit = on          # on | remote_apply | remote_write | local | off

# WAL buffer size (larger = fewer flushes under bursty writes)
wal_buffers = 64MB               # default: -1 (auto = 1/32 shared_buffers, capped 64MB)

# Maximum WAL retained before checkpoint forced
max_wal_size = 4GB               # larger = longer recovery, less I/O spiking

# Spread checkpoint I/O over this fraction of checkpoint interval
checkpoint_completion_target = 0.9   # 0.5–0.9, higher = smoother I/O
```

**Crash recovery:** On restart after an ungraceful shutdown, PostgreSQL reads the control file to find the last checkpoint position in WAL, then replays all WAL records from that point forward. This is why max_wal_size directly controls maximum recovery time: if max_wal_size = 4GB and WAL write speed is ~500MB/s, worst-case recovery is ~8 seconds. More WAL between checkpoints means longer recovery.

**synchronous_commit modes:**
```
on              → WAL flushed to local disk before ACK
remote_write    → WAL sent to standby (not necessarily flushed) before ACK
remote_apply    → WAL applied on standby before ACK (zero data loss)
local           → WAL flushed locally, async to standby
off             → ACK before WAL flush (up to ~200ms data loss risk)
```

---

## Q3: How does VACUUM prevent transaction ID wraparound, and what happens if it fails?

**Answer:**

PostgreSQL uses 32-bit transaction IDs (XIDs). With ~4 billion possible XIDs, a long-running cluster will eventually wrap around. If wraparound occurs without preparation, all tuples in the database become "in the future" and invisible — effectively destroying the entire database.

**Prevention mechanism — tuple freezing:**
VACUUM freezes tuples by replacing xmin with a special FrozenXID marker (XID 2), which is defined as "older than all transactions." A frozen tuple is always visible regardless of the current XID counter.

```sql
-- Check how close databases are to wraparound
SELECT datname,
       age(datfrozenxid) AS xid_age,
       2147483648 - age(datfrozenxid) AS xids_remaining,
       pg_size_pretty(pg_database_size(datname)) AS db_size
FROM pg_database
WHERE datallowconn = true
ORDER BY age(datfrozenxid) DESC;

-- PostgreSQL will force autovacuum (aggressive) at:
-- autovacuum_freeze_max_age (default 200M XID)
-- And will refuse ALL new transactions at:
-- 2^31 - 3 = 2,147,483,645 age (hard limit)
```

**Key thresholds:**
- `vacuum_freeze_min_age` (default 50M): Don't freeze tuples newer than this — they might still be needed for MVCC.
- `vacuum_freeze_table_age` (default 150M): Run aggressive VACUUM (freeze all pages) when table's age exceeds this.
- `autovacuum_freeze_max_age` (default 200M): Force autovacuum even if table has no dead tuples.

**If wraparound prevention fails:**
PostgreSQL will emit WARNING messages at 40M XIDs before the limit, then FATAL at 3M XIDs, then stop accepting new transactions entirely, with the message:
```
ERROR: database is not accepting commands to avoid wraparound data loss
HINT: Stop the postmaster and vacuum that database.
```

Recovery requires single-user mode with `vacuumdb --freeze`.

**Monitoring alert thresholds:**
```sql
-- Alert if any database is within 500M XIDs of wraparound
SELECT datname, age(datfrozenxid) AS age
FROM pg_database
WHERE age(datfrozenxid) > 1500000000;  -- 1.5 billion
```

---

## Q4: How do you tune autovacuum for a high-write OLTP table?

**Answer:**

Default autovacuum thresholds are designed for "average" tables but are often wrong for very large or very active tables. The trigger condition for autovacuum is:

```
n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × n_live_tup
```

With defaults (threshold=50, scale_factor=0.2), a 10M-row table requires 2,000,050 dead tuples before autovacuum triggers. In a high-write environment this creates enormous bloat windows.

**Per-table storage parameters (preferred over global changes):**
```sql
-- Aggressive autovacuum for a high-churn orders table
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor    = 0.01,   -- trigger at 1% dead tuples (was 20%)
  autovacuum_vacuum_threshold       = 1000,   -- plus absolute minimum 1000 dead tuples
  autovacuum_analyze_scale_factor   = 0.005,  -- update statistics at 0.5% change
  autovacuum_vacuum_cost_delay      = 2,      -- 2ms delay between cost limit hits (was 20ms)
  autovacuum_vacuum_cost_limit      = 400     -- cost budget per round (was 200)
);
```

**Cost-based delay explained:**
Autovacuum reads pages and charges a cost:
- `vacuum_cost_page_hit` = 1 (buffer cache hit)
- `vacuum_cost_page_miss` = 10 (disk read)
- `vacuum_cost_page_dirty` = 20 (dirty page write)

When accumulated cost exceeds `autovacuum_vacuum_cost_limit`, autovacuum sleeps for `autovacuum_vacuum_cost_delay`. On fast NVMe storage, the cost delays are often too conservative — reduce cost_delay to 2ms or even 0 for storage that can handle it.

**Global configuration:**
```ini
# postgresql.conf — for NVMe-backed OLTP instances
autovacuum_max_workers = 6              # default 3; more workers = more tables in parallel
autovacuum_vacuum_cost_delay = 2ms      # reduce from 20ms for fast storage
autovacuum_vacuum_cost_limit = 400      # increase from 200
log_autovacuum_min_duration = 250ms     # log slow autovacuum runs for visibility
```

---

## Q5: Explain the PostgreSQL checkpoint process and how to tune it for a high-write workload.

**Answer:**

A checkpoint is a consistency point where PostgreSQL guarantees all dirty shared_buffers pages have been written to their heap files. After a checkpoint, crash recovery only needs to replay WAL from the checkpoint position forward.

**Checkpoint trigger conditions:**
1. `max_wal_size` exceeded (WAL size since last checkpoint grew too large)
2. `checkpoint_timeout` elapsed (default 5 minutes)
3. `CHECKPOINT` command issued explicitly

**What the checkpoint process does:**
1. Identify all dirty buffers in shared_buffers
2. Write dirty buffers to heap/index files (spread over checkpoint_completion_target × interval)
3. Issue fsync() to flush OS buffers for all modified files
4. Write a checkpoint record to WAL
5. Update pg_control file with new checkpoint location

**Tuning for high-write OLTP (256GB RAM, NVMe storage):**
```ini
# postgresql.conf

shared_buffers = 64GB                     # 25% of RAM
wal_buffers = 64MB                        # WAL buffer in shared memory
max_wal_size = 16GB                       # Allow large WAL between checkpoints
min_wal_size = 2GB                        # Keep WAL files pre-allocated
checkpoint_completion_target = 0.9        # Spread writes over 90% of interval
checkpoint_timeout = 15min                # Longer interval = fewer checkpoints
                                          # Trade-off: longer crash recovery

# Monitor checkpoint frequency
log_checkpoints = on                      # Log each checkpoint with timing
```

**Diagnosing checkpoint pressure:**
```sql
-- Check checkpoint frequency and write duration
SELECT checkpoints_timed,
       checkpoints_req,                   -- checkpoints forced by max_wal_size (bad)
       checkpoint_write_time / 1000.0 AS write_secs,
       checkpoint_sync_time / 1000.0 AS sync_secs,
       buffers_checkpoint,
       buffers_clean,
       buffers_backend                    -- backends writing dirty pages (very bad)
FROM pg_stat_bgwriter;
```

If `checkpoints_req` >> `checkpoints_timed`, increase `max_wal_size`. If `buffers_backend` is high, the checkpoint and bgwriter cannot keep up — increase `bgwriter_lru_maxpages` and consider more aggressive checkpoint scheduling.

---

## Q6: How do you identify and kill blocking queries using PostgreSQL system views?

**Answer:**

A blocking chain occurs when one transaction holds a lock needed by another. PostgreSQL provides system views to diagnose the entire chain.

```sql
-- Full blocking chain query (PostgreSQL 9.6+)
WITH RECURSIVE lock_tree AS (
  -- Root: queries that are waiting for a lock
  SELECT
    w.pid AS waiting_pid,
    w.usename AS waiting_user,
    w.query AS waiting_query,
    w.state AS waiting_state,
    w.query_start AS waiting_start,
    b.pid AS blocking_pid,
    b.usename AS blocking_user,
    b.query AS blocking_query,
    b.state AS blocking_state,
    b.query_start AS blocking_start
  FROM pg_stat_activity w
  JOIN pg_stat_activity b
    ON b.pid = ANY(pg_blocking_pids(w.pid))
  WHERE cardinality(pg_blocking_pids(w.pid)) > 0
)
SELECT *,
       now() - waiting_start AS wait_duration,
       now() - blocking_start AS blocking_duration
FROM lock_tree
ORDER BY blocking_duration DESC;

-- Kill the blocker (graceful)
SELECT pg_cancel_backend(blocking_pid);

-- Kill the blocker (forceful — terminates the connection)
SELECT pg_terminate_backend(blocking_pid);
```

**Lock compatibility quick reference:**

| Mode held → | ACCESS SHARE | ROW SHARE | ROW EXCL | SHARE UPDATE EXCL | SHARE | SHARE ROW EXCL | EXCL | ACCESS EXCL |
|---|---|---|---|---|---|---|---|---|
| ACCESS SHARE     |   |   |   |   |   |   |   | X |
| ROW EXCLUSIVE    |   |   |   |   | X | X | X | X |
| ACCESS EXCLUSIVE | X | X | X | X | X | X | X | X |

DDL (ALTER TABLE, DROP TABLE) takes ACCESS EXCLUSIVE, blocking ALL concurrent operations including simple SELECTs. This is why long-running DDL in production should use lock_timeout:

```sql
-- Safe DDL with lock timeout
SET lock_timeout = '3s';
ALTER TABLE orders ADD COLUMN notes TEXT;
-- If lock not acquired in 3s, fails instead of blocking indefinitely
```
