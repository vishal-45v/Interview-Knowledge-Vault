# Chapter 04 — Transactions and ACID: Diagram Explanations

Visual representations of MVCC, isolation levels, locking, deadlocks, and WAL.

---

## Diagram 1: MVCC Row Versioning (xmin/xmax)

```
  orders heap file — physical layout after a series of operations

  ┌─────────────────────────────────────────────────────────────────────────┐
  │ Heap Page 88                                                            │
  ├──────┬──────┬─────────┬──────────────────────────────────────────────── │
  │ slot │ xmin │ xmax    │ data                                            │
  ├──────┼──────┼─────────┼──────────────────────────────────────────────── │
  │  1   │ 1000 │ 0       │ id=10, status='pending',  amount=99.00  (live) │
  │  2   │ 1001 │ 2005    │ id=11, status='pending',  amount=150.00 (dead) │
  │  3   │ 2005 │ 0       │ id=11, status='shipped',  amount=150.00 (live) │
  │  4   │ 1003 │ 1004    │ id=12, status='cancelled',amount=50.00  (dead) │
  │  5   │ 1004 │ 0       │ id=12, status='refunded', amount=50.00  (live) │
  └──────┴──────┴─────────┴──────────────────────────────────────────────── │

  Legend:
  xmin = transaction that created this row version
  xmax = transaction that superseded this row (DELETE or UPDATE)
  xmax=0 means this is the current live version

  Visibility check for transaction T=2006 (snapshot: all TXIDs < 2006 are committed):
  slot 1: xmin=1000 (visible), xmax=0 (not deleted)    → VISIBLE ✓
  slot 2: xmin=1001 (visible), xmax=2005 (visible)     → NOT VISIBLE (deleted)
  slot 3: xmin=2005 (visible), xmax=0 (not deleted)    → VISIBLE ✓
  slot 4: xmin=1003 (visible), xmax=1004 (visible)     → NOT VISIBLE (deleted)
  slot 5: xmin=1004 (visible), xmax=0 (not deleted)    → VISIBLE ✓

  VACUUM can reclaim slots 2 and 4 (both dead, no active snapshot needs them)
```

---

## Diagram 2: Isolation Levels — What Each Transaction Sees

```
  Timeline of events:
  T=1: Transaction A starts (isolation level varies)
  T=2: Transaction B starts, reads balance=100
  T=3: Transaction B updates balance to 75, COMMITS
  T=4: Transaction C starts, inserts new row (id=999), COMMITS
  T=5: Transaction A re-reads balance
  T=6: Transaction A re-runs query for all rows

  ┌──────────────────┬──────────────────────────┬──────────────────────────┐
  │ Isolation Level  │ T=5: A re-reads balance  │ T=6: A re-runs query     │
  ├──────────────────┼──────────────────────────┼──────────────────────────┤
  │ READ COMMITTED   │ 75 (sees B's commit)      │ includes row id=999      │
  │                  │ Non-repeatable read ✗     │ Phantom read ✗           │
  ├──────────────────┼──────────────────────────┼──────────────────────────┤
  │ REPEATABLE READ  │ 100 (sees A's snapshot)  │ excludes row id=999      │
  │                  │ Repeatable read ✓         │ No phantom (PG extends)  │
  ├──────────────────┼──────────────────────────┼──────────────────────────┤
  │ SERIALIZABLE     │ 100 (sees A's snapshot)  │ excludes row id=999      │
  │                  │ Repeatable read ✓         │ No phantom ✓             │
  │                  │ + detects r/w dep cycles  │ May fail with ser. error │
  └──────────────────┴──────────────────────────┴──────────────────────────┘

  Snapshot timing:
  READ COMMITTED:  new snapshot taken at START OF EACH STATEMENT
  REPEATABLE READ: snapshot taken at START OF TRANSACTION, reused for all statements
  SERIALIZABLE:    same as REPEATABLE READ + SSI conflict detection
```

---

