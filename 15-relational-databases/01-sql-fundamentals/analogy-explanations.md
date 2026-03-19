# Chapter 01 — SQL Fundamentals: Analogy Explanations

Making hard SQL concepts click through everyday analogies.

---

## Analogy 1: SQL Query Execution Order — The Restaurant Kitchen

**The Story:**
You walk into a restaurant and hand the waiter an order form filled out like this:
"I want (SELECT) grilled salmon, FROM the dinner menu, WHERE it is not sold out,
GROUP BY course, HAVING at least 2 options available, ORDER BY price."

The waiter does not process your form top to bottom. Instead, the kitchen works
in a specific order: first they identify which menu applies (FROM), then they
check what's available tonight (WHERE), then they group items by course (GROUP BY),
then they check if each course has enough options (HAVING), then they plate your
dish (SELECT — final column computation), then they arrange the plates in order
(ORDER BY), and finally they send out only the number you requested (LIMIT).

**Connection:**
The written SQL order is for human readability. The execution order is:
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT.
This is why you cannot use a SELECT alias in a WHERE clause — the kitchen hasn't
named the dish yet when it's deciding which ingredients make the cut.

```sql
-- "adjusted" alias is born in SELECT, but WHERE runs before SELECT
SELECT salary * 1.1 AS adjusted FROM employees WHERE adjusted > 50000; -- FAILS
SELECT salary * 1.1 AS adjusted FROM employees WHERE salary * 1.1 > 50000; -- OK
```

---

## Analogy 2: JOIN Types — The School Dance

**The Story:**
Picture a school dance with students and teachers as two separate lists.

- **INNER JOIN:** Only pairs where both a student and a teacher showed up and
  were matched. Wallflowers on either side are excluded.
- **LEFT JOIN:** All students appear in the photo, but if a student has no
  matching teacher partner, their "partner" spot is blank (NULL).
- **RIGHT JOIN:** All teachers appear, students with no partner get a NULL spot.
- **FULL OUTER JOIN:** Every person on both lists appears, blanks wherever no
  partner was found.
- **CROSS JOIN:** Every student dances with every teacher exactly once. 30
  students × 4 teachers = 120 pairs in the photo.

**Connection:**
```sql
-- LEFT JOIN: all orders, even those with no matching product (deleted products)
SELECT o.id, p.name
FROM orders o
LEFT JOIN products p ON o.product_id = p.id;
-- orders with deleted products show p.name = NULL

-- FULL OUTER JOIN: detect data integrity gaps
SELECT o.id, p.id
FROM orders o
FULL OUTER JOIN products p ON o.product_id = p.id
WHERE o.id IS NULL OR p.id IS NULL;
-- Shows unmatched rows on either side
```

---

## Analogy 3: NULL — The Sticky Note That Says "Unknown"

**The Story:**
Imagine a filing cabinet where each drawer holds a piece of information about
a person. Some drawers have a sticky note that says "Unknown" rather than an
actual value. Now someone asks: "Is this person's age equal to 25?" The sticky
note says unknown, so the answer is not "yes" and not "no" — it is "I don't know."

Now imagine: "Is the unknown person's age NOT 25?" Still "I don't know."
"Is the unknown age equal to another unknown age?" Still "I don't know."

The filing clerk's rule: any question about a sticky note returns "I don't know"
(UNKNOWN). And the ultimate rule: only TRUE lets a row through the WHERE filter.
UNKNOWN is treated the same as FALSE.

**Connection:**
```sql
-- Both comparisons return UNKNOWN, not TRUE
SELECT * FROM users WHERE age = NULL;      -- 0 rows, use IS NULL
SELECT * FROM users WHERE age != NULL;     -- 0 rows, use IS NOT NULL

-- The NOT IN NULL trap: one NULL in the list poisons the whole IN check
SELECT * FROM products WHERE id NOT IN (1, 2, NULL);
-- Expands to: id != 1 AND id != 2 AND id != NULL
-- id != NULL → UNKNOWN → entire AND chain → UNKNOWN → row excluded
-- Result: 0 rows!
```

---

## Analogy 4: Correlated Subquery — The Picky Librarian

**The Story:**
Imagine a library with 10,000 books. A librarian is asked: "For each book, find
whether there is another book by the same author that was published more recently."
The librarian picks up book #1, walks to the catalog, searches all books by that
author, finds the answer, and comes back. Then picks up book #2 and repeats the
entire walk. 10,000 books = 10,000 full catalog searches.

An efficient assistant would instead: group all books by author once, find the
latest publication date per author, then compare each book against that precomputed
table. One catalog scan total.

**Connection:**
```sql
-- Correlated: 10,000 subquery executions for 10,000 employees
SELECT name, salary
FROM employees e
WHERE salary > (SELECT AVG(salary) FROM employees WHERE dept = e.dept);

-- Efficient: one aggregation pass
WITH dept_avg AS (SELECT dept, AVG(salary) avg FROM employees GROUP BY dept)
SELECT e.name, e.salary
FROM employees e JOIN dept_avg d ON e.dept = d.dept
WHERE e.salary > d.avg;
```

