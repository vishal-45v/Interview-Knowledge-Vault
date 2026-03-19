# Chapter 03 — Indexes and Performance: Analogy Explanations

Making B-tree internals, query planning, and index strategies concrete through
everyday analogies.

---

## Analogy 1: B-tree Index — The Library Catalog System

**The Story:**
Imagine a massive library with 10 million books arranged randomly on shelves by
acquisition date (the heap). To find "Clean Code by Robert Martin," you could
walk every shelf — that is a sequential scan.

The library has a card catalog. At the top is a master index (root node) with
drawers labeled A-F, G-L, M-R, S-Z. You open the M-R drawer (internal node).
Inside are dividers by second letter: Ma-Mc, Md-Mk, Ml-Mz. You go to the Ml
section (another internal node) and find Martin. The card for that book gives
you the exact shelf and position number (TID — tuple identifier). You walk
directly to that shelf and pull the book.

That is an index scan: root → internal node(s) → leaf node → heap fetch.
The catalog does not contain the book content, only pointers.

**Connection:**
```sql
-- B-tree index lookup: 3-4 page reads regardless of table size
SELECT * FROM books WHERE author = 'Martin, Robert';
-- 1. Read root page: find 'Ma-Mz' pointer
-- 2. Read internal page: find 'Mar-' range pointer
-- 3. Read leaf page: find 'Martin, Robert', get TID (page 4821, slot 3)
-- 4. Read heap page 4821: retrieve the actual book row
```

A covering index is like a card catalog that also has a summary of each book
printed on the card — no need to walk to the shelf at all.

---

## Analogy 2: Composite Index Column Order — The Telephone Directory

**The Story:**
A phone directory is sorted by last name first, then first name. To find everyone
named "Smith" is easy — flip to the S section and scan a few pages. To find
everyone named "John" is impossible — you must scan the entire directory because
the directory is not sorted by first name.

To find "John Smith" is easy — go to S, find Smith, then scan for John within
the Smiths. The directory is effectively a composite index on (last_name, first_name).

The **leftmost prefix rule**: you can search efficiently if your first filter
matches the leftmost column(s) of the sort order.

**Connection:**
```sql
CREATE INDEX phone_dir ON contacts (last_name, first_name);

-- Fast: uses index (starts from left)
WHERE last_name = 'Smith'
WHERE last_name = 'Smith' AND first_name = 'John'

-- Slow: cannot use index (no last_name filter)
WHERE first_name = 'John'

-- Solution: create a separate index for first-name-only searches
CREATE INDEX ON contacts (first_name);
```

If you only had first_name searches and last_name searches separately, you
would want TWO separate single-column indexes, not one composite index.

---

## Analogy 3: Partial Index — The VIP Guest List

**The Story:**
A nightclub with 100,000 members in their database. 99% have status = 'standard'
and 1% have status = 'vip'. Every night, the door staff must quickly look up
the VIP list. Maintaining a full alphabetical index of all 100,000 members is
wasteful for this purpose — 99% of the index is useless for VIP lookups.

The smart approach: maintain a separate small list of only the 1,000 VIP members.
This is a partial index — an index that covers only a subset of rows.

**Connection:**
```sql
-- Full index: 100,000 rows indexed, but only 1% are useful for VIP queries
CREATE INDEX ON members (name);  -- massive, mostly unused for VIP lookups

-- Partial index: only 1,000 rows indexed
CREATE INDEX idx_vip_members ON members (name) WHERE status = 'vip';
-- 100x smaller, 100x faster lookups for WHERE status = 'vip' AND name = ?

-- Real-world uses of partial indexes:
-- Index only active (non-deleted) rows:
CREATE INDEX ON orders (customer_id) WHERE deleted_at IS NULL;

-- Index only unpaid invoices:
CREATE INDEX ON invoices (due_date) WHERE paid_at IS NULL;

-- Index only recent data (avoids indexing 10 years of old data):
CREATE INDEX ON events (user_id) WHERE created_at > '2024-01-01';
```

---

## Analogy 4: Table Bloat — The Warehouse with Sticky Price Tags

**The Story:**
A warehouse stores boxes of products. When an item's price changes, workers don't
open the box and replace the price tag on the item — instead they put a new box
next to the old one with a "current price" sticker, and put a "void" sticker on
the old box. The old box still takes up shelf space.

A janitor (autovacuum) comes around periodically and moves the voided boxes to
an empty section of the shelf, making that space available for new boxes. But
the janitor never physically removes shelves — the warehouse footprint stays the same.

Only a full warehouse reorganization (VACUUM FULL / pg_repack) physically
consolidates everything and returns unused shelving to the building.

**Connection:**
```sql
-- UPDATE in PostgreSQL: creates a new row version, marks old as "dead"
UPDATE orders SET status = 'shipped' WHERE id = 12345;
-- New row version created at new location
-- Old row version marked with xmax (visibility "dead" after commit)
-- Dead row stays in heap file, taking space, bloating indexes

-- Autovacuum: marks dead space reusable (doesn't shrink file)
VACUUM orders;
-- Like the janitor rearranging boxes — space is reused by future inserts

-- VACUUM FULL: rewrites entire table (dangerous on production — full lock)
VACUUM FULL orders;

-- Production-safe compaction:
-- Use pg_repack: rewrites table online with only brief lock at the end
```

---

## Analogy 5: Hash Join vs Nested Loop — Speed Dating vs Looking Through a Phonebook

