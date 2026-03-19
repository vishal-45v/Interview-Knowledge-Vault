# Query Optimization — Structured Answers

Complete interview-ready answers with SQL, EXPLAIN output interpretation, and architectural reasoning.

---

## Q1: Walk through EXPLAIN ANALYZE output interpretation for a slow query.

**Answer:**

Given a slow query, the complete analysis approach is:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT c.name, COUNT(o.id) as order_count, SUM(o.total) as revenue
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE o.created_at >= '2024-01-01'
  AND c.region = 'US'
GROUP BY c.name
ORDER BY revenue DESC
LIMIT 100;
```

**Sample output and interpretation:**
```
Limit  (cost=125000.00..125000.25 rows=100 width=48)
       (actual time=45823.000..45823.025 rows=100 loops=1)
  ->  Sort  (cost=125000.00..125100.00 rows=40000 width=48)
            (actual time=45823.000..45823.010 rows=100 loops=1)
        Sort Key: (sum(o.total)) DESC
        Sort Method: quicksort  Memory: 1024kB   ← GOOD (in-memory)
        Buffers: shared hit=142 read=89023       ← 89K disk reads (BAD)
        ->  HashAggregate  (cost=...) (actual rows=40000 loops=1)
              ->  Hash Join  (cost=...) (actual rows=2800000 loops=1)
                    Hash Cond: (o.customer_id = c.id)
                    Buffers: shared hit=142 read=89023
                    ->  Seq Scan on orders  (cost=0.00..42000.00 rows=3000000)
                                            (actual rows=2800000 loops=1)
                          Filter: (created_at >= '2024-01-01')
                          Rows Removed by Filter: 7200000   ← PROBLEM: 72% wasted
                    ->  Hash on customers   (actual rows=45000 loops=1)
                          Buckets: 65536  Batches: 1  Memory Usage: 3842kB
  Planning Time: 12.345 ms
  Execution Time: 45823.456 ms
```

**Reading the key signals:**

| Signal | Value | Interpretation |
|---|---|---|
| Seq Scan + high "Rows Removed by Filter" | 7.2M removed | Missing index on orders.created_at |
| shared_blks_read=89023 | 89K × 8KB = ~700MB disk I/O | orders not cached, cold data |
| Sort Method: quicksort Memory | Good | Sort fits in work_mem |
| actual rows / estimated rows ratio | Check each node | Misestimate causes wrong plan |

**Fixes for this plan:**

```sql
-- Add partial index for date range queries
CREATE INDEX CONCURRENTLY idx_orders_created_region
ON orders (created_at, customer_id)
WHERE created_at >= '2024-01-01';

-- If orders.created_at is frequently filtered, a regular index:
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders (created_at);

-- After creating index, run ANALYZE to update statistics
ANALYZE orders;
```

**The key diagnostic questions for any EXPLAIN output:**
1. Are row estimates close to actuals? (within 10x is acceptable; 1000x is a problem)
2. Are there sequential scans on large tables with high "Rows Removed by Filter"?
3. Are sort operations spilling to disk? (Sort Method: external merge Disk: XXkb)
4. Are hash joins using multiple batches? (Batches > 1 means hash table spilled to disk)
5. Are there nested loops with high loops counts? (loops=1000000 at 1ms each = 1000 seconds)
6. What is the shared_blks_read vs shared_blks_hit ratio? (high read = cold data, I/O bound)

---

## Q2: Explain all three join algorithms and when to choose each.

**Answer:**

**Nested Loop Join:**
```
For each row in outer_relation:
    For each matching row in inner_relation (via index or full scan):
        emit combined row
