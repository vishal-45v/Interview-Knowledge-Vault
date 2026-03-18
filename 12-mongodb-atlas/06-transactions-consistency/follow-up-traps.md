# Chapter 06 — Transactions & Consistency: Follow-Up Traps

---

## Trap 1: "MongoDB Transactions Work Like SQL Transactions"

**The interviewer asks**: "How are MongoDB transactions the same as PostgreSQL transactions?"

**Wrong answer**: "They're basically the same — start, commit, rollback, ACID guarantees."

**Why it's nuanced**:

```javascript
// Key DIFFERENCES from SQL transactions:

// 1. MongoDB transactions have a 60-second default lifetime limit
// SQL transactions can run indefinitely (for better or worse)
// → Design MongoDB transactions to complete in < 1 second

// 2. MongoDB transactions have a 16MB oplog entry limit
// SQL: no such limit on a single transaction's change set
// → Don't update 100,000 documents in a single MongoDB transaction

// 3. MongoDB: DDL inside transactions is limited
// SQL: CREATE TABLE / ALTER TABLE inside transactions is common
// → Cannot create indexes inside MongoDB transactions

// 4. No SAVEPOINT support in MongoDB
// SQL: SAVEPOINT sp1; ROLLBACK TO sp1; — partial transaction rollback
// MongoDB: no partial rollback — it's all or nothing

// 5. MongoDB transactions have overhead even for small operations
// SQL: lightweight transactions on single-row operations
// MongoDB: even simple two-doc transactions add commit latency (~5-15ms)
// The advice: use MongoDB's document model to AVOID transactions where possible

// The philosophical difference:
// SQL assumes transactions for all multi-row operations
// MongoDB assumes single-document atomicity covers 90% of cases
// MongoDB transactions are the escape hatch, not the default tool

// When a MongoDB interviewer asks about transactions:
// The RIGHT framing is: "Can I design the schema so I DON'T need a transaction?"
// If yes: embed the related data in one document → atomic by default
// If no: then use a transaction
```

---

## Trap 2: "withTransaction() Retries Automatically, So My Code Is Safe"

**The interviewer asks**: "You're using `withTransaction()` with retry logic. What happens if your callback has a side effect — like sending an HTTP request to a payment API?"

**Wrong answer**: "`withTransaction()` handles everything — I don't need to worry about it."

**Why it's wrong**:

```javascript
// DANGEROUS: side effect inside withTransaction() callback
await session.withTransaction(async () => {
  await db.orders.insertOne({ customerId, items, status: "placed" }, { session })

  // THIS CAN BE CALLED MULTIPLE TIMES if the transaction retries!
  const paymentResult = await stripe.charges.create({
    amount: 5000,
    currency: "usd",
    customer: customerId
  })
  // If the transaction gets WriteConflict and retries:
  // → stripe.charges.create is called AGAIN
  // → Customer is charged TWICE!

  await db.payments.insertOne({ chargeId: paymentResult.id, ... }, { session })
})

// CORRECT: side effects OUTSIDE the transaction
// Step 1: Create pending order atomically
let orderId
const session = client.startSession()
try {
  await session.withTransaction(async () => {
    const result = await db.orders.insertOne(
      { customerId, items, status: "pending_payment" },
      { session }
    )
    orderId = result.insertedId
  })
} finally {
  session.endSession()
}

// Step 2: Process payment AFTER transaction commits
const paymentResult = await stripe.charges.create({
  amount: 5000,
  currency: "usd",
  customer: customerId,
  idempotencyKey: orderId.toString()  // Stripe idempotency key prevents double charge
})

// Step 3: Update order status (new transaction — idempotent)
await db.orders.updateOne(
  { _id: orderId, status: "pending_payment" },
  { $set: { status: "paid", paymentId: paymentResult.id } }
)

// Rule: withTransaction() callback must be IDEMPOTENT
// Side effects (emails, HTTP calls, queue publishes) go AFTER the transaction
```

---

## Trap 3: "Read Concern `majority` Means I'm Reading the Latest Data"

**The interviewer asks**: "If I use `readConcern: majority`, am I always reading the most up-to-date data?"

**Wrong answer**: "Yes — majority read concern returns the latest committed data."

**Why it's wrong**:

