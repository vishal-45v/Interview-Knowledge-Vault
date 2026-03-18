# Chapter 06 — Transactions & Consistency: Structured Answers

---

## Answer 1: Explain MongoDB's ACID Guarantees

**Point**: MongoDB provides ACID guarantees at the document level by default, and full multi-document ACID transactions since MongoDB 4.0.

**Reason**: The document model often eliminates the need for multi-document transactions — related data is embedded, and a single document update is always atomic. When true multi-document atomicity is needed, transactions provide it.

**Example**:

```javascript
// A: Atomicity — all or nothing
// Single-doc atomic: no transaction needed
await db.products.updateOne(
  { _id: id },
  { $inc: { quantity: -1, soldCount: 1 }, $set: { lastSoldAt: new Date() } }
)

// Multi-doc atomic: use transaction
const session = client.startSession()
await session.withTransaction(async () => {
  await db.accounts.updateOne({ _id: fromId }, { $inc: { balance: -amount } }, { session })
  await db.accounts.updateOne({ _id: toId },   { $inc: { balance: +amount } }, { session })
})

// C: Consistency — schema validators enforce invariants
db.runCommand({ collMod: "orders", validator: { $jsonSchema: { required: ["total"] } } })

// I: Isolation — snapshot isolation via MVCC
// Concurrent transactions see consistent snapshots, not each other's uncommitted data

// D: Durability — j: true + w: majority
await db.orders.insertOne({ ... }, { writeConcern: { w: "majority", j: true } })
// Data persists on disk on majority of nodes — survives crashes
```

**Pitfall**: Overusing transactions. Single-document operations are atomic by default — wrapping them in transactions adds overhead without benefit.

---

## Answer 2: How Do You Choose the Right Write Concern?

**Point**: Write concern is a sliding scale between performance and durability. Match it to the criticality of the data being written.

**Example**:

```javascript
// w: 0 — Logging, metrics, analytics
await db.clickEvents.insertOne({ userId, page, ts: new Date() },
  { writeConcern: { w: 0 } })   // fire and forget — OK to lose a click event

// w: 1 — Default for most app writes
await db.sessions.updateOne({ token }, { $set: { lastSeen: new Date() } })
// Session refresh — if it rolls back, user just gets logged out. Acceptable.

// w: majority — Financial, user-facing confirmations
await db.payments.insertOne({ amount: 5000, status: "processed" },
  { writeConcern: { w: "majority", j: true } })
// Cannot accept rollback on a payment record

// w: majority + j: true — Compliance, audit logs
await db.gdprAuditLog.insertOne({ action: "deletion", userId, at: new Date() },
  { writeConcern: { w: "majority", j: true } })
// Must survive power outage AND primary failure

// Latency cost in a 3-node replica set:
// w:0         ≈  0ms  (no wait)
// w:1         ≈  2ms  (primary RTT)
// w:1, j:true ≈  3ms  (primary disk flush)
// w:majority  ≈ 10ms  (replication RTT)
// w:majority, j:true ≈ 11ms
```

---

## Answer 3: Explain Snapshot Isolation and MVCC in MongoDB

**Point**: MongoDB uses Multi-Version Concurrency Control (MVCC) in WiredTiger. When a transaction reads data, it sees a snapshot of the database at the transaction's start time. Concurrent writes create new versions; the transaction never sees them.

**Example**:

```javascript
// T1 starts at time 100, T2 starts at time 102

// T1 reads account: sees { balance: 1000 } (snapshot at T=100)
// T2 reads account: sees { balance: 1000 } (snapshot at T=102, same — no commits)

// T1 writes: { balance: 900 } — new version created in WiredTiger
// T1 commits at T=105

// T2 reads account AGAIN — still sees { balance: 1000 }
// T2's snapshot was taken at T=102, before T1 committed at T=105
// T2 will NEVER see T1's write within T2's current snapshot

// This prevents:
// Dirty reads:         T2 cannot see T1's uncommitted data (balance mid-write)
// Non-repeatable reads: T2 reads account twice, always gets same value
// Phantom reads:       T2's range scans return consistent results throughout

// If T2 now also writes to the same account:
// → T2's write conflicts with T1's committed write
// → WriteConflict error (112) → withTransaction() retries T2 automatically

// What MVCC costs:
// - WiredTiger must keep OLD versions of documents while transactions are active
// - Long-running transactions → WiredTiger cache grows with old versions
// - Eventually: cache evictions → performance degradation
// Keep transactions short: < 1 second ideal, < 10 seconds maximum

const session = client.startSession()
session.startTransaction({ readConcern: { level: "snapshot" } })
const a = await db.accounts.findOne({ _id: id1 }, { session })
const b = await db.accounts.findOne({ _id: id2 }, { session })
// a and b are from the SAME snapshot — consistent view of the world
await session.commitTransaction()
session.endSession()
```

