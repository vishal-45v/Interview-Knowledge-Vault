# Chapter 05 — Advanced SQL: Analogy Explanations

Making window functions, recursive CTEs, JSONB, and advanced PostgreSQL features
concrete through everyday analogies.

---

## Analogy 1: Window Functions — The Scoreboard That Doesn't Eliminate Players

**The Story:**
At a tennis tournament, after every match the scoreboard updates each player's
standing: their current rank, their total career wins, their wins compared to
the previous tournament. But every player remains on the scoreboard — being ranked
doesn't eliminate you. The scoreboard computes positions RELATIVE TO ALL PLAYERS
without removing any of them from the list.

This is the essence of window functions. An aggregate function with GROUP BY
would tell you the top-ranked player and eliminate everyone else. A window
function tells each player their rank while keeping all players in the result.

**Connection:**
```sql
-- GROUP BY: collapses rows — tells you the max, loses the individual rows
SELECT department, MAX(salary) FROM employees GROUP BY department;
-- Returns one row per department, individual employee rows are gone

-- Window function: keeps all rows, computes relative position
SELECT
    name,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    MAX(salary) OVER (PARTITION BY department) AS dept_max
FROM employees;
-- All 500 employees are in the result; each knows their own rank
-- PARTITION BY is like "within their own department's scoreboard"
```

---

## Analogy 2: Window Frame — The Sliding Newspaper Archive Window

**The Story:**
A researcher at a newspaper archive looks at articles through a physical viewing
window frame. The frame can be adjusted:

- **UNBOUNDED PRECEDING AND CURRENT ROW:** The frame starts at the very first
  article ever published and ends at today's article. Good for calculating
  cumulative article counts over time.
- **7 PRECEDING AND CURRENT ROW (ROWS):** Show only the 8 most recent articles
  (by position on the shelf).
- **RANGE BETWEEN INTERVAL '7 days' PRECEDING AND CURRENT ROW:** Show all articles
  published within the last 7 days from the current article's date. If 3 articles
  were published on the same day, all 3 are in the window.
- **UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING:** The entire archive is always
  in view — gives the same global total for every row.

**Connection:**
```sql
SELECT
    article_date,
    title,
    -- Cumulative count of articles up to this date
    COUNT(*) OVER (
        ORDER BY article_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_count,

    -- Rolling 7-day article count by date value (not row count)
    COUNT(*) OVER (
        ORDER BY article_date
        RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
    ) AS rolling_7day_count,

    -- Total articles ever (same for every row)
    COUNT(*) OVER () AS total_ever
FROM articles
ORDER BY article_date;
```

---

## Analogy 3: Recursive CTE — The Family Tree Research

**The Story:**
A genealogist starts research with a known ancestor (the anchor). For each person
found, they look up that person's children in the records. For each child found,
they look up that child's children. They continue until no more records are found
in the archive.

They keep a notebook tracking everyone found so far (to avoid researching the
same person twice if records have cycles). At the end, they have a complete family
tree from the original ancestor down through all generations.

**Connection:**
```sql
WITH RECURSIVE family_tree AS (
    -- Anchor: start with the known founder
    SELECT id, name, parent_id, 0 AS generation, ARRAY[id] AS lineage
    FROM persons WHERE name = 'John Adams'

    UNION ALL

    -- Recursive: find each person's children
    SELECT p.id, p.name, p.parent_id, ft.generation + 1, ft.lineage || p.id
    FROM persons p
    JOIN family_tree ft ON p.parent_id = ft.id
    WHERE NOT (p.id = ANY(ft.lineage))  -- prevent cycles in corrupted data
      AND ft.generation < 15            -- max 15 generations deep
)
SELECT
    REPEAT('  ', generation) || name AS family_tree,
    generation
FROM family_tree
ORDER BY lineage;
-- lineage array gives correct sibling ordering
```