```javascript
// readConcern: majority returns data that has been committed to a MAJORITY of nodes
// This is NOT necessarily the latest data — it's the majority-committed data

// Example:
// Primary: { balance: 1000 }  ← just updated (not yet replicated)
// Secondary 1: { balance: 900 }  ← previous value (majority-committed)
// Secondary 2: { balance: 900 }  ← previous value (majority-committed)

// Read with majority concern returns: { balance: 900 }
// The $1000 update hasn't reached majority yet!

// readConcern: majority guarantees:
// ✓ Data won't be rolled back (it's on majority)
// ✓ Durability — this data is persistent
// ✗ NOT necessarily the freshest (could be slightly behind)

// If you need THE freshest data (accept rollback risk):
db.accounts.findOne({ _id: id }, { readConcern: { level: "local" } })
// Returns: { balance: 1000 } ← latest, but could roll back

// If you need guaranteed-latest AND guaranteed-durable:
db.accounts.findOne({ _id: id }, { readConcern: { level: "linearizable" } })
// Returns: { balance: 1000 } ← latest AND guaranteed durable
// But: very slow (must confirm with majority that it's still primary)

// Real-world: for most financial reads, majority is the right choice
// "slightly behind but guaranteed safe" > "latest but could roll back"
```

---

## Trap 4: "Transactions Prevent All Concurrency Issues"

**The interviewer asks**: "If I use transactions, do I need to worry about concurrent access?"

**Wrong answer**: "Transactions handle all concurrency — I don't need to think about it."

**Why it's wrong**:

```javascript
// MongoDB uses SNAPSHOT ISOLATION, not SERIALIZABLE isolation
// Snapshot isolation prevents dirty reads, non-repeatable reads, phantom reads
// But it does NOT prevent all anomalies

// WRITE SKEW (snapshot isolation does NOT prevent this):
// Two transactions both check a constraint and both pass, but together they violate it

// Example: booking system — at least 1 doctor must be on call
// 2 doctors are on call: Dr. Alice, Dr. Bob
// TX1 (Alice takes day off): reads on-call list → 2 doctors → OK → removes Alice
// TX2 (Bob takes day off):   reads on-call list → 2 doctors → OK → removes Bob
// Both commit → 0 doctors on call! Constraint violated!

// Both transactions saw the correct snapshot (2 doctors), but together they're wrong
// This is write skew — snapshot isolation doesn't catch it

// FIX: add a counter document and check it, or use serializable reads

// Using $inc on a separate constraint document:
await session.withTransaction(async () => {
  // Atomically decrement and check constraint
  const roster = await db.roster.findOneAndUpdate(
    { _id: "on_call", count: { $gt: 1 } },  // must have > 1 before removing
    { $inc: { count: -1 } },
    { session }
  )
  if (!roster) throw new Error("Must maintain at least 1 on-call doctor")

  // Remove doctor from on-call list
  await db.oncall.deleteOne({ doctorId: ObjectId(doctorId) }, { session })
})
// Now TX1 decrements 2→1, TX2 tries to decrement 1→0 but count > 1 check fails
// TX2 is aborted → constraint maintained

// Other concurrency issues transactions DON'T fully prevent:
// 1. Application-level logic errors (incorrect use of snapshot data)
// 2. Hotspot contention (high-traffic documents still cause write conflicts/queuing)
// 3. Deadlocks (two transactions waiting for each other's locks)
//    → withTransaction() detects and retries, but still represents wasted work
```

---

## Trap 5: "You Can Use Transactions on Standalone MongoDB"

**The interviewer asks**: "For development, I run a standalone MongoDB instance (not replica set). Can I use transactions?"

**Wrong answer**: "Transactions work on any MongoDB instance."

**Why it's wrong**:

