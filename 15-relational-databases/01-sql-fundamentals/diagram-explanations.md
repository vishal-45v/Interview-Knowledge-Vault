# Chapter 01 — SQL Fundamentals: Diagram Explanations

Visual representations of SQL internals, execution flow, and set operations.

---

## Diagram 1: Logical SQL Execution Order

Shows the pipeline through which a SELECT query passes from text to result set.

```
 Written SQL                        Logical Execution Order
 ──────────────────                 ───────────────────────────────────────────
 SELECT col, agg()       Step 6 ◄── SELECT  (project columns, create aliases)
 FROM   table_a          Step 1 ◄── FROM    (identify base tables)
 JOIN   table_b          Step 2 ◄── JOIN    (combine tables on condition)
 WHERE  condition        Step 3 ◄── WHERE   (filter individual rows)
 GROUP BY col            Step 4 ◄── GROUP BY (form groups of rows)
 HAVING agg() > val      Step 5 ◄── HAVING  (filter groups by aggregate)
 ORDER BY col            Step 7 ◄── ORDER BY (sort rows — aliases usable here)
 LIMIT  n                Step 8 ◄── LIMIT   (truncate to requested count)

                         ┌─────────────────────────────────────────────────┐
                         │  WHERE: alias "adjusted_salary" does NOT exist  │
                         │  ORDER BY: alias "adjusted_salary" DOES exist   │
                         └─────────────────────────────────────────────────┘
```

**Key rule:** Each step receives output from the previous step only. An alias
created in step 6 (SELECT) cannot be seen by steps 1–5 (WHERE, GROUP BY, HAVING),
but IS visible to step 7 (ORDER BY).

---

## Diagram 2: JOIN Types — Venn Diagram Representation

```
  employees table (E)          departments table (D)
  ┌──────────────────┐         ┌──────────────────┐
  │ id │ name │ d_id │         │ id │ dept_name    │
  ├────┼───────┼──────┤         ├────┼─────────────┤
  │  1 │ Alice │    1 │         │  1 │ Engineering  │
  │  2 │ Bob   │    2 │         │  2 │ Marketing    │
  │  3 │ Carol │ NULL │         │  3 │ HR           │
  └──────────────────┘         └──────────────────┘

  INNER JOIN (E.d_id = D.id)      LEFT JOIN (E.d_id = D.id)
  ┌─────────┬─────────────┐       ┌─────────┬─────────────┐
  │ name    │ dept_name   │       │ name    │ dept_name   │
  ├─────────┼─────────────┤       ├─────────┼─────────────┤
  │ Alice   │ Engineering │       │ Alice   │ Engineering │
  │ Bob     │ Marketing   │       │ Bob     │ Marketing   │
  └─────────┴─────────────┘       │ Carol   │ NULL        │
  Carol excluded (no dept)        └─────────┴─────────────┘
  HR excluded (no employees)      HR still excluded

  FULL OUTER JOIN                 RIGHT JOIN (D on right)
  ┌─────────┬─────────────┐       ┌─────────┬─────────────┐
  │ name    │ dept_name   │       │ name    │ dept_name   │
  ├─────────┼─────────────┤       ├─────────┼─────────────┤
  │ Alice   │ Engineering │       │ Alice   │ Engineering │
  │ Bob     │ Marketing   │       │ Bob     │ Marketing   │
  │ Carol   │ NULL        │       │ NULL    │ HR          │
  │ NULL    │ HR          │       └─────────┴─────────────┘
  └─────────┴─────────────┘

          E only   Both   D only
          ┌──────┬──────┬──────┐
  LEFT  → │  ██  │  ██  │      │
  INNER → │      │  ██  │      │
  RIGHT → │      │  ██  │  ██  │
  FULL  → │  ██  │  ██  │  ██  │
          └──────┴──────┴──────┘
```

---

## Diagram 3: NULL Three-Valued Logic Truth Tables

```
  AND  │ TRUE  │ FALSE │ UNKNOWN      OR   │ TRUE  │ FALSE │ UNKNOWN
  ─────┼───────┼───────┼────────      ─────┼───────┼───────┼────────
  TRUE │ TRUE  │ FALSE │ UNKNOWN      TRUE │ TRUE  │ TRUE  │ TRUE
  FALS │ FALSE │ FALSE │ FALSE        FALS │ TRUE  │ FALSE │ UNKNOWN
  UNK  │ UNKNOWN│ FALSE│ UNKNOWN      UNK  │ TRUE  │UNKNOWN│ UNKNOWN

  NOT
  ─────────────────────
  NOT TRUE    → FALSE
  NOT FALSE   → TRUE
  NOT UNKNOWN → UNKNOWN   ← critical! negation does not resolve UNKNOWN

  WHERE clause filter rule:
  ┌──────────────────────────────────────────────────────┐
  │  Only rows where the condition evaluates to TRUE     │
  │  pass through. FALSE and UNKNOWN are both rejected.  │
  └──────────────────────────────────────────────────────┘

  NOT IN trap with NULL:
  id NOT IN (1, 2, NULL)
  → id <> 1 AND id <> 2 AND id <> NULL
  → TRUE   AND TRUE   AND UNKNOWN
  → UNKNOWN
  → Row REJECTED  (even though id = 5 is clearly not 1 or 2)
```

