# Chapter 03 — Indexes and Performance: Structured Answers

Complete, interview-ready answers for the most critical indexing and performance
questions at the senior engineer level.

---

## Q1: Describe the internal structure of a B-tree index and how it handles
a point lookup.

**Answer:**

A B-tree (Balanced Tree) index has three types of nodes: root, internal (branch),
and leaf.

**Root Node:** The top of the tree. Contains separator keys and pointers to child
nodes. For a small index the root may also be a leaf.

**Internal Nodes:** Contain separator keys (lower bounds for each subtree) and
pointers to child nodes at the next level. They do not contain actual row pointers.

**Leaf Nodes:** Contain the actual index key values and pointers (TIDs — tuple
identifiers = heap page number + slot number) to the actual rows in the table.
Leaf nodes are linked in a doubly-linked list, enabling efficient range scans
without returning to the root.

```
          Root Node
    ┌──────────────────────────┐
    │  [50]  [150]  [300]      │
    └───┬────────┬────────┬────┘
        │        │        │
  ┌─────┴──┐  ┌──┴─────┐  ┌──┴──────┐
  │ [10,30]│  │[70,100]│  │[200,250]│  Internal nodes
  └──┬──┬──┘  └──┬──┬──┘  └──┬──┬───┘
     │  │        │  │         │  │
  [Leaf][Leaf] [Leaf][Leaf] [Leaf][Leaf]
  key→TID      key→TID      key→TID     Leaf nodes (linked ◄──►)
```

**Point lookup algorithm:**
1. Start at root. Compare search key against separator values.
2. Follow pointer to appropriate child node.
3. Repeat until reaching a leaf node. O(log N) page reads.
4. At leaf: find the key, extract TID, fetch the actual row from the heap.
5. If index-only scan: leaf has all needed columns, heap fetch skipped.

**Height of a B-tree:** Typically 3-4 levels for tables up to hundreds of millions
of rows. Each internal page in PostgreSQL (8KB) can hold ~200-400 pointers. A
3-level tree can index 200^3 = 8 million keys; a 4-level tree handles ~1.6 billion.

```sql
-- Check index height (number of levels):
SELECT * FROM bt_metap('idx_orders_customer_id');
-- Returns: root page, levels (height), fastroot, etc.
-- levels=3 means 4 levels including root (levels is depth-1)
```

---

## Q2: Explain composite index column ordering and the leftmost prefix rule
with practical examples.

**Answer:**

A composite index stores keys sorted by the first column, then by the second
column within each first-column value, then by the third column within that, etc.
The result is a lexicographically ordered structure.

The **leftmost prefix rule**: a query can use the index if it provides equality
conditions on a leading prefix of the index columns, followed optionally by a
range condition on the next column.

```sql
CREATE INDEX idx_orders ON orders (customer_id, status, created_at);

-- CAN use this index (full key prefix):
WHERE customer_id = 42 AND status = 'pending' AND created_at > '2024-01-01'

-- CAN use this index (partial prefix — equality on first column):
WHERE customer_id = 42

-- CAN use this index (equality on first two, range on third):
WHERE customer_id = 42 AND status = 'pending' AND created_at BETWEEN X AND Y

-- CAN use this index (equality on first, range on second):
WHERE customer_id = 42 AND status > 'p'
-- BUT: created_at cannot be filtered via index (range on status "breaks" the chain)

-- CANNOT use this index (missing leftmost column):
WHERE status = 'pending'
WHERE status = 'pending' AND created_at > '2024-01-01'

-- CANNOT use this index (only third column):
WHERE created_at > '2024-01-01'
```

**Column ordering strategy:**
- Put equality-filtered columns first (highest cardinality first within the
  equality set, for better page pruning).
- Put range-filtered columns last.
- Put ORDER BY columns after equality filters to enable index sort (avoid Sort node).

```sql
-- Query: WHERE customer_id = 42 AND status = 'pending' ORDER BY created_at DESC
-- Optimal index:
CREATE INDEX ON orders (customer_id, status, created_at DESC);
-- Result: no Sort operation in EXPLAIN — index delivers rows pre-sorted
```

---

## Q3: How do you read and interpret EXPLAIN ANALYZE output?

**Answer:**

EXPLAIN ANALYZE runs the query, measures actual execution, and reports both
estimates and actuals.

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;
```

Sample output:
```
Index Scan using idx_orders_customer on orders
  (cost=0.56..1234.89 rows=450 width=64)
  (actual time=0.112..8.432 rows=432 loops=1)
  Index Cond: (customer_id = 42)
  Buffers: shared hit=15 read=32
Planning Time: 0.312 ms
Execution Time: 8.617 ms
```

**Field by field:**

`cost=0.56..1234.89`:
- 0.56 = startup cost (cost to return first row) in arbitrary planner units
- 1234.89 = total cost to return all rows
- Not milliseconds — these are cost units relative to seq_page_cost=1.0

`rows=450`:
- Planner's estimate of rows this node returns. Based on statistics.

`width=64`:
- Estimated average row width in bytes.

`actual time=0.112..8.432`:
- 0.112ms = time to return first actual row
- 8.432ms = total wall time for this node (cumulative, includes children)

`actual rows=432`:
- Actual rows returned. Compare to estimated rows=450 — close, planner is good.
- If estimated=1 and actual=450,000 — statistics are stale, run ANALYZE.

`loops=1`:
- How many times this node was executed. For nested loop inners, this is the
  outer row count. Total actual rows = actual rows × loops.

`Buffers: shared hit=15 read=32`:
- hit=15: 15 pages served from shared_buffers (no disk I/O)
- read=32: 32 pages read from disk
- High read count on a repeated query = cache thrashing or working set > shared_buffers

**Common patterns to spot:**

```
-- Bad: estimate vs actual mismatch → stale statistics
rows=1 actual rows=450000  → Run ANALYZE or increase statistics target

