# Chapter 06 — Transactions & Consistency: Theory Questions

---

## Q1: What ACID properties does MongoDB support?

MongoDB supports full ACID guarantees at the single-document level natively, and full multi-document ACID transactions since version 4.0 (replica sets) and 4.2 (sharded clusters).

**Atomicity**: Single-document operations are always atomic — all field updates succeed or all fail. Multi-document transactions are atomic: either all operations commit or all roll back on abort/error.

**Consistency**: Documents move from one valid state to another. Schema validators (JSON Schema) enforce consistency rules. Transactions ensure invariants across multiple documents (e.g., total balance must sum to constant).

**Isolation**: MongoDB uses snapshot isolation for transactions. Each transaction sees a consistent snapshot of the data as of its start time, regardless of concurrent operations.

**Durability**: With `j: true` (journal: true) write concern, writes are durable to disk before acknowledgment. With `w: majority`, writes are durable on a majority of replica set members.

```javascript
// Single-document atomicity — no transaction needed
db.products.updateOne(
  { _id: productId },
  {
    $inc: { quantity: -1, soldCount: 1 },
    $set: { lastSoldAt: new Date() }
  }
)
// All three field changes succeed or all fail — atomic by default

// Multi-document ACID transaction
const session = client.startSession()
try {
  await session.withTransaction(async () => {
    await db.accounts.updateOne({ _id: fromId, balance: { $gte: 100 } },
      { $inc: { balance: -100 } }, { session })

    await db.accounts.updateOne({ _id: toId },
      { $inc: { balance: 100 } }, { session })

    await db.txnLog.insertOne({
      from: fromId, to: toId, amount: 100, at: new Date()
    }, { session })
    // All 3 ops succeed or all 3 roll back
  })
} finally {
  await session.endSession()
}
```

---

## Q2: How do MongoDB transactions work? What is a ClientSession?

A **ClientSession** is the object that tracks a transaction's state. All operations within a transaction must use the same session object.

```javascript
// Step-by-step breakdown
const session = client.startSession()

// Manually start/commit/abort
session.startTransaction({
  readConcern:  { level: "snapshot" },
  writeConcern: { w: "majority", j: true },
  maxCommitTimeMS: 10000                    // fail if commit takes > 10s
})

try {
  const sender = await db.wallets.findOneAndUpdate(
    { _id: fromId, balance: { $gte: amount } },
    { $inc: { balance: -amount } },
    { session, returnDocument: "after" }     // must pass session!
  )
  if (!sender) throw new Error("Insufficient funds")

  await db.wallets.updateOne(
    { _id: toId },
    { $inc: { balance: amount } },
    { session }
  )

  await session.commitTransaction()
} catch (err) {
  await session.abortTransaction()          // rolls back all ops in the session
  throw err
} finally {
  session.endSession()                      // release session resources
}

// RECOMMENDED: withTransaction() handles retries automatically
await session.withTransaction(async () => {
  // ... operations with { session } ...
}, {
  readConcern: { level: "snapshot" },
  writeConcern: { w: "majority" }
})
// withTransaction() auto-retries on transient errors (TransientTransactionError)
// auto-retries commitTransaction on UnknownTransactionCommitResult
```

**What happens internally**:
1. Session records a transaction start snapshot timestamp (logical session clock)
2. All reads within the session see data as of the snapshot point (snapshot isolation)
3. All writes go to the WiredTiger write buffer with transaction ID tags
4. On `commitTransaction`: operations are durably written and made visible
5. On `abortTransaction`: all buffered writes are discarded

---

## Q3: What are the limitations of multi-document transactions?

