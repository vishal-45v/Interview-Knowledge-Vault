# Chapter 03 — Indexes and Performance: Diagram Explanations

Visual representations of B-tree structure, scan types, query plan anatomy,
and index type comparisons.

---

## Diagram 1: B-tree Index Structure

```
  B-tree index on orders.customer_id
  (Simplified: branching factor reduced for clarity)

  Level 0 (Root):
  ┌────────────────────────────────────────┐
  │  [ 1000 | 5000 | 10000 | 50000 ]      │
  └───┬────────┬────────┬────────┬─────────┘
      │        │        │        │
  Level 1 (Internal nodes):
  ┌───┴──┐  ┌──┴───┐  ┌──┴───┐  ┌──┴──────┐
  │ 100  │  │ 2000 │  │ 7500 │  │ 25000   │
  │ 500  │  │ 3000 │  │ 8000 │  │ 30000   │
  └──┬───┘  └──┬───┘  └──┬───┘  └──┬──────┘
     │          │          │          │
  Level 2 (Leaf nodes — contain actual TID pointers):
  ┌──────────────────────────────────────────────────────────────────────┐
  │  [42 → (pg88,slot3)] │ [43 → (pg12,slot1)] │ [44 → (pg88,slot7)]   │
  │  [45 → (pg91,slot2)] │ [46 → (pg15,slot4)] │ ...                   │
  └───────────────────────────────────────────────────────────────────────
  ◄────────────────────── doubly-linked leaf chain ──────────────────────►
       (enables efficient range scans without returning to root)

  Query: WHERE customer_id = 44
  Path: Root[10000>] → Internal[<100] → Leaf[find 44] → Heap page 88 slot 7

  Query: WHERE customer_id BETWEEN 42 AND 46
  Path: Root → Internal → First leaf containing 42, then traverse links rightward
        until 46 is exceeded. No need to return to root for range.
```

---

## Diagram 2: Index Scan vs Bitmap Scan vs Sequential Scan Decision

```
  Table: orders, 10M rows, 128,000 pages

  ┌──────────────────────────────────────────────────────────────────────┐
  │  Query: WHERE customer_id = 42                                       │
  │  Estimated result: ~150 rows (0.0015% selectivity)                  │
  │                                                                      │
  │  Index Scan:                                                         │
  │  Root → Leaf → fetch 150 rows via random heap access                │
  │  Cost: ~150 random page reads ← CHOSEN: very selective              │
  └──────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────┐
  │  Query: WHERE customer_id IN (1,2,3,...,5000)                        │
  │  Estimated result: ~75,000 rows (0.75% selectivity)                 │
  │                                                                      │
  │  Bitmap Index Scan:                                                  │
  │  1. Scan index, build bitmap of all matching heap pages              │
  │     [page 1: yes, page 2: no, page 3: yes, ...]                    │
  │  2. Sort bitmap pages                                                │
  │  3. Fetch heap pages in sequential order (less random I/O)          │
  │  Cost: index scan + bitmap overhead + sorted heap reads ← CHOSEN    │
  └──────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────┐
  │  Query: WHERE status = 'active' (80% of rows)                       │
  │  Estimated result: 8,000,000 rows                                   │
  │                                                                      │
  │  Sequential Scan:                                                    │
  │  Read ALL 128,000 pages left to right, test each row                │
  │  Cost: 128,000 sequential page reads ← CHOSEN: random I/O worse    │
  │  Reason: random reads to fetch 80% of pages via index               │
  │  would require ~102,400 random reads — more than 128,000 seq reads  │
  └──────────────────────────────────────────────────────────────────────┘

  Threshold (approx): index scan → seq scan around 5-15% selectivity
  (lower for HDD due to high random I/O cost; higher for SSD)
```

---

## Diagram 3: Composite Index — Column Order Impact