```
- Complexity: O(n × m) without index, O(n × log m) with index on inner
- Memory: O(1) — no materialization needed
- Best when: outer is small (< 1000 rows) AND inner has an index on join key
- Planner chooses: when inner can be index-scanned efficiently per outer row

```sql
-- Nested loop is ideal: find orders for a specific customer (small outer)
SELECT o.id, o.total FROM customers c
JOIN orders o ON c.id = o.customer_id  -- index on orders.customer_id used
WHERE c.id = 42;
-- outer: 1 row | inner: indexed lookup → nested loop at its best
```

**Hash Join:**
```
Build phase:  scan inner_relation, build hash table on join key → O(m)
Probe phase:  scan outer_relation, probe hash table → O(n)
```
- Complexity: O(n + m)
- Memory: O(m) — inner relation must fit in work_mem (or spills to disk in batches)
- Best when: both relations are large, no useful sort order exists, no inner index
- Cannot be used for non-equi-joins (e.g., JOIN ON a > b)

```sql
-- Hash join is ideal: large unindexed join
SELECT p.name, SUM(oi.quantity) FROM products p
JOIN order_items oi ON p.id = oi.product_id  -- 50M rows, no useful index
GROUP BY p.name;
-- Build hash table of 50M order_items, probe with products
```

**Merge Join:**
```
Sort both relations on join key (if not already sorted)
Merge-scan both sorted sequences simultaneously → O((n+m) log(n+m)) with sort
                                                  O(n+m) if already sorted
```
- Complexity: O(n + m) if pre-sorted (e.g., index scan produces sorted order)
- Memory: sort buffer (work_mem) if sort is needed
- Best when: both relations are pre-sorted on the join key (via index scans) OR result is needed sorted
- Ideal for: range join conditions, large sorted results, avoiding hash table spill

```sql
-- Merge join is ideal: both tables indexed on join key, sorted output needed
SELECT c.name, o.id FROM customers c
JOIN orders o ON c.id = o.customer_id
ORDER BY c.id;  -- both can be index-scanned in order → merge join with no sort cost
```

**Choosing which join to use manually (diagnostic only):**
```sql
-- Force a specific join type for testing
SET enable_nestloop = off;
SET enable_hashjoin = off;
-- Only merge join remains; compare EXPLAIN costs
EXPLAIN SELECT ...;
SET enable_nestloop = on;
SET enable_hashjoin = on;
```

---

## Q3: How do you diagnose and fix N+1 query problems using pg_stat_statements?

**Answer:**

**Identifying N+1 in production:**
```sql
-- Find queries called proportionally to other queries (N+1 pattern)
WITH query_stats AS (
  SELECT
    queryid,
    left(query, 100) AS query_snippet,
    calls,
    round(mean_exec_time::numeric, 3) AS mean_ms,
    round(total_exec_time::numeric / 1000, 1) AS total_secs
  FROM pg_stat_statements
  WHERE calls > 1000
  ORDER BY calls DESC
)
SELECT * FROM query_stats
WHERE query_snippet LIKE '%WHERE id = %'      -- single-row lookups
   OR query_snippet LIKE '%WHERE %_id = $1%'  -- FK lookups
LIMIT 20;
```

**Classic N+1 pattern in ORM (ActiveRecord example):**
```ruby
# N+1: generates 1 + N queries
customers = Customer.where(region: 'US').limit(100)
customers.each { |c| c.orders.count }  # 1 query per customer!
```

**The SQL it generates:**
```sql
-- Query 1 (1 time):
SELECT * FROM customers WHERE region = 'US' LIMIT 100;

-- Query 2..101 (100 times, once per customer):
SELECT * FROM orders WHERE customer_id = 1;
SELECT * FROM orders WHERE customer_id = 2;
-- ... 98 more identical queries with different ids
```

**The correct JOIN-based rewrite:**
```sql
-- Single query returning all needed data
SELECT
  c.id,
  c.name,
  COUNT(o.id) AS order_count,
  MAX(o.created_at) AS last_order_date
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
WHERE c.region = 'US'
GROUP BY c.id, c.name
LIMIT 100;
```

**In Rails: eager loading (includes/joins):**
```ruby
# Fix: eager load with a single JOIN query
customers = Customer.where(region: 'US')
                    .includes(:orders)
                    .limit(100)
```

**Real impact calculation:**
- 100 customers × 1ms per order query = 100ms database time
- But also: 100 network round trips × 1ms network = 100ms extra latency
- With connection pool exhaustion: if all 100 queries need connections simultaneously, and pool_size=20, 80 queries wait → cascading latency amplification

---

## Q4: Describe how PostgreSQL collects and uses statistics for query planning.

**Answer:**

PostgreSQL's planner statistics are stored in pg_statistic (raw) and pg_stats (readable view).

**Statistics collected per column:**
```sql
-- View statistics for a specific column
SELECT
  attname,
  null_frac,         -- fraction of NULLs
  avg_width,         -- average byte width
  n_distinct,        -- estimated distinct values (negative = fraction of rows)
  most_common_vals,  -- MCV list (up to statistics_target values)
  most_common_freqs, -- frequency of each MCV
  histogram_bounds,  -- bounds for equal-frequency histogram buckets
  correlation        -- physical/logical ordering correlation (-1 to 1)
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