```javascript
// LIMITATION 1: 60-second default timeout (configurable)
// Transactions that run longer than transactionLifetimeLimitSeconds are auto-aborted
// Default: 60 seconds
// Set lower in application; never design transactions that run for minutes

// LIMITATION 2: 16MB transaction op log entry limit
// All changes in a transaction must fit in a single oplog entry
// Practical: don't update thousands of documents in one transaction

// LIMITATION 3: Transactions cannot create or drop collections
// ERROR: creating a new collection inside a transaction
await session.withTransaction(async () => {
  await db.createCollection("newColl")   // ERROR: not allowed
})
// Exception: implicit collection creation on insert IS allowed in MongoDB 4.4+
await session.withTransaction(async () => {
  await db.newColl.insertOne({ x: 1 })  // OK in 4.4+: creates collection if not exists
})

// LIMITATION 4: Cannot use $out or $merge in aggregation within a transaction
// ERROR:
await session.withTransaction(async () => {
  await db.orders.aggregate([
    { $group: { _id: "$status", count: { $sum: 1 } } },
    { $out: "orderStats" }   // ERROR: $out not allowed in transaction
  ], { session })
})

// LIMITATION 5: DDL operations blocked during active transactions
// An active transaction holds locks. Long-running transactions can block index builds.

// LIMITATION 6: Performance overhead
// Transactions are more expensive than single-document writes:
// - Snapshot isolation requires maintaining multiple versions of data (MVCC)
// - Two-phase commit protocol over the network for distributed transactions
// - Lock contention if hot documents are in multiple concurrent transactions

// LIMITATION 7: Sharded cluster extra overhead
// Cross-shard transactions require a two-phase commit coordinator
// The mongos picks the first shard touched as the coordinator
// Network round-trips increase with each additional shard involved

// When NOT to use transactions:
// - Single-document updates (atomic by default)
// - Read-only operations (use read concern directly)
// - High-throughput inserts (transactions add commit overhead)
// - Cases where eventual consistency is acceptable
```

---

## Q4: Explain read concerns in MongoDB. When do you use each?

Read concern controls the consistency and isolation level of data returned by a query.

```javascript
// read concern: "local" (default for most operations)
db.orders.find({ status: "pending" }).readConcern("local")
// Returns the most recent data available on the queried node
// May include data that has NOT yet been replicated to a majority
// Risk: if primary crashes before replicating, these reads may "roll back"
// Use: non-critical reads where freshest possible data is needed

// read concern: "available" (fastest, weakest — for sharded clusters)
db.orders.find({ status: "pending" }).readConcern("available")
// Same as "local" for replica sets
// On sharded clusters: may return orphaned documents (chunks migrating between shards)
// Use: very rarely — only when absolute maximum throughput > data correctness

// read concern: "majority"
db.orders.find({ status: "pending" }).readConcern("majority")
// Returns only data that has been acknowledged by a majority of replica set members
// Data returned will NOT roll back — it is permanently committed
// Slight staleness: minority secondaries may not have it yet, but majority does
// Use: financial data, anything where rollback would be a problem
// Performance: slower than "local" (must wait for majority ack point)

// read concern: "linearizable" (strongest)
db.orders.findOne({ _id: orderId }).readConcern("linearizable")
// Reads reflect ALL writes completed before the read starts
// Guarantees: read your own writes even with primary failover in between
// Only for single document reads (point queries) — not for ranged queries
// Performance: very slow — must confirm with majority that it's still primary
// Use: cases requiring absolute linearizability (distributed locks, counters)

// read concern: "snapshot" (for transactions)
session.startTransaction({ readConcern: { level: "snapshot" } })
// All reads in the transaction see a consistent snapshot from transaction start
// Reads do NOT see changes made by other transactions that committed after this one started
// Use: ONLY inside transactions (required for multi-document consistency)

// Summary table:
// Level        | Freshness    | Consistency  | Performance  | Rollback risk
// local        | Freshest     | Low          | Fastest      | Yes
// available    | Freshest     | Lowest       | Fastest      | Yes (orphans possible)
// majority     | Slightly old | High         | Medium       | No
// linearizable | Guaranteed   | Highest      | Slowest      | No
// snapshot     | Point-in-time| Transactional| Medium       | No (in transaction)
```

---

## Q5: What is causal consistency? How does MongoDB implement it?

**Causal consistency** guarantees that if operation B is causally dependent on operation A, any reader will see B's effects only after (or at the same time as) A's effects — regardless of which replica set member they read from.

The classic problem: write to primary (w:majority), then immediately read from a secondary — and the secondary hasn't replicated yet so the write appears "missing."

