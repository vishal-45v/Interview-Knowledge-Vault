# Chapter 01 — SQL Fundamentals: Theory Questions

Senior-level questions covering SELECT internals, JOIN mechanics, NULL semantics,
subquery behavior, set operations, and aggregate edge cases.

---

1. What is the logical order of SQL clause evaluation? How does it differ from
   the written order, and why does this matter when referencing aliases defined
   in the SELECT list?

2. Explain the difference between WHERE and HAVING. Can you use an aggregate
   function in a WHERE clause? Can you use a non-aggregated column in a HAVING
   clause without including it in GROUP BY?

3. What is the difference between a correlated subquery and an uncorrelated
   subquery? How does the database engine execute each one, and what are the
   performance implications?

4. When would you choose EXISTS over IN, and when would IN outperform EXISTS?
   What happens to IN vs EXISTS behavior when the subquery returns NULL values?

5. Explain what happens when NULL appears in a NOT IN subquery. Give an example
   where NOT IN returns no rows unexpectedly due to NULL.

6. What is the semantic difference between INNER JOIN and a comma-separated FROM
   with a WHERE clause? Are they always equivalent?

7. Explain the difference between LEFT JOIN and LEFT OUTER JOIN. What does a
   RIGHT JOIN produce that cannot be rewritten as a LEFT JOIN without changing
   table order?

8. What does a CROSS JOIN produce, and when is it legitimately useful? How many
   rows does a CROSS JOIN between a 1000-row table and a 500-row table produce?

9. How does a SELF JOIN work? Give a practical use case where a SELF JOIN is the
   most natural solution.

10. What is the difference between UNION and UNION ALL? When does UNION ALL
    produce different results than UNION? What is the performance impact of each?

11. Explain how NULL behaves in aggregate functions like COUNT, SUM, AVG, MIN,
    and MAX. What does COUNT(*) count that COUNT(column) does not?

12. What is the difference between COALESCE and NULLIF? Give an example where
    NULLIF prevents a division-by-zero error.

13. How does GROUP BY interact with NULL? If a column contains NULL values, are
    all NULL values grouped together or treated as distinct?

14. Explain the difference between RANK(), DENSE_RANK(), and ROW_NUMBER(). If
    three rows tie for first place, what does each function return for positions
    1, 2, and 3?

15. What is a covering index query, and how does the SQL you write affect whether
    the optimizer can use one? Give an example SELECT statement that benefits
    from a covering index.

16. Explain INTERSECT and EXCEPT. What implicit operation does INTERSECT perform
    that raw JOIN does not? How does EXCEPT differ from NOT IN when NULL values
    are present in the data?

17. What is the difference between implicit and explicit type casting in SQL?
    Give an example where implicit casting silently produces wrong results or
    causes a performance problem by preventing index use.

18. How does DISTINCT work under the hood? Is SELECT DISTINCT equivalent to
    GROUP BY on all selected columns? Are there cases where they produce
    different results?

19. What does LIMIT with OFFSET do, and why is deep pagination (e.g.,
    LIMIT 20 OFFSET 1000000) a serious performance problem? What is the
    keyset pagination pattern that replaces it?

20. Explain the CASE expression syntax (simple vs searched). Can CASE expressions
    be used in ORDER BY, GROUP BY, and aggregate functions? Give one non-trivial
    example of each.
