# Replication and High Availability — Theory Questions

Senior engineer / database architect level. No answers provided — use for self-assessment.

---

1. Explain PostgreSQL streaming replication at the protocol level. What are the roles of the WAL sender process on the primary and the WAL receiver process on the standby? What is the replication slot and why was it introduced? What problem does it solve that was present before replication slots existed?

2. What is the difference between synchronous and asynchronous replication in PostgreSQL? What are the synchronous_commit modes (on, remote_write, remote_apply, local, off) and what durability guarantees does each provide? What happens to write throughput and latency at each level?

3. How do you monitor replication lag in PostgreSQL? What system views and functions (pg_stat_replication, pg_current_wal_lsn, pg_wal_lsn_diff) reveal lag? What is the difference between write_lag, flush_lag, and replay_lag? At what lag threshold should an alert fire?

4. What is logical replication in PostgreSQL? How does it differ from streaming (physical) replication? What can logical replication do that physical replication cannot (different PostgreSQL versions, filtered tables, transformations)? What WAL level is required and what DDL changes are not replicated?

5. Describe the pg_basebackup utility. What does it do, what connection permissions are required, and how is it used to set up a new streaming replica? What is the difference between pg_basebackup -Fp (file format) and -Ft (tar format)?

6. Explain Point-in-Time Recovery (PITR) in PostgreSQL. What are the required components (base backup + WAL archive)? How does the recovery.conf/postgresql.conf recovery_target_time work? What is the difference between inclusive and exclusive recovery targets?

7. What is Patroni and what problem does it solve? How does it use a distributed consensus system (etcd, Consul, ZooKeeper) to manage leader election? What is the difference between a switchover and a failover in Patroni? What is the "split-brain" scenario and how does Patroni prevent it?

8. Describe table partitioning strategies in PostgreSQL (RANGE, LIST, HASH). For each strategy, give a concrete use case, describe how partition pruning works, and explain the trade-offs in terms of maintenance overhead, query performance, and data distribution.

9. What are the performance implications of declarative partitioning in PostgreSQL? How many partitions is too many? What happens to query planning time as the number of partitions grows? How do indexes work on partitioned tables — are they inherited automatically?

10. What is a replication slot in PostgreSQL? What are the two types (physical and logical)? What danger does an unconsumed replication slot pose to the primary server, and how would you monitor for this condition? What parameter limits WAL retention for slots?

11. Explain pg_dump and pg_dumpall. What are the differences between custom format (-Fc), directory format (-Fd), plain SQL format (-Fp), and tar format (-Ft)? Which format should you use for large databases and why? How does pg_restore's -j flag help with restore performance?

12. What is continuous WAL archiving to S3 (or similar object storage)? What does archive_command do, and what is the difference between archive_command and archive_library (PostgreSQL 15+)? How does restore_command work during recovery?

13. Define RTO (Recovery Time Objective) and RPO (Recovery Point Objective). For a PostgreSQL setup with: asynchronous streaming replication + hourly base backups + continuous WAL archiving, what is the theoretical maximum RPO and how is RTO bounded?

14. What is a Foreign Data Wrapper (FDW) in PostgreSQL? How does postgres_fdw allow querying remote PostgreSQL databases? What are the performance implications of JOINs across FDW tables versus local tables? What is predicate pushdown in the context of FDW?

15. Describe the Citus extension for horizontal scaling. What are coordinator nodes and worker nodes? How does Citus distribute tables (distributed vs reference vs local)? What types of queries are accelerated by Citus and which queries still require cross-shard operations?

16. What is the "hot standby" feature in PostgreSQL? How do you enable read queries on a physical standby? What are the limitations of queries on a standby (no DDL, no writes, conflict with primary VACUUM)? What is a hot standby conflict and how does hot_standby_feedback prevent it?

17. Explain semi-synchronous replication concepts (PostgreSQL's synchronous_standby_names). What is ANY 1 (standby1, standby2) versus FIRST 1 (standby1, standby2) in the synchronous_standby_names configuration? How does quorum-based synchronous replication work?

18. What is the purpose of wal_keep_size (formerly wal_keep_segments) in PostgreSQL? How does it interact with replication slots? What happens to a replica that falls too far behind when wal_keep_size is the only mechanism keeping WAL files (no replication slot)?

19. What is a switchover versus a failover in a high availability PostgreSQL cluster? What are the steps for a planned switchover with zero data loss? What data loss risk exists during an unplanned failover from an asynchronous replica?

20. How does read scaling work with PostgreSQL read replicas? What routing mechanisms exist (application-level, HAProxy, PgBouncer, pgpool-II)? What consistency guarantees does an application reading from a replica have, and what application patterns are safe versus unsafe when reads may return stale data?
