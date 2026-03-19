# Query Optimization — Follow-Up Traps

Tricky follow-up questions revealing gaps in understanding of the query planner and optimization techniques.

---

## Trap 1: "If an index exists on a column, the planner will always use it for queries filtering on that column, right?"

**What most people say:** Yes, indexes exist to be used.

**Correct answer:** The planner uses an index only when it estimates the index scan cost is lower than the sequential scan cost. For a query returning 30%+ of table rows, a sequential scan is often cheaper because sequential I/O is much faster than random I/O (seq_page_cost=1 vs random_page_cost=4 by default). The planner also won't use an index when: a function is applied to the indexed column (WHERE lower(email) = $1 won't use index on email), the data type doesn't match without implicit cast, or statistics are stale causing poor selectivity estimates. The fix for SSDs: set random_page_cost=1.1 (near-random access speed) to make index scans relatively more attractive.

```sql
-- Check if random_page_cost is appropriate for your storage
SHOW random_page_cost;       -- should be 1.1 for NVMe, 2.0 for SSD, 4.0 for HDD

-- Force planner to show its cost estimate
EXPLAIN (ANALYZE, BUFFERS)
  SELECT id FROM users WHERE email = 'alice@example.com';
-- Look for: "Index Scan cost=0.57..8.59 rows=1 width=8"
--       vs: "Seq Scan cost=4502.00..4512.00 rows=100 width=8"
```

---

## Trap 2: "CTEs are always good for readability without any performance downside in modern PostgreSQL."

**What most people say:** CTEs are just syntactic sugar and are optimized away.

**Correct answer:** In PostgreSQL 11 and earlier, CTEs acted as an optimization fence — they were ALWAYS materialized as a separate subquery, preventing the planner from pushing predicates into them or merging them with the outer query. A CTE referenced once with a WHERE clause in the outer query would execute the full CTE first, then filter. In PostgreSQL 12+, CTEs that are referenced once and have no side effects are inlined by default. But MATERIALIZED keyword forces the old fence behavior, and NOT MATERIALIZED forces inlining. The trap: code migrated from PG11 to PG12 may behave differently if CTEs have side effects (e.g., CTEs with INSERT/UPDATE use the fence for correctness, not performance).

```sql
-- PostgreSQL 12+: both forms are explicit
-- Forced materialization (pre-12 behavior):
WITH expensive AS MATERIALIZED (
  SELECT * FROM large_table WHERE expensive_condition
)
SELECT * FROM expensive WHERE id = 5;

-- Forced inlining (allow predicate pushdown):
WITH filtered AS NOT MATERIALIZED (
  SELECT * FROM large_table
)
SELECT * FROM filtered WHERE id = 5;  -- planner can push id=5 into the scan
```

---

## Trap 3: "EXPLAIN ANALYZE shows actual_rows = estimated_rows, so the plan is optimal."

**What most people say:** If estimates match actuals, the plan is good.

**Correct answer:** Row count accuracy is necessary but not sufficient for plan optimality. A plan can have accurate row estimates and still be slow because of: wrong join ordering (still correct estimates but wrong sequence), choice of sort method (external merge vs quicksort), buffer cache state (cold vs warm), network transfer time for wide rows, or suboptimal use of parallelism. The most important EXPLAIN ANALYZE fields to check after row estimates are: Buffers (shared_blks_read vs hit), Sort Method (external merge = spilling to disk), actual time on individual nodes (identify the bottleneck node), and loops (a node running 1M times with 0.01ms each = 10 seconds total, hidden by per-loop timing).

---

## Trap 4: "Hash joins are always better than nested loop joins for large tables."

**What most people say:** Hash joins scale better and should be preferred.

**Correct answer:** Nested loop joins are actually preferred (and faster) when the outer relation is small and the inner join key is indexed. A nested loop with an inner index scan is O(n × log m) where n is the outer result set size. If n is small (1-100 rows), this can dramatically outperform building a hash table over the inner relation. Hash joins require reading all of the inner relation into memory (work_mem) to build the hash table. If work_mem is insufficient, the hash join spills to disk (Batches > 1 in EXPLAIN) and can become dramatically slower than a nested loop. The planner should make this decision correctly given accurate statistics — but a common tuning mistake is disabling nested loops globally when they are genuinely bad for large-table joins, inadvertently hurting the small-lookup case.

