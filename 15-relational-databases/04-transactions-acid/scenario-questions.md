# Chapter 04 — Transactions and ACID: Scenario Questions

Production scenarios requiring deep understanding of concurrency, locking,
isolation levels, and transaction failure modes.

---

1. Your payment service processes concurrent payment requests. Occasionally, a
   customer's balance goes negative even though the code checks the balance before
   deducting. The code reads the balance, checks it is sufficient, then deducts
   it in a separate UPDATE. The application runs at READ COMMITTED isolation.
   What is the exact concurrency anomaly occurring, and what are two different
   database-level solutions?

2. A job queue system stores tasks in a `jobs` table. Multiple worker processes
   run the query `SELECT * FROM jobs WHERE status = 'pending' ORDER BY created_at
   LIMIT 1 FOR UPDATE`. Under load, workers are all grabbing the same job, causing
   duplicate processing. How do you fix this at the SQL level?

3. Your application has a "seat reservation" system. Two users simultaneously try
   to book the last seat on a flight. Both check availability, see 1 seat
   available, and both proceed to book. Both get confirmation emails. The airplane
   is now overbooked. Walk through exactly what went wrong at the transaction level
   and provide the correct SQL to prevent this.

4. A developer set `synchronous_commit = off` on a write-heavy financial
   transactions table "for performance." One month later, the server crashes
   and after recovery, 300 transaction records are missing from the last 30
   seconds. The developer insists committed transactions should be durable.
   What happened, and what should have been done instead?

5. Your production database is experiencing lock waits. The monitoring shows
   `pg_locks` has rows in `granted = false` that have been waiting for over 5
   minutes. The DBA sees that the blocking transaction is a long-running ETL
   job that started 3 hours ago. What lock types are likely involved, what is
   the impact on application users, and how do you resolve this immediately and
   prevent it long-term?

6. After deploying a migration that adds a column with a default value using
   `ALTER TABLE orders ADD COLUMN notes TEXT DEFAULT ''`, your application becomes
   unresponsive for 30 seconds. Thousands of timeout errors appear. What is
   happening at the lock level, and how should this migration have been written
   differently? (PostgreSQL pre-11 vs PostgreSQL 11+ behavior matters here.)

7. A microservices architecture has Service A updating `accounts` and Service B
   updating `ledger_entries`, both within distributed operations that span the
   two services. Service A commits, then Service B crashes before committing.
   The ledger is now out of sync with accounts. You cannot use a single RDBMS
   transaction across both services. What patterns exist to handle this, and
   what are the trade-offs?

8. A developer reports that their transaction is seeing "stale data." They read
   a row at the start of a long transaction, do some computation, then re-read
   the same row at the end and get the same old values even though other
   transactions have updated it in the meantime. They are running at REPEATABLE
   READ isolation. Is this a bug? What should they do if they need fresh data
   mid-transaction?

9. Your database has been running for 2 years without issues, but suddenly
   autovacuum is running constantly and the monitoring shows `age(datfrozenxid)`
   is at 1.8 billion. Your DBA says "we have a potential transaction ID wraparound
   risk." Explain what is happening, why it is dangerous, and what the emergency
   mitigation steps are.

10. Two transaction patterns in your application frequently deadlock:
    Pattern A: UPDATE accounts WHERE id=1, then UPDATE accounts WHERE id=2
    Pattern B: UPDATE accounts WHERE id=2, then UPDATE accounts WHERE id=1
    These run concurrently hundreds of times per second. How do you eliminate
    the deadlocks without changing the isolation level or adding explicit table
    locks?