---

## Analogy 5: HAVING vs WHERE — Quality Control vs Shipping Filter

**The Story:**
A factory produces widgets. The production line has two checkpoints:

**Checkpoint 1 (WHERE):** On the assembly line itself. Each individual widget is
inspected as it comes off the belt. Defective widgets are removed before they
ever get counted or boxed.

**Checkpoint 2 (HAVING):** At the loading dock. Entire boxes are evaluated. If
a box contains fewer than 10 widgets or has a total weight under 5kg, the whole
box is held back.

You cannot move the loading dock filter to the assembly line (you can't check
if a box has 10 widgets while building the first widget), and you cannot move
the assembly line filter to the loading dock (you've already packed the defective
widgets by then — wasteful).

**Connection:**
```sql
-- WHERE filters individual rows (assembly line) — runs before GROUP BY
-- HAVING filters groups (loading dock) — runs after GROUP BY
SELECT department, AVG(salary), COUNT(*)
FROM employees
WHERE hire_date > '2018-01-01'    -- Individual row filter: only recent hires
GROUP BY department
HAVING COUNT(*) > 5;              -- Group filter: only departments with >5 recent hires
```

Putting a non-aggregate condition in HAVING instead of WHERE is legal but wasteful:
the database groups all the discarded rows before realizing it should have dropped them.

---

## Analogy 6: UNION vs JOIN — Two Reports vs One Report

**The Story:**
Imagine two spreadsheets: "Q1 Sales" and "Q2 Sales." Both have the same columns:
Region, Product, Amount.

**UNION** is like stacking the two spreadsheets on top of each other: you get
one taller spreadsheet with the same columns. The rows are appended vertically.

**JOIN** is like merging two spreadsheets side by side using a common key: you
get a wider spreadsheet. "Q1 Sales" and "Q1 Targets" joined on Region+Product
gives you one row per Region+Product with both actual and target columns.

**Connection:**
```sql
-- UNION: vertical stacking of rows
SELECT product_id, revenue, 'Q1' AS quarter FROM q1_sales
UNION ALL
SELECT product_id, revenue, 'Q2' AS quarter FROM q2_sales;

-- JOIN: horizontal merging of columns
SELECT q1.product_id, q1.revenue AS q1_rev, q2.revenue AS q2_rev
FROM q1_sales q1
JOIN q2_sales q2 ON q1.product_id = q2.product_id;
```

UNION requires the same number and type of columns. JOIN requires a matching key.
They solve completely different problems and are not interchangeable.

---

## Analogy 7: Aggregate Functions and NULL — The Average Score With Missing Tests

**The Story:**
A teacher has 30 students. Five students were absent for a test and have no score
(NULL). She asks: "What is the average score?"

Two approaches:
1. Treat absent students as scoring 0. Sum all 30 grades, divide by 30.
2. Only average those who took the test. Sum the 25 scores, divide by 25.

SQL always uses approach 2. AVG, SUM, MIN, MAX, and COUNT(column) all ignore NULLs.
This produces a higher average, which may or may not be what the teacher wants.
If she wants approach 1, she must use COALESCE to explicitly replace NULLs with 0.

**Connection:**
```sql
-- These produce different results when scores has NULLs:
SELECT AVG(score) FROM test_results;                     -- ignores NULL rows
SELECT AVG(COALESCE(score, 0)) FROM test_results;        -- treats NULL as 0
SELECT SUM(score) / COUNT(*) FROM test_results;          -- also treats NULL as 0

-- COUNT(*) vs COUNT(column)
SELECT COUNT(*) AS all_rows, COUNT(score) AS scored_rows FROM test_results;
-- all_rows = 30, scored_rows = 25
```

---

## Analogy 8: EXISTS vs IN — Bouncer vs Guest List Search

**The Story:**
A nightclub with 1,000 people in the queue. The bouncer is told to let in only
people whose names appear on the VIP list.

**IN approach:** The bouncer takes the entire VIP list (say 500 names), memorizes
it, then checks each person in the queue against the full list.

**EXISTS approach:** For each person, the bouncer sends a runner into the club to
check "is there at least one entry for this person in the VIP folder?" The runner
stops as soon as the first match is found — even if the person appears 50 times.

EXISTS short-circuits: as soon as it finds one matching row, it stops looking.
IN with a large subquery must materialize the entire subquery result first.

**The NULL danger with NOT IN:** Imagine the VIP list has one smudged entry that
reads "???" (NULL). The bouncer says "I can't verify this person isn't on the list
because there's an unreadable entry" — and turns away EVERYONE. That's NOT IN
with NULL in the subquery: returns zero rows.

```sql
-- EXISTS: stops at first match, safe with NULLs
SELECT name FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);

-- NOT EXISTS: safe for "has no orders" check
SELECT name FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```
