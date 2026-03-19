# Chapter 05 — Advanced SQL: Diagram Explanations

Visual representations of window function frames, recursive CTE execution,
JSONB indexing, and ROLLUP output structure.

---

## Diagram 1: Window Function Frame Specification

```
  Table: sales(date, amount) — ordered by date
  Data: 2024-01-01/$100, 2024-01-02/$200, 2024-01-03/$150, 2024-01-04/$300, 2024-01-05/$250

  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW (running total):
  ┌────────────┬────────┬──────────────────────────────────────────────────┐
  │ date       │ amount │ window rows for CURRENT ROW                     │
  ├────────────┼────────┼──────────────────────────────────────────────────┤
  │ 2024-01-01 │  100   │ [100]                       SUM = 100            │
  │ 2024-01-02 │  200   │ [100, 200]                  SUM = 300            │
  │ 2024-01-03 │  150   │ [100, 200, 150]             SUM = 450            │
  │ 2024-01-04 │  300   │ [100, 200, 150, 300]        SUM = 750            │
  │ 2024-01-05 │  250   │ [100, 200, 150, 300, 250]   SUM = 1000           │
  └────────────┴────────┴──────────────────────────────────────────────────┘

  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW (3-row sliding window):
  ┌────────────┬────────┬──────────────────────────────────────────────────┐
  │ date       │ amount │ window rows for CURRENT ROW                     │
  ├────────────┼────────┼──────────────────────────────────────────────────┤
  │ 2024-01-01 │  100   │ [100]                       SUM = 100            │
  │ 2024-01-02 │  200   │ [100, 200]                  SUM = 300            │
  │ 2024-01-03 │  150   │ [100, 200, 150]             SUM = 450  ← full    │
  │ 2024-01-04 │  300   │ [200, 150, 300]             SUM = 650  ← window  │
  │ 2024-01-05 │  250   │ [150, 300, 250]             SUM = 700  ← slides  │
  └────────────┴────────┴──────────────────────────────────────────────────┘

  ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING (partition total):
  ┌────────────┬────────┬──────────────────────────────────────────────────┐
  │ date       │ amount │ window: entire partition = SUM always 1000       │
  ├────────────┼────────┼──────────────────────────────────────────────────┤
  │ 2024-01-01 │  100   │ [100,200,150,300,250]       SUM = 1000           │
  │ 2024-01-02 │  200   │ [100,200,150,300,250]       SUM = 1000           │
  │ ... (same for all rows)                                                │
  └────────────┴────────┴──────────────────────────────────────────────────┘
```

---

## Diagram 2: ROW_NUMBER vs RANK vs DENSE_RANK

```
  Table: employees ordered by salary DESC within department 'Engineering'
  Salaries: 150k, 150k, 130k, 120k, 120k, 100k

  ┌─────────┬────────┬────────────┬────────┬────────────┐
  │ name    │ salary │ ROW_NUMBER │ RANK   │ DENSE_RANK │
  ├─────────┼────────┼────────────┼────────┼────────────┤
  │ Alice   │ 150000 │     1      │   1    │     1      │
  │ Bob     │ 150000 │     2      │   1    │     1      │  ← tie at 150k
  │ Carol   │ 130000 │     3      │   3    │     2      │  ← ROW_NUMBER never ties
  │ Dave    │ 120000 │     4      │   4    │     3      │  ← RANK skips 2 (no rank 2)
  │ Eve     │ 120000 │     5      │   4    │     3      │  ← tie at 120k
  │ Frank   │ 100000 │     6      │   6    │     4      │  ← RANK skips 5
  └─────────┴────────┴────────────┴────────┴────────────┘

  ROW_NUMBER:   always unique (1,2,3,4,5,6)   - ties broken arbitrarily
  RANK:         ties share a rank, next rank is skipped (1,1,3,4,4,6)
  DENSE_RANK:   ties share a rank, no gaps (1,1,2,3,3,4)

  Use cases:
  ROW_NUMBER  → Need unique row identifiers for pagination or deduplication
  RANK        → Sports/competition rankings ("tied for 3rd place, 5th place next")
  DENSE_RANK  → Grouping by rank level ("how many distinct salary levels exist?")

  SQL:
  SELECT name, salary,
      ROW_NUMBER()  OVER (PARTITION BY dept ORDER BY salary DESC) AS row_num,
      RANK()        OVER (PARTITION BY dept ORDER BY salary DESC) AS rnk,
      DENSE_RANK()  OVER (PARTITION BY dept ORDER BY salary DESC) AS dense_rnk
  FROM employees;
```

