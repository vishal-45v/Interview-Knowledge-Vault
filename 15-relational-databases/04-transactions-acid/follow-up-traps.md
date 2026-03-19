# Chapter 04 — Transactions and ACID: Follow-Up Traps

Tricky follow-up questions that reveal whether a candidate truly understands
MVCC internals, isolation level semantics, and locking subtleties.

---

## Trap 1: "SERIALIZABLE isolation prevents all concurrency anomalies, so you
should always use it."

**What most people say:** Yes, SERIALIZABLE is safest, always use it.

**Correct answer:** SERIALIZABLE prevents all three read phenomena and serialization
anomalies, but it comes with costs:
1. More transactions fail with serialization errors (`ERROR: could not serialize
   access due to concurrent update`). Application code must retry these.
2. PostgreSQL's SSI (Serializable Snapshot Isolation) tracks read/write dependencies
   which consumes memory and CPU.
3. Throughput can be significantly lower under high contention.

The correct approach: use the weakest isolation level that still prevents the
anomalies your application cannot tolerate. Most applications work correctly
at READ COMMITTED. Financial applications with balance operations may need
REPEATABLE READ or explicit row-level locking (SELECT FOR UPDATE).

```sql
-- Application must handle serialization failures at SERIALIZABLE level
LOOP:
  BEGIN;
  -- do operations
  COMMIT;
  -- If fails with:
  -- ERROR: could not serialize access due to concurrent update
  -- → retry the entire transaction
```

---

## Trap 2: "READ UNCOMMITTED in PostgreSQL lets you read uncommitted changes
from other transactions."

**What most people say:** Yes, READ UNCOMMITTED is the lowest isolation level
and allows dirty reads.

**Correct answer:** No. PostgreSQL's MVCC implementation makes dirty reads
impossible at any isolation level. READ UNCOMMITTED in PostgreSQL behaves exactly
like READ COMMITTED — it never reads uncommitted data from other transactions.
PostgreSQL effectively has only three distinct isolation levels: READ COMMITTED,
REPEATABLE READ, and SERIALIZABLE.

```sql
-- PostgreSQL: these behave identically
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Both: dirty reads are impossible because MVCC never exposes uncommitted versions
```

This is a common trap when candidates have memorized the SQL standard's four
isolation levels without knowing the implementation-specific behavior.

---

## Trap 3: "A ROLLBACK undoes all changes and returns the database to its
previous state. So ROLLBACK is always safe."

**What most people say:** Yes, ROLLBACK is the safe escape hatch.

**Correct answer:** ROLLBACK does undo all changes within the transaction, but:
1. Sequences/serials: sequence increments are NOT rolled back. If your transaction
   inserted a row with id=500 and rolled back, the next insert gets id=501, not 500.
   Gaps in auto-increment sequences are permanent.
2. Notifications: `NOTIFY` issued inside a rolled-back transaction is discarded.
   But some external systems may have already acted on interim states.
3. External side effects: if your application sent an email or made an API call
   during a transaction and then rolled back, those external actions are NOT rolled back.

```sql
BEGIN;
INSERT INTO orders (id, customer_id) VALUES (DEFAULT, 42);  -- Gets id=500
SELECT pg_notify('new_order', '500');  -- NOTIFY is rolled back
-- Crash or application error
ROLLBACK;
-- id=500 is gone from orders
-- But the SEQUENCE is now at 501. Next order gets id=501, not 500.
-- Gap: id=500 never exists in the table, but id 501 follows 499.
```

Always design systems to tolerate gaps in sequential IDs.

---

## Trap 4: "SELECT FOR UPDATE locks the rows it reads, preventing concurrent
updates. So it guarantees I can safely update those rows."

**What most people say:** Yes, FOR UPDATE locks the row until commit.

**Correct answer:** Mostly true, with an important nuance: if no rows match
the WHERE clause, nothing is locked. If you rely on FOR UPDATE to prevent
concurrent inserts of a new row that satisfies some condition, you're not
protected — FOR UPDATE only locks existing rows.