---

## Answer 4: How Do You Implement Exactly-Once Message Processing?

**Point**: Exactly-once processing requires idempotency checking — recording that an event has been processed, atomically with the processing itself, and checking before re-processing.

**Example**:

```javascript
// Pattern: "Mark as processed" atomically with side-effect-free work
// Side effects (emails, HTTP calls) go AFTER the idempotency-protected section

async function processOrderEvent(event) {
  const session = client.startSession()

  let alreadyProcessed = false

  try {
    await session.withTransaction(async () => {
      // Try to insert a "processed" marker — unique on event ID
      try {
        await db.processedEvents.insertOne(
          { _id: event.id, processedAt: new Date() },
          { session }
        )
      } catch (err) {
        if (err.code === 11000) {
          // Already processed — skip safely
          alreadyProcessed = true
          return
        }
        throw err
      }

      if (!alreadyProcessed) {
        // Do the actual work (idempotent DB writes only)
        await db.orders.updateOne(
          { _id: event.orderId },
          { $set: { status: "confirmed", confirmedAt: new Date() } },
          { session }
        )
      }
    })
  } finally {
    session.endSession()
  }

  // Side effects OUTSIDE transaction
  if (!alreadyProcessed) {
    await sendConfirmationEmail(event.orderId)
  }
}

db.processedEvents.createIndex({ _id: 1 }, { unique: true })
db.processedEvents.createIndex({ processedAt: 1 }, { expireAfterSeconds: 604800 }) // 7 days
```

---

## Answer 5: Walk Through a Distributed Transaction on a Sharded Cluster

**Point**: Sharded cluster transactions use a two-phase commit protocol coordinated by a "coordinator shard." The protocol ensures atomicity even across multiple shards and despite node failures.

**Example**:

```javascript
// Cluster topology:
// Shard A: contains orders collection (sharded on customerId)
// Shard B: contains inventory collection (sharded on productId)
// mongos: router

const session = client.startSession()
await session.withTransaction(async () => {
  // Operation 1 → routed to Shard A
  await db.orders.insertOne({
    customerId: "alice",
    items: [{ productId: "widget", qty: 1 }],
    status: "placed"
  }, { session })

  // Operation 2 → routed to Shard B
  await db.inventory.updateOne(
    { productId: "widget", qty: { $gte: 1 } },
    { $inc: { qty: -1 } },
    { session }
  )
  // If this fails: transaction aborts → Shard A's insertOne is rolled back
})

// INTERNAL 2PC FLOW:
// 1. mongos routes Op1 to Shard A, Op2 to Shard B
// 2. mongos designates Shard A as "coordinator" (first shard touched)
// 3. Application calls commitTransaction
// 4. mongos sends PREPARE to Shard A and Shard B
// 5. Shard A: writes prepare record to oplog, votes YES
//    Shard B: writes prepare record to oplog, votes YES
// 6. Coordinator (Shard A) writes COMMIT record to its oplog (point of no return)
// 7. Coordinator sends COMMIT to Shard B
// 8. Both shards apply changes and release locks
// 9. mongos returns success to application

// If Shard A crashes AFTER step 6 but before step 7:
// On Shard A recovery: sees commit record → re-sends COMMIT to Shard B
// Transaction still completes correctly!

// Performance overhead vs single-shard:
// Extra round trips: 2 (prepare + commit acknowledgment)
// Extra oplog writes: 3 (prepare on each shard + commit on coordinator)
// Recommendation: design sharding keys to keep related data on same shard
```

---

## Answer 6: Causal Consistency — Read Your Own Writes Across Nodes

**Point**: Causal consistency ensures that within a client session, every operation sees the results of previous operations in that session, even when reads are routed to secondaries.

**Example**:

