# Chapter 01 — MongoDB Fundamentals: Structured Answers

Model answers you can adapt for technical interviews. Each answer follows the PREP framework: **Point → Reason → Example → Point (summary)**.

---

## Answer 1: "Explain MongoDB's document model and why it differs from RDBMS"

**Point**: MongoDB stores data as self-describing BSON documents in collections, not rows in tables.

**Reason**: This enables storing related data together (denormalization), supports varying structures per document, and maps naturally to how application code represents objects (JSON/objects), reducing the object-relational impedance mismatch.

**Example**:
```javascript
// RDBMS approach — 3 tables, multiple JOINs needed
// TABLE users: id, first_name, last_name, email
// TABLE addresses: id, user_id, street, city, zip
// TABLE phone_numbers: id, user_id, number, type

// MongoDB approach — one document, one read
{
  _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
  firstName: "Alice",
  lastName: "Johnson",
  email: "alice@example.com",
  address: {
    street: "123 Main St",
    city: "Austin",
    state: "TX",
    zip: "78701"
  },
  phones: [
    { type: "mobile", number: "+1-512-555-0100" },
    { type: "work", number: "+1-512-555-0200" }
  ]
}

// Single query, no JOIN
db.users.findOne({ email: "alice@example.com" })
```

**Summary**: The document model reduces read latency by co-locating related data, enables horizontal scaling (each document is self-contained), and eliminates migration overhead for schema changes since MongoDB enforces no schema by default.

**Caveats to mention**: The document model trades off data duplication for read performance. Updates to duplicated data require updating multiple documents. Multi-document atomicity requires explicit transactions (MongoDB 4.0+).

---

## Answer 2: "What is BSON and what BSON types matter in production?"

**Point**: BSON is MongoDB's binary serialization format — a superset of JSON with richer types and binary encoding for faster traversal.

**Reason**: JSON is text-based and has limited type support (one number type, no dates, no binary). BSON adds types needed for real applications while keeping documents compact.

**Example**:
```javascript
db.products.insertOne({
  // String — UTF-8
  name: "Widget Pro",

  // Decimal128 — CRITICAL for financial values
  // NEVER use double for money
  price: NumberDecimal("29.99"),
  taxRate: NumberDecimal("0.0875"),

  // Int32 / Int64 — for counts
  stockCount: NumberInt(500),
  totalSold: NumberLong("1000000"),

  // Date — stored as UTC milliseconds since epoch
  // NEVER store dates as strings — loses sorting and range query ability
  listedAt: new Date(),
  expiresAt: ISODate("2025-12-31T23:59:59Z"),

  // ObjectId — 12-byte unique identifier with embedded timestamp
  createdBy: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),

  // Binary — for encrypted fields, file data
  encryptedSSN: BinData(6, "base64encodedciphertext"),

  // Boolean
  isActive: true,

  // Null — field exists but has no value
  discontinuedAt: null,

  // Array — ordered list of any BSON type
  tags: ["electronics", "featured"],

  // Embedded document
  dimensions: { width: 10.5, height: 5.2, unit: "cm" }
})
```

**Summary**: The most important BSON type choice in production is using `NumberDecimal` for financial values instead of `double` (avoids floating point errors), and `Date` for timestamps instead of strings (enables range queries and date aggregation functions).

---

## Answer 3: "Explain write concerns with examples of when you'd use each level"

**Point**: Write concern controls how many replica set members must acknowledge a write before the driver returns success.

**Reason**: Higher write concern = more durability but higher latency. The right choice depends on the data's criticality and the acceptable durability trade-off.

**Example**:
```javascript
// w:0 — Fire and forget
// Use for: high-frequency non-critical telemetry (click events, page views)
db.clickEvents.insertOne(
  { userId: id, page: "/home", ts: new Date() },
  { writeConcern: { w: 0 } }
)
// Returns immediately. If server crashes, event is lost. That's acceptable.

// w:1 — Primary acknowledged (default)
// Use for: most user-facing writes where replica availability is acceptable
db.userProfiles.updateOne(
  { _id: userId },
  { $set: { theme: "dark" } },
  { writeConcern: { w: 1 } }
)
// If primary crashes before replicating, change may be lost on failover.

// w:majority — Most voting members acknowledged
// Use for: financial transactions, user account creation, any critical data
db.payments.insertOne(
  {
    userId: ObjectId("..."),
    amount: NumberDecimal("500.00"),
    status: "completed",
    ts: new Date()
  },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
// j:true ensures the journal is flushed to disk.
// wtimeout prevents indefinite waiting if a secondary is down.

// w:majority + j:true is the gold standard for financial data.
// A payment confirmed with w:majority will survive:
// - Primary crash
// - Single secondary failure
// The data exists on a majority of nodes, committed to journal.
```

