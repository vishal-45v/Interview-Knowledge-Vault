# Chapter 04 — Transactions and ACID: Structured Answers

Complete, interview-ready answers for the most critical transaction and concurrency
questions at the senior engineer level.

---

## Q1: Explain ACID properties with real failure-mode examples.

**Answer:**

**Atomicity:** All operations in a transaction succeed, or none of them do.
Failure mode without it: a money transfer deducts $100 from Account A (first
UPDATE), then the server crashes before adding $100 to Account B (second UPDATE).
$100 disappears. With atomicity, the entire transfer rolls back.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- Both succeed, or ROLLBACK reverts both
```

**Consistency:** A transaction brings the database from one valid state to
another. All constraints, triggers, and rules must be satisfied after the commit.
Failure mode: a transaction adds an order item referencing product_id=999 which
doesn't exist (FK violation). Consistency enforcement means this transaction
fails with a constraint violation rather than creating orphaned data.

**Isolation:** Concurrent transactions must not interfere with each other in
ways that produce incorrect results. Failure mode (dirty read without isolation):
Transaction A updates a row and then rolls back. Transaction B, running
concurrently, read the uncommitted value from A and used it to compute a balance.
The value B read never actually existed. With proper isolation, B never sees
A's uncommitted work.

**Durability:** Once a transaction commits, it persists even through crashes.
Failure mode: a payment processes successfully, the server crashes 10ms later,
and the payment is gone. With durability (WAL + fsync), the commit record is
persisted to disk before returning success to the client. On recovery, the WAL
is replayed to restore all committed transactions.

---

## Q2: Explain all isolation levels, the anomalies each prevents, and which
isolation level is appropriate for different workloads.

**Answer:**

The three read anomalies:
- **Dirty read:** Reading uncommitted data from another transaction.
- **Non-repeatable read:** Re-reading a row within a transaction and seeing
  changed data because another transaction committed an update.
- **Phantom read:** Re-executing a query within a transaction and seeing new
  rows that another transaction inserted and committed.

```
                      Dirty    Non-Repeatable  Phantom
Isolation Level       Read     Read            Read
──────────────────────────────────────────────────────
READ UNCOMMITTED      Possible  Possible        Possible
READ COMMITTED        Prevented Possible        Possible
REPEATABLE READ       Prevented Prevented       Possible*
SERIALIZABLE          Prevented Prevented       Prevented

* PostgreSQL's REPEATABLE READ also prevents phantom reads (stronger than SQL standard)
```

**READ COMMITTED (PostgreSQL default):**
Each statement sees a fresh snapshot of committed data. Non-repeatable reads
are possible — a row read twice in the same transaction can show different values
if another transaction committed between the two reads.

```sql
-- READ COMMITTED: common source of the "lost update" bug
-- Use explicit locking when needed:
BEGIN;
SELECT balance FROM accounts WHERE id = 42 FOR UPDATE;  -- lock acquired
UPDATE accounts SET balance = balance - 50 WHERE id = 42;
COMMIT;
```

**REPEATABLE READ:**
The snapshot is taken at the start of the transaction. All reads within the
transaction see the same consistent view. New rows inserted by concurrent
transactions are NOT visible. Appropriate for: reports that span multiple queries
and must see consistent data, balance checks followed by updates.

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT SUM(balance) FROM accounts;  -- takes snapshot here
-- ... do some computation ...
SELECT SUM(balance) FROM accounts;  -- returns same result, not affected by concurrent updates
COMMIT;
```

**SERIALIZABLE:**
Equivalent to executing transactions one at a time. PostgreSQL uses SSI
(Serializable Snapshot Isolation) which detects dangerous read/write dependency
cycles. Transactions may fail with serialization errors and must be retried.
Appropriate for: financial calculations where even snapshot-level anomalies
could produce wrong results (e.g., bank constraint: total deposits must exceed
total loans — a write by one transaction depends on a read by another).

---

## Q3: Explain PostgreSQL's MVCC implementation with xmin and xmax.