```javascript
// Without causal consistency: write to primary, read from secondary, see stale data
const session = client.startSession({ causalConsistency: true })

// Write with strong durability
await db.profiles.updateOne(
  { userId: ObjectId(userId) },
  { $set: { avatarUrl: newUrl } },
  { session, writeConcern: { w: "majority" } }
)
// Session records: "operation time = T"

// Read — possibly on a secondary
const profile = await db.profiles.findOne(
  { userId: ObjectId(userId) },
  {
    session,
    readPreference: "secondary",
    readConcern: { level: "majority" }
  }
)
// MongoDB sends afterClusterTime: T in the read command
// Secondary waits until it has replicated data up to time T
// Returns: { avatarUrl: newUrl } ← sees the write!

session.endSession()

// Four guarantees of causal consistency:
// 1. Read your writes
// 2. Monotonic reads (read at time T, next read returns T or later)
// 3. Monotonic writes (writes don't "go backward")
// 4. Writes follow reads (a write happens after the data it read was current)
```

---

## Answer 7: Transaction Error Handling — TransientTransactionError vs UnknownTransactionCommitResult

**Point**: Two distinct error labels require different handling strategies. Confusing them can cause either silent data duplication or unnecessary failures.

**Example**:

```javascript
async function robustTransaction(session, txnFn) {
  while (true) {
    session.startTransaction({
      readConcern:  { level: "snapshot" },
      writeConcern: { w: "majority" }
    })

    try {
      await txnFn(session)

      // Inner retry loop for commit (only for UnknownTransactionCommitResult)
      while (true) {
        try {
          await session.commitTransaction()
          return   // success
        } catch (commitErr) {
          if (commitErr.hasErrorLabel("UnknownTransactionCommitResult")) {
            // Commit may or may not have succeeded
            // Safe to retry ONLY the commit (not the entire txn)
            console.log("Retrying commit...")
            continue
          }
          throw commitErr   // non-retryable commit error
        }
      }

    } catch (txnErr) {
      await session.abortTransaction()

      if (txnErr.hasErrorLabel("TransientTransactionError")) {
        // Safe to retry the ENTIRE transaction from scratch
        // (write conflict, network error, etc.)
        console.log("Retrying transaction due to transient error...")
        continue
      }
      throw txnErr   // application error — do not retry
    }
  }
}

// withTransaction() does this automatically:
await session.withTransaction(txnFn)  // preferred — handles all error cases

// Key distinction:
// TransientTransactionError:
//   → Transaction definitely did NOT commit
//   → Safe to retry entire callback from scratch
//   → Examples: WriteConflict (112), network timeout on op, primary election

// UnknownTransactionCommitResult:
//   → Transaction MAY OR MAY NOT have committed (network timeout on commit)
//   → ONLY retry the commitTransaction() call
//   → Do NOT re-run the callback (could duplicate work)
```

---

## Answer 8: When to Use Transactions vs. Atomic Single-Document Operations

**Point**: MongoDB's document model is designed to make transactions unnecessary for the majority of use cases. The rule is: can I embed the related data? If yes, I avoid the transaction entirely.

**Example**:

```javascript
// CASE 1: Shopping cart — NO transaction needed
// The entire cart is ONE document
db.carts.updateOne(
  { userId: ObjectId(userId), "items.productId": ObjectId(productId) },
  { $inc: { "items.$.quantity": 1 } }
)
// OR add new item:
db.carts.updateOne(
  { userId: ObjectId(userId) },
  { $push: { items: { productId: ObjectId(productId), qty: 1, price: 29.99 } } }
)
// Single document → atomic by default → no transaction

// CASE 2: Blog post + view count — NO transaction needed
db.posts.updateOne(
  { _id: ObjectId(postId) },
  {
    $inc: { viewCount: 1 },
    $set: { lastViewedAt: new Date() }
  }
)
// All changes in ONE document → atomic

// CASE 3: Transfer between accounts — MUST use transaction
// Two different documents, both must succeed
await session.withTransaction(async () => {
  await db.accounts.updateOne({ _id: fromId }, { $inc: { balance: -amount } }, { session })
  await db.accounts.updateOne({ _id: toId },   { $inc: { balance: +amount } }, { session })
})
// Cannot embed both accounts in one document — they're independent entities

// CASE 4: Order + inventory — CAN avoid transaction with careful schema
// Option A (with transaction):
await session.withTransaction(async () => {
  await db.orders.insertOne({ items }, { session })
  await db.inventory.updateOne({ productId }, { $inc: { qty: -1 } }, { session })
})

// Option B (atomic single op — no transaction):
const result = await db.inventory.findOneAndUpdate(
  { productId, qty: { $gte: 1 } },    // check and decrement atomically
  { $inc: { qty: -1 } }
)
if (!result) throw new Error("Out of stock")
// Then insert the order (single-doc insert is atomic)
await db.orders.insertOne({ items, inventoryReserved: true })
// If order insert fails: inventory is already decremented
// Need compensating action or accept rare inconsistency (depends on business rules)
// For strict atomicity: use transaction
```

