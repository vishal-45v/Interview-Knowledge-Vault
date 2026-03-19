# Query Optimization — Theory Questions

Senior engineer / database architect level. No answers provided — use for self-assessment.

---

1. Describe the complete lifecycle of a SQL query in PostgreSQL: from the client sending a string to rows being returned. What are the five stages (parse, analyze, rewrite, plan, execute) and what does each stage produce as output?

2. Explain the cost model PostgreSQL uses when comparing query plans. What are seq_page_cost, random_page_cost, cpu_tuple_cost, cpu_index_tuple_cost, and cpu_operator_cost? Why is random_page_cost = 4.0 the default, and when should you change it for SSD storage?

3. What statistics does PostgreSQL collect per column and per table for query planning? Explain the purpose of n_distinct, the MCV (Most Common Values) list, the histogram, and the correlation statistic. How does each affect the planner's row count estimation?

4. What is the statistics target in PostgreSQL and what does ALTER TABLE t ALTER COLUMN c SET STATISTICS N do? What is the default value (100), the maximum (10000), and under what specific circumstances would you increase it for a particular column?

5. Describe all three join algorithms available to the PostgreSQL query planner: nested loop, hash join, and merge join. For each algorithm, explain the time complexity, memory requirements, when the planner chooses it, and what conditions (sort order, data size, join predicate type) favor it.

6. What are the planner method configuration parameters (enable_seqscan, enable_hashjoin, enable_mergejoin, enable_nestloop, etc.)? When is it appropriate to use these in production versus treating them as diagnostic tools only? What is the risk of permanently disabling a join type globally?

7. Explain partition pruning in PostgreSQL. How does the planner determine which partitions to scan for a given query? What is the difference between compile-time (static) pruning and runtime pruning (for parameterized queries)? What query patterns can defeat partition pruning?

8. Describe how parallel query works in PostgreSQL. What are the roles of the Gather node, parallel workers, and the leader process? What parameters control parallelism (max_parallel_workers_per_gather, min_parallel_table_scan_size, parallel_setup_cost)? What types of operations can be parallelized and which cannot?

9. What is a CTE (Common Table Expression) fence in PostgreSQL and how did behavior change in PostgreSQL 12? What does the MATERIALIZED keyword do and when does it help or hurt performance? When would you deliberately use WITH x AS (NOT MATERIALIZED) or WITH x AS (MATERIALIZED)?

10. Explain the N+1 query problem in the context of an ORM like ActiveRecord or Hibernate. Write a SQL example of N+1 occurring and the JOIN-based rewrite that eliminates it. What is the impact on database connection usage and round-trip latency?

11. What is connection pooling and why is it necessary for PostgreSQL specifically? What are the three PgBouncer pool modes (session, transaction, statement)? What are the restrictions of each mode and which is recommended for most OLTP applications?

12. What is a prepared statement in PostgreSQL? How does it differ from a plain query execution? What is generic plan vs custom plan caching, and what parameter controls how PostgreSQL decides which to use after the first few executions?

13. Explain the EXPLAIN ANALYZE output. What do "actual rows," "actual loops," "rows removed by filter," "Buffers: shared hit/read," and "Sort Method: quicksort Memory: Xkb / external merge Disk: Xkb" tell you about query performance?

14. What are the most useful columns in pg_stat_statements for identifying problematic queries? How would you use mean_exec_time, stddev_exec_time, total_exec_time, and calls together to prioritize optimization work? What does high stddev relative to mean suggest?

15. Describe the subquery vs JOIN performance debate. Under what circumstances is a correlated subquery genuinely slower than an equivalent JOIN? When does PostgreSQL automatically transform a subquery into a join (subquery unnesting)? When does it fail to unnest and what can you do?

16. What is an index-only scan in PostgreSQL, and what are the two conditions that must be true for the planner to choose it? How does the Visibility Map interact with index-only scans? What happens when a heap fetch is required during an index-only scan?

17. What is the difference between EXPLAIN (ANALYZE) and EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)? When would you use JSON format output versus text format? What additional information does VERBOSE provide that the default output omits?

18. What are extended statistics in PostgreSQL (CREATE STATISTICS)? What problems do they solve that per-column statistics cannot? Explain the three types: ndistinct, dependencies, and MCV for multi-column combinations. Give an example of a query that would benefit from each.

19. What is the executor's "bitmapped heap scan" and when does the planner choose it over a plain index scan? How does it combine multiple indexes (bitmap AND / bitmap OR)? What are the memory implications and what happens when the bitmap exceeds work_mem?

20. Describe the query planner's handling of OR conditions in WHERE clauses. Why can an OR condition prevent index usage in some databases? How does PostgreSQL handle OR with BitmapOr plans, and when would you rewrite an OR query as a UNION ALL for better performance?