```
  Table: orders (customer_id, status, created_at, amount)

  INDEX A: (customer_id, status, created_at)  ← optimal for typical queries
  INDEX B: (status, customer_id, created_at)  ← wrong for most queries

  Query 1: WHERE customer_id = 42 AND status = 'pending'
  INDEX A: ██████████ seeks directly to (42, 'pending') leaf ← used
  INDEX B: full status scan, then customer_id filter within ← partial use

  Query 2: WHERE customer_id = 42
  INDEX A: ██████████ seeks to customer_id=42 subtree ← used
  INDEX B: must scan entire index for customer_id=42 ← NOT used (leftmost is status)

  Query 3: WHERE status = 'pending'
  INDEX A: CANNOT use (status is not the leftmost column) ← NOT used
  INDEX B: ██████████ seeks directly to status='pending' ← used

  Composite key sort order visualization (INDEX A):
  ┌──────────────────────────────────────────────────────────────────┐
  │ customer_id=1, status='cancelled', created_at=2022-01-01         │
  │ customer_id=1, status='cancelled', created_at=2022-03-15         │
  │ customer_id=1, status='pending',   created_at=2024-01-01         │
  │ customer_id=2, status='active',    created_at=2023-05-01         │
  │ customer_id=42, status='pending',  created_at=2024-01-10  ◄─ seek│
  │ customer_id=42, status='pending',  created_at=2024-02-20         │
  │ customer_id=42, status='shipped',  created_at=2024-01-05         │
  │ customer_id=100, ...                                             │
  └──────────────────────────────────────────────────────────────────┘
  The range WHERE customer_id=42 AND status='pending' is a contiguous
  block in this sorted structure → efficient range scan
```

---

## Diagram 4: EXPLAIN ANALYZE Plan Tree Anatomy

```
  Query: SELECT u.name, COUNT(o.id) FROM users u JOIN orders o ON u.id = o.user_id
         WHERE u.created_at > '2024-01-01' GROUP BY u.name ORDER BY 2 DESC LIMIT 10

  Plan tree (read bottom-up: leaves execute first):

  Limit (cost=45000..45000 rows=10)  ← Step 6: take 10 rows
  └── Sort (cost=44800..45000)        ← Step 5: sort by count desc
      └── HashAggregate               ← Step 4: GROUP BY u.name
          └── Hash Join               ← Step 3: join users and orders
              ├── [Build side] Hash
              │   └── Seq Scan on users               ← Step 1: scan users
              │       Filter: (created_at > '2024-01-01')  ← WHERE applied here
              │       Rows removed by filter: 1800000
              └── [Probe side] Seq Scan on orders      ← Step 2: scan orders

  Reading costs:
  ┌──────────────────────────────────────────────────────────────────┐
  │ cost=0.00..45000.00:                                             │
  │   LEFT value = startup cost (cost to first row)                  │
  │   RIGHT value = total cost (cost to last row)                    │
  │   Units: abstract planner cost units (seq_page_cost = 1.0)      │
  │                                                                  │
  │ actual time=0.5..8432.1:                                         │
  │   LEFT = ms to first row, RIGHT = ms to last row                 │
  │                                                                  │
  │ rows=450 vs actual rows=4500:                                    │
  │   10x underestimate → planner chose wrong join strategy          │
  │   → Run ANALYZE or increase statistics target                    │
  │                                                                  │
  │ loops=50000:                                                     │
  │   Node executed 50,000 times (nested loop inner side)           │
  │   Total actual rows = actual_rows_per_loop × loops              │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Diagram 5: Index Types Comparison

```
  ┌───────────────┬────────────┬────────────┬────────────┬────────────┐
  │ Index Type    │ B-tree     │ Hash       │ GIN        │ BRIN       │
  ├───────────────┼────────────┼────────────┼────────────┼────────────┤
  │ Best for      │ Everything │ = equality │ Arrays,    │ Time-series│
  │               │            │ only       │ JSONB, FTS │ append-only│
  ├───────────────┼────────────┼────────────┼────────────┼────────────┤
  │ Supports      │ =,<,>,     │ = only     │ @>,?,&&    │ =,<,>,     │
  │ operators     │ LIKE 'x%', │            │ containment│ BETWEEN    │
  │               │ BETWEEN    │            │            │            │
  ├───────────────┼────────────┼────────────┼────────────┼────────────┤
  │ Supports      │ YES        │ NO         │ NO         │ NO         │
  │ ORDER BY      │            │            │            │            │
  ├───────────────┼────────────┼────────────┼────────────┼────────────┤
  │ Size (approx) │ Medium     │ Medium     │ Large      │ Tiny       │
  │               │ ~20% table │ ~25% table │ (inverted) │ ~1% table  │
  ├───────────────┼────────────┼────────────┼────────────┼────────────┤
  │ Write speed   │ Fast       │ Fast       │ Slow       │ Very fast  │
  │               │            │            │ (batch OK) │            │
  ├───────────────┼────────────┼────────────┼────────────┼────────────┤
  │ Data must be  │ No         │ No         │ No         │ YES        │
  │ physically    │            │            │            │ correlated │
  │ correlated    │            │            │            │            │
  └───────────────┴────────────┴────────────┴────────────┴────────────┘

  BRIN structure (min/max per 128-page block range):
  ┌───────────────────────────────────────────────────────────────────┐
  │ Pages 1-128:     event_time MIN=2020-01-01  MAX=2020-01-15        │
  │ Pages 129-256:   event_time MIN=2020-01-15  MAX=2020-02-01        │
  │ Pages 257-384:   event_time MIN=2020-02-01  MAX=2020-03-01        │
  │ ...                                                               │
  │ Query: WHERE event_time BETWEEN '2020-01-20' AND '2020-01-25'     │
  │ Blocks 1-128:   range [Jan 1, Jan 15] → no overlap → SKIP        │
  │ Blocks 129-256: range [Jan 15, Feb 1] → overlap! → SCAN          │
  │ Blocks 257+:    range [Feb 1, ...] → no overlap → SKIP           │
  └───────────────────────────────────────────────────────────────────┘
  For 1TB table: B-tree index ~20GB, BRIN index ~1MB
