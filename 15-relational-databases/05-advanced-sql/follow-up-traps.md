# Chapter 05 — Advanced SQL: Follow-Up Traps

Tricky follow-up questions that reveal whether a candidate truly understands
advanced SQL semantics, window function frames, and PostgreSQL-specific behaviors.

---

## Trap 1: "A CTE is just a named subquery — it's for readability only and has
no performance implications."

**What most people say:** Yes, CTEs are just syntactic sugar for subqueries.

**Correct answer:** In PostgreSQL 11 and earlier, CTEs acted as an optimization
fence — the planner could not inline them and must materialize them (execute once
and cache the result). This prevented the planner from pushing down predicates
into the CTE, sometimes causing worse performance than equivalent subqueries.

In PostgreSQL 12+, CTEs are inlined by default unless they are recursive,
contain side effects (INSERT/UPDATE/DELETE), or are referenced more than once
(in which case materialization is beneficial to avoid re-execution).

```sql
-- PostgreSQL 11: CTE is always materialized, predicate NOT pushed down
WITH recent AS (
    SELECT * FROM orders WHERE order_date > '2024-01-01'  -- full scan here
)
SELECT * FROM recent WHERE customer_id = 42;  -- filter after materialization

-- PostgreSQL 12+: CTE inlined unless flagged
-- Equivalent to: SELECT * FROM orders WHERE order_date > '2024-01-01' AND customer_id = 42
-- Index on (customer_id, order_date) can now be used

-- Force materialization when you WANT the optimization fence (e.g., complex recursive):
WITH MATERIALIZED expensive AS (...)
SELECT * FROM expensive;

-- Force inlining (PostgreSQL 12+ default behavior):
WITH NOT MATERIALIZED cheap AS (...)
SELECT * FROM cheap;
```

---

## Trap 2: "Window functions are evaluated after WHERE, so you can filter
on a window function result in the WHERE clause."

**What most people say:** Yes, I can add WHERE rank = 1 to filter window results.

**Correct answer:** No. Window functions are evaluated after WHERE, GROUP BY,
and HAVING but before ORDER BY. You CANNOT filter on a window function result
in a WHERE or HAVING clause. You must wrap the query in a subquery or CTE.

```sql
-- FAILS: rank() is not yet computed when WHERE runs
SELECT *, RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk
FROM products
WHERE rnk = 1;  -- ERROR: column "rnk" does not exist

-- CORRECT: wrap in subquery or CTE
WITH ranked AS (
    SELECT *, RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk
    FROM products
)
SELECT * FROM ranked WHERE rnk = 1;

-- OR as subquery:
SELECT * FROM (
    SELECT *, RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk
    FROM products
) ranked
WHERE rnk = 1;
```

---

## Trap 3: "ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING gives the
reverse running total — the sum from the current row to the end."

**What most people say:** Yes, that's the reverse running total.

**Correct answer:** Correct, but the tricky part is that FIRST_VALUE and
LAST_VALUE have non-obvious default frame behavior. LAST_VALUE with the default
frame gives the last value within the current window frame, which by default
is `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — meaning LAST_VALUE
returns the current row's value, not the last row in the partition!

```sql
-- TRAP: LAST_VALUE appears to not work
SELECT
    name,
    salary,
    LAST_VALUE(name) OVER (PARTITION BY dept ORDER BY salary) AS last_name
FROM employees;
-- Returns the current employee's name, not the highest-paid employee's name!
-- Because default frame ends at CURRENT ROW

