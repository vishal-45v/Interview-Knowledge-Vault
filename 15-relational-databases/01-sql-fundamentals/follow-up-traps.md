# Chapter 01 — SQL Fundamentals: Follow-Up Traps

Tricky follow-up questions that separate candidates who truly understand SQL
semantics from those who just memorize syntax.

---

## Trap 1: "Is SELECT DISTINCT the same as GROUP BY on all columns?"

**What most people say:** Yes, they produce the same result — both eliminate
duplicate rows.

**Correct answer:** They are semantically equivalent for deduplication when no
aggregates are involved, but they are NOT the same in execution or purpose.
`GROUP BY` enables aggregation; `DISTINCT` does not. More importantly, when
you add ORDER BY, the behavior diverges. Consider:

```sql
-- This works: ordering by a selected column
SELECT DISTINCT department, salary
FROM employees
ORDER BY department;

-- This fails: cannot ORDER BY a non-selected column with DISTINCT
SELECT DISTINCT department
FROM employees
ORDER BY salary;  -- ERROR in strict SQL: salary not in SELECT list
```

Also, `GROUP BY` can be used with HAVING to filter groups; `DISTINCT` cannot.
Use `DISTINCT` only for deduplication; use `GROUP BY` for aggregation.

---

## Trap 2: "What does NOT IN return when the subquery contains a NULL?"

**What most people say:** It filters out rows that match — just like NOT IN
without NULLs.

**Correct answer:** It returns zero rows (or behaves unexpectedly). This is one
of the most dangerous NULL traps in SQL.

```sql
-- Suppose status_blacklist contains (NULL, 'banned')
SELECT * FROM users
WHERE status NOT IN (SELECT status FROM status_blacklist);
-- Returns ZERO rows even for perfectly valid users!
```

Why: `NOT IN` expands to `status != NULL AND status != 'banned'`. Any comparison
with NULL returns UNKNOWN, and UNKNOWN in a WHERE clause is treated as FALSE.
The fix: use NOT EXISTS or filter NULLs explicitly.

```sql
-- Safe alternative
SELECT * FROM users u
WHERE NOT EXISTS (
    SELECT 1 FROM status_blacklist sb
    WHERE sb.status = u.status
);
```

---

## Trap 3: "You said LEFT JOIN returns all rows from the left table. So if I
filter WHERE right_table.column = 'X', I still get all left rows, right?"

**What most people say:** Yes, the LEFT JOIN guarantees the left table rows.

**Correct answer:** Wrong. A WHERE condition on the right table's column converts
the LEFT JOIN into an effective INNER JOIN, because NULL (produced for non-matching
left rows) does not satisfy `= 'X'`, so those rows are filtered out.

```sql
-- This is NOT a true LEFT JOIN behavior:
SELECT u.id, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.amount > 100;   -- Filters out users with no orders!

-- Correct: move the filter into the JOIN condition
SELECT u.id, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.amount > 100;
```

The second query returns all users; non-matching orders columns are NULL.

---

## Trap 4: "COUNT(*) and COUNT(column) both count rows, right?"

**What most people say:** Yes, they both count the number of rows.

**Correct answer:** `COUNT(*)` counts every row including NULLs. `COUNT(column)`
counts only rows where that column is NOT NULL. This difference is critical.

```sql
SELECT
    COUNT(*)              AS total_rows,         -- e.g., 1000
    COUNT(email)          AS rows_with_email,    -- e.g., 847
    COUNT(DISTINCT email) AS unique_emails       -- e.g., 832
FROM users;
```

Also: `COUNT(DISTINCT column)` counts distinct non-NULL values. Aggregates like
SUM and AVG also silently ignore NULLs — AVG divides by the count of non-NULL
values, not total rows, which can produce misleading averages.

---

## Trap 5: "UNION removes duplicates, so it's safer than UNION ALL — you should
always use UNION."

**What most people say:** Yes, UNION is safer because it deduplicates.

**Correct answer:** UNION performs a sort or hash-based deduplication across the
entire result set, which is expensive. Unless you genuinely need deduplication,
UNION ALL is faster and correct. Using UNION when you don't need deduplication
is a performance anti-pattern that surprises engineers when they see it on large
result sets.

```sql
-- Anti-pattern: UNION where UNION ALL is correct
-- (These two sets should not produce duplicates by design)
SELECT id FROM active_users
UNION
SELECT id FROM archived_users;

-- Better: if you know no user can be in both tables
SELECT id FROM active_users
UNION ALL
SELECT id FROM archived_users;
```

Also: UNION enforces column count and type compatibility, but UNION ALL does too.
The deduplication is the only semantic difference.

---

## Trap 6: "Can you use a column alias in the WHERE clause?"

**What most people say:** Yes, just define it in SELECT and reference it in WHERE.

**Correct answer:** No — you cannot reference a SELECT alias in the WHERE clause.
This is because WHERE is evaluated before SELECT in the logical query processing
order. The alias does not exist yet when WHERE runs.