```javascript
// Without causal consistency:
// Thread 1: writes user's profile update (w: majority)
await db.users.updateOne({ _id: userId }, { $set: { bio: "New bio" } },
  { writeConcern: { w: "majority" } })

// Thread 2 (different connection, secondary read):
const user = await db.users.findOne({ _id: userId },
  { readPreference: "secondary" })
// user.bio might still be old! Secondary may not have the write yet.
// This violates "read your own writes" even though w:majority was used

// WITH causal consistency (client session):
const session = client.startSession({ causalConsistency: true })

// Write with majority concern
await db.users.updateOne(
  { _id: userId },
  { $set: { bio: "New bio" } },
  { session, writeConcern: { w: "majority" } }
)

// Read (even on secondary) will see the write above
const user = await db.users.findOne(
  { _id: userId },
  { session, readPreference: "secondary", readConcern: { level: "majority" } }
)
// user.bio = "New bio" — guaranteed!
// Even though we read from a secondary!

// How it works internally:
// 1. After each write, session records the operation's clusterTime and operationTime
// 2. When next read is issued, session sends afterClusterTime in the command
// 3. The queried node waits until it has replicated data up to that operationTime
// 4. Only then does it return the response

// Causal consistency guarantees:
// - Read your own writes: reads always reflect prior writes in the same session
// - Monotonic reads: a read never returns older data than a previous read
// - Monotonic writes: writes never go backward in causal order
// - Writes follow reads: a write in session S always happens after the last read in S
```

---

## Q6: What are retryable writes? What errors trigger a retry?

**Retryable writes** (MongoDB 3.6+) allow the driver to automatically retry certain write operations exactly once if they fail due to a transient network error or primary failover — without risk of duplicating data.

```javascript
// Enable retryable writes (default in MongoDB drivers 4.0+)
const client = new MongoClient(uri, { retryWrites: true })

// Retryable write operations:
// - insertOne, insertMany (ordered)
// - updateOne, replaceOne
// - deleteOne
// - findOneAndUpdate, findOneAndReplace, findOneAndDelete
// - bulkWrite (only if all ops are individually retryable)

// NOT retryable:
// - insertMany with ordered: false (partial success problem)
// - updateMany, deleteMany (not idempotent — retry could double-apply)

// Error types that trigger a retry:
// 1. TransientTransactionError (label attached to error)
// 2. Primary failover (NotWritablePrimary error) — driver retries on new primary
// 3. Network timeout on write (driver can't confirm if write succeeded)

// How retryable writes prevent duplicates:
// MongoDB tracks each write with a server-side "lsid" (logical session ID)
// and "txnNumber" (transaction number — incremented per write)
// If the same write arrives twice, MongoDB checks: "have I seen this txnNumber for this session?"
// YES → return the original result (idempotent response)
// NO  → apply the write and record it

// Example: write fails mid-flight (network timeout)
try {
  await db.orders.insertOne({ orderId: "123", status: "placed" })
  // Network dies — driver doesn't know if the write succeeded
  // With retryWrites: true, driver retries
  // If server got the first write: returns the original _id (no duplicate)
  // If server didn't get it: applies now
} catch (err) {
  // Error only propagates if retry also fails
}

// Retryable writes require:
// - Replica set (standalone MongoDB does not support retryable writes)
// - MongoDB 3.6+ server
// - retryWrites: true in MongoClient options (default in modern drivers)
```

---

## Q7: What is the difference between write concern `w:1`, `w:majority`, and `w:0`?

Write concern controls how many replica set members must acknowledge a write before the driver returns success.