```sql
-- Checking availability and locking:
SELECT * FROM seats WHERE flight_id = 100 AND status = 'available' LIMIT 1 FOR UPDATE;
-- Locks the returned row. Safe for concurrent seat booking.

-- NOT protected: concurrent INSERT of a new 'available' seat
-- A concurrent transaction that INSERTs a new seat row is not blocked.
-- FOR UPDATE only locks rows that exist at the time of the SELECT.

-- For protecting against phantom inserts, you need REPEATABLE READ or
-- SERIALIZABLE isolation level, or a table-level lock.
```

Also: `SELECT FOR UPDATE` on a row that another transaction has already locked
will WAIT by default. Use `NOWAIT` to fail fast:
```sql
SELECT * FROM jobs WHERE id = 42 FOR UPDATE NOWAIT;
-- Raises: ERROR: could not obtain lock on row in relation "jobs"
-- Instead of hanging
```

---

## Trap 5: "If two transactions deadlock, PostgreSQL kills both of them."

**What most people say:** Yes, both transactions are killed to break the deadlock.

**Correct answer:** No. PostgreSQL kills only ONE of the two transactions
(the "victim"). The victim transaction is rolled back and receives an error.
The surviving transaction continues normally and its lock is granted.

```
Transaction A: holds lock on row 1, wants lock on row 2 → waiting
Transaction B: holds lock on row 2, wants lock on row 1 → waiting
Deadlock detected by PostgreSQL deadlock detector (runs every deadlock_timeout ms)
Result: One transaction is chosen as victim → ROLLBACK
        Other transaction acquires the previously held lock and continues
```

```sql
-- Application must handle deadlock errors and retry:
-- ERROR: deadlock detected
-- DETAIL: Process 12345 waits for ShareLock on transaction 6789;
--         blocked by process 67890.

-- Prevention: always acquire locks in a consistent order
-- Transaction A and B must always lock row 1 first, then row 2 (never 2 then 1)
UPDATE accounts SET balance = balance - 100 WHERE id = LEAST(src, dst);
UPDATE accounts SET balance = balance + 100 WHERE id = GREATEST(src, dst);
```

---

## Trap 6: "MVCC means readers never block writers and writers never block readers.
This means there's no locking overhead."

**What most people say:** Correct — MVCC eliminates locking overhead entirely.

**Correct answer:** MVCC eliminates read/write conflicts but does NOT eliminate
all locking. Write/write conflicts still require locks. In PostgreSQL:
- Row-level write locks are still acquired for UPDATE, DELETE, SELECT FOR UPDATE.
- Table-level DDL operations (ALTER TABLE, TRUNCATE, etc.) acquire heavy locks.
- MVCC generates dead row versions which have their own overhead (vacuum).

