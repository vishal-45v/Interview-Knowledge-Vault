# Chapter 04 — Transactions and ACID: Analogy Explanations

Making transactions, MVCC, isolation levels, and locking concrete through
everyday analogies.

---

## Analogy 1: ACID — The Bank Vault

**The Story:**
Imagine a bank vault with strict rules:

**Atomicity:** The bank has a rule: "All safety deposit boxes in a single
transaction are either all opened or none are opened." You can't open box A,
move valuables to box B, and then fail before closing box A. Either the full
exchange completes or nothing changes.

**Consistency:** The vault enforces a rule: "Total value in all boxes must never
go negative." If a transaction would violate this, the vault doors refuse to lock
and the transaction is reversed.

**Isolation:** Two bank employees working with different customers' boxes at the
same time cannot see each other's in-progress work. Employee A emptying box 1
does not show as empty to Employee B until A has finished and recorded the change.

**Durability:** Every completed transaction is recorded in a steel-reinforced
ledger bolted to the floor. Even if the building burns down, the ledger survives
and the state can be restored.

**Connection:**
```sql
BEGIN;  -- Vault doors seal
UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- Remove cash
UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- Add cash
COMMIT;  -- Both recorded in WAL (ledger), vault doors open
-- Crash between BEGIN and COMMIT? ROLLBACK on recovery — neither change persists
```

---

## Analogy 2: MVCC — The Library with Laminated Cards

**The Story:**
Imagine a library where every book page is laminated. When a librarian needs to
update a page (e.g., a map that needs a new road), they don't erase and redraw
on the original. Instead, they create a new laminated copy of the page with the
change, and put it next to the old one. The old card says "superseded by version 2."

A reader who started reading the book before the update was filed sees the old
page — it's still there, untouched. A reader who starts after the update was filed
sees the new page. Nobody blocks anyone.

A janitor (autovacuum) comes by periodically and removes old laminated cards that
no active reader can possibly need anymore.

**Connection:**
```sql
-- MVCC in action:
-- Reader T100 starts: snapshot taken, sees all rows committed before T100
-- Writer T200 updates order #42: creates new row version, marks old as "dead after T200"
-- Reader T100 still in progress: sees OLD version of order #42 (from its snapshot)
-- Reader T300 starts after T200 commits: sees NEW version of order #42

-- The "dead" old version is cleaned up by VACUUM (the janitor)
-- when no transaction's snapshot is old enough to need it

-- Check which transaction is holding up vacuum (the oldest active snapshot):
SELECT pid, xact_start, state, query FROM pg_stat_activity
WHERE xact_start = (SELECT MIN(xact_start) FROM pg_stat_activity WHERE xact_start IS NOT NULL);
```

---

## Analogy 3: Isolation Levels — The Newsroom

**The Story:**
A newsroom has editions that represent the database at a point in time.

**READ COMMITTED:** Every time a journalist writes a new paragraph, they go to
the newsroom and grab the freshest available edition. They might see a different
edition mid-article — the front page changed between paragraph 1 and paragraph 5.
This is fine for most articles but risky for investigative pieces requiring
consistent facts throughout.

**REPEATABLE READ:** The journalist picks up ONE edition at the start of their
work session and uses only that edition all day, even as newer editions are
printed. They have a consistent view — the same facts from start to finish.
New articles (phantom rows) added to that edition do not appear for them.

**SERIALIZABLE:** The editor reviews every journalist's work as if they were
written one at a time, in some order. If two journalists' pieces contradict
each other (serialization anomaly), one is held and must be rewritten.

**Connection:**
```sql
-- READ COMMITTED: re-reads see updated data mid-transaction
-- Re-read of the same row can return different values
SELECT balance FROM accounts WHERE id = 42;  -- returns 100
-- (Another transaction commits: sets balance to 50)
SELECT balance FROM accounts WHERE id = 42;  -- returns 50!
-- Non-repeatable read: two reads, two different values

-- REPEATABLE READ: snapshot locked at transaction start
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 42;  -- returns 100
-- (Another transaction commits: sets balance to 50)
SELECT balance FROM accounts WHERE id = 42;  -- still returns 100 (snapshot!)
COMMIT;
```

---

## Analogy 4: Deadlock — The Two-Lane Bridge

**The Story:**
Imagine a narrow two-lane bridge where neither lane is wide enough for two trucks
passing each other. Truck A drives from the left and stops in the middle waiting
for Truck B (going right) to back up. Truck B also drives to the middle and stops
waiting for Truck A to back up. Neither will move. This is a deadlock.

The bridge authority (PostgreSQL deadlock detector) notices after a timeout that
no one is moving. It orders one truck (the "victim") to back up immediately,
allowing the other truck to cross. The backed-up truck must try again.

**Prevention:** Establish a rule that all trucks must always enter from the left
side first. This way, if Truck A is crossing left-to-right, Truck B waits at
the left end, never entering from the right. One consistent order eliminates
the circular dependency.

**Connection:**
```sql
-- Deadlock scenario:
-- T1: UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- locks row 1
-- T1: UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- waits for row 2
-- T2: UPDATE accounts SET balance = balance - 50 WHERE id = 2;   -- locks row 2
-- T2: UPDATE accounts SET balance = balance + 50 WHERE id = 1;   -- waits for row 1
-- → Deadlock! PostgreSQL kills one, the other proceeds.

-- Prevention: always lock in consistent order (e.g., by ID)
UPDATE accounts SET balance = balance - 100 WHERE id = LEAST(1, 2);   -- id=1 first
UPDATE accounts SET balance = balance + 100 WHERE id = GREATEST(1, 2); -- id=2 second
-- T1 and T2 both lock row 1 first → one waits, other proceeds → no deadlock
```