```javascript
// Transactions REQUIRE a replica set (or sharded cluster, which is built on replica sets)
// Standalone MongoDB does NOT support multi-document transactions

// If you try on a standalone:
await session.withTransaction(async () => {
  await db.orders.insertOne({ ... }, { session })
})
// Error: "Transaction numbers are only allowed on a replica set member or mongos"

// For LOCAL DEVELOPMENT: run a single-node replica set
// This gives you full transaction support without the overhead of 3 nodes

// mongod configuration for local dev (replica set of 1 node):
// mongod --replSet "rs0" --dbpath /data/db

// Then in mongosh:
rs.initiate()
// Initiates a 1-node replica set

// mongod.conf equivalent:
// replication:
//   replSetName: "rs0"

// With Docker Compose for local dev:
// services:
//   mongo:
//     image: mongo:7.0
//     command: ["--replSet", "rs0", "--bind_ip_all"]
//     ports: ["27017:27017"]

// After starting, init the replica set:
// db.adminCommand({ replSetInitiate: { _id: "rs0", members: [{ _id: 0, host: "localhost:27017" }] } })

// The fix for existing standalone deployments:
// Must convert to replica set — not a hot swap, requires planned maintenance
// Atlas: all Atlas clusters are replica sets (even the free M0 tier)
```

---

## Trap 6: "Aborting a Transaction Is Fast — It Just Discards Changes"

**The interviewer asks**: "If a transaction aborts, it just discards the in-memory changes, right? So aborts are free?"

**Wrong answer**: "Yes — abort is instant, no overhead."

**Why it's nuanced**:

```javascript
// Abort IS generally fast — MongoDB discards uncommitted writes
// But there are OVERHEAD costs:

// 1. Lock release latency
// All locks held by the transaction must be released on abort
// Other operations waiting on those locks are unblocked
// → High abort rate = high contention = cascading slowdowns

// 2. Network round-trip to send ABORT
// For sharded clusters: coordinator must send ABORT to all participant shards
// Each participating shard must acknowledge the ABORT
// → Cross-shard abort is NOT instant

// 3. Oplog overhead for distributed abort (sharded clusters)
// The coordinator writes an abort record to the oplog even on abort
// → Write overhead on abort, not zero cost

// 4. WriteConflict retries burn CPU
// If transaction T1 conflicts with T2, T2 is aborted and retried
// On retry, T2 might conflict again → retry loop
// → Under high contention, retry CPU cost dominates

// 5. Client-side overhead
// Driver must handle the error, call abortTransaction(), and notify the application
// → Still involves network calls and error handling code

// Monitoring abort rate:
db.serverStatus().transactions
// {
//   totalStarted: 10000,
//   totalCommitted: 9800,
//   totalAborted: 180,      ← high abort rate = contention problem
//   totalContactedParticipants: 15000,
//   currentOpen: 12
// }

// High abort rate causes:
// - Increased latency for successful transactions (waiting for retries)
// - Write throughput reduction
// - Lock contention cascades
// Fix: optimize hot documents, batch operations, reduce transaction scope
```

---

## Trap 7: "Linearizable Read Concern Is Perfect for All Critical Reads"

**The interviewer asks**: "I need perfectly consistent reads — should I just always use `readConcern: linearizable`?"

**Wrong answer**: "Yes — linearizable gives you the strongest guarantees so it's always the best choice."

**Why it's wrong**:

```javascript
// readConcern: linearizable has SEVERE limitations:

// 1. ONLY works for single document point queries
db.accounts.findOne({ _id: id })        // ✓ linearizable works
db.accounts.find({ status: "active" })  // ✗ linearizable NOT SUPPORTED for multi-doc queries

// 2. ONLY works when reading from primary
// You cannot use linearizable on secondaries or with read preference secondary
db.accounts.findOne(
  { _id: id },
  { readPreference: "secondary", readConcern: { level: "linearizable" } }
)
// Error: "linearizable read concern is not supported with secondaryPreferred"

// 3. Very slow (adds ~10-50ms per query)
// MongoDB must verify with majority that it's still the primary before responding
// → If network is slow, every linearizable read waits for majority confirmation
// → Under network partition: may wait indefinitely (eventual timeout)

// 4. maxTimeMS is REQUIRED with linearizable
// Without it, a partitioned primary might block forever
db.accounts.findOne(
  { _id: id },
  { readConcern: { level: "linearizable" }, maxTimeMS: 5000 }
)
// Without maxTimeMS: your app hangs until partition resolves

// When to actually use linearizable:
// - Distributed lock check: "is this lock document still unclaimed?"
// - Leader election: "am I the only node that thinks it's elected?"
// - Exactly-once processing check: "was this event already processed?"

// For most financial reads, majority is sufficient:
// majority = durable + non-rollback-able + fast
// linearizable = additionally guarantees external consistency across multiple clients
// The extra guarantee of linearizable is rarely needed in practice
```

