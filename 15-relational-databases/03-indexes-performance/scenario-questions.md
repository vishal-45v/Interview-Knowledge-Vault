# Chapter 03 — Indexes and Performance: Scenario Questions

Production performance scenarios requiring deep knowledge of indexing strategies,
query planning, and PostgreSQL internals.

---

1. A query `SELECT * FROM orders WHERE customer_id = 12345 AND status = 'pending'`
   runs in 50ms on a table with 100M rows. After a new feature launches, traffic
   increases and the query slows to 8 seconds. The table has an index on
   `(customer_id)` and another on `(status)`. EXPLAIN shows a bitmap AND of
   two index scans. What is happening, why did it degrade, and what index change
   would fix it?

2. Your team adds 15 new indexes to a critical orders table to speed up various
   analytical queries. The next day, the on-call engineer reports that INSERT
   performance has dropped by 60% and the batch import job that used to take
   20 minutes now takes 2 hours. What is causing this and how do you diagnose
   which indexes are actually being used vs being wasted?

3. A query on a `users` table using `WHERE LOWER(email) = LOWER($1)` ignores
   the index on `email` and does a full sequential scan of 50M rows. The
   developer says "but we have an index on email!" How do you fix this without
   requiring a collation change, and what is the general principle?

4. Your PostgreSQL server has 128GB of RAM. The `shared_buffers` is set to 128MB
   (the default). Your largest table is 40GB. Every morning after autovacuum runs,
   query performance degrades for 15 minutes as the buffer cache is "cold."
   What configuration changes do you make, and why does autovacuum evict pages
   from the buffer cache?

5. You run EXPLAIN ANALYZE on a query and see:
   `Seq Scan on orders (cost=0.00..2500000.00 rows=1 actual rows=450000)`
   The estimated rows is 1 but actual rows is 450,000. What is causing this huge
   discrepancy, and what are two different ways to fix it?

6. You have a time-series events table with 2 billion rows. All queries filter
   by a date range (`WHERE event_time BETWEEN X AND Y`) and the data is inserted
   in chronological order. Someone proposes a B-tree index on `event_time`.
   You think there might be a better option. What is it, and under what conditions
   does it apply?

7. A background analytics query runs every hour and takes 45 minutes, frequently
   conflicting with user-facing queries. It reads 80% of a large table and
   cannot be made faster with indexes. How do you reduce its impact on
   concurrent OLTP queries? What PostgreSQL features help here?

8. After a major deployment, you notice that a previously fast query is now
   using a sequential scan where it used to use an index scan. Nothing in the
   query or schema changed. EXPLAIN shows the index still exists. What are
   the three most likely causes, and how do you investigate each one?

9. You have a `notifications` table with 500M rows. 99% of rows have
   `status = 'sent'`. Only 1% have `status = 'pending'`. All application
   queries query `WHERE status = 'pending'`. The existing index on `status`
   is not being used by the planner. What type of index would fix this, and
   why is the regular index ineffective?

10. A developer proposes adding a composite index `(tenant_id, user_id, created_at,
    status, amount)` to cover multiple different query patterns across a
    multi-tenant SaaS application. Evaluate this proposal — what are the risks,
    and how would you systematically determine the right indexes for multiple
    query patterns on one table?