The correct statement: MVCC eliminates READ/WRITE conflicts (readers don't block
writers and writers don't block readers for concurrent reads). Write/write
conflicts on the same row still serialize via row-level locks.

```sql
-- This still blocks:
-- Transaction A: UPDATE orders SET status = 'shipped' WHERE id = 42;
-- Transaction B: UPDATE orders SET amount = 99  WHERE id = 42;
-- Transaction B waits until A commits or rolls back — row-level lock held by A
```

---

## Trap 7: "You should use SERIALIZABLE isolation to prevent the 'lost update'
problem."

**What most people say:** Yes, the lost update anomaly requires SERIALIZABLE.

**Correct answer:** Lost updates can be prevented at REPEATABLE READ (not just
SERIALIZABLE) and also at READ COMMITTED with explicit row locking. Using
SERIALIZABLE for this is overkill and may cause unnecessary serialization failures.

```sql
-- Lost update problem at READ COMMITTED:
-- T1: reads balance = 100
-- T2: reads balance = 100
-- T1: writes balance = 100 - 50 = 50 (commits)
-- T2: writes balance = 100 + 30 = 130 (commits, overwriting T1's update!)
-- Final: 130, expected: 80

-- Fix 1: SELECT FOR UPDATE (pessimistic, any isolation level)
BEGIN;
SELECT balance FROM accounts WHERE id = 42 FOR UPDATE;  -- acquire lock
UPDATE accounts SET balance = balance - 50 WHERE id = 42;
COMMIT;

-- Fix 2: Atomic update (no read-modify-write cycle needed)
UPDATE accounts SET balance = balance - 50 WHERE id = 42;
-- Single statement: no lost update possible, database handles atomicity

-- Fix 3: REPEATABLE READ detects the conflict:
-- T2's UPDATE sees that balance was modified since its snapshot → error
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- T2 gets: ERROR: could not serialize access due to concurrent update
```

---

## Trap 8: "BEGIN TRANSACTION is the proper way to start a transaction."

**What most people say:** Yes, BEGIN TRANSACTION starts a new transaction.

**Correct answer:** `BEGIN` and `BEGIN TRANSACTION` are synonymous in PostgreSQL.
More importantly, PostgreSQL is in autocommit mode by default. Every statement
that is NOT inside an explicit BEGIN block is automatically committed. This means:

```sql
-- Without BEGIN: each statement is its own transaction (autocommit)
UPDATE orders SET status = 'shipped' WHERE id = 1;  -- auto-committed
UPDATE orders SET status = 'shipped' WHERE id = 2;  -- auto-committed separately

-- With BEGIN: statements are in one transaction
BEGIN;
UPDATE orders SET status = 'shipped' WHERE id = 1;
UPDATE orders SET status = 'shipped' WHERE id = 2;
COMMIT;  -- Both committed atomically or neither
```

Also: in PostgreSQL there is no `START TRANSACTION` keyword (unlike MySQL). The
standard `BEGIN` or `BEGIN TRANSACTION` or `BEGIN WORK` are all equivalent.

The real trap: in application frameworks that use connection pools, a forgotten
`COMMIT` can hold a transaction open indefinitely, blocking vacuum and eventually
causing table bloat or even transaction ID wraparound.

---

## Trap 9: "The WAL guarantees durability. Once committed, data can never be lost."

**What most people say:** Correct — committed means durable.

**Correct answer:** Durability is guaranteed only if `synchronous_commit = on`
(the default). With `synchronous_commit = off` or `local`, commits return to
the client before the WAL is flushed to disk. A crash in that window loses
committed transactions.

Additionally, durability is only as good as the underlying storage's flush
behavior. If the storage controller has a write cache that lies about fsync
completing, WAL cannot provide true durability. This is why PostgreSQL requires
reliable fsync semantics — some cloud providers (EBS gp2 volumes historically)
have had issues with this.

```sql
-- Check current synchronous_commit setting:
SHOW synchronous_commit;
-- 'on' = WAL flushed before COMMIT returns (fully durable)
-- 'local' = WAL flushed to local disk only (replicas may be behind)
-- 'off' = COMMIT returns before WAL flush (up to ~200ms of data loss possible)
-- 'remote_write' = WAL sent to replica but not flushed there
-- 'remote_apply' = WAL applied on replica before COMMIT returns

-- Per-transaction override for low-value, high-throughput data:
BEGIN;
SET LOCAL synchronous_commit = off;  -- Only affects this transaction
INSERT INTO analytics_events ...;
COMMIT;
```

---

## Trap 10: "Advisory locks are automatically released when the query finishes."

**What most people say:** Yes, like row locks, they're released automatically.

**Correct answer:** It depends on whether you use session-level or transaction-level
advisory locks. Session-level advisory locks persist until explicitly released or
the session ends — they are NOT released at COMMIT or ROLLBACK.

```sql
-- SESSION-LEVEL: persists until released or session closes (DANGEROUS!)
SELECT pg_advisory_lock(12345);
COMMIT;  -- Lock is still held after commit!
-- Must explicitly release:
SELECT pg_advisory_unlock(12345);

-- TRANSACTION-LEVEL: released automatically at COMMIT/ROLLBACK (safer)
SELECT pg_advisory_xact_lock(12345);
COMMIT;  -- Lock automatically released here

-- Common bug: using pg_advisory_lock in a loop without matching unlock
-- Eventually exhausts max_locks_per_transaction limit
```

Use `pg_advisory_xact_lock` (transaction-level) by default for safety.
Use session-level only when you explicitly need the lock to survive transaction
boundaries (e.g., distributed cron job coordination).