**Summary**: Use `w:majority` with `j:true` for anything you can't afford to lose. Use `w:1` as the default for most application writes. Use `w:0` only for truly ephemeral, high-frequency non-critical data.

---

## Answer 4: "How does a MongoDB replica set election work?"

**Point**: Replica set elections use a modified Raft consensus protocol to elect a new primary when the current primary becomes unavailable.

**Reason**: Automated failover prevents manual intervention during outages, and the Raft-based protocol guarantees that only one primary exists at any time (prevents split-brain).

**Example**:
```javascript
// Setup: 3-member replica set (primary P, secondaries S1, S2)
// rs.status() shows normal state:
{
  set: "rs0",
  members: [
    { name: "mongo1:27017", stateStr: "PRIMARY",   health: 1 },
    { name: "mongo2:27017", stateStr: "SECONDARY", health: 1 },
    { name: "mongo3:27017", stateStr: "SECONDARY", health: 1 }
  ]
}

// Step 1: Primary becomes unavailable (crash/network partition)
// Step 2: Secondaries don't receive heartbeat for electionTimeoutMillis (default 10s)
// Step 3: S1 or S2 transitions to CANDIDATE state, increments term number
// Step 4: Candidate requests votes from other members
// Step 5: Each member votes YES if:
//   - It hasn't voted in this term yet
//   - The candidate's oplog is at least as up-to-date as theirs
// Step 6: Candidate needs majority of votes (2 out of 3)
// Step 7: Winner transitions to PRIMARY, begins accepting writes

// Manually trigger an election (step down current primary)
rs.stepDown(60)  // step down, don't re-elect for 60 seconds

// Force a specific member to become primary (by setting priority)
const cfg = rs.conf()
cfg.members[1].priority = 2  // make member[1] highest priority
rs.reconfig(cfg)
```

**Timeline**:
- Heartbeat interval: 2 seconds
- Election timeout: 10 seconds
- Election + new primary ready: typically 10–30 seconds
- Applications with `retryWrites: true` will automatically retry during this window

---

## Answer 5: "Explain the CAP theorem and where MongoDB fits"

**Point**: MongoDB is a CP (Consistent + Partition Tolerant) system by default, prioritizing data consistency over availability during network partitions.

**Reason**: During a partition, MongoDB's primary will step down if it cannot reach a majority of voting members, preventing writes rather than risking a split-brain scenario where two nodes accept conflicting writes.

**Example**:
```javascript
// Scenario: 3-node replica set (1 primary, 2 secondaries) across 2 data centers
// Network partition splits: [Primary + Secondary1] | [Secondary2]

// With w:majority:
// Primary can still reach majority (itself + S1 = 2 out of 3)
// Writes CONTINUE

// But if partition is: [Primary] | [Secondary1 + Secondary2]
// Primary cannot reach majority
// Primary STEPS DOWN (no writes accepted)
// S1 or S2 elects new primary (they have majority: 2 out of 3)
// Old primary (now secondary) rejects writes

// This prevents split-brain:
// There is NEVER two nodes simultaneously accepting writes as primary

// MongoDB can be tuned toward availability (at cost of consistency):
// w:1 (only primary acknowledgment) + reading from secondaries
// = higher availability, but risk of reading stale data or lost writes on failover

// Example of tuning for analytics workload (accept stale reads):
db.events.find({ date: { $gte: yesterday } })
  .readPref("secondaryPreferred")
  .readConcern("available")  // fastest, may include orphan docs
```

**Summary**: MongoDB's default configuration favors consistency (CP). You can shift toward availability by relaxing write concerns (`w:1`) and using secondary reads, but must accept the possibility of stale reads or write loss during failover.

---

## Answer 6: "What is the MongoDB oplog and why does its size matter?"

**Point**: The oplog (`local.oplog.rs`) is a special capped collection that records all data-modifying operations, used by secondaries to replicate from the primary.

**Reason**: The oplog is the heartbeat of replication. Its size determines how long a secondary can be offline before it falls too far behind to resync incrementally.