**The Story:**
**Nested Loop (Phonebook approach):** For each of the 5,000 employees, you look
up their department in the department phonebook (an alphabetical index). Each
lookup takes a second. 5,000 × 1 second = 5,000 seconds total. Efficient only
if the phonebook lookup is fast (indexed inner side) and the outer side is small.

**Hash Join (Speed dating):** Before the event starts, you build a seating chart
(hash table) of all department representatives arranged by department ID. Then
each of the 5,000 employees walks in and immediately finds their table in O(1).
Total: time to build the chart + 5,000 × instant lookup.

Best when both sides are large and an index isn't available. The hash table must
fit in memory (`work_mem`).

**Connection:**
```sql
-- Nested loop: good when orders is small and there's an index on customers
SELECT o.id, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id;
-- If orders has 100 rows and customers has an index: nested loop wins

-- Hash join: good when both tables are large
-- PostgreSQL chooses this automatically when hash is estimated faster
-- Increase work_mem to allow larger hash tables:
SET work_mem = '256MB';  -- Per-query, per-operation
-- This can convert a disk-spilling hash join to an in-memory one
```

---

## Analogy 6: Index Selectivity — The Encyclopedia vs the Word "the"

**The Story:**
Imagine an encyclopedia's back-of-book index. If you look up "mitochondria,"
the index points to 3 pages — very selective. If you look up "the," it would
point to every single page — completely non-selective.

A database planner works the same way. An index on `country_code` in a US-only
application (where 99% of rows have 'US') is like indexing the word "the" —
it points to almost every row, so using the index is actually slower than
just reading the book front-to-back (sequential scan).

**Connection:**
```sql
-- High selectivity (use index): finds very few rows
SELECT * FROM users WHERE user_id = 42;           -- 1 row out of 10M
SELECT * FROM orders WHERE order_number = 'X-99'; -- 1 row out of 50M

-- Low selectivity (sequential scan is better): finds too many rows
SELECT * FROM users WHERE country = 'US';   -- 98% of 10M users
SELECT * FROM users WHERE is_active = TRUE; -- 90% of users

-- Check selectivity estimate:
SELECT n_distinct, correlation FROM pg_stats
WHERE tablename = 'users' AND attname = 'country';
-- n_distinct: number of distinct values (negative = fraction of total)
-- correlation: 1.0 = perfectly sequential, 0 = random (affects index vs seq scan)
```

---

## Analogy 7: EXPLAIN ANALYZE — The Race Car Telemetry Dashboard

**The Story:**
A race engineer does not just time the car's final lap. They attach sensors to
every component: engine temperature, fuel consumption per corner, tire pressure
at each wheel. After the race, the telemetry dashboard shows where time was lost —
a corner where the driver braked too early, a straight where the engine lagged.

EXPLAIN ANALYZE is the query's telemetry. You see not just the total execution
time but the cost of each node in the plan tree: which table scan was slow,
which join consumed the most time, where the planner's estimates were wrong.

**Connection:**
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT u.name, COUNT(o.id)
FROM users u
JOIN orders o ON u.id = o.user_id
GROUP BY u.name;

-- Sample output interpretation:
-- HashAggregate (actual time=1234.5 loops=1)  ← GROUP BY node
--   -> Hash Join (actual time=800.2 loops=1)  ← join node, 800ms
--        -> Seq Scan on orders (actual rows=5000000 loops=1)
--           Buffers: shared hit=1000 read=40000  ← 40K disk reads = slow!
--        -> Hash
--             -> Seq Scan on users (actual rows=200000 loops=1)
--                Buffers: shared hit=5000 read=0  ← all cached = fast

-- Diagnosis: orders not in buffer cache, needs more shared_buffers
-- or sequential scan on orders needs better indexing for this query pattern
```

Each node is like a telemetry sensor. High "read" vs "hit" = cache miss.
Large discrepancy between rows estimate and actual rows = stale statistics.

---

## Analogy 8: BRIN Index — The Neighborhood Map

**The Story:**
Imagine a city where a crime database stores incidents in the order they were
reported. The database does not have exact addresses for every crime, but it
knows that crimes 1 through 10,000 happened in the North district, crimes
10,001 through 20,000 in the South district, etc. A neighborhood map divides
the database into geographic regions.

If you're asked to find crimes in the South district, you can skip the North
district entirely — you know it contains none. This is a BRIN (Block Range Index):
it stores the minimum and maximum values of a column for each block range (128
pages by default), allowing the database to skip entire block ranges that cannot
contain matching rows.

**Connection:**
```sql
-- BRIN works ONLY when data is physically correlated with column values
-- Time-series data inserted chronologically is perfectly correlated
CREATE INDEX ON sensor_readings USING brin (recorded_at);
-- The BRIN index is tiny: one entry per 128 pages (vs one per row for B-tree)
-- For a 1TB time-series table: B-tree index ~20GB, BRIN index ~1MB

-- BRIN fails for randomly ordered data (UUID primary keys, shuffled data):
-- Minimum and maximum of every block range would be nearly the same (full range)
-- making every block range a potential match — BRIN becomes useless

-- Check correlation to decide B-tree vs BRIN:
SELECT attname, correlation FROM pg_stats WHERE tablename = 'sensor_readings';
-- correlation near 1.0 or -1.0 → BRIN effective
-- correlation near 0.0 → use B-tree
```
