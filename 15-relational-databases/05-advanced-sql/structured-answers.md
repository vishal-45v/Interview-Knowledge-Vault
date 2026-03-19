# Chapter 05 — Advanced SQL: Structured Answers

Complete, interview-ready answers for the most important advanced SQL questions
at the senior engineer level.

---

## Q1: Explain window functions — what they do, key clauses, and the frame
specification in detail.

**Answer:**

A window function computes a value for each row based on a set of related rows
(the window), without collapsing the rows into a single output row the way
GROUP BY does. Each row in the result keeps its own identity.

**Basic syntax:**
```sql
function_name() OVER (
    [PARTITION BY column(s)]
    [ORDER BY column(s)]
    [frame_clause]
)
```

**PARTITION BY:** Divides the result set into independent partitions. The window
function is computed separately within each partition. Without PARTITION BY,
all rows form one partition.

```sql
-- Running total per customer (not across all customers)
SELECT
    customer_id,
    order_date,
    amount,
    SUM(amount) OVER (PARTITION BY customer_id ORDER BY order_date) AS running_total
FROM orders;
```

**ORDER BY in OVER:** Defines the order within the partition. Also activates the
default frame: `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

**Frame clause:** Defines which rows in the partition contribute to the computation
for each row.

```sql
-- Different frame specifications:
SUM(amount) OVER (PARTITION BY cust ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)  -- running total
    -- includes all rows from partition start to current row (by position)

SUM(amount) OVER (PARTITION BY cust ORDER BY date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)  -- 7-row rolling sum

SUM(amount) OVER (PARTITION BY cust ORDER BY date
    RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW)  -- 7-day rolling sum
    -- uses value-based range: all rows within 7 days of current row's date

SUM(amount) OVER (PARTITION BY cust
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)  -- total per partition
    -- same as SUM(amount) OVER (PARTITION BY cust)
