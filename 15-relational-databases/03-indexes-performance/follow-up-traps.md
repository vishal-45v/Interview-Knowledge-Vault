# Chapter 03 — Indexes and Performance: Follow-Up Traps

Tricky follow-up questions revealing whether a candidate understands index
mechanics deeply vs having surface-level knowledge.

---

## Trap 1: "Adding an index always makes queries faster."

**What most people say:** Yes, indexes speed up lookups.

**Correct answer:** Indexes make reads faster for selective queries but hurt
write performance and can actually make some reads slower. The planner may choose
an index scan over a sequential scan when a sequential scan would be faster
(e.g., when selecting 40% of a table — reading random pages is worse than a
single sequential pass). Additionally, an index on a low-cardinality column
(e.g., boolean, gender with 2 values) is almost never used by the planner and
wastes space and write overhead.

```sql
-- Worst case: index on boolean column (50% selectivity)
CREATE INDEX ON users (is_active);  -- Planner almost never uses this
-- For WHERE is_active = TRUE: index returns ~50% of 10M rows
-- Reading 5M random pages is far slower than one sequential scan

-- When an index scan beats sequential scan:
-- Rule of thumb: selectivity < 5-10% of table rows for random-access storage
-- On SSDs: threshold is higher because random I/O penalty is smaller

-- Check if an index is being used:
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0;  -- These indexes have never been used
```

---

## Trap 2: "A composite index on (a, b) can be used for WHERE b = X."

**What most people say:** Wait, is that the leftmost prefix rule? No, you need
to start from the left, so WHERE b alone can't use it.

