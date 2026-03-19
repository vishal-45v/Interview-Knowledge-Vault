# Query Optimization — Diagram Explanations

ASCII diagrams illustrating query planning, execution, and optimization concepts.

---

## Diagram 1: PostgreSQL Query Processing Pipeline

```
Client sends SQL string
        │
        ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PARSER                                                             │
│  Input:  "SELECT o.id, c.name FROM orders o JOIN customers c        │
│           ON c.id = o.customer_id WHERE o.total > 100"              │
│  Output: Parse Tree (abstract syntax tree)                          │
│          SelectStmt { targetList, fromClause, whereClause }         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ parse tree
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ANALYZER / SEMANTIC ANALYSIS                                       │
│  - Resolve table names to OIDs (orders → 16384, customers → 16385) │
│  - Resolve column names and types                                   │
│  - Validate that o.total exists and is numeric                      │
│  Output: Query tree with OIDs, types, and resolved names            │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ query tree
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  REWRITER                                                           │
│  - Apply view definitions (expand views into base table queries)    │
│  - Apply row-level security policies (add WHERE tenant_id = $1)     │
│  - Apply rules (PostgreSQL's rule system)                           │
│  Output: Rewritten query tree (may be different from input)         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ rewritten query tree
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  PLANNER / OPTIMIZER                                                │
│  - Generate all possible access paths for each relation             │
│  - Estimate costs using pg_stats + cost model parameters            │
│  - Enumerate join orderings (dynamic programming for ≤8 tables)    │
│  - Choose cheapest plan tree                                        │
│  Output: Plan tree (PlanNode hierarchy with cost estimates)         │
│                                                                     │
│  Plan tree example:                                                 │
│  HashJoin (cost=1234..5678)                                         │
│  ├── Index Scan on orders using idx_total (cost=0.57..2341)         │
│  └── Hash on customers (cost=1234..1234)                            │
│      └── Seq Scan on customers (cost=0..1000)                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ plan tree
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  EXECUTOR                                                           │
│  - Walk the plan tree, executing each node                          │
│  - Fetch rows from storage via buffer manager                       │
│  - Apply filters, joins, sorts, aggregations                        │
│  Output: Result rows → sent to client                               │
└─────────────────────────────────────────────────────────────────────┘
```

**Key insight:** The planner and executor are completely separate. The planner's cost numbers are ESTIMATES based on statistics. EXPLAIN shows planned costs; EXPLAIN ANALYZE shows both planned and actual values, revealing estimation errors.

---

## Diagram 2: Join Algorithm Selection Decision Tree

```
  Start: Need to join Table A (nA rows) and Table B (nB rows) on key K
                              │
              ┌───────────────┼───────────────┐
              │                               │
         nA is small?                    nA is large?
         (< ~1000 rows)                  (> 10K rows)
              │                               │
              ▼                               ▼
    Does B have an index on K?     Is work_mem large enough
         │            │            to hold B in memory?
        YES           NO                │         │
         │             │               YES        NO
         ▼             ▼                │          │
    NESTED LOOP   Can we sort       HASH JOIN   Both A and B
    (index scan   A and B on K                  sortable on K?
     on inner)   cheaply?                            │
                    │     │                    YES        NO
                   YES    NO                   │          │
                    │     │              MERGE JOIN   External
                    ▼     ▼              (if pre-      merge
               MERGE   HASH JOIN        sorted by     (slow!)
               JOIN    (last resort)    index scan)

  Cost summary:
  ┌──────────────┬────────────────────┬────────────────────────────────┐
  │ Algorithm    │ Complexity         │ Memory                         │
  ├──────────────┼────────────────────┼────────────────────────────────┤
  │ Nested Loop  │ O(nA × log nB)     │ O(1) — no materialization      │
  │              │ with index         │                                │
  │ Hash Join    │ O(nA + nB)         │ O(nB) — inner in hash table    │
  │              │                    │ (spills to disk if > work_mem) │
  │ Merge Join   │ O(nA + nB)         │ O(sort buffer) if not pre-sort │
  │              │ if pre-sorted      │ O(1) if both index-scanned     │
  └──────────────┴────────────────────┴────────────────────────────────┘
```

---

## Diagram 3: EXPLAIN ANALYZE — Reading the Output

