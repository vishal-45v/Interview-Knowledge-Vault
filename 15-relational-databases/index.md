# Relational Databases — Interview Knowledge Vault

Senior engineer and database architect level preparation for RDBMS interviews.
Covers PostgreSQL internals, SQL mastery, design patterns, and production war stories.

---

## Chapters

### Chapter 01 — SQL Fundamentals (`01-sql-fundamentals/`)
Core SQL syntax and semantics at depth: SELECT execution order, JOIN mechanics,
NULL behavior in aggregates and comparisons, correlated vs uncorrelated subqueries,
EXISTS vs IN performance traps, set operations (UNION/INTERSECT/EXCEPT), CASE
expressions, and all aggregate functions. Covers the "why" behind syntax rules,
not just the "what".

### Chapter 02 — Database Design (`02-database-design/`)
Entity-Relationship modeling, all normal forms (1NF through 4NF) with functional
dependency theory, denormalization trade-offs, surrogate vs natural keys, constraint
types and their enforcement guarantees, ON DELETE/UPDATE behaviors, star vs snowflake
schema for analytics workloads, soft deletes, audit columns, and temporal table
patterns.

### Chapter 03 — Indexes and Performance (`03-indexes-performance/`)
B-tree internals (page splits, leaf node chaining, height), hash/GiST/GIN/BRIN
index types, composite index column ordering (leftmost prefix rule), covering indexes
with INCLUDE, partial and expression indexes, index selectivity, write overhead
trade-offs, EXPLAIN/EXPLAIN ANALYZE output interpretation, join algorithms (nested
loop, hash join, merge join), planner statistics, table bloat, VACUUM, and REINDEX.

### Chapter 04 — Transactions and ACID (`04-transactions-acid/`)
ACID guarantees and what each property actually means in practice, all four isolation
levels and the read phenomena each prevents, PostgreSQL MVCC implementation
(xmin/xmax, row versions, snapshot isolation), optimistic vs pessimistic locking,
SELECT FOR UPDATE/FOR SHARE, deadlock detection and prevention strategies, row-level
vs table-level vs advisory locks, two-phase locking theory, and WAL durability
mechanics.

### Chapter 05 — Advanced SQL (`05-advanced-sql/`)
Window functions with frame specifications (ROWS vs RANGE), all ranking functions,
LAG/LEAD for time-series, CTEs including recursive CTEs for tree/graph traversal,
LATERAL joins, GROUPING SETS/ROLLUP/CUBE for multi-dimensional aggregation, FILTER
on aggregates, PostgreSQL JSONB operators and indexing, full-text search
(tsvector/tsquery/GIN), arrays and UNNEST, generated columns, materialized views,
upsert with INSERT ON CONFLICT, and the MERGE statement.

### Chapter 06 — PostgreSQL Internals (`06-postgresql-internals/`)
The PostgreSQL storage engine at page level (8KB pages, heap files, page layout,
line pointers, tuple headers), TOAST for large values, buffer manager and shared
buffers, background workers (autovacuum, bgwriter, checkpointer, walwriter),
lock manager internals, catalog structure (pg_class, pg_attribute, pg_index),
extension architecture, and how DDL changes are transactional in PostgreSQL.

### Chapter 07 — Query Optimization (`07-query-optimization/`)
How the PostgreSQL planner works (parse, rewrite, plan, execute), cost model
(seq_page_cost, random_page_cost, cpu_tuple_cost), join reordering, partition
pruning, CTE optimization fence, statistics targets, extended statistics,
plan hints (pg_hint_plan), and common anti-patterns (N+1, implicit casts,
function on indexed column, missing indexes on FK columns).

### Chapter 08 — Replication and High Availability (`08-replication-ha/`)
Streaming replication vs logical replication, synchronous vs asynchronous modes,
replication lag monitoring, read replicas and replica lag, failover strategies,
patroni/pgBouncer architecture, connection pooling (transaction vs session mode),
WAL archiving, point-in-time recovery (PITR), backup strategies, and RPO/RTO
trade-offs.

### Chapter 09 — Database Security (`09-database-security/`)
PostgreSQL role system (GRANT/REVOKE, role inheritance, pg_hba.conf), row-level
security (RLS) policies, column-level privileges, SSL/TLS configuration, data
encryption at rest vs in transit, audit logging (pgaudit extension), PII handling
patterns, SQL injection prevention (parameterized queries, prepared statements),
and compliance patterns (GDPR, SOC2, HIPAA data isolation).

### Chapter 10 — MySQL Specifics (`10-mysql-specifics/`)
InnoDB storage engine internals, clustered vs secondary indexes in InnoDB, the
MySQL query cache (and why it was removed), strict vs lenient SQL mode, character
set and collation pitfalls, MySQL replication (binlog formats: STATEMENT/ROW/MIXED),
GTID-based replication, MySQL partitioning, and key behavioral differences from
PostgreSQL that trip up engineers switching between the two.