**Answer:**

Every row tuple in PostgreSQL's heap contains system columns:

- **xmin:** Transaction ID (XID) of the transaction that inserted this row version.
- **xmax:** Transaction ID of the transaction that deleted or replaced this row.
  Zero (0) if the row is not deleted/updated.
- **ctid:** Physical location of the row (page number, slot number).

```sql
-- Inspect system columns:
SELECT xmin, xmax, ctid, id, status
FROM orders WHERE id = 42;
-- xmin=5001 xmax=0 ctid=(88,3) id=42 status='pending'
```

**How an UPDATE works:**
```
Before UPDATE:
  Heap page 88, slot 3: xmin=5001, xmax=0, id=42, status='pending'

Transaction 6000 executes: UPDATE orders SET status='shipped' WHERE id=42;

After UPDATE:
  Heap page 88, slot 3: xmin=5001, xmax=6000, id=42, status='pending'  ← old version
  Heap page 91, slot 1: xmin=6000, xmax=0,    id=42, status='shipped'  ← new version
```

**Visibility rule:** A row version is visible to transaction T if:
- `xmin` committed before T's snapshot was taken (row was inserted before T's snapshot)
- `xmax` is either 0 (not deleted) OR `xmax` had not committed when T's snapshot was taken

```sql
-- Transaction 5999 (started before 6000 committed) sees:
-- Row at slot 3: xmin=5001 (committed), xmax=6000 (not yet committed) → VISIBLE
-- Row at slot 1: xmin=6000 (not yet committed) → NOT VISIBLE
-- T5999 sees: status='pending'

-- Transaction 6001 (started after 6000 committed) sees:
-- Row at slot 3: xmax=6000 (committed) → NOT VISIBLE (deleted)
-- Row at slot 1: xmin=6000 (committed) → VISIBLE
-- T6001 sees: status='shipped'
```

**Why this enables non-blocking reads:** A reader never modifies or checks a
lock on the row — it simply checks whether the xmin/xmax values are visible
in its snapshot. Writers write new tuple versions to new locations; they don't
block readers of old versions.

---

## Q4: How do you fix the classic "balance deduction race condition" (lost update)?

**Answer:**

This is one of the most common concurrency bugs in financial applications.

**The bug:**
```sql
-- Application code (READ COMMITTED isolation):
-- Step 1: Read current balance
SELECT balance FROM accounts WHERE id = 42;  -- returns 100

-- Step 2: Application computes new balance: 100 - 50 = 50

-- Step 3: Write the new balance
UPDATE accounts SET balance = 50 WHERE id = 42;
-- If two concurrent requests both read 100 and both write 50,
-- the final balance is 50 instead of 0. One deduction is "lost."
```

**Fix 1: Atomic UPDATE (no read needed)**
```sql
-- No application-level read-modify-write cycle:
UPDATE accounts
SET balance = balance - 50
WHERE id = 42 AND balance >= 50;  -- also enforce constraint

-- Check affected rows = 1 to detect concurrent depletion:
IF affected_rows = 0:
    raise InsufficientFunds()
```

**Fix 2: SELECT FOR UPDATE (pessimistic locking)**
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 42 FOR UPDATE;
-- Other transactions trying to UPDATE or FOR UPDATE on row 42 will WAIT here
-- Balance is guaranteed not to change while we hold this lock
UPDATE accounts SET balance = balance - 50 WHERE id = 42;
COMMIT;
```

**Fix 3: Optimistic locking with version column**
```sql
ALTER TABLE accounts ADD COLUMN version INT NOT NULL DEFAULT 0;

-- Read with version:
SELECT balance, version FROM accounts WHERE id = 42;
-- Returns: balance=100, version=5

-- Update only if version hasn't changed:
UPDATE accounts SET balance = 50, version = 6
WHERE id = 42 AND version = 5;  -- fails if concurrent update changed version to 6