```
EXPLAIN (ANALYZE, BUFFERS) SELECT c.name, o.total
FROM customers c JOIN orders o ON c.id = o.customer_id
WHERE c.region = 'EU' AND o.total > 500;

Output:
┌─────────────────────────────────────────────────────────────────────────┐
│ Hash Join                                                               │
│   (cost=2834.00..12483.25 rows=4500 width=48)            ← ESTIMATED   │
│   (actual time=45.234..892.456 rows=3821 loops=1)        ← ACTUAL      │
│   Hash Cond: (o.customer_id = c.id)                                     │
│   Buffers: shared hit=1423 read=8934                     ← I/O stats   │
│                                                                         │
│   → Seq Scan on orders o                                               │
│     (cost=0.00..9823.00 rows=90000)                                     │
│     (actual time=0.012..423.123 rows=87432 loops=1)                     │
│     Filter: (total > 500)                                               │
│     Rows Removed by Filter: 412568                       ← WASTE       │
│     Buffers: shared hit=823 read=8102                                   │
│                                                                         │
│   → Hash on customers c                                                 │
│     (cost=2584.00..2584.00 rows=20000)                                  │
│     (actual time=44.892..44.892 rows=18432 loops=1)                     │
│     Buckets: 32768  Batches: 1  Memory Usage: 2048kB     ← GOOD        │
│     → Seq Scan on customers c                                           │
│       (cost=0.00..2484.00 rows=20000)                                   │
│       (actual time=0.008..31.234 rows=18432 loops=1)                    │
│       Filter: (region = 'EU')                                           │
│       Rows Removed by Filter: 81568                                     │
│       Buffers: shared hit=600 read=832                                  │
│                                                                         │
│ Planning Time: 3.456 ms                                                 │
│ Execution Time: 892.789 ms                                              │
└─────────────────────────────────────────────────────────────────────────┘

  Diagnostic interpretation:
  ┌──────────────────────────────────┬────────────────────────────────────┐
  │ Signal                           │ Problem / Action                   │
  ├──────────────────────────────────┼────────────────────────────────────┤
  │ Rows Removed by Filter: 412568   │ 82% of orders scanned wasted.      │
  │ (on orders Seq Scan)             │ Add index on orders.total          │
  ├──────────────────────────────────┼────────────────────────────────────┤
  │ shared read=8934                 │ ~70MB of disk I/O on this query.   │
  │                                  │ orders not cached in shared_buffers│
  ├──────────────────────────────────┼────────────────────────────────────┤
  │ Batches: 1, Memory: 2048kB       │ Hash table fits in work_mem. Good. │
  │                                  │ If Batches > 1 → work_mem too low  │
  ├──────────────────────────────────┼────────────────────────────────────┤
  │ estimated rows=4500 actual=3821  │ Reasonable estimate (within 15%)   │
  │                                  │ Statistics are reasonably fresh    │
  └──────────────────────────────────┴────────────────────────────────────┘
```

---

## Diagram 4: PgBouncer Connection Pooling Architecture

```
  Without PgBouncer:
  ─────────────────
  1000 app threads ──────────────────────────────► 1000 PostgreSQL backends
  (each occupies 1 backend regardless of activity)    (each = OS process, ~8MB RAM)
                                                      Total: ~8GB RAM + lock contention

  With PgBouncer (transaction mode):
  ──────────────────────────────────

  App Layer                PgBouncer              PostgreSQL
  ─────────────           ──────────             ────────────
  App1 ─────────────────► │        │◄────────── Backend #1 (active)
  App2 ─────────────────► │  Pool  │◄────────── Backend #2 (active)
  App3 ──(waiting)──────► │ Queue  │◄────────── Backend #3 (active)
  App4 ──(waiting)──────► │        │            Backend #4 (idle, reusable)
  App5 ──(waiting)──────► │  max   │            Backend #5 (idle, reusable)
  ...                     │client  │
  App1000 ──(waiting)───► │ =5000  │
                          │        │
                          │  pool  │
                          │ size   │
                          │  =25   │
                          └────────┘

  Timeline (transaction mode):
  ─────────────────────────────
  t=0ms:   App1 requests connection → gets Backend #1
  t=5ms:   App1 sends COMMIT → Backend #1 returned to pool
  t=5ms:   App2 (was waiting) → gets Backend #1 immediately
  t=8ms:   App2 sends COMMIT → Backend #1 returned to pool
  ...
  Backend #1 serves hundreds of apps per second, idle only microseconds between

  Pool mode comparison:
  ┌──────────────┬────────────────────────┬──────────────────────────────────┐
  │ Mode         │ Connection returned    │ Restrictions                     │
  ├──────────────┼────────────────────────┼──────────────────────────────────┤
  │ session      │ When client disconnects│ None — full session semantics    │
  │ transaction  │ After each COMMIT/ROLLBACK│ No session-level state        │
  │ statement    │ After each statement   │ No multi-statement transactions  │
  └──────────────┴────────────────────────┴──────────────────────────────────┘
```

---

