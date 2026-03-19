# Chapter 01 — SQL Fundamentals: Structured Answers

Complete, interview-ready answers for the most commonly asked SQL fundamentals
questions at the senior engineer level.

---

## Q1: Walk me through the logical order of SQL query execution.

**Answer:**

SQL is written in one order but evaluated in a different logical order. Understanding
this order explains every "why can't I use this alias here?" question.

```
Written order:         Logical evaluation order:
1. SELECT              1. FROM          (identify source tables)
2. FROM                2. JOIN / ON     (combine tables)
3. JOIN                3. WHERE         (filter rows)
4. WHERE               4. GROUP BY      (form groups)
5. GROUP BY            5. HAVING        (filter groups)
6. HAVING              6. SELECT        (compute output columns, aliases created)
7. ORDER BY            7. DISTINCT      (deduplicate)
8. LIMIT/OFFSET        8. ORDER BY      (sort — aliases now available)
                       9. LIMIT/OFFSET  (paginate)
```

**Consequences:**
- You cannot reference a SELECT alias in WHERE (alias doesn't exist yet).
- You CAN reference a SELECT alias in ORDER BY (evaluated after SELECT).
- HAVING filters after GROUP BY, so it can use aggregate results.
- Window functions run after GROUP BY/HAVING but before ORDER BY.

```sql
-- Demonstrates the order: this works
SELECT
    department,
    AVG(salary) AS avg_sal
FROM employees
WHERE hire_date > '2020-01-01'   -- WHERE sees raw columns, not aliases
GROUP BY department
HAVING AVG(salary) > 60000       -- HAVING can use aggregate
ORDER BY avg_sal DESC;           -- ORDER BY can use SELECT alias
```

---

## Q2: Explain EXISTS vs IN vs JOIN — when to use each, and the NULL trap.

**Answer:**

These three constructs often return identical results but have critical semantic
and performance differences.

**EXISTS:**
- Short-circuits on the first match. Does not return any data from the subquery.
- Handles NULLs safely (only cares if a row exists, not what value it has).
- Best when the subquery has high selectivity and you only need to check existence.

```sql
-- "Which customers have placed at least one order?"
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

**IN:**
- Evaluates the subquery first and builds a value list, then checks membership.
- DANGER: if the subquery returns any NULL, NOT IN returns zero rows.
- Fine for static, small value lists; for large subqueries, EXISTS is usually faster.

```sql
-- Safe: subquery guaranteed non-null
SELECT * FROM products WHERE category_id IN (1, 2, 3);

-- DANGEROUS: if any category.parent_id is NULL, this returns nothing
SELECT * FROM categories WHERE id NOT IN (
    SELECT parent_id FROM categories
);
-- Fix: filter nulls
SELECT * FROM categories WHERE id NOT IN (
    SELECT parent_id FROM categories WHERE parent_id IS NOT NULL
);
```

**JOIN:**
- Best when you also need data from the joined table.
- Can produce duplicates if the join key is not unique in the joined table — a
  common bug when using INNER JOIN as an existence check.

```sql
-- BUG: if one customer has 5 orders, they appear 5 times
SELECT c.id, c.name
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id;

-- Fix with DISTINCT or EXISTS instead
SELECT DISTINCT c.id, c.name
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id;
```

**Rule of thumb:**
- Need data from both tables → JOIN
- Checking existence only → EXISTS
- Membership in a small known list → IN
- Checking NON-membership → NOT EXISTS (never NOT IN with subquery)

---

## Q3: Explain all JOIN types with examples and when each is appropriate.

**Answer:**

```sql
-- Setup for all examples
CREATE TABLE employees (id INT, name TEXT, dept_id INT);
CREATE TABLE departments (id INT, name TEXT);
```

**INNER JOIN:** Returns only rows with a match in both tables.
```sql
SELECT e.name, d.name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;
-- Employees with no department assignment are excluded.
-- Departments with no employees are excluded.
```

**LEFT JOIN (LEFT OUTER JOIN):** All rows from left table; NULLs for non-matching right.
```sql
SELECT e.name, d.name
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;
-- All employees returned; d.name is NULL for employees with no department.
```

**RIGHT JOIN:** Mirror of LEFT JOIN. Rarely used — rewrite as LEFT JOIN with tables swapped.

**FULL OUTER JOIN:** All rows from both tables; NULLs where no match exists.
```sql
SELECT e.name, d.name
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.id;
-- Shows unmatched employees (no dept) AND unmatched departments (no employees).
```

**CROSS JOIN:** Cartesian product — every combination of rows.
```sql
-- Legitimate use: generate all size/color combinations
SELECT s.size, c.color
FROM sizes s
CROSS JOIN colors c;
-- 4 sizes × 6 colors = 24 rows
```

**SELF JOIN:** Table joined to itself. Requires aliases.
```sql
-- Find employees and their managers (both in the same table)
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

## Q4: How do NULL values behave in SQL, and what are all the functions for
handling them?

**Answer:**

NULL represents the absence of a value — it is not zero, not empty string, not
false. NULL participates in three-valued logic: TRUE, FALSE, UNKNOWN.

**Key behaviors:**

```sql
-- NULL in comparisons always yields UNKNOWN
SELECT NULL = NULL;     -- UNKNOWN (not TRUE!)
SELECT NULL != 5;       -- UNKNOWN
SELECT NULL > 0;        -- UNKNOWN

-- Use IS NULL / IS NOT NULL
SELECT * FROM t WHERE col IS NULL;
SELECT * FROM t WHERE col IS NOT NULL;

-- IS NOT DISTINCT FROM: NULL-safe equality
SELECT * FROM t WHERE col1 IS NOT DISTINCT FROM col2;
-- This treats NULL = NULL as TRUE
```

**COALESCE:** Returns the first non-NULL argument.
```sql
-- Replace NULL with a default
SELECT COALESCE(phone, email, 'no contact') AS contact FROM users;
-- Returns phone if non-null, else email if non-null, else 'no contact'
```

**NULLIF:** Returns NULL if two arguments are equal; otherwise returns first argument.
```sql
-- Prevent division by zero
SELECT total_sales / NULLIF(num_transactions, 0) AS avg_transaction
FROM daily_stats;
-- If num_transactions = 0, returns NULL instead of raising an error
```

**NULL in aggregates:**
```sql
SELECT
    COUNT(*)          AS total_rows,     -- counts all rows (no NULL skipping)
    COUNT(score)      AS scored_rows,    -- skips NULL scores
    AVG(score)        AS avg_score,      -- AVG skips NULLs: SUM(score)/COUNT(score)
    SUM(score)        AS sum_score,      -- returns NULL if ALL values are NULL
    COALESCE(SUM(score), 0) AS safe_sum  -- handle all-NULL case
FROM game_results;
```

**NULL in GROUP BY:**
```sql
-- NULLs ARE grouped together (unlike NULL = NULL comparisons)
SELECT department, COUNT(*) FROM employees GROUP BY department;
-- All employees with NULL department are placed in one group labeled NULL
```

---

## Q5: Explain correlated subqueries — execution model, performance, and when
to refactor them.

**Answer:**

A correlated subquery references a column from the outer query. This means it
cannot be evaluated once — it must be re-executed for each row of the outer query.

```sql
-- Correlated: references e.department from outer query
-- Executed once per employee row — O(n) subquery executions
SELECT e.name, e.salary
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department = e.department  -- correlation here
);
```

For a table with 10,000 employees across 20 departments, this subquery runs
10,000 times. The query is O(n) in correlated subquery executions.

**Refactor to a join:**
```sql
-- Calculates each department average exactly once — O(1) aggregation pass
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary
FROM employees e
JOIN dept_avg d ON e.department = d.department
WHERE e.salary > d.avg_sal;
```

**When correlated subqueries are acceptable:**
- Small outer result sets (e.g., checking one specific record).
- EXISTS subqueries: the optimizer often converts these to semi-joins and
  may execute them more efficiently than the naive row-by-row model suggests.
- When the database optimizer rewrites the correlated subquery internally
  (modern optimizers are smart about this for common patterns).

**Rule:** Always check EXPLAIN. If you see "SubPlan" in PostgreSQL's output for
a correlated subquery applied to a large table, it's a red flag for optimization.

---

## Q6: What is the difference between UNION, UNION ALL, INTERSECT, and EXCEPT?

**Answer:**

All four are set operations. They combine result sets from two SELECT statements
that must have the same number of columns and compatible types.

**UNION:** Combines results and removes duplicates (like DISTINCT on the combined set).
```sql
SELECT city FROM customers
UNION
SELECT city FROM suppliers;
-- Returns each distinct city once, regardless of which table it came from
```

**UNION ALL:** Combines results without removing duplicates. Always faster than UNION.
```sql
SELECT product_id, amount FROM online_sales
UNION ALL
SELECT product_id, amount FROM store_sales;
-- Returns every row; duplicates retained. Use when deduplication is not needed.
```

**INTERSECT:** Returns only rows that appear in BOTH result sets.
```sql
SELECT user_id FROM newsletter_subscribers
INTERSECT
SELECT user_id FROM paying_customers;
-- Users who are both subscribers AND paying customers
-- Note: implicitly removes duplicates within each set, like UNION
```

**EXCEPT (MINUS in Oracle):** Returns rows from the first set that do not appear
in the second set.
```sql
SELECT user_id FROM all_users
EXCEPT
SELECT user_id FROM deleted_users;
-- Active users only

-- Key difference from NOT IN: EXCEPT handles NULLs safely
-- NULL EXCEPT NULL → no rows (the NULL rows cancel out)
-- NOT IN with NULL subquery → empty result set (dangerous)
```

**Performance note:** UNION, INTERSECT, and EXCEPT all require sorting or hashing
to find duplicates. UNION ALL skips this step entirely. On large datasets the
difference can be an order of magnitude.

---

## Q7: Explain CASE expressions — simple vs searched, and advanced uses.

**Answer:**

CASE is SQL's conditional expression. It returns a value (not a boolean) and can
be used anywhere an expression is valid.

**Simple CASE:** Matches a single expression against multiple values.
```sql
SELECT
    order_id,
    CASE status
        WHEN 'P' THEN 'Pending'
        WHEN 'S' THEN 'Shipped'
        WHEN 'D' THEN 'Delivered'
        WHEN 'C' THEN 'Cancelled'
        ELSE 'Unknown'
    END AS status_label
FROM orders;
```

**Searched CASE:** Evaluates arbitrary boolean conditions.
```sql
SELECT
    name,
    salary,
    CASE
        WHEN salary < 40000  THEN 'Junior'
        WHEN salary < 80000  THEN 'Mid-level'
        WHEN salary < 130000 THEN 'Senior'
        ELSE 'Staff+'
    END AS level
FROM employees;
```

**In ORDER BY (custom sort):**
```sql
SELECT * FROM tickets
ORDER BY
    CASE priority
        WHEN 'CRITICAL' THEN 1
        WHEN 'HIGH'     THEN 2
        WHEN 'MEDIUM'   THEN 3
        ELSE                 4
    END;
```

**In GROUP BY (bucketed histogram):**
```sql
SELECT
    CASE
        WHEN age < 18 THEN 'Under 18'
        WHEN age < 30 THEN '18-29'
        WHEN age < 50 THEN '30-49'
        ELSE '50+'
    END AS age_bucket,
    COUNT(*) AS user_count
FROM users
GROUP BY 1;  -- GROUP BY the first SELECT expression
```

**In aggregate (conditional counting):**
```sql
-- Count how many orders are in each status without separate queries
SELECT
    COUNT(CASE WHEN status = 'P' THEN 1 END) AS pending,
    COUNT(CASE WHEN status = 'S' THEN 1 END) AS shipped,
    COUNT(CASE WHEN status = 'D' THEN 1 END) AS delivered
FROM orders;
-- This is the pre-FILTER clause pattern; same as COUNT(*) FILTER (WHERE status='P')
```