---

## Diagram 3: Recursive CTE Execution Model

```
  Query: Find all descendants of employee #1 in the hierarchy

  WITH RECURSIVE hierarchy AS (
      SELECT id, name, manager_id, 0 AS depth FROM employees WHERE id = 1
      UNION ALL
      SELECT e.id, e.name, e.manager_id, h.depth + 1
      FROM employees e JOIN hierarchy h ON e.manager_id = h.id
  )

  Execution steps:
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 0: Anchor query executes                                           │
  │ Working table: [(1, 'CEO', NULL, 0)]                                   │
  │ Result so far: [(1, 'CEO', NULL, 0)]                                   │
  └──────────────────────────────────┬──────────────────────────────────────┘
                                     ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 1: Recursive query joins working table to employees                │
  │ Finds: manager_id = 1 → employees 2 (VP Eng), 3 (VP Mktg)             │
  │ Working table: [(2, 'VP Eng', 1, 1), (3, 'VP Mktg', 1, 1)]            │
  │ Result so far: [(1,'CEO',0), (2,'VP Eng',1), (3,'VP Mktg',1)]         │
  └──────────────────────────────────┬──────────────────────────────────────┘
                                     ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step 2: Recursive query joins new working table (depth=1 rows)          │
  │ Finds: manager_id = 2 → [4,5,6]; manager_id = 3 → [7,8]               │
  │ Working table: [(4,'Dir1',2,2),(5,'Dir2',2,2),...,(8,'Mgr2',3,2)]     │
  │ Result accumulates: 8 rows total                                        │
  └──────────────────────────────────┬──────────────────────────────────────┘
                                     ▼
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Step N: Recursive query finds no new rows (leaf employees have no      │
  │ reports → JOIN produces empty set)                                      │
  │ Working table: empty → STOP                                            │
  └─────────────────────────────────────────────────────────────────────────┘
  Final result: entire tree starting from employee #1

  Key: The working table at each step contains ONLY the newly added rows,
  not the entire accumulated result. This is why UNION ALL is used (not UNION)
  — deduplication at each step would be expensive and usually unnecessary.
```

---

## Diagram 4: JSONB GIN Index — How Containment Search Works

```
  Documents in events table (data JSONB column):
  ┌─────┬─────────────────────────────────────────────────────────────┐
  │ id  │ data                                                        │
  ├─────┼─────────────────────────────────────────────────────────────┤
  │  1  │ {"type":"click","page":"/home","user_id":42}                │
  │  2  │ {"type":"purchase","page":"/checkout","user_id":42,"amt":99}│
  │  3  │ {"type":"click","page":"/product","user_id":100}            │
  │  4  │ {"type":"view","page":"/home","user_id":100}                │
  └─────┴─────────────────────────────────────────────────────────────┘

  GIN Index internal structure (inverted index):
  ┌──────────────────────────────────┬──────────────────┐
  │ Key (JSON path + value)          │ Document IDs     │
  ├──────────────────────────────────┼──────────────────┤
  │ "type" = "click"                 │ {1, 3}           │
  │ "type" = "purchase"              │ {2}              │
  │ "type" = "view"                  │ {4}              │
  │ "page" = "/home"                 │ {1, 4}           │
  │ "page" = "/checkout"             │ {2}              │
  │ "page" = "/product"              │ {3}              │
  │ "user_id" = 42                   │ {1, 2}           │
  │ "user_id" = 100                  │ {3, 4}           │
  │ "amt" = 99                       │ {2}              │
  └──────────────────────────────────┴──────────────────┘

  Query: WHERE data @> '{"type":"click","user_id":42}'
  1. Look up "type"="click" → {1, 3}
  2. Look up "user_id"=42   → {1, 2}
  3. INTERSECT: {1, 3} ∩ {1, 2} = {1}
  4. Heap fetch row 1 → return

  Cost: two GIN lookups + set intersection + one heap fetch
  Without index: full table scan + JSONB parse for every row
```

---

## Diagram 5: ROLLUP Output Structure