**How each statistic affects estimation:**

1. **n_distinct**: If n_distinct = -0.01, about 1% of rows are distinct (high cardinality). The planner estimates GROUP BY selectivity and join fanout using this.

2. **MCV list**: For a status column with values ['pending', 'shipped', 'cancelled'], the MCV list stores the frequencies (e.g., [0.70, 0.20, 0.10]). A query WHERE status = 'pending' gets 70% selectivity from MCV — very accurate.

3. **Histogram**: For continuous values (created_at, price), the histogram stores bucket boundaries. Values between buckets get linear interpolation — which fails for skewed distributions.

4. **Correlation**: If correlation = 1.0, rows are perfectly ordered by this column on disk. The planner can estimate that an index range scan will access sequential pages (cheap). If correlation = 0 (random), each index entry requires a separate heap page read (expensive).

```sql
-- Increase statistics target for a skewed column
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
-- default is 100, meaning up to 100 MCV entries and 100 histogram buckets
-- For 50,000 customers with highly skewed order distribution, 100 is too few

ANALYZE orders;  -- rebuild statistics with new target
```

**Extended statistics for correlated columns:**
```sql
-- Column statistics are independent by default
-- If WHERE region = 'US' AND product_type = 'software' are correlated
-- (US always has software), independent estimates multiply incorrectly

CREATE STATISTICS orders_region_type (dependencies)
ON region, product_type FROM orders;

ANALYZE orders;
-- Now planner knows these columns are functionally dependent
```

---

## Q5: How do you use pg_stat_statements to prioritize optimization work?

**Answer:**

```sql
-- Complete optimization triage query
SELECT
  queryid,
  calls,
  round(mean_exec_time::numeric, 2)    AS mean_ms,
  round(stddev_exec_time::numeric, 2)  AS stddev_ms,
  round(total_exec_time::numeric/1000/60, 2) AS total_mins,
  round(rows / NULLIF(calls, 0)::numeric, 1)  AS avg_rows_returned,
  shared_blks_hit + shared_blks_read   AS total_blks_accessed,
  round(shared_blks_hit::numeric /
    NULLIF(shared_blks_hit + shared_blks_read, 0) * 100, 1) AS cache_hit_pct,
  shared_blks_read                     AS disk_reads,
  left(query, 120)                     AS query
FROM pg_stat_statements
WHERE calls > 100
  AND mean_exec_time > 1  -- ignore sub-millisecond queries
ORDER BY total_exec_time DESC
LIMIT 30;
```

**Interpreting the results to prioritize work:**

| Pattern | Interpretation | Fix |
|---|---|---|
| High calls + low mean_ms + low cache_hit_pct | High-frequency index scan hitting cold data | Warm cache, better connection pooling, caching layer |
| Low calls + high mean_ms | Expensive analytical query | Query rewrite, better plan, partitioning |
| High calls + high mean_ms + stddev ≈ mean | Consistently slow; not a cache issue | Index, rewrite, materialized view |
| High calls + low mean_ms + high stddev | Usually fast but sometimes slow | Lock waits, skewed data, cache eviction |
| Large avg_rows_returned + simple query | Returning too much data | Add LIMIT, paginate, narrow projection |

**Finding queries that improved/regressed after a deployment:**
```sql
-- Save a snapshot before deployment:
CREATE TABLE pss_baseline AS SELECT * FROM pg_stat_statements;

-- After deployment, compare mean_exec_time changes:
SELECT
  s.queryid,
  b.mean_exec_time AS before_ms,
  s.mean_exec_time AS after_ms,
  round(s.mean_exec_time - b.mean_exec_time, 2) AS delta_ms,
  left(s.query, 100) AS query
FROM pg_stat_statements s
JOIN pss_baseline b USING (queryid)
WHERE abs(s.mean_exec_time - b.mean_exec_time) > 10  -- >10ms regression
ORDER BY delta_ms DESC;
```