## Diagram 3: Deadlock Detection and Resolution

```
  Time →
  ────────────────────────────────────────────────────────────────────────
  T1: BEGIN                         T2: BEGIN
  T1: UPDATE accounts SET ...       T2: UPDATE accounts SET ...
      WHERE id = 1                      WHERE id = 2
      ↑ T1 acquires row lock on id=1    ↑ T2 acquires row lock on id=2

  T1: UPDATE accounts SET ...       T2: UPDATE accounts SET ...
      WHERE id = 2                      WHERE id = 1
      ↓                                 ↓
      WAITING for T2 to release id=2    WAITING for T1 to release id=1

  ──────── DEADLOCK ─────────────────────────────────────────────────────
  Circular dependency:
  T1 → waiting for → T2's lock on id=2
  T2 → waiting for → T1's lock on id=1

  PostgreSQL deadlock detector runs every deadlock_timeout (default: 1 second):
  Detects the cycle → picks one transaction as victim (typically the one
  that would be cheapest to rollback)

  RESULT:
  T1: ERROR: deadlock detected → application must retry
  T2: Continues, acquires lock on id=1, completes successfully

  ──────── PREVENTION ──────────────────────────────────────────────────
  Rule: always acquire locks in a consistent order

  T1: lock id=LEAST(1,2)=1 first, then id=GREATEST(1,2)=2
  T2: lock id=LEAST(2,1)=1 first, then id=GREATEST(2,1)=2

  Now both try to lock id=1 first → T2 waits while T1 holds id=1
  → no cycle → T1 completes → T2 acquires id=1 → T2 completes
  No deadlock.
```

---

## Diagram 4: Write-Ahead Log (WAL) Flow

```
  Client Application
       │
       │ SQL: UPDATE orders SET status='shipped' WHERE id=42
       ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │ PostgreSQL Backend Process                                        │
  │                                                                   │
  │  1. Execute UPDATE:                                               │
  │     - Fetch page from disk (or shared_buffers)                   │
  │     - Mark old row as dead (xmax = current XID)                  │
  │     - Write new row version to the page                          │
  │     - Page is now "dirty" (modified in memory, not yet on disk)  │
  │                                                                   │
  │  2. Write WAL record to WAL buffer:                              │
  │     "At LSN X: page Y slot Z was updated from 'pending' to      │
  │      'shipped' by XID 7890"                                      │
  │                                                                   │
  │  3. COMMIT called:                                               │
  │     - Write COMMIT record to WAL buffer                          │
  │     - FLUSH WAL buffers to disk (fsync) ←── DURABILITY POINT     │
  │     - Return success to client                                   │
  └───────────────────────────────────────────────────────────────────┘
       │                              │
       │                              ▼
       │                    ┌──────────────────┐
       │                    │   WAL Files      │  ← Sequential writes
       │                    │   (pg_wal/)      │    (fast, append-only)
       │                    │   - LSN-based    │
       │                    │   - Replicated   │
       │                    └──────────────────┘
       │
       ▼ (later, during checkpoint or bgwriter)
  ┌──────────────────┐
  │  Data Files      │  ← Random writes (slower)
  │  (base/ dir)     │    Written after WAL, not before
  └──────────────────┘

  CRASH RECOVERY:
  Crash ──► Replay WAL from last checkpoint ──► Committed transactions restored
           ──► Uncommitted transactions ignored (no COMMIT record in WAL)

  STREAMING REPLICATION:
  Primary WAL ──► WAL Sender ──► Network ──► WAL Receiver ──► Standby
                                              (applies WAL records to standby data)
```

---

## Diagram 5: Lock Mode Compatibility Matrix (PostgreSQL)