---

## Trap 8: "Transactions Lock the Entire Collection"

**The interviewer asks**: "Won't a transaction that updates 10 documents lock the entire collection and block all other writes?"

**Wrong answer** (common misconception): "Yes, MongoDB transactions acquire collection-level locks."

**The truth**:

```javascript
// MongoDB WiredTiger uses document-level locking (row-level locking)
// A transaction ONLY locks the specific documents it writes to
// Other documents in the same collection are NOT blocked

// Example:
// Transaction A: updates Order #123
// Transaction B: updates Order #456 (same collection, different document)
// → B is NOT blocked! They work on different documents simultaneously

// What DOES get locked:
// - The specific document being written (exclusive lock for the write duration)
// - Index entries being modified (shared/exclusive depending on operation)
// - At the database level: an intent lock (allows concurrent operations)

// What CAN cause blocking:
// 1. Two transactions write to the SAME document (WriteConflict — one wins, one retries)
// 2. DDL operations (createIndex, drop) — these wait for all transactions to finish
// 3. Collection-level locks for metadata operations

// Checking lock waits:
db.currentOp({ waitingForLock: true })

// Real-world contention pattern:
// "Hot document" problem: all transactions update the SAME counter document
// This creates a single-document bottleneck regardless of transaction isolation
// Fix: use $inc (atomic), or shard the counter across multiple documents

// Monitoring:
db.serverStatus().locks
// Shows lock acquisition/wait stats per lock mode
// LockMode: R = shared read, W = exclusive write, r = intent read, w = intent write
```

---

## Trap 9: "A Transaction That Reads Then Writes Is Safe From Concurrent Modifications"

**The interviewer asks**: "I read a document's balance, check if it's sufficient, then write a debit. This is in a transaction — is it safe?"

**The subtle answer**:

```javascript
// SCENARIO: read-then-write in a transaction
await session.withTransaction(async () => {
  const account = await db.accounts.findOne({ _id: id }, { session })
  // account.balance = 1000

  if (account.balance < 100) throw new Error("Insufficient funds")

  await db.accounts.updateOne(
    { _id: id },
    { $inc: { balance: -100 } },
    { session }
  )
})

// Is this safe? Mostly YES — with an important caveat:
// Snapshot isolation: you read the snapshot value, then write
// If ANOTHER transaction writes to account._id BETWEEN your read and your write:
// → Your write triggers a WriteConflict error (detected by MVCC)
// → withTransaction() retries automatically
// → On retry, you re-read the current balance (not the stale snapshot)
// → Correct result

// HOWEVER: the check + decrement pattern above can be simplified to be safer:
// Instead of read → check → write, use ATOMIC conditional write:
await session.withTransaction(async () => {
  const account = await db.accounts.findOneAndUpdate(
    {
      _id: id,
      balance: { $gte: 100 }    // condition AND update in one atomic operation
    },
    { $inc: { balance: -100 } },
    { session, returnDocument: "after" }
  )
  if (!account) throw new Error("Insufficient funds")
})
// This is more efficient:
// - No read-then-write round trip
// - The condition is evaluated atomically with the update
// - WriteConflict is less likely (the operation is shorter)
// - Simpler code

// When the read-then-write pattern IS needed:
// Complex logic between read and write:
// - Calculations based on multiple documents
// - Business rules requiring multiple field checks
// - Cross-document invariants
// In these cases: transactions + snapshot isolation + automatic WriteConflict retry = safe
```

---

## Trap 10: "Transactions Are Transparent to the Aggregation Pipeline"

**The interviewer asks**: "Can I run any aggregation pipeline inside a transaction?"

**Wrong answer**: "Yes — all aggregation stages work inside transactions."

**The restrictions**:

```javascript
// Aggregation stages NOT allowed inside transactions:

// 1. $out — writes results to a new/existing collection
await session.withTransaction(async () => {
  await db.orders.aggregate([
    { $group: { _id: "$status", count: { $sum: 1 } } },
    { $out: "orderStats" }    // ERROR: not allowed inside transaction
  ], { session })
})

// 2. $merge — merges results into an existing collection
await session.withTransaction(async () => {
  await db.orders.aggregate([
    { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
    { $merge: { into: "customerStats" } }   // ERROR inside transaction
  ], { session })
})

// 3. $lookup referencing a sharded collection (in most configurations)
// Cross-shard $lookup in transactions has significant restrictions

// 4. $indexStats — returns index usage statistics (not allowed in transactions)

// What IS allowed:
await session.withTransaction(async () => {
  // $match, $group, $project, $sort, $unwind, $addFields, etc. — all fine
  const results = await db.orders.aggregate([
    { $match: { customerId: ObjectId(id) } },
    { $group: { _id: "$status", total: { $sum: "$amount" } } }
  ], { session }).toArray()

  // Use results for conditional writes
  const total = results.find(r => r._id === "completed")?.total || 0
  await db.customers.updateOne(
    { _id: ObjectId(id) },
    { $set: { lifetimeValue: total } },
    { session }
  )
})

// Summary: read-only aggregations are fine in transactions
// Write-producing aggregation stages ($out, $merge) must be OUTSIDE transactions
// Solution: run the aggregation first, then use the results inside a transaction
```

---

## Trap 11: "retryWrites: true Protects Against All Write Failures"

**The interviewer asks**: "I have `retryWrites: true`. Am I fully protected against write failures?"

**Wrong answer**: "Yes — retryable writes handle all failure scenarios."

**What retryable writes do NOT protect against**:

```javascript
// retryWrites: true ONLY retries on TransientTransactionError:
// - Primary failover (NotPrimary error)
// - Network timeout where write result is unknown
// - Server selection timeout during the write

// retryWrites does NOT retry on:

// 1. Application errors (business logic errors)
//    - "Insufficient funds" → not retried (not transient)
//    - Validation errors → not retried

// 2. Duplicate key errors (11000)
//    - Cannot safely retry: already succeeded
//    - Or: genuinely duplicate — retrying won't help

// 3. Write concern failures (WriteConcernError)
//    - If w:majority fails due to insufficient replica set members
//    - → retryable write will retry, might fail again

// 4. Shard key violations on updateMany/deleteMany
//    - These operations are NOT retryable (not idempotent)
//    - updateMany({ status: "pending" }, $set: ... ) → NOT retried on failure
//    - deleteMany({ expiredAt: { $lt: now } }) → NOT retried

// 5. Standalone MongoDB (no replica set)
//    - retryWrites: true is ignored on standalone

// Operations that ARE retryable:
// - insertOne, insertMany (ordered:true only)
// - updateOne, replaceOne
// - deleteOne
// - findOneAndUpdate, findOneAndDelete, findOneAndReplace
// - Single-op bulkWrite

// Best practices beyond retryWrites:
// - Add idempotency keys to critical insertions
// - Use writeConcern: majority for durability
// - Handle business errors separately from transient errors
// - For updateMany/deleteMany: implement your own retry with state tracking
```

---

## Trap 12: "Snapshot Read Concern Works Outside Transactions"

**The interviewer asks**: "Can I use `readConcern: snapshot` for a regular find operation outside of a transaction?"

**Wrong answer**: "Yes — snapshot isolation is available for all reads."

**The truth**:

```javascript
// readConcern: "snapshot" REQUIRES a transaction (or causal session in specific cases)
// It is NOT available for standalone reads outside a transaction

// WRONG:
db.orders.find({ status: "pending" }).readConcern("snapshot")
// Error: "readConcern snapshot is only supported in a transaction"

// CORRECT: snapshot inside a transaction
const session = client.startSession()
session.startTransaction({ readConcern: { level: "snapshot" } })
const orders = await db.orders.find({ status: "pending" }, { session }).toArray()
const customers = await db.customers.find({}, { session }).toArray()
// Both reads see the same point-in-time snapshot
// No phantom reads between the two find() calls
await session.commitTransaction()
session.endSession()

// If you want snapshot-like reads outside a transaction:
// Use readConcern: "majority" — it gives you a committed snapshot (slightly older)
// It's not identical to snapshot isolation but provides consistent committed reads

// Alternative: $lookup in a single aggregation stage
// MongoDB evaluates the pipeline atomically at a single point in time
// (though not with the same guarantees as a full snapshot isolation transaction)

// The practical rule:
// For consistent multi-document reads at the SAME logical time:
// → Use a transaction with readConcern: snapshot
// For single-document reads with durability guarantee:
// → readConcern: majority (or linearizable for strictest)
```