**Example**:
```javascript
// Check current oplog size and coverage window
db.getReplicationInfo()
// Output:
{
  logSizeMB: 4096,    // 4GB oplog
  usedMB: 3800,
  timeDiff: 172800,   // oplog covers 2 days of operations
  timeDiffHours: 48,
  tFirst: ISODate("2024-01-13T10:00:00Z"),
  tLast: ISODate("2024-01-15T10:00:00Z"),
  now: ISODate("2024-01-15T11:00:00Z")
}

// Check replication lag on secondaries
rs.printSecondaryReplicationInfo()
// Output:
// source: mongo2:27017
//   syncedTo: Mon Jan 15 2024 10:58:00 GMT+0000
//   0 secs (0 hrs) behind the primary

// Danger zone: if a secondary's lag approaches the oplog window:
// Secondary transitions to RECOVERING state
// Must perform a FULL initial sync (potentially hours/days for large deployments)

// Increase oplog size (requires maintenance window):
db.adminCommand({ replSetResizeOplog: 1, size: 8192 })  // 8GB

// Typical oplog size recommendations:
// Development: 1GB (default for small deployments)
// Production: minimum 10GB, aim for 24-72 hour coverage
// High write volume: monitor oplog window; increase as needed
```

**Key insight**: On Atlas, the oplog is managed automatically. On self-managed deployments, monitor `db.getReplicationInfo().timeDiff` and alert when the window drops below 24 hours.

---

## Answer 7: "What are Change Streams and how would you use them in production?"

**Point**: Change Streams provide a real-time event stream of data changes (insert/update/delete/replace) on collections, databases, or deployments, using the oplog as their underlying source.

**Reason**: They enable reactive architectures — instead of polling for changes, applications subscribe and receive events in real time with exactly-once delivery guarantees (when using resume tokens correctly).

**Example**:
```javascript
// Production-grade change stream consumer (Node.js)
const { MongoClient } = require("mongodb")

async function startOrderChangeStream(mongoClient) {
  const db = mongoClient.db("ecommerce")

  // Load resume token from persistent storage (Redis/MongoDB)
  let resumeToken = await loadResumeToken()

  const pipeline = [
    {
      $match: {
        // Only watch order status changes
        operationType: { $in: ["insert", "update", "replace"] },
        $or: [
          { "fullDocument.status": "paid" },
          { "updateDescription.updatedFields.status": "paid" }
        ]
      }
    }
  ]

  const options = {
    fullDocument: "updateLookup",  // include full post-update document
    ...(resumeToken && { resumeAfter: resumeToken })
  }

  const changeStream = db.collection("orders").watch(pipeline, options)

  changeStream.on("change", async (change) => {
    try {
      // Process the event
      await triggerFulfillmentWorkflow(change.fullDocument)

      // Save resume token AFTER successful processing
      resumeToken = change._id
      await saveResumeToken(resumeToken)
    } catch (err) {
      // Don't advance the resume token — retry on restart
      console.error("Failed to process change:", err)
    }
  })

  changeStream.on("error", async (err) => {
    console.error("Change stream error:", err)
    // Restart the change stream
    await startOrderChangeStream(mongoClient)
  })
}
```

**Summary**: Save the resume token persistently after each successful event processing. This guarantees exactly-once processing on restart. Change Streams require a replica set — not available on standalone mongod.

---

## Answer 8: "Explain ObjectId structure and when to use custom _id values"

**Point**: ObjectId is a 12-byte BSON type with an embedded creation timestamp, serving as MongoDB's default primary key with guaranteed global uniqueness.

**Reason**: ObjectIds are generated client-side in the driver, enabling ID generation without server round-trips and making them safe for distributed insert scenarios.

