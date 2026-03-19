# Chapter 04 — Transactions and ACID: Theory Questions

Senior-level questions covering ACID guarantees, isolation levels, MVCC internals,
locking, and WAL durability.

---

1. Define the four ACID properties. For each property, give a concrete real-world
   example of what would go wrong if that property were violated. Do not just
   recite the definition — explain the failure mode.

2. What is the difference between a dirty read, a non-repeatable read, and a
   phantom read? Which isolation levels prevent each, and which isolation levels
   allow each?

3. PostgreSQL supports READ COMMITTED, REPEATABLE READ, and SERIALIZABLE. It
   technically supports READ UNCOMMITTED as well. What does READ UNCOMMITTED
   actually do in PostgreSQL, and how is this different from other databases?

4. Explain Multi-Version Concurrency Control (MVCC). How does it allow multiple
   transactions to read and write concurrently without the reader blocking the
   writer? What is the trade-off compared to a lock-based approach?

5. What are `xmin` and `xmax` in a PostgreSQL row tuple? How does PostgreSQL
   use these values to determine row visibility for a given transaction?

6. What is a snapshot in the context of MVCC? When is the snapshot taken for
   READ COMMITTED vs REPEATABLE READ transactions, and how does this difference
   explain the different anomaly prevention properties?

7. What is the difference between optimistic and pessimistic locking? When is
   each approach more appropriate? What is the classic lost update problem and
   how does each locking strategy handle it?

8. What does `SELECT FOR UPDATE` do? What does `SELECT FOR SHARE` do? How do
   they differ in what they block? What is `SELECT FOR UPDATE SKIP LOCKED`
   used for?

9. What is a deadlock? How does PostgreSQL detect deadlocks? What happens when
   PostgreSQL detects a deadlock — which transaction is killed? How do you
   prevent deadlocks in application code?

10. What is the difference between row-level locking and table-level locking?
    What table-level lock modes exist in PostgreSQL? Which DDL operations acquire
    which lock modes, and why does this matter for zero-downtime deployments?

11. What is two-phase locking (2PL)? How does it guarantee serializability?
    What is the difference between strict 2PL and standard 2PL? How does 2PL
    differ from MVCC in its approach to concurrency?

12. What is the Write-Ahead Log (WAL) and how does it implement the Durability
    property of ACID? What happens if the server crashes after a COMMIT but
    before the data pages are flushed to disk?

13. What is a SAVEPOINT? How does it work, and what are its use cases? What
    happens to work done after a SAVEPOINT when you ROLLBACK TO SAVEPOINT?

14. What is the serialization anomaly? How does PostgreSQL's SERIALIZABLE
    isolation level (SSI — Serializable Snapshot Isolation) detect and prevent
    it, and how is this different from traditional lock-based serializability?

15. What is `synchronous_commit` in PostgreSQL? What are the valid settings,
    and what durability vs performance trade-off does each represent? When would
    you ever set synchronous_commit = off?

16. What is an advisory lock in PostgreSQL? How is it different from a regular
    row or table lock? Give a use case where advisory locks are the right tool.

17. What is transaction ID wraparound in PostgreSQL, and why is it one of the
    most catastrophic failures possible? What does autovacuum do to prevent it,
    and what are the early warning signs?

18. What happens to open transactions when a PostgreSQL primary fails and a
    standby is promoted? What guarantee does the WAL + synchronous replication
    give about committed transactions on the new primary?