The notebook = the `lineage` array. Research continues until no new descendants
are found (UNION ALL produces no new rows). The generation limit is a safety net
in case the genealogical records have loops.

---

## Analogy 4: LATERAL Join — The Personalized Recommendation Engine

**The Story:**
An online bookstore wants to show each customer "their last 3 purchases" on their
homepage. A regular JOIN can combine customers with all purchases, but getting
exactly 3 per customer requires something special.

A regular subquery in SELECT can only return one value. A regular JOIN cannot
be told "stop after 3 rows per customer." A LATERAL join is like giving each
customer their own personal shopper: "For customer A, go to the purchases shelf
and bring back the top 3. For customer B, go to the same shelf and bring back
their top 3." Each customer gets a personalized result from the same query logic.

**Connection:**
```sql
-- Regular JOIN cannot do "top N per group" cleanly:
-- Would need a subquery, window function, or LATERAL

-- LATERAL: personalized query per outer row
SELECT c.name, recent.*
FROM customers c
LEFT JOIN LATERAL (
    SELECT p.title, p.purchase_date, p.amount
    FROM purchases p
    WHERE p.customer_id = c.id      -- reference outer table: this is the LATERAL part
    ORDER BY p.purchase_date DESC
    LIMIT 3
) recent ON true;  -- ON true: always include, even if subquery returns 0 rows

-- The subquery runs ONCE PER customer row, each time with a different c.id
-- Without LATERAL: you'd need a window function approach or correlated subquery
--                  that can only return one value at a time
```

The key phrase: "reference outer table column inside the subquery" — that's
what makes it LATERAL. A regular subquery cannot do this.

---

## Analogy 5: JSONB — The Flexible Filing Cabinet

**The Story:**
A regular SQL table is like a fixed office desk with labeled drawers: one drawer
for "name," one for "email," one for "phone." Every desk in the building is
identical. If you want to add a "fax number" drawer, you must renovate every
single desk in the building (ALTER TABLE — schema migration).

A JSONB column is like one large deep drawer labeled "everything else." Each
person can put a different collection of items in their drawer: one person might
have their business card, a photo, and a contract. Another might have just a
sticky note. The drawer can hold anything, structured however they like.

The GIN index is like a concordance — a master list of every key-value pair
across all drawers, so you can instantly find "everyone with a business card"
without opening each drawer individually.

**Connection:**
```sql
-- Schema migration for regular columns: ALTER TABLE (potentially locks table)
ALTER TABLE users ADD COLUMN fax_number TEXT;

-- JSONB: no migration needed, just start inserting new keys
UPDATE users SET profile = profile || '{"fax_number": "555-9999"}' WHERE id = 42;
-- Other users don't need to have 'fax_number' at all — completely optional

-- GIN index enables fast search across all profiles
CREATE INDEX ON users USING gin (profile);
SELECT id, name FROM users WHERE profile @> '{"subscription_tier": "premium"}';
-- Finds all users with premium tier, regardless of other profile contents

-- Trade-off: no type enforcement, no compile-time column validation
-- Lost: foreign key constraints, strong typing, optimized storage per column
```

---

## Analogy 6: Materialized View vs Regular View — The Photograph vs the Window

**The Story:**
A regular view is like a window in your office. When you look through it, you
see the current state of the world outside in real time. The view is always up
to date. But every time you look, you're actually doing the work of looking
(the underlying query runs).

A materialized view is like a photograph of the view outside your window, printed
and hung on the wall. It is fast to look at (no computation). But it shows
yesterday's weather. You must periodically retake the photo (REFRESH) to update it.
During the photo session, the wall photo is temporarily unavailable (unless you
use CONCURRENTLY, which lets you keep the old photo visible until the new one is ready).