```
  Lock modes used by common operations:

  Operation                         Lock Mode Acquired
  ──────────────────────────────────────────────────────
  SELECT                            AccessShareLock (lightest)
  SELECT FOR UPDATE / FOR SHARE     RowShareLock
  INSERT, UPDATE, DELETE            RowExclusiveLock
  CREATE INDEX                      ShareLock
  CREATE INDEX CONCURRENTLY         ShareUpdateExclusiveLock
  VACUUM (non-FULL)                 ShareUpdateExclusiveLock
  ALTER TABLE ADD COLUMN (pre-PG11) AccessExclusiveLock (heaviest!)
  ALTER TABLE ADD COLUMN w/DEFAULT  AccessExclusiveLock (pg11+ can avoid this)
  DROP TABLE                        AccessExclusiveLock
  TRUNCATE                          AccessExclusiveLock
  REINDEX (non-concurrent)          AccessExclusiveLock

  Compatibility (✓ = compatible, ✗ = conflicts/waits):
  ┌─────────────────────────┬──────┬──────┬──────┬──────┐
  │ Held →                  │ AS   │ RS   │ RE   │ AE   │
  │ Requested ↓             │      │      │      │      │
  ├─────────────────────────┼──────┼──────┼──────┼──────┤
  │ AccessShare (SELECT)    │  ✓   │  ✓   │  ✓   │  ✗   │
  │ RowShare (FOR UPDATE)   │  ✓   │  ✓   │  ✓   │  ✗   │
  │ RowExclusive (DML)      │  ✓   │  ✓   │  ✓   │  ✗   │
  │ AccessExclusive (DDL)   │  ✗   │  ✗   │  ✗   │  ✗   │
  └─────────────────────────┴──────┴──────┴──────┴──────┘

  Key insight: DDL (AccessExclusiveLock) conflicts with ALL lock modes,
  including plain SELECT. This is why ALTER TABLE on a busy table causes
  all queries to queue behind it → application appears hung.

  Solution: Use LOCK_TIMEOUT + retries for DDL in production:
  SET lock_timeout = '2s';
  ALTER TABLE orders ADD COLUMN notes TEXT;
  -- Will fail fast if it can't acquire lock in 2s → retry
```

---

## Diagram 6: Optimistic vs Pessimistic Locking Timeline

```
  Scenario: Two users editing the same order concurrently

  PESSIMISTIC LOCKING (SELECT FOR UPDATE)
  ───────────────────────────────────────
  User A                            User B
  │                                 │
  ├─ BEGIN                          ├─ BEGIN
  ├─ SELECT ... FOR UPDATE ──────── │  (waits)
  │  (lock acquired)                │  BLOCKED
  │                                 │  .
  ├─ update form shown to User A    │  .
  │  (User A takes 30 seconds)      │  waiting...
  │                                 │  waiting...
  ├─ UPDATE orders SET ...          │  waiting...
  ├─ COMMIT ────────────────────── ►│  lock released
                                    ├─ SELECT ... FOR UPDATE
                                    │  (now gets lock, sees A's changes)
                                    ├─ UPDATE, COMMIT

  Problem: User B is blocked for 30 seconds (the duration of User A's form fill)
  Good for: short, automated transactions (milliseconds)

  OPTIMISTIC LOCKING (version column)
  ────────────────────────────────────
  User A                            User B
  │                                 │
  ├─ SELECT id,data,version=5 ──────├─ SELECT id,data,version=5
  │  (no lock)                      │  (no lock, both read simultaneously)
  │                                 │
  │  (form fill, 30 seconds)        │  (form fill, 30 seconds)
  │                                 │
  ├─ UPDATE ... WHERE version=5 ────│─ UPDATE ... WHERE version=5
  │  version becomes 6 ✓            │  0 rows affected ✗ (version is now 6!)
  ├─ COMMIT                         │
                                    ├─ application detects conflict
                                    ├─ re-read order (version=6)
                                    ├─ merge or show conflict to user
                                    ├─ retry UPDATE WHERE version=6

  Benefit: No blocking. 30-second form fill doesn't lock anyone out.
  Cost: Occasional retries when conflicts happen.
  Best for: web forms with human interaction (high read-to-write latency)
```