```

**ROWS vs RANGE:**
- ROWS: physical rows counted (N rows before/after)
- RANGE: value range (all rows where ORDER BY column is within N units)
- For dates: RANGE with INTERVAL is the natural choice for time-series windows
- For arbitrary rankings: ROWS gives predictable row counts

---

## Q2: Write a comprehensive window function query solving a real analytical problem.

**Answer:**

Business requirement: For each product, for each month, show: monthly revenue,
running total YTD revenue, previous month's revenue, month-over-month growth %,
and rank within the product category for that month.

```sql
WITH monthly_data AS (
    SELECT
        p.category,
        p.name AS product_name,
        DATE_TRUNC('month', o.order_date) AS month,
        SUM(oi.quantity * oi.unit_price) AS monthly_revenue
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.id
    JOIN products p ON oi.product_id = p.id
    WHERE o.order_date >= '2024-01-01'
    GROUP BY p.category, p.name, DATE_TRUNC('month', o.order_date)
)
SELECT
    category,
    product_name,
    month,
    monthly_revenue,

    -- Running total within the year for this product
    SUM(monthly_revenue) OVER (
        PARTITION BY category, product_name
        ORDER BY month
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS ytd_revenue,

    -- Previous month's revenue for this product
    LAG(monthly_revenue, 1, 0) OVER (
        PARTITION BY category, product_name
        ORDER BY month
    ) AS prev_month_revenue,

    -- Month-over-month growth percentage
    ROUND(
        (monthly_revenue - LAG(monthly_revenue, 1) OVER (
            PARTITION BY category, product_name ORDER BY month
        )) / NULLIF(LAG(monthly_revenue, 1) OVER (
            PARTITION BY category, product_name ORDER BY month
        ), 0) * 100, 2
    ) AS mom_growth_pct,

    -- Rank within category for this month (highest revenue = rank 1)
    RANK() OVER (
        PARTITION BY category, month
        ORDER BY monthly_revenue DESC
    ) AS category_rank_this_month

FROM monthly_data
ORDER BY category, product_name, month;
```

**Interview-ready explanation:** This uses three separate window definitions.
The running total uses an explicit ROWS frame. LAG accesses the previous row
in the same partition's ordering. RANK uses a different partition (category +
month) from the running total (category + product). All three window functions
operate on the same underlying rows from the CTE — the CTE is materialized once
and all window computations reference it.

---

## Q3: Explain recursive CTEs with a complete tree traversal example.

**Answer:**

A recursive CTE has two parts joined by UNION ALL:
1. **Anchor member:** Initial result set (base case), executed once.
2. **Recursive member:** References the CTE itself, executed repeatedly until
   no new rows are produced.

```sql
-- Organization hierarchy traversal
-- employees(id, name, manager_id, title)

WITH RECURSIVE org_tree AS (
    -- Anchor: start from the CEO (no manager)
    SELECT
        id,
        name,
        title,
        manager_id,
        0 AS depth,
        ARRAY[id] AS path,           -- track visited IDs for cycle detection
        name AS breadcrumb           -- build path string
    FROM employees
    WHERE manager_id IS NULL  -- root node(s)

    UNION ALL

    -- Recursive: join each manager's row to their direct reports
    SELECT
        e.id,
        e.name,
        e.title,
        e.manager_id,
        ot.depth + 1,
        ot.path || e.id,             -- append current ID to path
        ot.breadcrumb || ' > ' || e.name
    FROM employees e
    INNER JOIN org_tree ot ON e.manager_id = ot.id
    WHERE NOT (e.id = ANY(ot.path))  -- cycle guard: skip if already visited
      AND ot.depth < 20              -- depth limit: safety net for corrupt data
)
SELECT
    REPEAT('  ', depth) || name AS indented_name,  -- visual indentation
    title,
    depth,
    breadcrumb,
    array_length(path, 1) AS hierarchy_level
FROM org_tree
ORDER BY path;  -- order by path array ensures correct tree order
```

**Key concepts to emphasize:**
- `UNION ALL` is almost always used in recursive CTEs (not UNION). Using UNION
  adds deduplication overhead at each step.
- The recursive member's join condition defines the traversal relationship.
- `ARRAY[id]` path tracking handles cycles. PostgreSQL 14+ has native `CYCLE` syntax.
- Execution: anchor produces N rows → recursive member joins each to its children →
  new rows added back to the working table → repeat until no new rows → final result.

**For graphs (not just trees):** Replace the self-join with graph edge traversal
and add cycle detection for non-tree structures.

---

## Q4: How does PostgreSQL JSONB work, and how do you index and query it efficiently?

**Answer:**

**JSON vs JSONB:**
- `JSON`: stored as-is, preserves whitespace, duplicate keys, key order. Parsing
  on every read. Faster writes, slower reads.
- `JSONB`: parsed, stored in decomposed binary format. Deduplicates keys (last wins),
  does not preserve key order or whitespace. Slower writes, faster reads. Supports GIN indexing.

**Common operators:**
```sql
-- Sample row: data = '{"type": "click", "user": {"id": 42, "country": "US"}, "tags": ["a","b"]}'

-- Operator: -> (returns JSON/JSONB, not text)
data -> 'type'           -- returns: "click" (JSONB)
data -> 'user' -> 'id'   -- returns: 42 (JSONB)

-- Operator: ->> (returns text)
data ->> 'type'           -- returns: click (TEXT)
data -> 'user' ->> 'id'   -- returns: 42 (TEXT)

-- Operator: #> (path with array notation, returns JSONB)
data #> '{user,id}'       -- returns: 42 (JSONB)

-- Operator: #>> (path, returns text)
data #>> '{user,country}' -- returns: US (TEXT)

-- Containment: @> (does the left JSON contain the right JSON?)
data @> '{"type": "click"}'        -- TRUE
data @> '{"user": {"id": 42}}'     -- TRUE (nested containment)

-- Key existence: ?
data ? 'type'                       -- TRUE (key 'type' exists)
data ? 'missing_key'                -- FALSE

-- Array containment:
data->'tags' @> '["a"]'             -- TRUE
```

**Indexing strategies:**
```sql
-- GIN index: enables @>, ?, ?|, ?& operators (containment and existence)
CREATE INDEX idx_events_gin ON events USING gin (data);
-- Covers: WHERE data @> '{"type": "click"}'
-- Covers: WHERE data ? 'user_id'
-- Does NOT cover: WHERE data->>'type' = 'click' (path extraction)

-- Expression index: for specific path extractions
CREATE INDEX idx_events_type ON events ((data->>'type'));
-- Now: WHERE data->>'type' = 'click' uses this index

-- Expression index with cast: for numeric comparisons
CREATE INDEX idx_events_amount ON events (((data->>'amount')::numeric));
-- Now: WHERE (data->>'amount')::numeric > 100 uses this index

-- GIN with jsonb_path_ops: smaller index, supports @> only (not ?)
CREATE INDEX idx_events_path ON events USING gin (data jsonb_path_ops);
```

**Updating JSONB without overwriting:**
```sql
-- Merge/update one field using ||  (concatenation operator, right key wins)
UPDATE events SET data = data || '{"status": "processed"}' WHERE id = 42;
-- Adds/updates "status" field without touching other fields

-- Remove a key:
UPDATE events SET data = data - 'temp_field' WHERE id = 42;

-- Update nested field:
UPDATE events
SET data = jsonb_set(data, '{user, country}', '"GB"')
WHERE id = 42;
```

---

## Q5: Explain GROUPING SETS, ROLLUP, and CUBE with a practical multi-dimensional
reporting example.

**Answer:**

These extensions to GROUP BY allow multiple grouping combinations in a single
query, replacing what previously required multiple queries joined with UNION ALL.

**GROUPING SETS:** Explicit list of grouping combinations.
```sql
-- Without GROUPING SETS (three separate queries + UNION):
SELECT region, product, SUM(revenue) FROM sales GROUP BY region, product
UNION ALL
SELECT region, NULL, SUM(revenue) FROM sales GROUP BY region
UNION ALL
SELECT NULL, NULL, SUM(revenue) FROM sales;

-- With GROUPING SETS (equivalent, single scan):
SELECT region, product, SUM(revenue) AS total_revenue
FROM sales
GROUP BY GROUPING SETS (
    (region, product),   -- subtotal per region-product combination
    (region),            -- subtotal per region (across all products)
    ()                   -- grand total (across everything)
);
```

**ROLLUP:** Hierarchical rollup. `ROLLUP(a, b, c)` = `GROUPING SETS((a,b,c),(a,b),(a),())`.
Generates N+1 grouping sets from N columns — each level rolls up by removing
the last column.

```sql
SELECT year, quarter, month, SUM(revenue)
FROM sales
GROUP BY ROLLUP (year, quarter, month)
ORDER BY year NULLS LAST, quarter NULLS LAST, month NULLS LAST;
-- Produces: subtotals for (year,quarter,month), (year,quarter), (year), and grand total
```

**CUBE:** All possible combinations. `CUBE(a, b)` = `GROUPING SETS((a,b),(a),(b),())`.
For N columns: 2^N grouping sets.

```sql
SELECT region, product, channel, SUM(revenue)
FROM sales
GROUP BY CUBE (region, product, channel);
-- 2^3 = 8 grouping sets: all combinations of the three dimensions
-- Full pivot table in one query
```

**Distinguishing NULL grouping from data NULLs using GROUPING():**
```sql
SELECT
    region,
    product,
    SUM(revenue),
    GROUPING(region) AS region_is_rollup,   -- 1 = this row is a rollup total
    GROUPING(product) AS product_is_rollup  -- 1 = this row is a rollup total
FROM sales
GROUP BY ROLLUP (region, product);
-- Use GROUPING() to distinguish "subtotal row" from "data had NULL region"

-- Better presentation:
SELECT
    COALESCE(region, CASE WHEN GROUPING(region) = 1 THEN '-- ALL --' ELSE NULL END) AS region,
    COALESCE(product, CASE WHEN GROUPING(product) = 1 THEN '-- ALL --' ELSE NULL END) AS product,
    SUM(revenue)
FROM sales
GROUP BY ROLLUP (region, product);
```

---

## Q6: Explain PostgreSQL full-text search — tsvector, tsquery, indexing, and ranking.

**Answer:**

Full-text search in PostgreSQL is based on two types:
- `tsvector`: a document representation — a sorted list of lexemes (normalized words)
  with their positions, with stop words removed.
- `tsquery`: a search query — lexemes combined with `&` (AND), `|` (OR), `!` (NOT),
  and `<->` (phrase/proximity).

```sql
-- Creating tsvectors:
SELECT to_tsvector('english', 'The quick brown fox jumped over the lazy dog');
-- 'brown':3 'dog':9 'fox':4 'jump':5 'lazi':8 'quick':2
-- "the" and "over" are English stop words → removed
-- "jumped" → "jump" (stemming)

-- Creating tsqueries:
SELECT to_tsquery('english', 'quick & brown');  -- AND query
SELECT to_tsquery('english', 'quick | fast');   -- OR query
SELECT to_tsquery('english', '!slow');          -- NOT query
SELECT phraseto_tsquery('english', 'quick brown fox');  -- phrase
SELECT websearch_to_tsquery('english', '"quick brown" fox');  -- web-style query

-- Matching with @@ operator:
SELECT to_tsvector('english', 'The quick brown fox') @@ to_tsquery('english', 'quick & fox');
-- returns TRUE
```

**Storing tsvector and indexing:**
```sql
-- Option 1: generated column (auto-updated)
ALTER TABLE posts
ADD COLUMN search_vector tsvector
GENERATED ALWAYS AS (
    to_tsvector('english', COALESCE(title, '') || ' ' || COALESCE(body, ''))
) STORED;

CREATE INDEX idx_posts_fts ON posts USING gin (search_vector);

-- Option 2: separate column + trigger (more control over weighting)
ALTER TABLE posts ADD COLUMN search_vector tsvector;

CREATE OR REPLACE FUNCTION update_search_vector() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.body, '')), 'B');
    -- 'A' weight = title matches rank higher than 'B' weight = body matches
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_search_vector_update
BEFORE INSERT OR UPDATE ON posts
FOR EACH ROW EXECUTE FUNCTION update_search_vector();

CREATE INDEX idx_posts_search ON posts USING gin (search_vector);
```

**Querying with ranking:**
```sql
SELECT
    id,
    title,
    ts_rank(search_vector, query) AS rank,
    ts_headline('english', body, query,
        'StartSel=<mark>, StopSel=</mark>, MaxWords=30') AS excerpt
FROM posts,
     websearch_to_tsquery('english', 'database performance') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 10;
-- ts_rank: relevance score (0 to 1)
-- ts_headline: highlighted excerpt with matching terms wrapped in <mark> tags
```