**Correct answer:** Correct that it generally cannot use the index for WHERE b=X
alone. The leftmost prefix rule means the index can be used for any prefix of
the key columns: WHERE a=1, or WHERE a=1 AND b=2, but NOT WHERE b=2 alone.
However, there is a nuance: if the optimizer can use an **index-only scan** or
if it has a very small result set estimate, it might use the index with a skip
scan (PostgreSQL 14+ has some index skip scan support, but it's limited). The
general rule holds: create a separate index on (b) if you need it.

```sql
CREATE INDEX idx_orders_cust_status ON orders (customer_id, status);
-- Usable: WHERE customer_id = 5
-- Usable: WHERE customer_id = 5 AND status = 'pending'
-- NOT usable: WHERE status = 'pending'  ← needs separate index on (status)
-- RANGE nuance: WHERE customer_id = 5 AND status LIKE 'pend%'  ← usable
-- RANGE nuance: WHERE customer_id > 5 AND status = 'pending'
--   ← index used for customer_id range, but status cannot be filtered via index
--   after a range scan on the first column
```

---

## Trap 3: "VACUUM removes dead rows and frees disk space."

**What most people say:** Yes, VACUUM cleans up deleted rows and reclaims space.

**Correct answer:** Regular VACUUM marks dead tuple space as reusable but does NOT
return space to the OS. The table file on disk does not shrink. Only VACUUM FULL
(which rewrites the table) returns space to the OS — but it requires an exclusive
lock on the table for its entire duration. On a 500GB table, VACUUM FULL can
lock the table for hours.

```sql
-- Regular VACUUM: marks space as reusable, does not shrink table file
VACUUM orders;  -- no lock, runs in background, safe for production

-- VACUUM FULL: rewrites table, returns disk space, requires exclusive lock
VACUUM FULL orders;  -- DANGEROUS on large production tables

-- Better alternative: pg_repack extension
-- Rewrites the table in the background without exclusive lock
-- SELECT * FROM pg_repack('orders');

-- Check table bloat:
SELECT relname, n_dead_tup, n_live_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY dead_pct DESC;
```

---

## Trap 4: "The INCLUDE columns in a covering index are just like adding more
columns to the index key."

**What most people say:** Yes, they both make the index covering, same thing.

**Correct answer:** There is an important difference. Key columns are stored
at every level of the B-tree and are used for ordering and filtering. INCLUDE
columns are stored ONLY at the leaf level and are not used for lookups or sorting.
This has two advantages:

1. INCLUDE columns can be of types that cannot be index key columns (e.g., very
   large text fields, arrays).
2. The index is smaller because non-key columns are not duplicated at every
   internal node, only at leaves.
3. INCLUDE columns cannot be used in WHERE, ORDER BY, or JOIN conditions via
   the index — they are fetch-only.

```sql
-- Key columns in index — usable in WHERE, ORDER BY
-- Included columns — only usable in SELECT (index-only scan avoids heap fetch)

CREATE INDEX idx_orders_covering ON orders (customer_id, status)
INCLUDE (amount, created_at);

-- This query is an index-only scan (no heap access):
SELECT amount, created_at
FROM orders
WHERE customer_id = 42 AND status = 'pending';

-- This CANNOT use the INCLUDE columns for filtering:
WHERE amount > 100  -- must still access heap or add amount to key columns
```

---

## Trap 5: "An index on a foreign key column is optional — the database doesn't
require it."

**What most people say:** Correct, PostgreSQL doesn't require it, it just
recommends it.

**Correct answer:** It is not required for correctness, but omitting it causes
serious performance problems on DELETE and UPDATE of parent rows. When you delete
a parent row, PostgreSQL must verify that no child rows reference it — which
requires scanning the child table's FK column. Without an index, this is a
sequential scan. On a 10M row orders table, deleting one customer without an
index on orders.customer_id causes a full sequential scan of the orders table
for every customer delete.

```sql
-- Always create an index on FK columns:
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id)
);
CREATE INDEX ON orders (customer_id);  -- Critical for DELETE FROM customers to be fast

-- Check for FK columns missing indexes:
SELECT conrelid::regclass AS fk_table, a.attname AS fk_column
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = c.conkey[1]
WHERE c.contype = 'f'
  AND NOT EXISTS (
    SELECT 1 FROM pg_index i
    WHERE i.indrelid = c.conrelid AND i.indkey[0] = a.attnum
  );
```

---

## Trap 6: "EXPLAIN shows the query plan. EXPLAIN ANALYZE actually runs the query."

**What most people say:** Correct, EXPLAIN just estimates, ANALYZE actually runs it.

**Correct answer:** Mostly correct but with a critical caveat: EXPLAIN ANALYZE
actually executes the query, including all side effects. If you run EXPLAIN ANALYZE
on an UPDATE or DELETE, it WILL modify the data. You must wrap it in a transaction
and rollback:

```sql
-- DANGEROUS: actually deletes rows
EXPLAIN ANALYZE DELETE FROM orders WHERE status = 'cancelled';

-- Safe pattern:
BEGIN;
EXPLAIN ANALYZE DELETE FROM orders WHERE status = 'cancelled';
ROLLBACK;  -- Always rollback when analyzing write queries
```

Also: EXPLAIN ANALYZE reports wall-clock time. For micro-benchmarking, run
the query multiple times (cache effects), and use `EXPLAIN (ANALYZE, BUFFERS)`
to see cache hit vs disk read ratios:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 42;
-- Shows: "Buffers: shared hit=X read=Y"
-- hit = served from shared_buffers (fast)
-- read = required disk I/O (slow)
```

---

## Trap 7: "A GIN index on a JSONB column makes all JSONB queries fast."

**What most people say:** Yes, add a GIN index and JSONB queries will be fast.

**Correct answer:** A GIN index on JSONB enables fast containment and existence
queries (`@>`, `?`, `?|`, `?&`) but does NOT help with all JSONB operations.
Specifically, path extraction queries using `->>` or `#>>` are NOT accelerated
by a default GIN index. You need an expression index for those.

```sql
-- GIN index: fast for containment
CREATE INDEX idx_events_data_gin ON events USING gin (data);

-- These queries USE the GIN index:
SELECT * FROM events WHERE data @> '{"type": "click"}';
SELECT * FROM events WHERE data ? 'user_id';

-- These queries do NOT use the GIN index:
SELECT * FROM events WHERE data->>'type' = 'click';  -- needs expression index
SELECT * FROM events WHERE (data->>'amount')::numeric > 100;  -- needs expression index

-- Expression index for path queries:
CREATE INDEX idx_events_type ON events ((data->>'type'));
CREATE INDEX idx_events_amount ON events (((data->>'amount')::numeric));
-- Now these queries use the expression indexes
```

---

## Trap 8: "Setting statistics target higher always improves query plans."

**What most people say:** Higher statistics mean better estimates, so it should
always improve things.

**Correct answer:** Higher statistics (ALTER TABLE ... ALTER COLUMN ... SET
STATISTICS N) give the planner more histogram buckets and MCV (Most Common Value)
entries for that column, which generally improves cardinality estimates. However:

1. ANALYZE takes longer because it must collect more samples.
2. The pg_statistic catalog grows, using more shared memory.
3. Planning time itself increases because the planner must process larger
   statistics structures.
4. For perfectly uniform distributions or very low-cardinality columns, higher
   statistics give no benefit.

```sql
-- Default statistics target: 100 (provides ~300 histogram buckets)
-- Increase for columns with complex distributions:
ALTER TABLE orders ALTER COLUMN amount SET STATISTICS 500;
ANALYZE orders;  -- Must re-run ANALYZE for new target to take effect

-- Check current statistics:
SELECT attname, attstattarget
FROM pg_attribute
WHERE attrelid = 'orders'::regclass AND attstattarget != -1;

-- Extended statistics for correlated columns (PostgreSQL 10+):
CREATE STATISTICS orders_customer_status ON customer_id, status FROM orders;
ANALYZE orders;
-- Now the planner understands that customer_id and status are correlated
-- (a customer tends to have orders in one status at a time)
```

---

## Trap 9: "Indexes on NULLable columns are useless because NULLs aren't indexed."

**What most people say:** Right, NULLs aren't indexed in most databases so
NULLable columns need special treatment.

**Correct answer:** In PostgreSQL, NULLs ARE stored in B-tree indexes (unlike
Oracle where NULL is not indexed in standard indexes). This means:

```sql
-- PostgreSQL B-tree index DOES store NULLs
CREATE INDEX ON orders (deleted_at);

-- This query CAN use the index:
SELECT * FROM orders WHERE deleted_at IS NULL;  -- Uses the index in PostgreSQL!

-- However, if the majority of rows have deleted_at IS NULL (99% are not deleted),
-- the planner may still choose seq scan due to low selectivity.
-- A partial index is more efficient:
CREATE INDEX idx_active_orders ON orders (id) WHERE deleted_at IS NULL;
-- This index only contains the 1% of rows that are active, making it tiny
-- and highly selective.
```

In Oracle and SQL Server, IS NULL cannot use a standard B-tree index — you need
a function-based index or a NULL bitmap index. PostgreSQL's behavior is different
and is a common portability trap.

---

## Trap 10: "Hash indexes are faster than B-tree for equality lookups."

**What most people say:** Yes, O(1) hash lookup vs O(log n) B-tree.

**Correct answer:** In theory, hash gives O(1) vs O(log n) for point lookups.
In practice, the difference is negligible because a B-tree on a large table
rarely exceeds 4-5 levels, meaning O(log n) is 4-5 page reads regardless of
table size. Hash indexes offer no benefit for range queries (`>`, `<`, `BETWEEN`),
sorting, or prefix matching. They are also larger than B-tree on most datasets
(due to hash collision chains and bucket overhead). The real-world recommendation:
use B-tree for almost everything. Use hash only when you can measure a genuine
point-lookup performance gain and the workload is exclusively equality operations.

```sql
-- Hash index: only equality, no range, no order
CREATE INDEX ON users USING hash (email);

-- B-tree: equality AND range AND sort
CREATE INDEX ON users (email);  -- B-tree by default

-- Performance is nearly identical for equality lookups in practice;
-- B-tree wins on versatility
```