```sql
-- This FAILS:
SELECT salary * 1.1 AS adjusted_salary
FROM employees
WHERE adjusted_salary > 50000;  -- ERROR: column "adjusted_salary" does not exist

-- Solutions:
-- 1. Repeat the expression
WHERE salary * 1.1 > 50000;

-- 2. Use a subquery or CTE
WITH calc AS (
    SELECT salary * 1.1 AS adjusted_salary FROM employees
)
SELECT * FROM calc WHERE adjusted_salary > 50000;
```

Note: some databases (MySQL) DO allow alias references in HAVING and ORDER BY
as an extension, but not in WHERE.

---

## Trap 7: "Does ORDER BY guarantee a stable sort for equal values?"

**What most people say:** Yes, ORDER BY sorts rows so they come back in the same
order every time.

**Correct answer:** No. When rows have identical values in all ORDER BY columns,
their relative order is non-deterministic and can change between executions or
even between pages of the same result. This causes "flickering" in paginated UIs.

```sql
-- Unstable: rows with same price can appear in any relative order
SELECT id, name, price FROM products ORDER BY price;

-- Stable: add a unique tiebreaker
SELECT id, name, price FROM products ORDER BY price, id;
```

This is especially important for keyset pagination where the next page cursor
is built from the last row of the previous page.

---

## Trap 8: "Is HAVING just WHERE for aggregate results?"

**What most people say:** Yes, HAVING filters after aggregation, just like WHERE
filters rows.

**Correct answer:** Mostly correct, but there is an important nuance: HAVING can
reference non-aggregated columns that are not in the GROUP BY — in some databases.
More importantly, HAVING without GROUP BY applies the filter to the entire table
treated as one group:

```sql
-- No GROUP BY: HAVING applies to the whole table as one group
SELECT SUM(amount) FROM orders HAVING SUM(amount) > 1000000;
-- Returns the sum only if total exceeds 1M, otherwise returns nothing.
```

Also: a WHERE on a non-aggregated column can always be moved before GROUP BY and
is more efficient (filters rows before grouping). HAVING should only contain
conditions that genuinely depend on aggregate values.

---

## Trap 9: "Self JOIN is just a regular JOIN where the table happens to be the same.
It works the same way."

**What most people say:** Yes, you just alias the table twice and write normal
join conditions.

**Correct answer:** Correct mechanically, but there is a critical trap: the join
condition. Without a proper inequality or asymmetric condition, you get unintended
rows including self-matches and duplicates.

```sql
-- Find all pairs of employees in the same department
-- Trap: this returns (Alice, Bob) AND (Bob, Alice) AND (Alice, Alice)
SELECT a.name, b.name
FROM employees a
JOIN employees b ON a.department_id = b.department_id;

-- Correct: use < to avoid mirrored pairs and self-matches
SELECT a.name, b.name
FROM employees a
JOIN employees b
  ON a.department_id = b.department_id
 AND a.id < b.id;
```

---

## Trap 10: "NULL = NULL is TRUE, right?"

**What most people say:** Yes, NULL equals NULL.

**Correct answer:** No. NULL = NULL evaluates to UNKNOWN, not TRUE. This is
three-valued logic: TRUE, FALSE, UNKNOWN. You must use IS NULL or IS NOT NULL.

```sql
-- This returns ZERO rows even if both columns are NULL:
SELECT * FROM t WHERE col1 = col2;  -- NULL = NULL → UNKNOWN → not selected

-- Correct way to match nulls:
SELECT * FROM t WHERE col1 IS NOT DISTINCT FROM col2;
-- IS NOT DISTINCT FROM treats NULL = NULL as TRUE
```

This matters heavily in JOIN conditions: rows where the join key is NULL on
both sides do NOT match in a standard JOIN.

---

## Trap 11: "LIMIT 10 OFFSET 10000 is fine for pagination — it just skips rows."

**What most people say:** It's a bit slow but fine, the database just skips
the first 10,000 rows.

**Correct answer:** The database cannot skip rows without reading them. It scans
through all 10,000 rows, discards them, and returns the next 10. At OFFSET 1,000,000
this becomes catastrophically slow. Also, if rows are inserted or deleted between
pages, the result set shifts, causing rows to appear twice or be skipped.

```sql
-- Anti-pattern: deep offset pagination
SELECT * FROM orders ORDER BY created_at LIMIT 20 OFFSET 500000;

-- Keyset pagination: use the last seen value as the cursor
SELECT * FROM orders
WHERE created_at < :last_seen_created_at
   OR (created_at = :last_seen_created_at AND id < :last_seen_id)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

Keyset pagination is O(log n) with a proper index, not O(n).

---

## Trap 12: "I can use an aggregate function directly in a WHERE clause if I
wrap it in a subquery."

**What most people say:** That's how you do it — subquery in WHERE with the aggregate.

**Correct answer:** Technically correct but often the wrong tool. Correlated
subqueries in WHERE that recalculate an aggregate for every row can be O(n^2).
A CTE or derived table that calculates the aggregate once is far better:

```sql
-- Potentially O(n^2): recalculates avg for every row
SELECT name, salary
FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees WHERE department = e.department);

-- Better: calculate once per department
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_salary
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary
FROM employees e
JOIN dept_avg d ON e.department = d.department
WHERE e.salary > d.avg_salary;
```
