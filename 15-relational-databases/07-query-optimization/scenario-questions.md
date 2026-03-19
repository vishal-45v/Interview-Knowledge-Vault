# Query Optimization — Scenario Questions

Real-world production situations requiring senior-level diagnosis and remediation.

---

1. Your team's most critical report query — a dashboard aggregation joining orders, customers, and products — suddenly degraded from 800ms to 4 minutes after a routine data load of 2 million new orders. EXPLAIN ANALYZE shows the planner switched from a hash join to a nested loop join on the largest table join. No schema changes were made. Walk through your complete diagnosis: what happened, what system views you query, what you find, and the exact SQL to fix it.

2. A microservices backend uses Hibernate ORM with lazy loading. In production, a request that fetches a "customer with their 50 most recent orders" generates 51 database queries (1 for the customer + 50 individual order fetches). The endpoint has p99 latency of 3.2 seconds. Engineers say "the individual queries are fast (1ms each)" and don't understand the problem. Explain the full technical impact, write the correct SQL JOIN that eliminates this, and describe how you would identify N+1 queries at scale using pg_stat_statements.

3. You receive an alert: a PostgreSQL query is consuming 95% CPU for 8 minutes and hasn't completed. EXPLAIN (ANALYZE, BUFFERS) shows: "Sort Method: external merge Disk: 45823kB" and "actual rows=2800000 estimated rows=420." What exactly happened, what are the two root causes, and provide the complete remediation plan including SQL and configuration changes.

4. A SaaS application uses per-tenant row filtering: every query has AND tenant_id = $1 appended by the application layer. The database has 50,000 tenants with 10,000 rows each (500M rows total). Index scans are extremely slow for large tenants (100,000+ rows) but fast for small ones. The application team asks whether they should partition the table by tenant_id. Analyze the trade-offs and describe the complete architectural decision including partition strategy, routing, and the query plan differences.

5. Your analytics team runs ad-hoc queries against a 2TB PostgreSQL table. Some queries run for 20+ minutes even though similar queries run in seconds. The slow queries all share a common pattern: they filter on a low-cardinality status column (4 values) but also group by a high-cardinality user_id column. EXPLAIN shows sequential scans on the full table. Design the complete index strategy and explain why each index type (B-tree, partial, covering) would or would not help for this specific pattern.

6. An application issues this query thousands of times per second:
   ```sql
   SELECT * FROM sessions WHERE token = $1 AND expires_at > NOW();
   ```
   The sessions table has 50M rows with an index on token. Despite the index existing, pg_stat_statements shows this query consumes 40% of total database CPU. EXPLAIN ANALYZE shows the index is being used but execution time is 8ms per call. Diagnose every possible cause (index type, data type, function on column, selectivity, network, result size) and provide a complete optimization checklist.

7. You need to implement a full-text search feature: "find all articles where the body contains the phrase 'machine learning' OR the tags array contains 'ML'." The articles table has 10 million rows. The current approach uses LIKE '%machine learning%' and = ANY(tags) which performs a full sequential scan. Design the complete search index strategy using PostgreSQL-native features (tsvector, GIN indexes, pg_trgm), write the optimized query, and explain the trade-offs.

8. A team reports that a query using a CTE runs in 45 seconds, but rewriting it as a subquery runs in 0.3 seconds. The query uses PostgreSQL 11. The CTE is referenced only once. Explain exactly why this difference exists (CTE optimization fence), what changed in PostgreSQL 12, and how to fix this in PostgreSQL 11 without changing the query structure. Also explain when the CTE fence behavior is actually desirable.

9. PgBouncer is configured in session mode with pool_size=50 for an application with 500 concurrent users. Operations team reports that during peak traffic, 300 users are waiting for connections with >10 second wait times. Diagnose the architecture problem, explain why session mode is causing this, and provide the complete PgBouncer reconfiguration (pool mode, pool size calculation, prepared statement handling, transaction boundaries in the application).

10. Your PostgreSQL query planner is consistently choosing a sequential scan on a 100M-row table despite an index existing on the filter column. EXPLAIN shows the estimated rows = 2,100,000 (21% of the table) but the actual rows = 847 (0.0008%). What are the three most likely causes of this estimation error, how do you diagnose each one using system views and EXPLAIN output, and what specific fixes would you apply in order of preference?
