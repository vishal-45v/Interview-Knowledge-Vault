# PostgreSQL Internals — Follow-Up Traps

Tricky follow-up questions exposing common misconceptions. Each trap has a specific wrong answer most people give, and the precise correct explanation.

---

## Trap 1: "If VACUUM reclaims dead tuples, why does my table size on disk not shrink?"

**What most people say:** VACUUM frees space and the table file gets smaller.

**Correct answer:** Regular VACUUM marks dead tuple space as reusable in the Free Space Map (FSM) but does NOT return pages to the operating system. The heap file on disk stays the same size or continues to grow. Only VACUUM FULL (or pg_repack) physically rewrites the table and returns space to the OS. This is why you can have a 100GB table with only 20GB of live data — the remaining 80GB is available for future inserts but invisible at the OS level. VACUUM solves the bloat-causes-slow-scans problem only when new writes fill the reclaimed space; it does not solve the disk-full problem.

```sql
-- Check bloat ratio without third-party tools
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS heap_size,
  n_dead_tup,
  n_live_tup,
  round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## Trap 2: "Does synchronous_commit=off mean committed transactions could be lost?"

**What most people say:** No, a committed transaction is always durable.

**Correct answer:** Yes. With synchronous_commit=off, a transaction receives a "COMMIT successful" acknowledgment before its WAL records are flushed to disk. If the server crashes within the window between the commit acknowledgment and the WAL flush (typically up to 3 × wal_writer_delay = ~600ms by default), those committed transactions are silently lost. The application receives no error — it believes the data was persisted. This is a deliberate trade-off for higher write throughput and is acceptable for event logging or metrics, but never for financial or user-critical data. It is NOT the same as the COMMIT failing — the application gets no signal that data was lost.

---

## Trap 3: "Increasing max_connections from 100 to 1000 will fix connection exhaustion, right?"

**What most people say:** Yes, more connections means more capacity.

**Correct answer:** Almost always wrong. Each PostgreSQL backend is a separate OS process consuming roughly 5–10MB of shared memory overhead plus work_mem allocations for active sorts/hashes. At 1000 connections with 64MB work_mem, worst-case memory is enormous. More critically, the lock manager and shared buffer manager use spinlocks protected by shared memory mutexes — with 1000 active processes, contention on these locks collapses CPU throughput. Benchmarks routinely show that PostgreSQL throughput peaks between 100–200 connections and declines sharply beyond that. The correct solution is PgBouncer in transaction-mode pooling: 1000 application connections multiplexed over 30–50 actual PostgreSQL connections gives better throughput and lower latency.

---

## Trap 4: "What happens to VACUUM when there is a long-running transaction?"

**What most people say:** VACUUM runs independently and clears dead tuples regardless.

**Correct answer:** VACUUM cannot remove any dead tuple that could still be visible to an open transaction. The oldest active transaction's xmin acts as the VACUUM horizon — VACUUM will not reclaim any tuple with xmax newer than this horizon, regardless of how dead it is to everyone else. A single idle-in-transaction connection that has been open for 6 hours will prevent VACUUM from reclaiming ANY dead tuples created in that 6-hour window, across the entire database. This is the single most common cause of runaway table bloat in production OLTP systems.

```sql
-- Find transactions blocking VACUUM (horizon blockers)
SELECT pid, usename, state, query_start,
       now() - query_start AS duration,
       left(query, 100) AS query_snippet
FROM pg_stat_activity
WHERE state IN ('idle in transaction', 'active')
  AND (now() - query_start) > interval '5 minutes'
ORDER BY query_start ASC;

-- Show XID age (how close to wraparound)
SELECT age(datfrozenxid) AS xid_age,
       2147483648 - age(datfrozenxid) AS xids_remaining
FROM pg_database WHERE datname = current_database();
```

---

## Trap 5: "work_mem × max_connections gives the total RAM PostgreSQL will use for sorts?"

**What most people say:** Yes, that is the correct formula.

**Correct answer:** The real maximum is dramatically higher. work_mem is allocated per sort/hash node, not per connection. A single complex query with N sort nodes allocates work_mem × N simultaneously. In parallel query mode, each parallel worker gets its own work_mem allocation. A query with 4 parallel workers each performing 2 hash joins could consume 8 × work_mem at once per connection. With 200 connections all running such queries: 200 × 8 × work_mem. The practical formula is: peak_memory = max_connections × max_parallel_workers_per_gather × estimated_nodes_per_query × work_mem. For a 200-connection instance with 64MB work_mem, realistic peak is 10–20GB, not 200 × 64MB = 12.8GB, because not all connections are active simultaneously.

---

## Trap 6: "Is TOAST completely transparent with no performance implications?"

**What most people say:** Yes, TOAST is totally transparent.

**Correct answer:** TOAST is syntactically transparent — you query TOASTed columns identically to regular ones. But the performance implications are significant and non-obvious: fetching a TOASTed value requires an additional random heap access on the TOAST table (pg_toast_<oid>). If a query projects a TOASTed column from millions of rows, PostgreSQL performs millions of additional random I/Os against the TOAST table. This is why SELECT * on wide tables with large text or jsonb columns in hot paths can be extremely slow even when the main table fits in shared_buffers. The fix: SELECT only the columns you need, and consider storing frequently-accessed subfields of large JSON in separate, non-TOASTed columns.

---

## Trap 7: "Does EXPLAIN ANALYZE actually run the query?"

**What most people say:** No, EXPLAIN just estimates without executing.

**Correct answer:** EXPLAIN alone does NOT execute the query. EXPLAIN ANALYZE DOES execute the query completely, including all side effects. Running EXPLAIN ANALYZE on a DELETE or UPDATE will delete/update the actual rows. Always wrap mutating EXPLAIN ANALYZE in a transaction:

```sql
-- Safe pattern for mutating statements
BEGIN;
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
  DELETE FROM events WHERE created_at < now() - interval '1 year';