```javascript
// w: 0 — Fire and forget
await db.logs.insertOne(
  { message: "Background task started", ts: new Date() },
  { writeConcern: { w: 0 } }
)
// Driver sends the write and immediately returns (no acknowledgment waited)
// If the primary crashes immediately after receiving the write but before writing to disk,
// the write is LOST — you'll never know
// Use: non-critical, high-volume logging where occasional loss is acceptable
// Performance: fastest (no round-trip wait)

// w: 1 — Primary acknowledged (default)
await db.products.updateOne(
  { _id: productId },
  { $inc: { quantity: -1 } },
  { writeConcern: { w: 1 } }
)
// Primary applies the write and sends acknowledgment
// Risk: primary crashes before replicating → secondaries roll back on election
//       (write is "rolled back" but application thought it succeeded)
// Use: most application writes where slight rollback risk is acceptable
// Performance: ~1-5ms (just the primary roundtrip)

// w: majority — Majority acknowledged
await db.accounts.updateOne(
  { _id: accountId },
  { $inc: { balance: -100 } },
  { writeConcern: { w: "majority", j: true } }
)
// Majority of voting members (e.g., 2 out of 3 in a 3-node set) must acknowledge
// Write is durable: even if primary crashes, majority has it → new primary has it
// Use: financial data, user-facing confirmations, anything requiring durability
// Performance: ~5-15ms (must wait for replication round-trip)

// j: true — Journal acknowledged
// Primary must write to the WiredTiger journal (journal is on disk)
// before acknowledging. Survives OS crashes.
await db.orders.insertOne(
  { orderId: "ORD-001", ... },
  { writeConcern: { w: "majority", j: true } }
)
// j: false → acknowledged after write is in memory (faster, but power loss = data loss)
// j: true  → acknowledged after write is journaled to disk (~1ms extra latency)

// Latency comparison (3-node replica set):
// w: 0         = 0ms    (no wait)
// w: 1         = 1-3ms  (primary RTT)
// w: 1, j:true = 2-5ms  (primary journal flush)
// w: majority  = 5-15ms (replication RTT to 2nd node)
// w: majority, j:true = 6-16ms

// Combining with read concern for strong consistency:
db.accounts.findOne({ _id: id }, { readConcern: { level: "majority" } })
// + all writes use writeConcern: { w: "majority" }
// = no rollback risk, no stale reads
```

---

## Q8: How does MongoDB implement snapshot isolation for transactions?

MongoDB uses **Multi-Version Concurrency Control (MVCC)** via the WiredTiger storage engine to implement snapshot isolation.

```javascript
// MVCC in WiredTiger:
// When a document is updated, WiredTiger does NOT overwrite the old value.
// Instead, it creates a new version of the document while keeping the old version.
// Old versions are kept until no active transaction needs them (then garbage-collected).

// Transaction snapshot:
// When a transaction starts, it records the current "snapshot" —
// a list of all in-progress transaction IDs visible at that moment.
// All reads in the transaction see:
//   - Committed data as of the snapshot timestamp
//   - The transaction's OWN uncommitted writes
//   - NOT other transactions' uncommitted writes
//   - NOT committed data from transactions that committed after snapshot was taken

// Example: two concurrent transactions
// T1 (balance transfer): starts at time 100
// T2 (interest calculation): starts at time 102

// T1: reads account A balance (sees value as of time 100: $1000)
// T2: reads account A balance (sees value as of time 102: $1000, same — nothing committed)
// T1: decrements account A balance by $100
// T1: commits at time 105

// T2: reads account A balance AGAIN
// → T2 still sees $1000 (its snapshot was at time 102, before T1 committed at 105)
// → T2 does NOT see T1's committed write (correct — snapshot isolation)

// Conflict detection (write-write conflicts):
// T1 and T2 both try to update account A
// T1 updates first → succeeds
// T2 tries to update the same document → WriteConflict error
// → T2 is automatically retried by withTransaction()

const accountId = ObjectId("...")

// Demonstrating snapshot isolation
const session = client.startSession()
await session.withTransaction(async () => {
  // Read at snapshot time
  const account = await db.accounts.findOne({ _id: accountId }, { session })
  // ... do calculations based on account.balance ...

  // Another transaction commits to account between the read and now
  // We still see our snapshot value — no phantom reads

  await db.accounts.updateOne(
    { _id: accountId, balance: account.balance },  // optimistic check
    { $set: { balance: newBalance } },
    { session }
  )
  // If another transaction updated balance between our read and this write:
  // → WriteConflict error → withTransaction() retries automatically
})
session.endSession()
```

---

## Q9: What is a write conflict in MongoDB transactions? How are they handled?

A **write conflict** occurs when two concurrent transactions attempt to modify the same document. MongoDB detects this using MVCC and raises a `WriteConflict` error on the second transaction.