-- FIX: extend frame to UNBOUNDED FOLLOWING
SELECT
    name,
    salary,
    LAST_VALUE(name) OVER (
        PARTITION BY dept
        ORDER BY salary
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_name
FROM employees;
-- Now correctly returns the name of the highest-paid employee in the department
```

FIRST_VALUE does not have this problem because the default frame starts at
UNBOUNDED PRECEDING, which already includes the first row.

---

## Trap 4: "RANGE BETWEEN 1 PRECEDING AND 1 FOLLOWING gives a 3-row sliding window."

**What most people say:** Yes, it includes the previous row, current row, and next row.

**Correct answer:** RANGE uses the values in the ORDER BY column, not row counts.
"1 PRECEDING" means "rows where the ORDER BY column value is within 1 unit less
than the current row's value." If multiple rows have the same ORDER BY value,
ALL of them fall within the range, giving potentially many more than 3 rows.

```sql
SELECT
    sale_date,
    amount,
    SUM(amount) OVER (
        ORDER BY sale_date
        RANGE BETWEEN INTERVAL '1 day' PRECEDING AND CURRENT ROW
    ) AS rolling_2day_sum
FROM daily_sales;
-- If three sales occurred on the same date: all three are in "CURRENT ROW" range
-- ROWS BETWEEN 2 PRECEDING AND CURRENT ROW gives exactly 3 rows regardless of values

-- Key distinction:
-- ROWS: count-based (N physical rows before/after current)
-- RANGE: value-based (rows within N units of current row's ORDER BY value)
```

---

## Trap 5: "A recursive CTE must always use UNION to prevent duplicates."

**What most people say:** Use UNION not UNION ALL in recursive CTEs to prevent
infinite loops.

**Correct answer:** UNION de-duplicates, but it does NOT prevent infinite loops
in all cases (only when cycles cause exact duplicate rows). UNION ALL is usually
the correct choice for recursive CTEs traversing graphs because:
1. UNION requires sorting/hashing every recursion level to de-duplicate, which
   is expensive.
2. If cycles exist, UNION may still loop (it de-dupes rows but if data changes
   slightly through traversal, same logical node can appear as different rows).
3. The correct protection against cycles is an explicit cycle detection mechanism:

```sql
-- UNION does NOT reliably prevent cycles in all cases
-- CORRECT approach: track visited nodes
WITH RECURSIVE hierarchy AS (
    -- Anchor: start from root
    SELECT id, name, parent_id, ARRAY[id] AS path, 0 AS depth
    FROM employees WHERE parent_id IS NULL

    UNION ALL

    -- Recursive: add children
    SELECT e.id, e.name, e.parent_id,
           path || e.id,  -- track path of visited IDs
           depth + 1
    FROM employees e
    JOIN hierarchy h ON e.parent_id = h.id
    WHERE NOT (e.id = ANY(path))  -- CYCLE PREVENTION: skip already-visited nodes
      AND depth < 20              -- DEPTH LIMIT: safety net
)
SELECT * FROM hierarchy;
```

PostgreSQL 14+ has native cycle detection syntax:
```sql
WITH RECURSIVE hierarchy AS (...)
CYCLE id SET is_cycle USING path
SELECT * FROM hierarchy WHERE NOT is_cycle;
```

---

## Trap 6: "LATERAL is just another name for a correlated subquery — they're the same."

**What most people say:** Yes, LATERAL is syntactic sugar for correlated subqueries.

**Correct answer:** LATERAL joins are more powerful than correlated subqueries:
1. A LATERAL subquery can return multiple rows (a correlated subquery in SELECT
   must return exactly one row or one value).
2. LATERAL can be used with JOIN, allowing all JOIN types (LEFT, INNER, etc.)
   with the full result set of the subquery.
3. LATERAL can reference multiple columns from the outer query at once.

```sql
-- Correlated subquery in SELECT: must return exactly 1 value
SELECT o.id, (SELECT MAX(amount) FROM order_items WHERE order_id = o.id) AS max_item
FROM orders o;

-- LATERAL: can return multiple rows and columns
SELECT o.id, top_items.*
FROM orders o
LEFT JOIN LATERAL (
    SELECT product_id, amount
    FROM order_items
    WHERE order_id = o.id
    ORDER BY amount DESC
    LIMIT 3
) top_items ON true;
-- Returns up to 3 rows per order — impossible with a scalar correlated subquery

-- Key LATERAL use case: "top N per group" without window functions
SELECT u.id, last_orders.*
FROM users u
LEFT JOIN LATERAL (
    SELECT id AS order_id, amount, order_date
    FROM orders
    WHERE user_id = u.id
    ORDER BY order_date DESC
    LIMIT 5
) last_orders ON true;
```

---

## Trap 7: "REFRESH MATERIALIZED VIEW is a good solution for near-real-time
data since it runs quickly."

**What most people say:** Yes, just set up a cron to refresh it frequently.

**Correct answer:** REFRESH MATERIALIZED VIEW acquires an exclusive lock on the
materialized view during the refresh, blocking all SELECT queries against it.
For a materialized view that takes 10 seconds to refresh and is refreshed every
minute, reads are blocked for ~17% of the time. This is often unacceptable.

The solution is `REFRESH MATERIALIZED VIEW CONCURRENTLY`, which:
1. Builds the new result in a temporary table.
2. Diffs the old and new results.
3. Applies only the changes with a minimal lock window.

But CONCURRENTLY requires a UNIQUE index on the materialized view, takes longer
total time, and uses more resources.

```sql
-- Blocking refresh (default):
REFRESH MATERIALIZED VIEW mv_daily_sales;
-- All SELECTs on mv_daily_sales block until this completes

-- Non-blocking (requires unique index):
CREATE UNIQUE INDEX ON mv_daily_sales (date, region_id);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales;
-- Reads continue during refresh; brief lock only at final swap

-- Automation: use pg_cron extension
CREATE EXTENSION pg_cron;
SELECT cron.schedule('refresh_mv', '0 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_sales');
```

---

## Trap 8: "INSERT ON CONFLICT DO UPDATE updates the row with the new values."

**What most people say:** Yes, ON CONFLICT UPDATE replaces the row with the
new INSERT values.

**Correct answer:** ON CONFLICT DO UPDATE gives you full control — you specify
exactly which columns to update and what values to use. The EXCLUDED pseudo-table
refers to the row that was attempted to be inserted (the new values). You can
use any expression, not just the new values — you can keep the old value, take
the max, concatenate, etc.

```sql
-- Full replacement (common but not always correct):
INSERT INTO user_stats (user_id, event_count)
VALUES (42, 5)
ON CONFLICT (user_id)
DO UPDATE SET event_count = EXCLUDED.event_count;  -- replaces with 5

-- Increment (accumulate):
INSERT INTO user_stats (user_id, event_count)
VALUES (42, 5)
ON CONFLICT (user_id)
DO UPDATE SET event_count = user_stats.event_count + EXCLUDED.event_count;
-- If existing count is 100, result is 105

-- Conditional update (only if new value is higher):
INSERT INTO high_scores (user_id, score)
VALUES (42, 1500)
ON CONFLICT (user_id)
DO UPDATE SET score = GREATEST(high_scores.score, EXCLUDED.score);
-- Keeps the higher of old and new score

-- Do nothing (ignore duplicates):
INSERT INTO events (id, data) VALUES (100, '{}')
ON CONFLICT (id) DO NOTHING;
```

---

## Trap 9: "NTILE(4) divides rows into 4 exactly equal buckets."

**What most people say:** Yes, NTILE(4) creates quartiles with equal row counts.

**Correct answer:** NTILE distributes rows as evenly as possible, but if the
total row count is not divisible by N, some buckets will have one more row than
others. The extra rows go to the earlier buckets.

```sql
-- 10 rows, NTILE(4):
-- Bucket 1: 3 rows (10/4 = 2.5, rounded up for first 2 buckets)
-- Bucket 2: 3 rows
-- Bucket 3: 2 rows
-- Bucket 4: 2 rows
-- Total: 10 rows ✓

SELECT
    user_id,
    revenue,
    NTILE(4) OVER (ORDER BY revenue DESC) AS quartile
FROM user_revenue;
-- quartile=1 is the top 25% (highest revenue), NOT the bottom 25%
-- The ORDER BY in OVER determines which end is bucket 1

-- Common mistake: assuming equal bucket sizes when planning threshold analysis
-- Use PERCENT_RANK() or CUME_DIST() if you need exact percentile positions
SELECT
    user_id,
    revenue,
    PERCENT_RANK() OVER (ORDER BY revenue) AS percentile  -- 0.0 to 1.0
FROM user_revenue;
```

---

## Trap 10: "You can always use a regular view instead of a materialized view,
since views are just stored queries."

**What most people say:** Yes, views and materialized views are interchangeable
except that materialized views cache results.

**Correct answer:** There are significant cases where regular views cause serious
performance problems that materialized views solve. More importantly, there are
also cases where materialized views cannot be used where views can:

1. Regular views can always reflect current data. Materialized views can only
   reflect data as of the last refresh — stale by design.
2. Regular views can be updatable (INSERT/UPDATE/DELETE through the view) if
   they meet certain criteria. Materialized views are never updatable.
3. Regular views participate in query planning — the planner can merge the view
   definition with the outer query and optimize holistically. Materialized views
   are always treated as physical tables, even if accessing them cold is slower.

```sql
-- Regular view: transparent to planner
CREATE VIEW active_orders AS SELECT * FROM orders WHERE status != 'cancelled';
-- Query: SELECT * FROM active_orders WHERE customer_id = 42;
-- Planner sees: SELECT * FROM orders WHERE status != 'cancelled' AND customer_id = 42
-- Can use index on (customer_id, status) ← planner can push predicates in

-- Materialized view: opaque to planner for updates
CREATE MATERIALIZED VIEW mv_active_orders AS SELECT * FROM orders WHERE status != 'cancelled';
-- Query: SELECT * FROM mv_active_orders WHERE customer_id = 42;
-- Planner must scan mv_active_orders, which has its own separate indexes
-- Cannot benefit from predicate pushdown into the source table
```

Use materialized views for: expensive aggregations, cross-database joins,
complex transformations that run slowly on raw data. Use regular views for:
permission facades, query simplification, business rule encapsulation.