**Connection:**
```sql
-- Regular view: real-time but potentially slow
CREATE VIEW daily_revenue AS
SELECT DATE_TRUNC('day', order_date), SUM(amount)
FROM orders GROUP BY 1;  -- Runs the full aggregation every time queried

-- Materialized view: fast but stale
CREATE MATERIALIZED VIEW mv_daily_revenue AS
SELECT DATE_TRUNC('day', order_date), SUM(amount)
FROM orders GROUP BY 1;  -- Aggregates 500M rows once and caches the result

-- Refresh options:
REFRESH MATERIALIZED VIEW mv_daily_revenue;  -- Blocks reads during refresh

-- Non-blocking refresh (requires unique index on the matview):
CREATE UNIQUE INDEX ON mv_daily_revenue (date_trunc);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_daily_revenue;
-- Old photo stays on wall until new one is ready → no downtime for readers
```

---

## Analogy 7: INSERT ON CONFLICT (Upsert) — The Post Office's Return-To-Sender Rule

**The Story:**
Imagine a post office that delivers packages. When a package arrives for an
address where someone already lives, the post office has a choice:

1. **DO NOTHING:** Return the package to sender (ignore the duplicate).
2. **DO UPDATE:** Check if the new package is better (newer, more important) than
   what's already there. If yes, swap it. If no, keep the existing one.

The post office has a "conflict resolution policy" that specifies exactly what
to do when two packages compete for the same address. The NEW package that caused
the conflict is available in the EXCLUDED pseudo-table — it's the returned package
you can inspect to decide what to do.

**Connection:**
```sql
-- User preference sync: event stream sends updates, duplicates are common
-- Rule: update only if the new event is more recent

INSERT INTO user_preferences (user_id, pref_key, pref_value, updated_at)
VALUES ($1, $2, $3, NOW())
ON CONFLICT (user_id, pref_key)           -- conflict target: unique constraint
DO UPDATE SET
    pref_value = EXCLUDED.pref_value,     -- use new value
    updated_at = EXCLUDED.updated_at      -- use new timestamp
WHERE user_preferences.updated_at < EXCLUDED.updated_at;  -- only if newer
-- If existing row is newer (clock skew): DO UPDATE WHERE is false → row unchanged

-- DO NOTHING: useful for idempotent inserts
INSERT INTO events (id, payload)
SELECT id, payload FROM incoming_batch
ON CONFLICT (id) DO NOTHING;
-- Duplicate event IDs from at-least-once delivery are silently ignored
```

---

## Analogy 8: ROLLUP — The Business Dashboard Drill-Down

**The Story:**
A business intelligence dashboard for a retail chain shows revenue at multiple
levels: every store, every region, every country, and a global total. These are
not four separate views — they are one unified "drill-down" report where each
level is a subtotal of its children.

ROLLUP generates exactly these drill-down subtotals by progressively stripping
off the rightmost dimension: you start with the finest granularity (store level)
and roll up to coarser granularities (region, country, grand total).

**Connection:**
```sql
-- Single query produces all drill-down levels:
SELECT
    country,
    region,
    store_id,
    SUM(revenue) AS total_revenue,
    COUNT(DISTINCT transaction_id) AS transaction_count
FROM sales
GROUP BY ROLLUP (country, region, store_id)
ORDER BY country NULLS LAST, region NULLS LAST, store_id NULLS LAST;

-- Result structure:
-- 'US' | 'East'  | store_1 | 15000 | 220   ← finest: per store
-- 'US' | 'East'  | store_2 | 18000 | 280
-- 'US' | 'East'  | NULL    | 33000 | 500   ← rollup: per region
-- 'US' | 'West'  | store_3 | 12000 | 150
-- 'US' | 'West'  | NULL    | 12000 | 150   ← rollup: per region
-- 'US' | NULL    | NULL    | 45000 | 650   ← rollup: per country
-- 'UK' | 'North' | store_5 | 9000  | 120
-- ...
-- NULL | NULL    | NULL    | 80000 | 1100  ← grand total

-- GROUPING(country) distinguishes NULL subtotal from NULL data value
```