---

## Answer 9: What Is a "Hot Document" and How Do Transactions Exacerbate It?

**Point**: A hot document is a single MongoDB document that is written to by many concurrent operations. Transactions holding write locks on hot documents create contention, queuing, and degraded throughput.

**Example**:

```javascript
// Hot document: a global product inventory counter
// 1,000 concurrent order placements, all updating the same product:
{ _id: "product-widget", availableQty: 500 }

// All 1,000 transactions try to update this one document simultaneously
// MongoDB serializes the writes → only one at a time → queue forms
// Result: 999 transactions waiting, latency spikes to seconds

// DIAGNOSIS:
db.currentOp({
  "ns": "mydb.inventory",
  waitingForLock: true
})
// Shows: hundreds of operations waiting for lock on product-widget

// FIX 1: Sharding (for reads)
// Shard inventory collection on productId → hot product goes to one shard
// Doesn't fully fix write contention within the shard

// FIX 2: Write patterns — use $inc (already the best atomic option)
// Already using $inc — that's fine. The hotspot is the document itself.

// FIX 3: Bucket inventory (reduces contention per bucket)
// Instead of one doc with qty: 500, use 5 docs with qty: 100 each
// Random selection of which bucket to decrement
{
  _id: "product-widget-bucket-1",
  productId: "product-widget",
  bucketQty: 100
}
// Random bucket selection spreads the write load:
const bucket = Math.floor(Math.random() * 5) + 1
await db.inventory.findOneAndUpdate(
  { _id: `product-widget-bucket-${bucket}`, bucketQty: { $gte: 1 } },
  { $inc: { bucketQty: -1 } }
)
// 5x less contention per document (5 independent lock queues)

// FIX 4: Atlas Global Clusters or separate inventory service with Redis
// Redis DECR is ~10x faster than MongoDB findOneAndUpdate for hot counters
// Sync to MongoDB periodically

// MONITORING: track lock wait time
db.serverStatus().globalLock.currentQueue
// { total: 0, readers: 0, writers: 0 }  ← should be near zero under normal load
// If writers > 10 sustained → investigate hot documents
```

---

## Answer 10: How Do Change Streams Interact With Transactions?

**Point**: Change streams receive transaction changes as a group — when a transaction commits, all its changes appear atomically as a single batch in the change stream, tagged with the same `lsid` and `txnNumber`.

**Example**:

```javascript
// Transaction that updates two documents
const session = client.startSession()
await session.withTransaction(async () => {
  await db.orders.insertOne({ customerId: "alice", total: 100 }, { session })
  await db.inventory.updateOne({ productId: "widget" }, { $inc: { qty: -1 } }, { session })
})

// Change stream consumer sees these events AFTER the transaction commits:
const stream = db.watch()

for await (const change of stream) {
  console.log(change)
  // Event 1: { operationType: "insert", ns: "mydb.orders", lsid: {...}, txnNumber: 5 }
  // Event 2: { operationType: "update", ns: "mydb.inventory", lsid: {...}, txnNumber: 5 }
  // Both have the SAME lsid and txnNumber → they're from the same transaction
  // Both appear AFTER the commit → you never see partial transaction state
}

// Handling transaction events in a change stream consumer:
for await (const change of stream) {
  if (change.lsid && change.txnNumber !== undefined) {
    // This change is part of a transaction
    // Group with other changes from same lsid + txnNumber
    console.log(`Transaction event: txn ${change.txnNumber}`)
  } else {
    // Regular (non-transactional) change
    console.log(`Non-transactional change`)
  }
}

// Important: aborted transactions produce NO change stream events
// Change streams only see committed data — exactly what consumers need
// Resume tokens work correctly across transaction boundaries:
const resumeAfter = lastChangeToken
db.watch([], { resumeAfter })   // picks up correctly from the resume point
```