## Diagram 5: Partition Pruning at Plan Time vs Runtime

```
  Table: events, partitioned by RANGE on occurred_at
  ┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
  │ events_2024_q1  │ events_2024_q2  │ events_2024_q3  │ events_2024_q4  │
  │ Jan–Mar 2024    │ Apr–Jun 2024    │ Jul–Sep 2024    │ Oct–Dec 2024    │
  │ 50M rows        │ 45M rows        │ 48M rows        │ 52M rows        │
  └─────────────────┴─────────────────┴─────────────────┴─────────────────┘

  Query 1: Static literal → COMPILE-TIME pruning
  ─────────────────────────────────────────────────
  WHERE occurred_at BETWEEN '2024-04-01' AND '2024-06-30'

  Planner sees:
  Append
  ├── Seq Scan on events_2024_q1  ← PRUNED (Jan–Mar, no overlap)
  ├── Seq Scan on events_2024_q2  ← SCANNED ✓ (Apr–Jun matches)
  ├── Seq Scan on events_2024_q3  ← PRUNED (Jul–Sep, no overlap)
  └── Seq Scan on events_2024_q4  ← PRUNED (Oct–Dec, no overlap)

  Only 45M rows scanned instead of 195M  → 4x improvement

  Query 2: Parameterized query → RUNTIME pruning
  ──────────────────────────────────────────────────
  WHERE occurred_at BETWEEN $1 AND $2

  Plan:
  Append
  ├── Seq Scan on events_2024_q1  ← checked at execution time
  ├── Seq Scan on events_2024_q2  ← checked at execution time
  ├── Seq Scan on events_2024_q3  ← checked at execution time
  └── Seq Scan on events_2024_q4  ← checked at execution time
  (runtime pruning eliminates non-matching partitions before scanning)

  Query 3: Function on key → PRUNING FAILS
  ──────────────────────────────────────────
  WHERE date_trunc('quarter', occurred_at) = '2024-04-01'

  Planner CANNOT determine which partition contains matching rows
  (the function might produce any value from any partition)

  Append
  ├── Seq Scan on events_2024_q1  ← SCANNED (can't prune)
  ├── Seq Scan on events_2024_q2  ← SCANNED (can't prune)
  ├── Seq Scan on events_2024_q3  ← SCANNED (can't prune)
  └── Seq Scan on events_2024_q4  ← SCANNED (can't prune)

  All 195M rows scanned → NO benefit from partitioning
```

---

## Diagram 6: Bitmap Index Scan — Combining Multiple Indexes

```
  Query: SELECT * FROM products WHERE price < 50 AND category = 'books'
  Available indexes: idx_price (on price), idx_category (on category)

  Standard approach: use only ONE index
  ──────────────────────────────────────
  Option A: Use idx_price alone → fetch 200K rows, filter by category (expensive if category rare)
  Option B: Use idx_category alone → fetch 50K rows, filter by price

  Bitmap scan approach: combine BOTH indexes
  ──────────────────────────────────────────

  Step 1: Bitmap Index Scan on idx_price (price < 50)
  ┌─────────────────────────────────────────────────────┐
  │ Bitmap (one bit per heap page):                     │
  │ Page 0:  1  (has rows with price < 50)              │
  │ Page 1:  0                                          │
  │ Page 2:  1                                          │
  │ Page 3:  1                                          │
  │ Page 4:  0                                          │
  │ ...                                                 │
  └─────────────────────────────────────────────────────┘

  Step 2: Bitmap Index Scan on idx_category (category = 'books')
  ┌─────────────────────────────────────────────────────┐
  │ Page 0:  1                                          │
  │ Page 1:  1                                          │
  │ Page 2:  0                                          │
  │ Page 3:  1                                          │
  │ Page 4:  0                                          │
  └─────────────────────────────────────────────────────┘

  Step 3: BitmapAnd (combine bitmaps with AND)
  ┌─────────────────────────────────────────────────────┐
  │ Page 0:  1 AND 1 = 1  ← fetch this page             │
  │ Page 1:  0 AND 1 = 0  ← skip                        │
  │ Page 2:  1 AND 0 = 0  ← skip                        │
  │ Page 3:  1 AND 1 = 1  ← fetch this page             │
  │ Page 4:  0 AND 0 = 0  ← skip                        │
  └─────────────────────────────────────────────────────┘

  Step 4: Bitmap Heap Scan — fetch only pages 0 and 3
  Recheck condition on actual tuples (because bitmap is page-level, not tuple-level)

  Memory note: If bitmap grows beyond work_mem,
  it becomes "lossy" (stores pages rather than exact tuple locations)
  → more heap pages fetched but correctness maintained
```
