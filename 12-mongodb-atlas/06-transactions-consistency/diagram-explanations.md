# Chapter 06 — Transactions & Consistency: Diagram Explanations

---

## Diagram 1: Transaction Lifecycle — States and Transitions

```
TRANSACTION STATE MACHINE
══════════════════════════════════════════════════════════════════════

                         client.startSession()
                                 │
                                 ▼
                        ┌──────────────┐
                        │   NO_TXN     │ ← idle session, no active txn
                        └──────────────┘
                                 │
                  session.startTransaction()
                                 │
                                 ▼
                        ┌──────────────┐
                        │   STARTING   │ ← transaction initiated
                        └──────────────┘
                                 │
                  first operation is sent to server
                                 │
                                 ▼
                        ┌──────────────┐
                        │    ACTIVE    │ ← operations executing
                        └──────────────┘
                        │              │
                        │              │
           commitTransaction()    abortTransaction()
           or error that is       or error that is
           not retryable          not retryable
                        │              │
                        ▼              ▼
               ┌─────────────┐  ┌─────────────┐
               │ COMMITTING  │  │  ABORTING   │
               └─────────────┘  └─────────────┘
                        │              │
              success / retry      success
                        │              │
                        ▼              ▼
               ┌─────────────┐  ┌─────────────┐
               │  COMMITTED  │  │   ABORTED   │
               └─────────────┘  └─────────────┘
                        │              │
                        └──────┬───────┘
                               ▼
                        ┌──────────────┐
                        │   NO_TXN     │ ← back to idle (session reusable)
                        └──────────────┘

withTransaction() error handling:
──────────────────────────────────────────────────────────────────────
ACTIVE state error:
  hasErrorLabel("TransientTransactionError")
  → abort → retry entire callback from STARTING
  → Examples: WriteConflict(112), NotPrimary, network timeout on op

COMMITTING state error:
  hasErrorLabel("UnknownTransactionCommitResult")
  → retry commitTransaction ONLY (not entire callback)
  → Transaction may already be committed — do NOT re-run operations!

Application error (no error label):
  → abort → propagate error to caller
  → Examples: "Insufficient funds", validation error
```

---

## Diagram 2: Write Concern Acknowledgment Flow

```
WRITE CONCERN: W:MAJORITY IN A 3-NODE REPLICA SET
══════════════════════════════════════════════════════════════════════

Application
    │
    │  insertOne({ amount: 5000 }, { w: "majority", j: true })
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ PRIMARY (Node 1)                                                 │
│                                                                  │
│  1. Receives write                                               │
│  2. Applies to WiredTiger storage                               │
│  3. Writes to journal (j:true → flushes to disk)               │
│  4. Adds to oplog: { op: "i", ns: "db.payments", o: {...} }    │
│  5. Sends ACK to application? NOT YET                          │
│     (waiting for majority of nodes to acknowledge)              │
└─────────────────────────────────────────────────────────────────┘
         │  oplog replication (async, continuous)
         ├──────────────────────►──────────────────────────────────┐
         │                                                         │
         ▼                                                         ▼
┌──────────────────────────┐               ┌──────────────────────────┐
│ SECONDARY (Node 2)       │               │ SECONDARY (Node 3)       │
│                          │               │                          │
│  1. Receives oplog entry │               │  1. Receives oplog entry │
│  2. Applies to storage   │               │  2. Applies to storage   │
│  3. Flushes to journal   │               │  (slightly slower)       │
│  4. Sends write ack     ◄│──── ack ────►│  4. Sends write ack     │
│     to primary          │               │     to primary           │
└──────────────────────────┘               └──────────────────────────┘
         │                                           │
         │──────────────── ack ──────────────────────│
                          │
                          ▼
                     MAJORITY (2 out of 3 nodes) have acknowledged!
                          │
                          ▼
APPLICATION ◄─── ACK sent back to driver ──────────────────────────
   │
   └── db.payments.insertOne returns: { acknowledged: true }

TOTAL TIME: ~10-15ms (includes primary write + replication RTT)

W:1 flow (shorter):
Application → Primary write + journal → Primary ACK → Application (~3ms)
W:0 flow (shortest):
Application → Primary receives → Primary ACK → Application (0ms — no wait)
```

---

## Diagram 3: MVCC — Snapshot Isolation Visualization