**Example**:
```javascript
// ObjectId anatomy
const id = ObjectId("64f1a2b3c4d5e6f7a8b9c0d1")
// 64f1a2b3 — Unix timestamp (hex) = seconds since epoch
// c4d5e6   — random value (per machine/process, avoids collisions)
// f7a8b9   — random + incrementing counter

id.getTimestamp()
// ISODate("2023-09-01T00:00:03.000Z")

// ObjectIds are sortable (newer > older)
// This means _id can serve as a creation-time sort with no extra field
db.logs.find().sort({ _id: 1 })  // oldest first
db.logs.find().sort({ _id: -1 }) // newest first (most common for recent events)

// Range query on ObjectId for time-based pagination
// "Give me all documents created after 2024-01-01"
const startOfYear = ObjectId.createFromTime(new Date("2024-01-01").getTime() / 1000)
db.events.find({ _id: { $gte: startOfYear } })

// Custom _id — when to use:
// 1. Natural unique key exists (no need for separate unique index)
db.countries.insertOne({ _id: "US", name: "United States" })
db.currencies.insertOne({ _id: "USD", symbol: "$", name: "US Dollar" })
db.configs.insertOne({ _id: "app_settings", theme: "dark", version: 3 })

// 2. Composite natural key
db.userSettings.insertOne({
  _id: { userId: ObjectId("..."), settingKey: "notifications" },
  value: true
})

// When NOT to use custom _id:
// - IDs might need to change (email as _id is risky — users change emails)
// - You can't guarantee uniqueness at write time
// - You need the timestamp encoding that ObjectId provides
```

---

## Answer 9: "Describe how MongoDB handles concurrent writes to the same document"

**Point**: MongoDB uses document-level locking in WiredTiger to handle concurrent writes, ensuring data integrity without requiring table-level locks.

**Reason**: WiredTiger (the default storage engine since MongoDB 3.2) uses optimistic concurrency control with multi-version concurrency control (MVCC), allowing reads to proceed without blocking writes.

**Example**:
```javascript
// Scenario: Two simultaneous updates to the same product document
// Update 1 (Thread A): decrement stock
db.products.updateOne(
  { _id: productId, stock: { $gt: 0 } },  // check condition
  { $inc: { stock: -1 } }                  // atomic decrement
)

// Update 2 (Thread B): update price
db.products.updateOne(
  { _id: productId },
  { $set: { price: NumberDecimal("24.99") } }
)

// What actually happens:
// 1. WiredTiger serializes writes to the same document
// 2. One update wins the document lock
// 3. Other update waits (brief lock duration — typically microseconds)
// 4. Both updates succeed, both changes are applied
// 5. No data loss, no corruption

// IMPORTANT: $inc is atomic — safe for counters
db.products.updateOne(
  { _id: productId },
  { $inc: { views: 1 } }  // safe — atomic increment, no read-modify-write race
)

// NOT safe (non-atomic read-modify-write in application code):
const product = await db.products.findOne({ _id: productId })
const newStock = product.stock - 1  // another thread may have changed stock already
await db.products.updateOne({ _id: productId }, { $set: { stock: newStock } })
// RACE CONDITION: between findOne and updateOne, another thread could have changed stock
```

---

## Answer 10: "What is the difference between read concern levels?"

**Point**: Read concern controls the consistency and isolation of data returned by read operations, from weakest (`local`) to strongest (`linearizable`).

**Reason**: Different use cases require different trade-offs between data freshness, isolation, and performance. Financial systems need linearizable reads; analytics dashboards can use local reads.

**Example**:
```javascript
// local — returns most recent data as seen by this node (default for reads)
// May return data that could be rolled back if primary fails
db.orders.findOne({ _id: orderId }, { readConcern: { level: "local" } })

// available — same as local for replica sets; may return orphan docs in sharded clusters
// Fastest, most available, used when consistency is not critical
db.events.find({}, { readConcern: { level: "available" } })

// majority — returns data acknowledged by majority of voting members
// Safe to read: this data will NOT be rolled back
db.payments.findOne(
  { _id: paymentId },
  { readConcern: { level: "majority" } }
)

// linearizable — strong consistency: reflects all majority-committed writes that
// completed before this read started. Only works on primary.
// Slowest — adds a verification round-trip
db.accounts.findOne(
  { _id: accountId },
  { readConcern: { level: "linearizable" }, maxTimeMS: 5000 }
)

// snapshot — reads a consistent snapshot of data at a point in time
// Used within multi-document transactions
const session = client.startSession()
session.startTransaction({ readConcern: { level: "snapshot" }, writeConcern: { w: "majority" } })
const balance = await db.accounts.findOne({ _id: accountId }, { session })
// All reads within this transaction see the same consistent snapshot
```

**Summary table**:

| Read Concern | Can Return Rolled-Back Data? | Works on Secondary? | Performance |
|---|---|---|---|
| local | Yes | Yes | Fastest |
| available | Yes (orphans possible in sharding) | Yes | Fastest |
| majority | No | Yes | Medium |
| linearizable | No | No (primary only) | Slowest |
| snapshot | No (within transaction) | Yes | Medium |