---

## Trap 5: "SELECT * is fine because the planner can push down the projection."

**What most people say:** The planner only retrieves what it needs regardless of SELECT *.

**Correct answer:** PostgreSQL does project early in some cases but SELECT * forces fetching all columns from the heap even when an index covering all needed columns exists. With SELECT * you cannot get an index-only scan — the heap must always be accessed for the non-indexed columns. Additionally, SELECT * on tables with wide TEXT or JSONB columns forces TOAST decompression and TOAST table access for every matching row. This can multiply I/O by 5-10x. The practical rule: always project only the columns you need in production queries, especially in high-frequency hot paths.

```sql
-- This CAN use an index-only scan (idx covers id, email):
SELECT id, email FROM users WHERE status = 'active';

-- This CANNOT use index-only scan regardless of indexes:
SELECT * FROM users WHERE status = 'active';
-- Must fetch heap page for every row to get all non-indexed columns
```

---

## Trap 6: "Running ANALYZE after a large data load is optional — autovacuum will handle it."

**What most people say:** Autovacuum runs ANALYZE automatically, no manual intervention needed.

**Correct answer:** Autovacuum does run ANALYZE automatically, but its trigger threshold is: changed rows > autovacuum_analyze_threshold + autovacuum_analyze_scale_factor × n_live_tup (default: 50 + 20% of table). For a 100M-row table, autovacuum ANALYZE won't trigger until 20,000,050 rows change. If you load 5 million new rows (5%), autovacuum won't ANALYZE — but your table's statistics are now based on a sample that represents only 95% of the actual distribution. The planner will use stale histograms for the new data, causing potential misestimates. Always run ANALYZE explicitly after significant bulk loads:

```sql
-- After a bulk load of significant data
ANALYZE orders;  -- updates statistics immediately

-- Or with verbosity to see what changed
ANALYZE VERBOSE orders;
```

---

## Trap 7: "Adding more indexes always improves query performance."

**What most people say:** More indexes = faster reads.

**Correct answer:** Every index has costs: INSERT/UPDATE/DELETE must maintain all indexes on the table, consuming CPU and I/O. Bloated index trees consume buffer cache space, evicting actual data pages. Too many indexes cause the planner to spend more time evaluating index alternatives. For write-heavy tables, indexes can cause more harm than good — each update that modifies an indexed column requires a B-tree modification and potentially a WAL record per index. The test: for every index on a table, check pg_stat_user_indexes.idx_scan — if an index has near-zero scans over 30 days but the table has millions of updates, it is purely costing you write performance. Remove it.

```sql
-- Find unused indexes (candidates for removal)
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND NOT indisprimary
  AND NOT indisunique
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

## Trap 8: "Parallel query is automatically used for slow queries — no configuration needed."

**What most people say:** PostgreSQL automatically parallelizes queries that would benefit.

**Correct answer:** Parallel query requires several conditions to be met simultaneously: the query must not be inside a transaction using serializable isolation, max_parallel_workers_per_gather > 0, the table must be above min_parallel_table_scan_size (default 8MB), and parallel_setup_cost + parallel_tuple_cost must be lower than the non-parallel plan cost. Many queries fail to parallelize because: they are short-running (setup cost exceeds savings), they join on non-parallel-safe functions, or they are called from within a PL/pgSQL function that was not marked PARALLEL SAFE. A common trap: a query runs single-threaded in production because the application uses a transaction wrapper around every query with BEGIN/COMMIT — which doesn't disable parallelism, but some ORM wrappers set session-level options that do.

---

## Trap 9: "If pg_stat_statements shows high total_exec_time for a query, that query is the bottleneck."

**What most people say:** High total_exec_time means that query needs to be optimized.

**Correct answer:** Total execution time is calls × mean_exec_time. A query with 1,000,000 calls at 0.5ms each has 500 seconds total_exec_time. A query with 10 calls at 45,000ms each also has 500 seconds total_exec_time. They need completely different optimizations. The first needs reduced call frequency (caching, N+1 elimination) or microsecond-level micro-optimization. The second needs fundamental query plan work. Always decompose total_exec_time into calls and mean_exec_time first, then look at stddev_exec_time: high stddev indicates inconsistent performance (sometimes fast, sometimes slow) often caused by cache misses, lock waits, or data skew — not a pure algorithmic problem.

```sql
-- Decompose pg_stat_statements for actionable insight
SELECT
  left(query, 80) AS query_snippet,
  calls,
  round(mean_exec_time::numeric, 2) AS mean_ms,
  round(stddev_exec_time::numeric, 2) AS stddev_ms,
  round(total_exec_time::numeric / 1000, 2) AS total_secs,
  shared_blks_read,
  shared_blks_hit,
  round(shared_blks_hit::numeric /
    NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 1) AS hit_pct
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