```
MULTI-VERSION CONCURRENCY CONTROL (MVCC) IN WIREDTIGER
══════════════════════════════════════════════════════════════════════

Document: accounts { _id: "alice", balance: 1000 }

Time →  T=0        T=10       T=20       T=30       T=40
        ─────────────────────────────────────────────────

WiredTiger internal version chain for "alice":
        ┌──────┐
        │ v1   │  balance: 1000    ← committed at T=0, original value
        │ T=0  │
        └──────┘
             │
             │ T1 (Transfer) starts at T=10, reads v1 (balance=1000)
             │ T2 (Interest) starts at T=15, reads v1 (balance=1000)
             │
             │ T1 updates balance to 900
             ▼
        ┌──────┐
        │ v2   │  balance: 900     ← T1 writes at T=20
        │ T=20 │                    (visible only to T1 until commit)
        └──────┘
             │
             │ T1 commits at T=25
             │ v2 now visible to new transactions
             │
             │ T2 STILL sees v1 (balance=1000)
             │ T2's snapshot is locked at T=15, before T1's commit
             │
             │ T2 tries to update alice.balance
             ▼
        ┌──────────────────────────────────────────────────────────┐
        │ WRITE CONFLICT!                                           │
        │ T2 tries to write to alice.balance                        │
        │ WiredTiger: "T1 already modified this document and        │
        │ committed after T2's snapshot was taken"                  │
        │ → WriteConflict error (code 112)                         │
        │ → withTransaction() retries T2 from scratch              │
        │ → T2 gets NEW snapshot at T=28, sees balance=900          │
        └──────────────────────────────────────────────────────────┘

GARBAGE COLLECTION:
When no active transaction needs v1 (balance=1000) anymore:
WiredTiger marks v1 for garbage collection
Next checkpoint or background sweep removes v1 from storage

MVCC versions accumulate when:
- Long-running transactions hold snapshots for minutes
- Under high write load with many concurrent transactions
→ WiredTiger cache fills with old versions → performance degrades
→ Keep transactions SHORT!
```

---

## Diagram 4: Two-Phase Commit for Sharded Cluster Transactions

```
DISTRIBUTED TRANSACTION: 2PC PROTOCOL
══════════════════════════════════════════════════════════════════════

Cluster Setup:
  Shard A: orders collection  (sharded on customerId)
  Shard B: inventory collection (sharded on productId)
  mongos: router

Application sends transaction:
  TX: insert order + decrement inventory

PHASE 1: PREPARE
──────────────────────────────────────────────────────────────────────

Application                    mongos
     │                           │
     │── withTransaction() ─────►│
                                 │
                                 │──── route insert ────────────────►│ Shard A (orders)
                                 │──── route update ────────────────►│ Shard B (inventory)
                                 │
                                 │  Designates Shard A as COORDINATOR
                                 │
                                 │◄── PREPARE request ─────────────◄│ (to both shards)
                                 │
                  Shard A         │         Shard B
                     │◄── PREPARE │ PREPARE ─────►│
                     │            │               │
               Write prepare      │         Write prepare
               record to oplog    │         record to oplog
               Lock documents     │         Lock documents
                     │            │               │
                     │──── VOTE YES ──────────────►
                     ◄──── VOTE YES ──────────────│

PHASE 2: COMMIT
──────────────────────────────────────────────────────────────────────

                  Shard A (Coordinator)
                     │
                  Receives both YES votes
                     │
                  WRITES COMMIT RECORD TO OWN OPLOG
                  (point of no return — transaction WILL commit)
                     │
                     ├────── COMMIT ─────────────────────────────────►│ Shard B
                     │                                                │
                     │                                         Apply changes
                     │                                         Release locks
                     │◄───── COMMIT ACK ────────────────────────────◄│
                     │
                  Apply changes
                  Release locks
                     │
                     ▼
                 mongos ──── SUCCESS ────► Application

FAILURE SCENARIO: Shard A crashes after writing COMMIT but before sending to Shard B
  On Shard A recovery:
    → Reads oplog, sees COMMIT record for txn-123
    → Re-sends COMMIT to Shard B
    → Transaction completes correctly
  Transaction atomicity is preserved even through coordinator failure!

IF SHARD A CRASHES BEFORE WRITING COMMIT RECORD:
  → No COMMIT record = transaction aborts
  → Shard B: sees PREPARE but no COMMIT → aborts
  → Both shards roll back → clean state
```

---

## Diagram 5: Read Concern Levels — What Data Is Returned

