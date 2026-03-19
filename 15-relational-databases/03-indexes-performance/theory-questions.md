# Chapter 03 — Indexes and Performance: Theory Questions

Senior-level questions covering B-tree internals, index types, query planning,
EXPLAIN output, and the operational realities of index management in production.

---

1. Describe the internal structure of a B-tree index. What is stored at the root,
   internal, and leaf nodes? How does a point query (equality search) traverse
   the tree, and what is the time complexity?

2. What is a page split in a B-tree index? What causes it, what does it cost,
   and how does it relate to index fragmentation? How does the choice of index
   key (sequential vs random) affect page split frequency?

3. What is the difference between a B-tree index scan, a bitmap index scan, and
   a sequential scan in PostgreSQL? Under what conditions does the planner choose
   each one?

4. Explain the leftmost prefix rule for composite indexes. If you have an index
   on (a, b, c), which of the following queries can use it: WHERE a=1, WHERE b=1,
   WHERE a=1 AND c=3, WHERE a=1 AND b=2 AND c=3, WHERE b=2 AND c=3? Explain why
   for each case.

5. What is a covering index? What is the INCLUDE clause in PostgreSQL, and how
   does it differ from adding extra columns to the index key? Why does the INCLUDE
   approach produce a better index in many cases?

6. What is a partial index? Give three real-world use cases where a partial index
   is significantly better than a full index, and explain the storage and
   performance benefits.

7. What is an expression index (functional index)? Give an example of a query
   that cannot use a regular column index but can use an expression index.

8. What is index selectivity and cardinality? How does the planner use statistics
   to decide whether to use an index? What is the threshold at which a sequential
   scan becomes cheaper than an index scan, and why?

9. What are the write-side costs of indexes? How does having 10 indexes on a
   table affect the performance of INSERT, UPDATE, and DELETE? What is the cost
   model the database uses to decide how many indexes are too many?

10. What is index bloat? What causes it, how do you detect it, and how do you
    fix it? What is the difference between REINDEX CONCURRENTLY and VACUUM in
    terms of their effect on indexes?

11. What is a hash index in PostgreSQL? When is it faster than a B-tree index,
    and when is it worse? What was the historical problem with PostgreSQL hash
    indexes before version 10?

12. What is a GIN (Generalized Inverted Index)? What types of data is it designed
    for, and how does it differ from a B-tree in structure? Give two specific
    PostgreSQL data types where GIN outperforms B-tree.

13. What is a BRIN (Block Range Index)? What table characteristic makes BRIN
    effective, and what table characteristic makes it useless? Compare BRIN size
    to B-tree size for a 100M row time-series table.

14. Explain EXPLAIN ANALYZE output. What do the following fields mean: "cost=X..Y",
    "rows=N", "actual time=X..Y", "actual rows=N", "loops=N"? What does a large
    discrepancy between estimated rows and actual rows tell you?

15. What are planner statistics in PostgreSQL? What is pg_statistic, and what
    does the ANALYZE command do? How does the statistics target (default 100)
    affect query planning quality, and when should you increase it?

16. What is table bloat? How does PostgreSQL's MVCC model cause bloat, and what
    does autovacuum do about it? What is the difference between VACUUM and
    VACUUM FULL in terms of locking and space reclamation?

17. What is a nested loop join, a hash join, and a merge join? What input
    characteristics (size, sorted order, indexes) favor each one? Can a merge
    join be performed without sorting both inputs first?

18. What is the difference between `random_page_cost` and `seq_page_cost` in
    PostgreSQL's cost model? How should you adjust these on an SSD-backed server
    vs an HDD-backed server, and what impact does the adjustment have on the
    planner's index vs sequential scan decisions?