```javascript
// Scenario: two concurrent transactions both modify account_123

// Transaction A (transfer $100 out)
session_A.withTransaction(async () => {
  await db.accounts.updateOne({ _id: "account_123" }, { $inc: { balance: -100 } }, { session: session_A })
  // T_A acquires a write lock on account_123
})

// Transaction B (apply fee $5) — starts concurrently
session_B.withTransaction(async () => {
  await db.accounts.updateOne({ _id: "account_123" }, { $inc: { balance: -5 } }, { session: session_B })
  // T_B tries to acquire a write lock on account_123
  // But T_A is already writing it!
  // → WriteConflict error thrown for T_B
  // → withTransaction() detects WriteConflict = TransientTransactionError label
  // → withTransaction() automatically retries T_B from the beginning
})

// WriteConflict error details:
// errorCode: 112
// label: "TransientTransactionError"
// The label tells the driver: "This error is transient — safe to retry"

// What withTransaction() does automatically:
// 1. Catches TransientTransactionError (WriteConflict, network errors, etc.)
// 2. Aborts the current transaction attempt
// 3. Sleeps briefly (exponential backoff in some drivers)
// 4. Retries the ENTIRE callback function from scratch
// 5. Up to a configurable max retry count

// Manual retry pattern (if not using withTransaction):
async function runWithRetry(txnFn, session, maxRetries = 3) {
  let attempt = 0
  while (attempt < maxRetries) {
    session.startTransaction()
    try {
      await txnFn(session)
      await session.commitTransaction()
      return
    } catch (err) {
      await session.abortTransaction()
      if (err.hasErrorLabel("TransientTransactionError") && attempt < maxRetries - 1) {
        attempt++
        continue   // retry
      }
      if (err.hasErrorLabel("UnknownTransactionCommitResult") && attempt < maxRetries - 1) {
        // Commit result unknown — retry the commit (NOT the whole transaction)
        attempt++
        try {
          await session.commitTransaction()
          return
        } catch (commitErr) {
          // handle commit retry error
        }
      }
      throw err   // non-retryable error
    }
  }
}
```

---

## Q10: When should you NOT use transactions in MongoDB?

```javascript
// CASE 1: Single-document updates (already atomic — no transaction needed)
// WRONG: wrapping a single-doc update in a transaction
await session.withTransaction(async () => {
  await db.products.updateOne(
    { _id: productId },
    { $inc: { quantity: -1, soldCount: 1 } }
  )
})
// RIGHT: use atomic operators directly (faster, simpler)
await db.products.updateOne(
  { _id: productId },
  { $inc: { quantity: -1, soldCount: 1 } }
)

// CASE 2: High-throughput insert pipelines
// Transactions add commit overhead (~5-10ms per transaction)
// If inserting 10,000 events per second in individual transactions:
// → 10,000 × 10ms = 100 seconds of commit overhead per second (impossible)
// RIGHT: batch inserts, accept eventual consistency for event logs

// CASE 3: Read-only operations (use read concern instead)
// WRONG:
await session.withTransaction(async () => {
  const order = await db.orders.findOne({ _id: orderId }, { session })
  const customer = await db.customers.findOne({ _id: order.customerId }, { session })
  return { order, customer }
})
// RIGHT: use a single $lookup aggregation or two sequential queries with read concern: majority

// CASE 4: Analytics / reporting pipelines
// Transactions hold snapshot isolation → can block garbage collection of old MVCC versions
// Long-running analytics transactions → WiredTiger cache pressure
// RIGHT: use Atlas Online Archive, Atlas Analytics Node, or a replica with read concern: snapshot

// CASE 5: Cross-collection operations that don't need atomicity
// Inserting an audit log doesn't need to be atomic with the main operation
// If the audit log insert fails, the main operation has already succeeded
// RIGHT: fire-and-forget the audit log with w:0 or handle failure separately

// WHEN TO USE TRANSACTIONS:
// ✓ Financial transfers between accounts
// ✓ Inventory reservation + order placement (both must succeed or neither)
// ✓ Document versioning (save new version + update "current version" pointer)
// ✓ Any operation touching 2+ documents where partial success is unacceptable
```

---

## Q11: What is the `maxTimeMS` option and how does it relate to transactions?

