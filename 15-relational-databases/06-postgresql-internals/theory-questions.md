# PostgreSQL Internals — Theory Questions

Senior engineer / database architect level. No answers provided — use these for self-assessment and study.

---

1. Describe the internal layout of a PostgreSQL heap page. What are the roles of the page header, line pointer array, and tuple data area? How does PostgreSQL use the pd_lower and pd_upper offsets to determine free space?

2. PostgreSQL defaults to 8KB pages. Under what circumstances would you compile PostgreSQL with 16KB or 32KB pages, and what are the trade-offs in terms of I/O efficiency, WAL amplification, and buffer pool fragmentation?

3. Explain the TOAST mechanism in full. When does PostgreSQL decide to TOAST a value? What are the four TOAST strategies (PLAIN, EXTERNAL, EXTENDED, MAIN), and how do they differ in compression and out-of-line storage behavior?

4. What are the four system columns xmin, xmax, cmin, and cmax present in every PostgreSQL tuple? What values do they hold, and how does the visibility check algorithm use them to determine whether a tuple is visible to a given transaction snapshot?

5. Walk through exactly what happens at the storage level when an UPDATE is executed in PostgreSQL. Why does PostgreSQL not perform in-place updates? How does this relate to MVCC and what is the consequence for table bloat over time?

6. What is a PostgreSQL transaction snapshot (SnapshotData)? What are xmin, xmax, and the xip list within a snapshot? How does the snapshot determine whether a tuple version is "in the past" or "in the future" relative to the current transaction?

7. Explain the Free Space Map (FSM) and the Visibility Map (VM). What does each track? How does VACUUM use these two structures? What is the significance of the "all-visible" bit and how does it enable index-only scans to skip heap fetches?

8. Describe the VACUUM process step by step. What is the difference between regular VACUUM and VACUUM FULL? What is the consequence of transaction ID (XID) wraparound, and how does aggressive VACUUM prevent it?

9. What is autovacuum cost-based delay? Explain the parameters autovacuum_vacuum_cost_delay, autovacuum_vacuum_cost_limit, and vacuum_cost_page_hit/miss/dirty. Why would you tune these differently on an OLTP system versus a data warehouse?

10. Explain PostgreSQL's Write-Ahead Logging (WAL) architecture. What does wal_level control (minimal, replica, logical)? What is the practical difference between fsync=on and synchronous_commit=off in terms of durability guarantees and crash recovery behavior?

11. What is a checkpoint in PostgreSQL? What sequence of operations does the checkpoint process perform? What do checkpoint_completion_target and max_wal_size control, and how do they affect I/O smoothing and recovery time after a crash?

12. Describe the role of the background writer (bgwriter). How does it differ from the checkpoint process? What do bgwriter_lru_maxpages and bgwriter_delay control, and when would high dirty-page eviction by bgwriter indicate a performance problem?

13. Explain the pg_locks system view. What lock modes exist from ACCESS SHARE through ACCESS EXCLUSIVE? Which locks conflict with which? How would you join pg_locks with pg_stat_activity to identify and diagnose a blocking chain in production?

14. What does shared_buffers control in PostgreSQL? Why is the common recommendation 25% of RAM, and why does PostgreSQL not benefit the same way as other databases (e.g., Oracle SGA) from a very large shared_buffers? How does effective_cache_size differ — what does setting it actually change in planner behavior?

15. Explain work_mem in PostgreSQL. Why is its total memory footprint not simply work_mem × max_connections? Under what conditions can a single query consume multiple work_mem allocations simultaneously? How would you set work_mem for a mixed OLTP/analytics workload without risking OOM?

16. What is logical replication in PostgreSQL? How do publications and subscriptions work internally? What WAL decoder plugin is used (pgoutput), and what categories of DDL changes are NOT replicated by the default logical replication system?

17. What are tablespaces in PostgreSQL? How do you create one, assign tables and indexes to it, and what operational benefits does placing indexes on a separate tablespace on a faster NVMe device provide? What happens to a tablespace's data when you drop it?

18. What does the pg_stat_statements extension track? What query normalization approach does it use for literal values? Which columns (mean_exec_time, stddev_exec_time, shared_blks_hit/read) are most useful for identifying problematic queries in a production system?

19. Describe the pg_trgm extension. What is a trigram, and how does pg_trgm enable fast LIKE '%pattern%' searches that normally cannot use a B-tree index? Which index types (GIN vs GiST) work with pg_trgm and when would you choose each?

20. What is the PostgreSQL process model? How does PostgreSQL handle connections — one backend process per connection versus a thread-per-connection model? What are the implications for memory usage, context switching overhead, and why does this architecture motivate the use of a connection pooler like PgBouncer?