---

## Trap 10: "Rewriting a subquery as a JOIN is always faster."

**What most people say:** JOINs are faster than subqueries because the planner handles them better.

**Correct answer:** Modern PostgreSQL (version 9.0+) can usually unnest subqueries and treat them equivalently to JOINs. The planner automatically rewrites many subquery patterns into join form. However, correlated subqueries (executed once per outer row) ARE generally slower than JOINs and should be rewritten. The real trap is EXISTS vs IN: IN with a subquery creates a semi-join and can be planned as a hash join; EXISTS also creates a semi-join. But NOT IN has different semantics from NOT EXISTS when NULLs are involved — they are NOT always equivalent rewrites.

```sql
-- These are semantically different when subquery can return NULL:
SELECT * FROM orders WHERE customer_id NOT IN (SELECT id FROM blacklist);
-- If blacklist contains a NULL, this returns ZERO rows (NULL comparison)

SELECT * FROM orders o WHERE NOT EXISTS (
  SELECT 1 FROM blacklist b WHERE b.id = o.customer_id
);
-- This correctly handles NULLs and is typically faster for large sets
```

---

## Trap 11: "Setting enable_seqscan=off in production permanently improves performance."

**What most people say:** If sequential scans are slow, disabling them forces better plans.

**Correct answer:** enable_seqscan=off is a diagnostic tool, not a production setting. When set, it does not actually disable sequential scans — it sets their cost to an astronomically high value. The planner will choose any available index over a sequential scan, even when the sequential scan would genuinely be faster (e.g., fetching 40% of table rows via random I/O). The consequences of leaving this on in production: queries that legitimately need full scans (analytics, VACUUM itself internally) will instead perform extremely slow index scans, potentially degrading performance by orders of magnitude. The correct approach: identify why the planner chose a sequential scan (stale stats? wrong random_page_cost? missing index?), fix the root cause, and never leave method-disable flags in production postgresql.conf.

---

## Trap 12: "Prepared statements always use the same query plan — which might be stale."

**What most people say:** Prepared statements cache one plan forever, which can be bad.

**Correct answer:** PostgreSQL's prepared statement plan caching is more nuanced. For the first 5 executions, PostgreSQL creates a custom plan for each execution using the actual parameter values — it does not cache a generic plan yet. After 5 executions, if the generic plan's estimated cost is within a factor of the average custom plan cost, PostgreSQL switches to a generic plan (one plan for all parameter values). This threshold is controlled by plan_cache_mode (auto, force_generic_plan, force_custom_plan). The generic plan is re-planned if statistics change significantly. In high-skew data scenarios (90% of orders for tenant_id=1, 10% for all others), a generic plan can be catastrophic — it assumes average selectivity and might choose wrong join type. Force custom plans for skewed workloads:

```sql
SET plan_cache_mode = force_custom_plan;  -- session level
-- or in postgresql.conf:
-- plan_cache_mode = force_custom_plan    -- global
```