---

## Analogy 5: Write-Ahead Log — The Notepad Before the Whiteboard

**The Story:**
An office uses a whiteboard to display the current project status. The team
has a rule: before erasing or writing anything on the whiteboard, you must first
write the change in the official notepad (the log), including what you're
changing and why. Only after writing in the notepad do you update the whiteboard.

If the power goes out while someone is updating the whiteboard, you can reconstruct
the exact state by replaying the notepad from the last checkpoint. Even if the
whiteboard is partially erased (data pages partially written), the notepad tells
you exactly what was committed and what wasn't.

**Connection:**
```sql
-- WAL is the notepad. Data pages are the whiteboard.
-- Sequence of a COMMIT:
-- 1. WAL record written to WAL buffer (in memory)
-- 2. WAL buffer flushed to disk (fsync) → notepad committed to permanent record
-- 3. COMMIT success returned to client
-- 4. (Later, maybe milliseconds later) data pages written to disk

-- Crash after step 2: notepad has the record → replay on recovery → no data loss
-- Crash after step 3 but before step 4: same → data pages rebuilt from notepad

-- Check WAL configuration:
SHOW wal_level;          -- minimal / replica / logical
SHOW synchronous_commit; -- on / off / local / remote_write / remote_apply
SHOW checkpoint_timeout; -- How often checkpoints run
```

---

## Analogy 6: SELECT FOR UPDATE vs Optimistic Locking — Checkout Counter

**The Story:**
Two shoppers want the last item on the shelf.

**SELECT FOR UPDATE (pessimistic):** Shopper A puts their hand on the item.
Now Shopper B cannot touch it — their hand would bump into Shopper A's. Shopper B
waits until Shopper A either puts the item in their cart (commits) or puts it back
(rolls back). Only one shopper can have the item at a time. This works well when
there are many competing shoppers (high contention) and each shopper decides quickly.

**Optimistic locking:** Both shoppers photograph the shelf (read the data with
a version number). Both go to the checkout. Shopper A checks out first and the
item is marked "sold" with version=2. Shopper B arrives at checkout, scans their
photo, and the system says "this item has changed since you photographed it
(version=2, you have version=1) — go back and see if it's still available."
Shopper B must retry. This works well when shoppers rarely compete for the same
item (low contention) and takes a long time deciding (long read-to-write window).

**Connection:**
```sql
-- High contention, quick decision (FOR UPDATE):
BEGIN;
SELECT id, quantity FROM inventory WHERE product_id = 101 FOR UPDATE;
-- Queue forms here for concurrent requests
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 101;
COMMIT;

-- Low contention, long user workflow (optimistic):
-- Read:
SELECT id, quantity, version FROM inventory WHERE product_id = 101;
-- (User spends 5 minutes filling checkout form)
-- Write:
UPDATE inventory SET quantity = quantity - 1, version = version + 1
WHERE product_id = 101 AND version = :original_version AND quantity > 0;
-- If 0 rows affected: concurrent update happened → show "item no longer available"
```

---

## Analogy 7: Transaction Savepoints — The Video Game Checkpoint

**The Story:**
In a long video game level, a player reaches a checkpoint. From this point,
if they die, they respawn at the checkpoint rather than restarting the entire
level. They can continue making progress from the checkpoint. But the entire
level must be completed (committed) for the progress to be saved permanently.

A SAVEPOINT works the same way. Within a transaction, you can mark a savepoint.
If later steps fail, you can roll back to the savepoint (respawn) without
losing all the work from the beginning of the transaction. But you still must
COMMIT the whole transaction for the work to be permanent.

**Connection:**
```sql
BEGIN;
INSERT INTO order_header (customer_id, total) VALUES (42, 0);
SAVEPOINT after_header;  -- Checkpoint set

INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 101, 2);
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 999, 1);
-- ^ This fails: product_id 999 doesn't exist (FK violation)

ROLLBACK TO SAVEPOINT after_header;  -- Respawn at checkpoint
-- order_header still exists in our transaction, bad order_items are undone

-- Try again with corrected items:
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 102, 1);
RELEASE SAVEPOINT after_header;  -- Remove the savepoint (no longer needed)
COMMIT;  -- Entire transaction committed: header + corrected items
```

---

## Analogy 8: Advisory Locks — The Office Conference Room Booking System

**The Story:**
A conference room booking system has rooms and a booking board. However, for
some special offline meetings, teams use a separate sticky-note system where
they place a note with their team name on the door saying "in use." The building
doesn't enforce this — it's voluntary, based on mutual agreement. Other teams
know to respect the sticky note because they also participate in this informal system.

Advisory locks work exactly this way. They have no automatic relationship to
any specific row or table. They're just a named mutex — any part of your
application that cooperates in the protocol respects the lock.

**Connection:**
```sql
-- Scenario: only one process should run the nightly report generation
-- Use advisory lock with a stable integer key (e.g., hash of 'nightly_report')

-- Process 1 acquires the lock:
SELECT pg_try_advisory_lock(hashtext('nightly_report'));
-- Returns TRUE: process 1 holds the lock, proceeds with report

-- Process 2 tries concurrently:
SELECT pg_try_advisory_lock(hashtext('nightly_report'));
-- Returns FALSE: another process holds the lock, this process exits gracefully

-- Process 1 finishes:
SELECT pg_advisory_unlock(hashtext('nightly_report'));

-- Key: advisory locks are only effective if ALL processes participate
-- A process that doesn't check the advisory lock can still run concurrently
-- They're a coordination mechanism, not an enforcement mechanism
```