```
READ CONCERN LEVELS: WHAT YOU SEE VS. WHEN
══════════════════════════════════════════════════════════════════════

Timeline of writes to accounts collection:
─────────────────────────────────────────────────────────────────────
T=0   { balance: 1000 }  committed to primary only
T=2   { balance: 950 }   committed to primary only (not replicated yet)
T=4   { balance: 950 }   replicated to secondary 1 (majority now has 950)
T=5   { balance: 930 }   written to primary, NOT yet replicated
T=6   Reader queries for balance ← WE ARE HERE
T=8   { balance: 930 }   replicated to secondary 1 (majority now has 930)

What each read concern returns at T=6:
────────────────────────────────────────────────────────────────────
READ CONCERN        │ Node queried  │ Returns │ Notes
────────────────────┼───────────────┼─────────┼──────────────────────
local               │ Primary       │ 930     │ Freshest, can roll back
local               │ Secondary     │ 950     │ May be behind primary
available           │ Any           │ 950/930 │ Like local, orphan risk
majority            │ Primary       │ 950     │ Majority-committed (T=4)
majority            │ Secondary     │ 950     │ Same majority-committed
linearizable        │ Primary only  │ 930     │ Confirmed latest, slow
snapshot (txn)      │ Primary       │ varies  │ Point-in-time snapshot

KEY INSIGHT:
  majority at T=6 returns 950, NOT 930
  because the 930 write (T=5) hasn't reached majority yet
  It WILL be majority at T=8
  → Slight staleness is the cost of "never rolls back" guarantee

ROLLBACK RISK:
  If primary crashes at T=5.5 (before T=6 replication of 930):
  local = 930 → ROLLED BACK to 950 (secondary becomes primary, has 950)
  majority = 950 → SAFE (majority never saw 930, so no rollback)
  → For financial data: always use majority to avoid rollback surprise

LATENCY COST (approximate):
  local: < 1ms (no confirmation needed)
  majority: 10-15ms (must confirm majority has it)
  linearizable: 15-50ms (must confirm primary is still primary)
```

---

## Diagram 6: Optimistic vs Pessimistic Locking

```
OPTIMISTIC vs PESSIMISTIC LOCKING COMPARISON
══════════════════════════════════════════════════════════════════════

PESSIMISTIC LOCKING (what transactions do for write-write conflicts):
──────────────────────────────────────────────────────────────────────

User A                        Database                        User B
  │                              │                              │
  │── update doc_123 ───────────►│                              │
  │  (acquires write lock)        │◄── update doc_123 ──────────│
  │                              │  BLOCKED: doc_123 is locked  │
  │   ... processing ...         │  waiting for lock release    │
  │                              │                              │
  │── commit ───────────────────►│                              │
  │◄─ success ───────────────────│                              │
  │  (releases lock)             │── doc_123 lock released ────►│
  │                              │  User B's update proceeds    │

OPTIMISTIC LOCKING (with version field — no transaction needed):
──────────────────────────────────────────────────────────────────────

User A                        Database                        User B
  │                              │                              │
  │── read doc_123 (v=7) ───────►│◄── read doc_123 (v=7) ──────│
  │◄─ { balance: 1000, v: 7 } ──│                              │
  │                              │◄── { balance: 1000, v: 7 } ─│
  │  ... user thinks for 5s ...  │   ... user thinks for 5s ... │
  │                              │                              │
  │── updateOne({_id, v:7}, ... ►│── updateOne({_id, v:7}, ... ►│
  │   $set: {bal: 900, v:8})     │  (arrives simultaneously)    │
  │                              │                              │
  │◄─ matched: 1, modified: 1 ──│ User A wins! doc now v=8      │
  │  SUCCESS                     │                              │
  │                              │── check: is v still 7?       │
  │                              │   NO! v is now 8             │
  │                              │◄─ matched: 0, modified: 0 ──│
  │                              │  User B gets ConflictError   │
  │                              │  Must re-fetch and re-apply  │

COMPARISON TABLE:
──────────────────────────────────────────────────────────────────────
                   Pessimistic          Optimistic
─────────────────────────────────────────────────────────────────────
Lock held during:  Entire operation     None (check at write time)
Concurrency:       Low (queuing)        High (no blocking)
Contention:        Blocking wait        Retry on conflict
Best for:          Write-heavy hotspot  Read-heavy, low write conflict
Overhead:          Lock management      Extra read + version check
MongoDB impl:      Transactions (MVCC)  { version: N } field + $inc
```
