# PostgreSQL Internals — Scenario Questions

Real-world production situations. Answer as a senior database engineer or architect.

---

1. Your production PostgreSQL database (300GB, 2000 writes/second) has table bloat of 60%. VACUUM runs every night but the dead tuple count keeps climbing. pg_stat_user_tables shows n_dead_tup growing faster than autovacuum can process them. What is your diagnosis, and what specific parameters would you change in postgresql.conf and per-table storage options to fix this without taking downtime?

2. An engineering team reports that their application begins throwing "ERROR: could not serialize access due to concurrent update" errors only during peak traffic (10,000 concurrent users). The database is set to REPEATABLE READ isolation. The errors appear on a specific ORDER UPDATE query. Walk through exactly why this error is occurring and what changes — at the application level, SQL level, and database configuration level — you would make to resolve it.

3. At 3 AM your monitoring fires: PostgreSQL is consuming 100% CPU, p99 query latency jumped from 5ms to 8 seconds, and pg_stat_activity shows hundreds of queries in "idle in transaction" state. There are no recent deployments. What are your first five diagnostic steps using only PostgreSQL system views, and what is your remediation plan?

4. Your data warehouse PostgreSQL instance (8TB) is experiencing 45-minute checkpoint cycles that cause massive I/O spikes, degrading OLAP query performance. The server has 512GB RAM, 64 CPU cores, and SSDs. Describe your complete postgresql.conf tuning approach — covering shared_buffers, wal_buffers, checkpoint_completion_target, max_wal_size, and effective_cache_size — with specific values and reasoning.

5. A critical customer-facing report query that JOINs 5 large tables (each 50M+ rows) suddenly degraded from 200ms to 45 seconds after a data load that added 5 million rows to one table. EXPLAIN ANALYZE shows the planner switched from an index scan + hash join to a sequential scan + nested loop. No schema or query changes were made. What happened, and what are the exact steps to diagnose and fix this?

6. Your PostgreSQL server crashed ungracefully (kernel OOM killer). On restart, recovery is taking 2 hours because it must replay 48GB of WAL. Your RTO requirement is 15 minutes. What configuration changes (checkpoint_completion_target, max_wal_size, wal_keep_size) would you make going forward, and what are the trade-offs of each? How would you reduce WAL volume without sacrificing durability?

7. A microservices platform is opening and closing database connections at 5,000 per second. pg_stat_activity shows max_connections (500) is constantly saturated, and applications are timing out waiting for connections. The team suggests raising max_connections to 2000. Explain why this is almost always the wrong solution, and describe the correct architecture using PgBouncer — including which pooling mode to use and how to configure pool_size correctly.

8. Your DBA ran VACUUM FULL on the largest table (800GB) during a maintenance window. The operation took 6 hours and held an ACCESS EXCLUSIVE lock the entire time, blocking all reads and writes. The table is now smaller but the incident was painful. What alternatives to VACUUM FULL exist (pg_repack, CLUSTER, CREATE TABLE AS SELECT), how do they work internally, and when would you use each?

9. You need to add a NOT NULL column with a default value to a 2-billion-row table in a zero-downtime production environment. A colleague says "ALTER TABLE ... ADD COLUMN ... DEFAULT ... NOT NULL" will rewrite the entire table and lock it for hours. Is this still true in PostgreSQL 11+? Explain the internal mechanism that changed, and describe the correct procedure for PostgreSQL 10 and earlier to achieve the same result without long locks.

10. Your security audit requires that all queries executed against the database be logged with the user, timestamp, query text, and duration — but only for queries taking longer than 100ms and only on specific sensitive tables. You cannot modify application code. Using only PostgreSQL features (pg_audit extension, log_min_duration_statement, log_statement), describe exactly how you would implement this requirement, including any limitations you would document in your design.