```
  Query: SELECT region, product, SUM(revenue) FROM sales
         GROUP BY ROLLUP (region, product)

  ROLLUP produces N+1 grouping levels for N columns:
  - Level 0: (region, product)  — finest granularity
  - Level 1: (region)           — subtotal per region
  - Level 2: ()                 — grand total

  Sample output:
  ┌─────────┬──────────────┬─────────┬──────────────────────────────────────┐
  │ region  │ product      │ revenue │ description                          │
  ├─────────┼──────────────┼─────────┼──────────────────────────────────────┤
  │ East    │ Widget A     │  15,000 │ Level 0: East + Widget A             │
  │ East    │ Widget B     │  22,000 │ Level 0: East + Widget B             │
  │ East    │ NULL         │  37,000 │ Level 1: East subtotal (all products)│
  │ West    │ Widget A     │  18,000 │ Level 0: West + Widget A             │
  │ West    │ Widget C     │  12,000 │ Level 0: West + Widget C             │
  │ West    │ NULL         │  30,000 │ Level 1: West subtotal               │
  │ NULL    │ NULL         │  67,000 │ Level 2: Grand total                 │
  └─────────┴──────────────┴─────────┴──────────────────────────────────────┘

  ROLLUP vs CUBE row counts:
  ┌─────────────────────────────────────────────────────────────────────────┐
  │ ROLLUP(a, b, c) → N+1 = 4 grouping sets:                               │
  │   (a,b,c), (a,b), (a), ()                                              │
  │                                                                         │
  │ CUBE(a, b, c) → 2^N = 8 grouping sets:                                 │
  │   (a,b,c), (a,b), (a,c), (b,c), (a), (b), (c), ()                     │
  │                                                                         │
  │ GROUPING SETS((a,b),(a),()):                                            │
  │   Explicit: exactly the grouping sets you list                          │
  │   Equivalent to ROLLUP(a, b) in this case                               │
  └─────────────────────────────────────────────────────────────────────────┘

  GROUPING() function to distinguish rollup NULLs from data NULLs:
  SELECT
      CASE WHEN GROUPING(region) = 1 THEN 'ALL REGIONS' ELSE region END AS region,
      CASE WHEN GROUPING(product) = 1 THEN 'ALL PRODUCTS' ELSE product END AS product,
      SUM(revenue)
  FROM sales
  GROUP BY ROLLUP (region, product);
```

---

## Diagram 6: INSERT ON CONFLICT (Upsert) Flow

```
  INSERT INTO user_stats (user_id, login_count, last_login)
  VALUES ($1, 1, NOW())
  ON CONFLICT (user_id)
  DO UPDATE SET
      login_count = user_stats.login_count + EXCLUDED.login_count,
      last_login = EXCLUDED.last_login;

  Execution flow:
  ┌───────────────────────────────────────────────────────────────────────┐
  │  Attempt INSERT of row (user_id=42, login_count=1, last_login=now)    │
  └──────────────────────────────┬────────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
               No conflict              Conflict on
                    │                   user_id=42
                    ▼                         │
  ┌─────────────────────────┐                 ▼
  │ Row inserted normally   │   ┌─────────────────────────────────────┐
  │ (user_id=42 is new)     │   │ EXCLUDED pseudo-table available:    │
  └─────────────────────────┘   │   EXCLUDED.user_id    = 42          │
                                │   EXCLUDED.login_count = 1          │
                                │   EXCLUDED.last_login  = now        │
                                │                                     │
                                │ Existing row:                       │
                                │   user_stats.user_id    = 42        │
                                │   user_stats.login_count = 15       │
                                │   user_stats.last_login  = yesterday│
                                │                                     │
                                │ DO UPDATE SET:                      │
                                │   login_count = 15 + 1 = 16         │
                                │   last_login = now (EXCLUDED value)  │
                                └─────────────────────────────────────┘

  Key rules:
  ┌─────────────────────────────────────────────────────────────────────┐
  │ ON CONFLICT requires: a unique constraint or unique index           │
  │ EXCLUDED: refers to the row that was being inserted (new values)   │
  │ table_name: refers to the existing row in the table (old values)   │
  │ WHERE clause in DO UPDATE: conditional update (only if condition)  │
  │ DO NOTHING: silently ignores the conflicting row                   │
  └─────────────────────────────────────────────────────────────────────┘
```