```

---

## Diagram 6: Index Bloat Lifecycle

```
  Time →
  ┌────────────────────────────────────────────────────────────────────┐
  │ T=0: Table is freshly loaded                                       │
  │ Live rows: 1,000,000  Dead rows: 0  Table size: 5GB               │
  │ [████████████████████████████████████ live rows ████████████████] │
  └────────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────────┐
  │ T=1: 6 months of heavy UPDATEs (e-commerce, status changes)       │
  │ Live rows: 1,000,000  Dead rows: 5,000,000  Table size: 25GB      │
  │ [██ live ██████████████████████ dead (bloat) ██████████████████]  │
  └────────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────────┐
  │ T=2: VACUUM runs (autovacuum or manual)                            │
  │ Live rows: 1,000,000  Dead rows: 0  Table size: 25GB (unchanged!) │
  │ [██ live ██ REUSABLE FREE SPACE (file not shrunk) ███████████████] │
  │                     ↑                                              │
  │              Dead space marked reusable but OS sees same file size │
  └────────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────────────┐
  │ T=3: pg_repack runs (or VACUUM FULL — but pg_repack is safer)     │
  │ Live rows: 1,000,000  Dead rows: 0  Table size: 5GB               │
  │ [████████████████ live rows ████████████████] ← file shrunk       │
  └────────────────────────────────────────────────────────────────────┘

  Commands:
  VACUUM           → marks dead space reusable, no lock, no shrink
  VACUUM ANALYZE   → vacuum + update statistics
  VACUUM FULL      → rewrites table, shrinks file, EXCLUSIVE LOCK (risky!)
  REINDEX CONCURRENTLY → rebuilds index without blocking reads/writes
  pg_repack        → rewrites table online, only brief lock at swap
```