ROLLBACK;
```

Also: always use EXPLAIN (ANALYZE, BUFFERS) not just EXPLAIN ANALYZE. The BUFFERS option reveals shared_blks_hit (cache hits), shared_blks_read (disk reads), shared_blks_dirtied, and local_blks — which distinguish I/O-bound from CPU-bound slowness. Without BUFFERS, an "actual time=45000ms" node could be slow for completely different reasons that you cannot diagnose.

---

## Trap 8: "The Visibility Map only benefits VACUUM — it has no query-time impact, right?"

**What most people say:** The VM is an internal VACUUM optimization detail.

**Correct answer:** The Visibility Map directly enables index-only scans, which is one of the most impactful query optimizations in PostgreSQL. When the planner chooses an index-only scan, it uses the VM to check whether a heap page is "all-visible" (all tuples on the page are visible to all transactions). If yes, the heap page does not need to be fetched — the index entry is sufficient. If the VM bit is not set, PostgreSQL must access the heap page to perform a visibility check even if the index covers all needed columns. A table with poor VACUUM coverage will have few all-visible pages, causing index-only scans to degrade into regular index scans with full heap access. Run VACUUM after large data loads to set VM bits before running analytical queries.

---

## Trap 9: "Can I use a partial index on an expression, and will the planner always pick it?"

**What most people say:** Yes, partial expression indexes work, and the planner will use them if they are more selective.

**Correct answer:** PostgreSQL supports partial indexes on expressions, but the planner uses a strict equivalence check — it will only use the index if the query's WHERE clause syntactically implies the index predicate. A type mismatch, extra function call, or logically-equivalent-but-not-syntactically-identical predicate will prevent usage. Common traps:

```sql
-- Index created with boolean literal
CREATE INDEX idx_active ON users (lower(email)) WHERE active = true;

-- This WILL use the index:
SELECT id FROM users WHERE lower(email) = 'a@b.com' AND active = true;

-- This will NOT use the index (integer, not boolean):
SELECT id FROM users WHERE lower(email) = 'a@b.com' AND active = 1;

-- This will NOT use the index (IS TRUE not same as = true in planner terms):
SELECT id FROM users WHERE lower(email) = 'a@b.com' AND active IS TRUE;
```

---

## Trap 10: "Adding shared_preload_libraries extensions like pg_stat_statements significantly slows down queries?"

**What most people say:** Yes, any shared_preload_libraries extension adds overhead to every query.

**Correct answer:** pg_stat_statements adds less than 1-2% overhead measured in production benchmarks. The mechanism is a shared-memory hash table keyed by a normalized query fingerprint (literals replaced with $N). After each query completes, a lightweight spinlock is acquired to update timing statistics — this happens once per query completion, not during execution. The normalization fingerprinting is O(query_length). The far greater danger is NOT loading pg_stat_statements: without it, you have zero historical query performance data and cannot diagnose production incidents after the fact. pg_stat_statements should be in shared_preload_libraries on every production PostgreSQL instance, period. The same is true for pg_stat_bgwriter and auto_explain for slow query logging.

---

## Trap 11: "Is the PostgreSQL buffer cache (shared_buffers) the only cache serving reads?"

**What most people say:** Yes, shared_buffers is the cache and effective_cache_size sets its size.

**Correct answer:** This is a two-part misconception. First, PostgreSQL data is cached in BOTH shared_buffers AND the OS page cache simultaneously — the same page can exist in both, consuming double the RAM. Second, effective_cache_size does NOT allocate or configure any memory — it is a pure planning hint that tells the planner how much total cache (shared_buffers + OS page cache) it can assume is available. Setting effective_cache_size = 75% of total RAM (the common recommendation) tells the planner that large sequential scans are likely to find data in cache, making index scans relatively more expensive in comparison. If you set effective_cache_size too low, the planner will prefer sequential scans over index scans even when the data is hot in cache.

---

## Trap 12: "WAL at wal_level=minimal is sufficient for a production primary with streaming replication?"

**What most people say:** Minimal WAL reduces I/O and is fine for performance-focused setups.

**Correct answer:** wal_level=minimal breaks streaming replication entirely. For streaming replication to a standby, you need at minimum wal_level=replica. For logical replication (publications/subscriptions), you need wal_level=logical. The wal_level=minimal setting omits from WAL the information needed to reconstruct row-level changes — it only logs enough to crash-recover the primary itself. The trade-off is real: logical WAL is larger than replica WAL, which is larger than minimal WAL, with corresponding I/O impacts. But in practice, the difference between replica and logical on modern NVMe storage is negligible, and most production systems should use logical to keep the option for logical replication available without a server restart.