-- Bad: large Sort node → missing index for ORDER BY
Sort  (cost=50000.00..52000.00)  → Add index with matching sort order

-- Bad: nested loop with high loops on inner side
Nested Loop (loops=50000)  → Hash Join or index on inner side needed

-- Good: Index Only Scan → all data served from index, no heap access
Index Only Scan (heap fetches: 0)  → Perfect covering index
```

---

## Q4: What is table and index bloat, how do you detect it, and how do you fix it?

**Answer:**

**Why bloat happens:** PostgreSQL's MVCC model never overwrites rows in place.
Every UPDATE creates a new row version and marks the old version as dead. Every
DELETE marks the row as dead. Dead rows remain in the heap and indexes until
VACUUM removes them. If VACUUM falls behind (heavy write load, long-running
transactions blocking VACUUM), dead rows accumulate — this is bloat.

**How bloat harms performance:**
- Larger table means more pages to scan (slower seq scan).
- Larger indexes mean more pages to traverse (slightly slower index scan).
- More pages means more buffer cache pressure.
- Write amplification: an UPDATE must write a new version AND maintain all indexes.

```sql
-- Detect table bloat (simplified approach using pgstattuple extension):
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('orders');
-- Returns: dead_tuple_percent, free_space, tuple_count, dead_tuple_count

-- Quick heuristic via pg_stat_user_tables:
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 1) AS dead_pct,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY dead_pct DESC NULLS LAST;

-- Detect index bloat:
SELECT indexrelname, idx_blks_read, idx_blks_hit
FROM pg_statio_user_indexes
ORDER BY idx_blks_read DESC;
```

**Fixing bloat:**

Option 1: VACUUM — marks dead space as reusable, no disk space returned, no lock.
```sql
VACUUM orders;              -- non-blocking, runs in background
VACUUM VERBOSE orders;      -- shows what it cleaned
VACUUM ANALYZE orders;      -- also updates statistics
```

Option 2: VACUUM FULL — rewrites the table, returns disk space, requires full lock.
```sql
VACUUM FULL orders;   -- BLOCKS all access for the entire duration. Use pg_repack instead.
```

Option 3: REINDEX CONCURRENTLY — rebuilds the index without blocking reads.
```sql
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;
-- Creates new index, swaps atomically, drops old. Safe for production.
-- Cannot be run inside a transaction block.
```

Option 4: pg_repack — rewrites table and all indexes with minimal locking. The
production-safe alternative to VACUUM FULL.

**Preventing bloat:**
- Tune autovacuum aggressiveness: lower `autovacuum_vacuum_scale_factor` for
  high-churn tables.
- Keep transactions short to avoid holding back the MVCC horizon.
- Monitor `age(relfrozenxid)` — if it approaches `autovacuum_freeze_max_age`,
  autovacuum will run aggressively.

---

## Q5: Explain the three join algorithms PostgreSQL uses and when each is chosen.

**Answer:**

**1. Nested Loop Join**

For each row in the outer table, scan the inner table for matches. Best when
the outer side is small and the inner side has an index on the join key.

```
Outer: 100 rows, Inner: 10M rows with index
Cost: 100 × (index scan cost ≈ a few ms each) = fast
```

```sql
-- EXPLAIN shows:
-- Nested Loop
--   -> Seq Scan on small_table (outer)
--   -> Index Scan on large_table using idx_key (inner, loops=100)
```

Worst case: outer has 50,000 rows, inner has no index → 50,000 × full scan.

**2. Hash Join**

Build a hash table from the smaller side (build side), then probe it with the
larger side (probe side). No index required. Best for large unsorted joins.

```
Memory: hash table of smaller side must fit in work_mem
Cost: O(N + M) where N, M are row counts of the two sides
```

```sql
-- EXPLAIN shows:
-- Hash Join (Hash Cond: (a.id = b.a_id))
--   -> Seq Scan on large_table (probe side)
--   -> Hash
--        -> Seq Scan on smaller_table (build side — hashed into memory)
```

If hash table exceeds `work_mem`: spills to disk (Hash Batches > 1 in EXPLAIN).
Increasing `work_mem` can convert a disk-spilling hash join to an in-memory one.

**3. Merge Join**

Both sides must be sorted on the join key. Merges them like a zip operation.
O(N log N) if sorting needed, O(N + M) if already sorted. Best when both sides
are large and either already sorted or have index-provided ordering.

```sql
-- EXPLAIN shows:
-- Merge Join (Merge Cond: (a.id = b.a_id))
--   -> Index Scan on table_a using idx_a_id (already sorted by id)
--   -> Index Scan on table_b using idx_b_a_id (already sorted by a_id)
```

**Decision matrix:**

```
┌──────────────────┬─────────────────┬────────────────┬────────────────┐
│ Condition        │ Nested Loop     │ Hash Join      │ Merge Join     │
├──────────────────┼─────────────────┼────────────────┼────────────────┤
│ Outer very small │ Best            │ Overkill       │ Overkill       │
│ Both sides large │ Poor (no index) │ Good           │ Good if sorted │
│ Index on inner   │ Excellent       │ Not needed     │ Not needed     │
│ Already sorted   │ N/A             │ Ignores sort   │ Ideal          │
│ Low memory       │ Fine            │ Needs work_mem │ Fine           │
└──────────────────┴─────────────────┴────────────────┴────────────────┘
```