-- If 0 rows affected: retry the entire read-modify-write cycle
```

**Best practice for financial systems:** Use atomic UPDATE with a constraint
check (Fix 1). It is the simplest, most performant, and correctly handles
concurrency without explicit locks. Reserve SELECT FOR UPDATE for cases where
you genuinely need to read a value, make application-level decisions based on it,
and then update.

---

## Q5: Explain the Write-Ahead Log (WAL) and how it provides durability.

**Answer:**

The WAL (called the redo log in other databases) is a sequential log of all
changes made to the database. The core principle: **changes are written to the
WAL before they are written to the actual data pages**.

**WAL write sequence:**
```
Application: COMMIT transaction T

1. All changes from T are written to WAL buffers in memory
2. WAL buffers are flushed to disk (fsync) → durable on disk
3. COMMIT returns success to client
4. Data pages may still be in memory (shared_buffers) — NOT yet on disk
```

**Why this is safe (the crash scenario):**
```
Crash happens between step 2 and 4:
- WAL is on disk (committed data is there)
- Data pages may be in wrong state or missing from disk

Recovery process:
1. PostgreSQL starts, finds uncommitted state in data files
2. Reads WAL from last checkpoint forward
3. Replays all WAL records for committed transactions
4. Rolls back (ignores) WAL records for uncommitted transactions
5. Database is consistent as of the last committed transaction
```

**Checkpoints:** Periodically, PostgreSQL writes all dirty pages from memory
to disk (a checkpoint). This limits how much WAL must be replayed on recovery.
`checkpoint_completion_target` spreads the I/O over time to avoid spikes.

```sql
-- Force a checkpoint (e.g., before a backup):
CHECKPOINT;

-- WAL-related settings:
-- wal_level: minimal | replica | logical (replica needed for streaming replication)
-- synchronous_commit: on | remote_apply | remote_write | local | off
-- max_wal_size: maximum WAL size before forcing a checkpoint (default 1GB)
-- wal_buffers: WAL buffer size in memory (default 1/32 of shared_buffers)

-- Check current WAL position:
SELECT pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());
```

**Replication and WAL:**
Streaming replication works by shipping WAL records from primary to standbys
in real time. The standby applies WAL records to maintain an identical copy
of the database. This is why WAL is also the replication mechanism — there is
no separate replication log.

---

## Q6: When and how should you use SELECT FOR UPDATE vs optimistic locking?

**Answer:**

**SELECT FOR UPDATE (pessimistic locking):**
Acquires a row-level exclusive lock immediately. Other transactions trying to
lock or update the row must wait until you release it (COMMIT/ROLLBACK).

Use when:
- Contention is high (multiple requests frequently update the same row)
- The critical section is short (lock held briefly)
- You cannot tolerate retry loops in the application

```sql
-- Seat booking with FOR UPDATE:
BEGIN;
SELECT * FROM seats WHERE id = 101 AND status = 'available' FOR UPDATE;
-- If no rows returned: seat already taken, ROLLBACK
-- If row returned: we hold the lock, no one else can book it
UPDATE seats SET status = 'booked', booked_by = :user_id WHERE id = 101;
COMMIT;
```

**Optimistic locking:**
No lock taken at read time. At write time, verify the version/timestamp has
not changed since you read. If changed, the update fails and you retry.

Use when:
- Contention is low (most transactions succeed without conflict)
- The read-to-write window is long (user spends 5 minutes filling a form)
- Retries are acceptable and easy to implement

```sql
-- Order editing with optimistic locking:
-- Read: SELECT id, amount, version FROM orders WHERE id = :id
-- User spends 3 minutes editing the order
-- Write:
UPDATE orders
SET amount = :new_amount, version = :version + 1
WHERE id = :id AND version = :version;
-- If 0 rows affected: someone else updated the order → show conflict error to user
```

**Performance comparison:**
- High contention: FOR UPDATE wins (no retry loops, lower total CPU)
- Low contention: optimistic locking wins (no lock overhead on read)
- Long-running user workflows: optimistic locking required (holding a row lock
  for the time a user spends on a form would cause severe contention)