---

## Q6: Explain partition pruning and when it fails.

**Answer:**

Partition pruning is the planner's ability to skip scanning partitions that cannot contain rows matching the query's WHERE clause.

```sql
-- Create range-partitioned table
CREATE TABLE events (
  id BIGSERIAL,
  occurred_at TIMESTAMPTZ NOT NULL,
  event_type TEXT,
  payload JSONB
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2024_q1 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
  FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');
CREATE TABLE events_2024_q3 PARTITION OF events
  FOR VALUES FROM ('2024-07-01') TO ('2024-10-01');
CREATE TABLE events_2024_q4 PARTITION OF events
  FOR VALUES FROM ('2024-10-01') TO ('2025-01-01');

-- Query with partition pruning working:
EXPLAIN SELECT * FROM events
WHERE occurred_at BETWEEN '2024-04-01' AND '2024-06-30';
-- Shows: Append → Seq Scan on events_2024_q2 ONLY
-- Partitions 2024_q1, q3, q4 are PRUNED

-- Query that DEFEATS partition pruning:
EXPLAIN SELECT * FROM events
WHERE date_trunc('quarter', occurred_at) = '2024-04-01';
-- Shows: Append → Seq Scan on ALL 4 partitions
-- Function on partition key prevents pruning!
```

**When partition pruning fails:**
1. Function applied to partition key: `WHERE date_trunc('month', occurred_at) = ...`
2. Implicit type cast: `WHERE occurred_at = '2024-06-15'` with mismatched types
3. OR conditions across partition boundaries that cannot be reduced
4. Non-immutable functions: `WHERE occurred_at > NOW() - interval '30 days'` (runtime pruning needed)

**Runtime pruning for parameterized queries:**
```sql
-- With a parameterized query, partition pruning happens at execution time
PREPARE find_events(TIMESTAMPTZ, TIMESTAMPTZ) AS
  SELECT * FROM events
  WHERE occurred_at BETWEEN $1 AND $2;

EXECUTE find_events('2024-04-01', '2024-06-30');
-- enable_partition_pruning = on allows runtime pruning
-- Planner creates a plan that checks partition bounds at execution
```

---

## Q7: What is the correct approach to connection pooling architecture for a high-traffic application?

**Answer:**

**Architecture overview:**
```
App instances (100+)
       │
       ▼ (thousands of connections)
  PgBouncer (transaction mode)
       │
       ▼ (20-50 connections)
  PostgreSQL primary
```

**PgBouncer configuration for OLTP:**
```ini
[pgbouncer]
pool_mode = transaction        # Return connection after each transaction
max_client_conn = 5000         # Max application connections
default_pool_size = 30         # Actual PostgreSQL backend connections per DB/user pair
min_pool_size = 5              # Keep 5 connections warm
reserve_pool_size = 5          # Emergency pool for spikes
reserve_pool_timeout = 3       # Wait 3s before using reserve pool
server_idle_timeout = 600      # Close idle backends after 10 min
server_lifetime = 3600         # Recycle connections every 1 hour
client_idle_timeout = 300      # Close idle client connections after 5 min
query_wait_timeout = 120       # Fail fast if no connection after 2 min

# Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
```

**Pool size calculation:**
```
Optimal pool_size = (cpu_cores × 2) + effective_spindle_count
For 16-core server with NVMe: 16 × 2 + 1 = 33 connections

Rule of thumb: Start at 25-50, measure pg_stat_activity active/idle ratio
If most connections are idle: pool_size too large, wasting memory
If applications waiting for connections: pool_size too small
```

**Limitations of transaction mode:**
```sql
-- These DO NOT WORK in PgBouncer transaction mode:
SET LOCAL client_encoding = 'UTF8';   -- session state reset after transaction
LISTEN channel_name;                   -- requires persistent connection
DECLARE cursor_name CURSOR FOR ...;   -- cursor tied to session
SELECT pg_advisory_lock(123);          -- advisory lock released when connection returned

-- Fix: use application-level alternatives or session mode for these features
-- Or: use PgBouncer's ignore_startup_parameters for SET commands
```