---

## Diagram 4: Correlated Subquery vs JOIN Execution Model

```
  CORRELATED SUBQUERY (slow for large tables)
  ────────────────────────────────────────────
  Outer table scan: 10,000 employee rows
  │
  ├── Row 1: emp.dept = 'Engineering'
  │         └── Inner query: SELECT AVG(salary) WHERE dept='Engineering' ──► avg₁
  │
  ├── Row 2: emp.dept = 'Marketing'
  │         └── Inner query: SELECT AVG(salary) WHERE dept='Marketing'   ──► avg₂
  │
  ├── Row 3: emp.dept = 'Engineering'
  │         └── Inner query: SELECT AVG(salary) WHERE dept='Engineering' ──► avg₁ (again!)
  │
  └── ... repeated 10,000 times → 10,000 subquery executions

  JOIN WITH CTE (fast)
  ────────────────────
  Step 1: Aggregate once
  ┌────────────────────────────────────┐
  │  dept_avg CTE                      │
  │  Engineering → avg₁  (computed 1x)│
  │  Marketing   → avg₂  (computed 1x)│
  │  HR          → avg₃  (computed 1x)│
  └────────────────────────────────────┘
            │
            ▼ hash join (O(n))
  Step 2: Join all 10,000 employees to their dept avg in one pass
  Each row: employee.salary > dept_avg.avg → filter → result

  Total subquery executions: 1 vs 10,000
```

---

## Diagram 5: Set Operations — UNION, INTERSECT, EXCEPT

```
  Set A: {1, 2, 3, 4, 5}      Set B: {3, 4, 5, 6, 7}

  UNION ALL (keeps all, no dedup):
  ┌─────────────────────────────┐
  │ 1  2  3  4  5  3  4  5  6  7│  ← all 10 rows
  └─────────────────────────────┘

  UNION (dedup, like DISTINCT on combined):
  ┌─────────────────────────┐
  │ 1  2  3  4  5  6  7    │  ← 7 rows
  └─────────────────────────┘

  INTERSECT (only in both):
        ┌─────────┐
        │ 3  4  5 │  ← 3 rows
        └─────────┘

  EXCEPT / A MINUS B (in A but not in B):
  ┌──────────┐
  │ 1  2     │  ← 2 rows
  └──────────┘

  Visual:
  A ┌────────────────────────────────┐ B
    │  1  2  │  3  4  5  │  6  7    │
    │ (A only)│ (A ∩ B)  │ (B only) │
    └────────────────────────────────┘
     EXCEPT   INTERSECT    (B EXCEPT A)
     ◄──────────────────────────────►
                 UNION

  SQL behavior: UNION/INTERSECT/EXCEPT remove duplicates from each input set
  before operating, similar to applying DISTINCT to each SELECT separately.
  UNION ALL is the only operator that preserves duplicates.
```

---

## Diagram 6: Pagination — OFFSET vs Keyset

```
  OFFSET PAGINATION (gets slower as page number grows)
  ──────────────────────────────────────────────────────
  Table: orders (10 million rows, sorted by created_at)

  Page 1: LIMIT 20 OFFSET 0        → reads rows 1–20,    returns 20
  Page 2: LIMIT 20 OFFSET 20       → reads rows 1–40,    returns rows 21–40
  Page 3: LIMIT 20 OFFSET 40       → reads rows 1–60,    returns rows 41–60
  ...
  Page 50000: LIMIT 20 OFFSET 999980
             → reads 1,000,000 rows, discards 999,980 → returns 20
             ↑ full table scan for deep pages

  ┌──────────────────────────────────────────────────────┐
  │  Cost grows linearly: O(offset + limit)              │
  │  Also: rows shift if inserts/deletes happen between  │
  │  page requests → rows skipped or duplicated          │
  └──────────────────────────────────────────────────────┘

  KEYSET PAGINATION (constant cost regardless of depth)
  ──────────────────────────────────────────────────────
  Client stores: last_seen = (created_at='2024-01-15', id=9821)

  Next page query:
  SELECT * FROM orders
  WHERE (created_at, id) < ('2024-01-15', 9821)
  ORDER BY created_at DESC, id DESC
  LIMIT 20;

  ┌──────────────────────────────────────────────────────┐
  │  Index seek → O(log n) to find starting point       │
  │  Reads exactly 20 rows. Same cost for page 1 or     │
  │  page 50,000.                                       │
  │  Robust to concurrent inserts/deletes.              │
  │  Limitation: cannot jump to arbitrary page number.  │
  └──────────────────────────────────────────────────────┘

  Required index for keyset:
  CREATE INDEX ON orders (created_at DESC, id DESC);
```
