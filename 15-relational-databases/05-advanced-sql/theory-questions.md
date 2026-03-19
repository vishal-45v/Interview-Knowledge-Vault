# Chapter 05 — Advanced SQL: Theory Questions

Senior-level questions covering window functions, CTEs, LATERAL joins, JSONB,
full-text search, and PostgreSQL-specific advanced features.

---

1. What is the difference between an aggregate function and a window function?
   Why does a window function not collapse rows the way GROUP BY does? What
   happens to the window frame when you omit the ORDER BY clause inside the
   OVER() clause?

2. Explain the PARTITION BY and ORDER BY clauses of a window function. What
   does each do, and what happens if you specify PARTITION BY but not ORDER BY,
   or ORDER BY but not PARTITION BY?

3. What is the difference between ROWS and RANGE in a window frame specification?
   Give a concrete example where ROWS BETWEEN and RANGE BETWEEN produce different
   results with the same numeric boundaries.

4. Explain the difference between ROW_NUMBER(), RANK(), and DENSE_RANK(). If
   rows have values [10, 10, 10, 20], what does each function return for each row?
   When would you choose one over the others?

5. What are LAG() and LEAD() functions? What is the third parameter they accept,
   and when is it useful? Give a real-world time-series use case where LAG is
   the natural solution.

6. What is a CTE (Common Table Expression)? How is it different from a subquery?
   What is the "optimization fence" behavior of CTEs in PostgreSQL, and how did
   this behavior change between PostgreSQL 11 and PostgreSQL 12?

7. Explain recursive CTEs. What are the two required components (anchor and
   recursive member)? What is the UNION ALL vs UNION difference within a
   recursive CTE, and what must be true to prevent infinite recursion?

8. What is a LATERAL join? How does it differ from a regular subquery? Give an
   example that is impossible to write without LATERAL.

9. What is GROUPING SETS? How does it relate to ROLLUP and CUBE? How many
   result set rows does `GROUP BY ROLLUP (a, b, c)` produce compared to
   `GROUP BY GROUPING SETS ((a,b,c), (a,b), (a), ())`?

10. What is the FILTER clause on an aggregate function? How does it differ from
    using CASE WHEN inside the aggregate function? Are there performance
    differences?

11. What is the difference between PostgreSQL's JSON and JSONB types? What are
    the storage, indexing, and query performance trade-offs between them?

12. What does the JSONB `@>` operator do? What does `?` do? What does `#>>` do?
    Which of these can be accelerated by a GIN index and which cannot?

13. How does PostgreSQL full-text search work? What is a tsvector, what is a
    tsquery, and how does the `@@` operator connect them? What is the role of
    `to_tsvector` and `to_tsquery`?

14. What is a materialized view? How does it differ from a regular view and a
    table? What are the trade-offs, and what is `REFRESH MATERIALIZED VIEW
    CONCURRENTLY`?

15. Explain INSERT ON CONFLICT (upsert). What is the difference between ON
    CONFLICT DO NOTHING and ON CONFLICT DO UPDATE? What must be specified for
    ON CONFLICT to work, and what is the EXCLUDED pseudo-table?

16. What is a generated column in PostgreSQL? What is the difference between
    a stored generated column and a virtual generated column? What are the
    limitations on what expressions a generated column can use?

17. What is TABLESAMPLE? What sampling methods does PostgreSQL support, and
    how do they differ? When would TABLESAMPLE be appropriate for production use?

18. What is the MERGE statement (PostgreSQL 15+)? How does it compare to the
    INSERT ON CONFLICT syntax? What use cases does MERGE handle that
    INSERT ON CONFLICT cannot?