```javascript
// maxTimeMS: abort if the operation takes longer than this many milliseconds
// Applies to: individual operations AND transactions

// Per-operation timeout
db.orders.find({ status: "pending" }).maxTimeMS(5000)  // abort after 5 seconds

// Per-transaction: maxCommitTimeMS (for the commit specifically)
session.startTransaction({ maxCommitTimeMS: 10000 })

// Global transaction limit: transactionLifetimeLimitSeconds
// Default: 60 seconds
// Set on mongod:
db.adminCommand({
  setParameter: 1,
  transactionLifetimeLimitSeconds: 30   // more aggressive
})
// If a transaction is still active after this many seconds → auto-aborted
// Application receives: "Transaction ... has been aborted"

// Clock timeout calculation:
// 60-second default means:
// Transaction started at T=0
// Application code runs slowly (large dataset processing) to T=65
// At T=60: mongod aborts the transaction automatically
// Application's next operation in the session throws: TransactionAbortedDueToTimeout

// Best practice:
// Keep transactions short — under 1 second ideally, never more than 10 seconds
// If a business process takes 30 seconds, do NOT put it all in one transaction
// Instead: break into smaller atomic steps with compensating transactions for rollback

// MongoDB 6.1+: clientOperationMaxTimeMS at the MongoClient level
const client = new MongoClient(uri, {
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 10000
})
```

---

## Q12: How do you implement an idempotent operation using transactions?

**Idempotency** means an operation can be applied multiple times without changing the result beyond the first application. Critical for safe retries.

```javascript
// Non-idempotent (BAD — retrying this creates duplicate orders)
async function placeOrder(orderData) {
  await db.orders.insertOne(orderData)
  await db.inventory.updateOne(
    { productId: orderData.productId },
    { $inc: { quantity: -orderData.qty } }
  )
}
// If network fails after insertOne but before updateOne → inconsistent state
// If we retry the entire function → duplicate order inserted!

// Idempotent with idempotency key (GOOD)
async function placeOrderIdempotent(orderData, idempotencyKey) {
  const session = client.startSession()

  try {
    await session.withTransaction(async () => {
      // Check if this exact request was already processed
      const existing = await db.orders.findOne(
        { idempotencyKey: idempotencyKey },
        { session }
      )
      if (existing) {
        // Already processed — return existing result (safe to retry!)
        return existing
      }

      // Check inventory
      const inventory = await db.inventory.findOneAndUpdate(
        {
          productId: orderData.productId,
          quantity: { $gte: orderData.qty }
        },
        { $inc: { quantity: -orderData.qty } },
        { session, returnDocument: "after" }
      )
      if (!inventory) throw new Error("Out of stock")

      // Insert order with idempotency key
      const order = await db.orders.insertOne({
        ...orderData,
        idempotencyKey: idempotencyKey,     // unique key from client
        status: "placed",
        createdAt: new Date()
      }, { session })

      return order
    })
  } finally {
    session.endSession()
  }
}

// Client generates idempotency key (UUID):
const key = crypto.randomUUID()
// First call:  processes normally, stores key
// Second call: finds existing order → returns same result
// db.orders.createIndex({ idempotencyKey: 1 }, { unique: true, sparse: true })
```

---

## Q13: What is the distributed transaction protocol used in sharded cluster transactions?

```javascript
// Two-Phase Commit (2PC) for distributed transactions across shards

// Setup: orders sharded on customerId, inventory sharded on productId
// A single transaction updates both → involves 2 shards

// PHASE 1: PREPARE
// 1. Application sends transaction to mongos
// 2. mongos routes operations to respective shards
//    - shard1 (orders): updateOne({ customerId: "alice", status: "pending" → "completed" })
//    - shard2 (inventory): updateOne({ productId: "widget", quantity -= 1 })
// 3. mongos designates shard1 as the "coordinator shard"
// 4. Coordinator sends PREPARE to all participant shards
// 5. Each shard votes YES or NO (YES = can commit, NO = something failed)
// 6. Each shard that votes YES locks its writes (cannot abort unilaterally)

// PHASE 2: COMMIT OR ABORT
// If ALL votes are YES:
//   7. Coordinator writes a commit record to its own oplog
//   8. Coordinator sends COMMIT to all participant shards
//   9. Each shard commits and releases locks
//   10. mongos acknowledges to the application

// If ANY vote is NO:
//   7. Coordinator writes an abort record
//   8. Sends ABORT to all participants
//   9. Each shard rolls back and releases locks

// The coordinator's commit record is critical:
// If coordinator crashes AFTER writing the commit record but before sending COMMIT:
//   → On recovery, coordinator sees the commit record → re-sends COMMIT
//   → Ensures atomicity even across node failures

// Performance overhead per cross-shard transaction:
// - 2 extra network round-trips (prepare + commit)
// - Coordinator overhead
// - Lock contention across shards during prepare phase

// In MongoDB driver code:
const session = client.startSession()
await session.withTransaction(async () => {
  // This is transparently distributed by mongos
  await db.orders.updateOne({ _id: orderId }, { $set: { status: "completed" } }, { session })
  await db.inventory.updateOne({ productId: "widget" }, { $inc: { qty: -1 } }, { session })
  // mongos handles the 2PC coordination automatically
})
```

