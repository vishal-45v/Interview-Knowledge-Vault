# MySQL Specifics — Theory Questions

Senior engineer / database architect level. No answers provided — use for self-assessment.

---

1. What are the major MySQL storage engines and what are the fundamental architectural differences between InnoDB and MyISAM? In what specific scenarios would you still choose MyISAM over InnoDB in 2024, and why is MyISAM effectively deprecated for new applications?

2. Describe the InnoDB buffer pool architecture. What is the buffer pool, how is it structured into chunks and instances (innodb_buffer_pool_instances), and what is the LRU list and midpoint insertion strategy? How does this differ from PostgreSQL's shared_buffers?

3. What is the InnoDB change buffer? What operations does it buffer (INSERT, DELETE, UPDATE secondary index pages)? When are buffered changes merged? What is the change buffer maximum size, and what problem does it solve?

4. Explain InnoDB redo logs and undo logs. What does each log contain and what specific purpose does each serve? What is the relationship between redo logs and MVCC? What are innodb_redo_log_capacity (MySQL 8.0.30+) and innodb_undo_tablespaces?

5. What is the InnoDB doublewrite buffer? What problem does it solve (partial page writes / torn pages)? In what storage scenarios can you safely disable it, and what is the performance impact of enabling it?

6. Explain MySQL's MVCC implementation. How does InnoDB use rollback segments and undo logs to provide multi-version reads? How does this differ architecturally from PostgreSQL's approach of storing old tuple versions in the heap? Which approach causes more I/O for OLTP workloads with heavy UPDATEs?

7. What is the MySQL binary log (binlog)? What are the three binlog formats (STATEMENT, ROW, MIXED) and what are the trade-offs of each in terms of data safety, replication fidelity, and storage size? What binlog format does MySQL 8.0 default to?

8. Describe MySQL replication (source/replica). What is the role of the binlog and the relay log? What is GTID replication and what problem does it solve compared to binary log file+position based replication? How do you identify and handle replication lag?

9. What is semi-synchronous replication in MySQL? How does it differ from traditional asynchronous replication? What is the "after commit" vs "after sync" semi-sync plugin variant, and which provides better durability guarantees?

10. How does MySQL's clustered index work? Why is the primary key in InnoDB also the clustered index (the physical row storage order)? What happens to secondary indexes — how do they reference rows, and what is the performance implication of using a large primary key (UUID vs BIGINT AUTO_INCREMENT)?

11. What is a covering index in MySQL? How does the "Using index" status in EXPLAIN indicate a covering index is being used? Why is the primary key always implicitly included at the end of every secondary index in InnoDB?

12. What is index merge optimization in MySQL? What are the three index merge algorithms (Union, Intersection, Sort-Union)? When does MySQL choose index merge versus using a single index, and when is index merge a sign of a missing composite index?

13. What are invisible indexes in MySQL 8.0? How are they different from disabled indexes? What is the use case for making an index invisible before dropping it? What optimizer switch controls whether invisible indexes are used?

14. What does MySQL's optimizer_switch variable control? How would you use it to diagnose whether a specific optimizer feature (hash_join, mrr, batched_key_access) is causing a performance problem? What is the risk of changing optimizer_switch globally?

15. What is MySQL's performance_schema? What types of instrumentation does it provide (statements, waits, locks, memory)? How is it different from sys schema? What is the overhead of enabling all performance_schema consumers, and how do you selectively enable only what you need?

16. Describe MySQL table partitioning. What partition types are supported (RANGE, LIST, HASH, KEY, subpartitioning)? What is MySQL KEY partitioning and how does it differ from RANGE? What are the limitations of MySQL partitioning compared to PostgreSQL's declarative partitioning?

17. What improvements did MySQL 8.0 introduce for JSON data type handling? What is the JSON_TABLE function and what problem does it solve? How does MySQL store JSON internally, and how do you create a functional index on a JSON path expression?

18. What is MySQL's EXPLAIN FORMAT=JSON output, and what additional information does it provide compared to traditional EXPLAIN? How do you use EXPLAIN ANALYZE (MySQL 8.0.18+)? What does "rows_examined_per_scan" reveal?

19. What are InnoDB lock monitoring views in performance_schema and sys schema? How would you use data_locks, data_lock_waits, and sys.innodb_lock_waits to diagnose a deadlock or blocking query? What is the difference between a table-level lock and a row-level lock in InnoDB?

20. What are the key architectural and feature differences between MySQL and PostgreSQL that matter most for choosing between them? Cover: MVCC implementation, JSON support, full-text search, partitioning, extensions/plugins, replication types, licensing, and cloud-native options.