---

## Q14: How do you handle the `UnknownTransactionCommitResult` error?

```javascript
// UnknownTransactionCommitResult:
// This error label means: "The commit may or may not have succeeded."
// It occurs when the driver sends commitTransaction but doesn't get a response
// (e.g., network timeout, primary failover during commit).

// CRITICAL: You CANNOT safely retry the entire transaction in this case.
// The transaction may have committed. Retrying would re-execute all operations
// from scratch → potential duplicates.

// The CORRECT action: retry ONLY the commitTransaction call (not the whole txn)

async function commitWithRetry(session) {
  let retries = 3
  while (retries > 0) {
    try {
      await session.commitTransaction()
      console.log("Transaction committed successfully")
      return
    } catch (err) {
      if (err.hasErrorLabel("UnknownTransactionCommitResult") && retries > 1) {
        console.log("Retrying commit — result was unknown...")
        retries--
        // Do NOT re-run the transaction operations!
        // Just retry the commit — MongoDB uses idempotent commit
        continue
      }
      throw err   // non-retryable commit error
    }
  }
}

// withTransaction() handles this automatically:
// - TransientTransactionError → retries entire callback
// - UnknownTransactionCommitResult → retries only commitTransaction
// This is why withTransaction() is the strongly recommended pattern

// Why is re-committing safe?
// MongoDB's commit operation is idempotent — the coordinator has already
// written the commit record. Re-sending commitTransaction simply acknowledges
// what already happened; it doesn't re-apply the operations.
```

---

## Q15: What are distributed locks in MongoDB and how do you implement them?

```javascript
// Distributed lock: ensure only ONE process across multiple servers executes
// a critical section at a time.

// Implementation using findOneAndUpdate with TTL
// Lock document schema:
// { _id: "lock-name", owner: "server-1:process-123", expiresAt: ISODate(...) }

db.locks.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })   // TTL index
db.locks.createIndex({ _id: 1 }, { unique: true })

async function acquireLock(lockName, owner, ttlSeconds) {
  const expiresAt = new Date(Date.now() + ttlSeconds * 1000)
  try {
    const result = await db.locks.findOneAndUpdate(
      {
        _id: lockName,
        $or: [
          { owner: { $exists: false } },    // no lock held
          { expiresAt: { $lt: new Date() } } // expired lock
        ]
      },
      {
        $set: { owner: owner, expiresAt: expiresAt }
      },
      {
        upsert: true,
        returnDocument: "after",
        writeConcern: { w: "majority" }      // must be durable!
      }
    )
    return result !== null   // true = lock acquired
  } catch (err) {
    if (err.code === 11000) return false   // duplicate key = another process got it
    throw err
  }
}

async function releaseLock(lockName, owner) {
  await db.locks.deleteOne(
    { _id: lockName, owner: owner },    // only release OWN lock
    { writeConcern: { w: "majority" } }
  )
}

// Usage
const lockOwner = `${os.hostname()}-${process.pid}-${Date.now()}`
const acquired = await acquireLock("generate-monthly-report", lockOwner, 300)  // 5 min TTL

if (acquired) {
  try {
    await generateReport()
  } finally {
    await releaseLock("generate-monthly-report", lockOwner)
  }
} else {
  console.log("Another process is already generating the report")
}

// TTL ensures locks auto-expire if the owning process crashes (prevents deadlock)
// The owner field prevents process B from releasing process A's lock
// readConcern: "linearizable" should be used when checking lock status for highest safety
```
