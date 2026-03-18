# Chapter 01 — MongoDB Fundamentals: Scenario Questions

---

**S1. Your team is migrating a legacy MySQL e-commerce application to MongoDB. The product table has 47 columns, many of which are NULL for most product types (a book has ISBN, a shoe has size, electronics have wattage). How do you model this in MongoDB?**

This is the **Polymorphic Pattern** use case. In MongoDB, you only store fields that exist for each document:

```javascript
// Book product
{
  _id: ObjectId("..."),
  type: "book",
  name: "MongoDB: The Definitive Guide",
  price: NumberDecimal("49.99"),
  isbn: "978-1491954461",
  author: "Shannon Bradshaw",
  pageCount: 514
}

// Shoe product
{
  _id: ObjectId("..."),
  type: "shoe",
  name: "Running Shoe Pro",
  price: NumberDecimal("89.99"),
  brand: "SportMax",
  sizes: [7, 8, 9, 10, 11, 12],
  colors: ["black", "white", "red"]
}

// Electronic product
{
  _id: ObjectId("..."),
  type: "electronic",
  name: "USB-C Charger",
  price: NumberDecimal("29.99"),
  wattage: 65,
  inputVoltage: "100-240V",
  certifications: ["UL", "CE", "FCC"]
}
```

Benefits:
- No NULL waste — only store relevant fields
- Single collection for all products (easy catalog queries)
- Add new product types without schema migrations
- Index on `type` for filtering

Query example:
```javascript
// Find all books under $30
db.products.find({ type: "book", price: { $lt: NumberDecimal("30.00") } })
```

---

**S2. A startup is building a real-time analytics dashboard that needs to display "last 24 hours of user activity" for their app. They have millions of events per day. Should they use MongoDB? How would you structure the data?**

Yes, with the right approach:

```javascript
// Raw event document
{
  _id: ObjectId("..."),  // ObjectId encodes timestamp — useful for range queries
  userId: ObjectId("user_id"),
  event: "page_view",
  page: "/products/widget-a",
  sessionId: "sess_abc123",
  metadata: {
    browser: "Chrome",
    os: "macOS",
    country: "US"
  },
  ts: ISODate("2024-01-15T14:30:00Z")
}

// TTL index to auto-expire events after 24 hours
db.events.createIndex({ ts: 1 }, { expireAfterSeconds: 86400 })

// Index for fast user+time queries
db.events.createIndex({ userId: 1, ts: -1 })

// Dashboard query: last 24h events for a user
const since = new Date(Date.now() - 86400000)
db.events.find(
  { userId: ObjectId("user_id"), ts: { $gte: since } },
  { event: 1, page: 1, ts: 1, _id: 0 }
).sort({ ts: -1 })
```

For the analytics aggregation:
```javascript
db.events.aggregate([
  { $match: { ts: { $gte: new Date(Date.now() - 86400000) } } },
  { $group: { _id: "$event", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

---

**S3. You have a replica set with one primary and two secondaries. The primary crashes. Walk through what happens.**

1. **Secondaries detect failure**: Each secondary monitors the primary via heartbeats (every 2 seconds). After 10 seconds without a heartbeat, a secondary initiates an election.

2. **Election begins**: The secondary with the highest `priority` (or most up-to-date oplog) calls for an election. It must receive votes from a majority of the replica set's voting members (2 out of 3 in this case).

3. **Voting**: Each voting member casts one vote. A candidate needs ⌈n/2⌉ + 1 votes to win. With 2 remaining members, the candidate needs 2 votes (majority of 3 = 2).

4. **New primary elected**: The winner becomes primary, applies any oplogs it was missing, and starts accepting writes.

5. **Write availability gap**: During election (~10-30 seconds), NO writes are accepted. Applications should implement retry logic.

```javascript
// In your application driver, configure retryable writes to handle this
const client = new MongoClient(uri, {
  retryWrites: true,      // automatically retry failed write ops once
  retryReads: true,       // automatically retry failed read ops once
  serverSelectionTimeoutMS: 30000  // wait up to 30s for server selection
})
```

---

**S4. A developer stores prices as JavaScript floats (regular `number` type) in MongoDB. After thousands of transactions, the total balance is off by a few cents. What went wrong and how do you fix it?**

JavaScript `number` is IEEE 754 double-precision float. Binary floats cannot represent many decimal fractions exactly:

```javascript
// The bug
db.transactions.insertOne({ amount: 0.1 + 0.2 })  // stores 0.30000000000000004

// After many transactions:
db.accounts.aggregate([
  { $match: { userId: ObjectId("...") } },
  { $group: { _id: null, balance: { $sum: "$amount" } } }
])
// Result: { balance: 99.9999999999983 }  // wrong!
```

Fix — use `NumberDecimal` (Decimal128):

```javascript
// Store as Decimal128
db.transactions.insertOne({
  userId: ObjectId("..."),
  amount: NumberDecimal("0.10"),
  type: "credit",
  ts: new Date()
})

// Decimal arithmetic is exact
db.accounts.aggregate([
  { $match: { userId: ObjectId("...") } },
  { $group: { _id: null, balance: { $sum: "$amount" } } }
])
// Result: { balance: NumberDecimal("100.00") }  // correct!
```

Migration strategy for existing data:
```javascript
db.transactions.find({ amount: { $type: "double" } }).forEach(doc => {
  db.transactions.updateOne(
    { _id: doc._id },
    { $set: { amount: NumberDecimal(doc.amount.toString()) } }
  )
})
```

---

**S5. Your company wants to implement audit logging — recording every document change in every collection. How would you use Change Streams to build this?**

```javascript
// Audit log collection
// Create with capped or TTL depending on retention needs
db.createCollection("auditLog")
db.auditLog.createIndex({ ts: 1 }, { expireAfterSeconds: 7776000 }) // 90 days

// Change stream watcher (run in a persistent Node.js service)
async function startAuditWatcher(db) {
  const pipeline = [
    {
      $match: {
        // Exclude the auditLog collection itself to avoid infinite loop!
        ns: { $not: { $regex: /^auditdb\.auditLog$/ } }
      }
    }
  ]

  const options = {
    fullDocument: "updateLookup",  // include full updated doc in update events
    resumeAfter: await getResumeToken()  // resume from last processed position
  }

  const changeStream = db.watch(pipeline, options)

  changeStream.on("change", async (change) => {
    await db.auditLog.insertOne({
      ts: new Date(),
      operationType: change.operationType,
      namespace: change.ns,
      documentKey: change.documentKey,
      fullDocument: change.fullDocument || null,
      updateDescription: change.updateDescription || null,
      resumeToken: change._id
    })
    await saveResumeToken(change._id)
  })
}
```

Key considerations:
- Always save the resume token so the watcher can restart from where it left off
- Use `fullDocument: "updateLookup"` for update events if you need the full post-update document
- Watch at the database level to capture all collections
- Run in a separate service with retry logic

---

**S6. A collection has 50 million documents. You run `db.collection.find({})` in mongosh. What happens? Is this safe?**

This is NOT safe in production. `find({})` returns a **cursor** that, without limits, will:

1. Batch through all 50 million documents
2. Hold a server-side cursor open (default timeout: 10 minutes)
3. Transfer all documents over the network
4. Potentially exhaust mongosh memory

Safe alternatives:
```javascript
// Use limit for sampling
db.users.find({}).limit(10)

// Use count first to understand scope
db.users.countDocuments({})

// Use aggregation with $sample for random sampling
db.users.aggregate([{ $sample: { size: 100 } }])

// Use projection to reduce document size
db.users.find({}, { email: 1, _id: 0 }).limit(100)

// Use explain() to check query plan without running the query
db.users.find({}).explain("executionStats")
```

In production code, always:
- Apply a filter (`$match`)
- Apply `.limit()` with pagination
- Apply projections to return only needed fields
- Use `allowDiskUse: true` for large aggregation pipelines

---

**S7. You need to store user profile pictures (average 200KB each) in your application. Should you store them in MongoDB? What are your options?**

The BSON document limit is **16MB**, so a 200KB image technically fits, but storing binary data in MongoDB is generally NOT recommended:

```javascript
// Option 1: Store as BinData in the document (not recommended for anything > 16KB)
db.users.updateOne(
  { _id: userId },
  { $set: { profilePic: BinData(0, base64EncodedImage) } }
)
// Problem: bloats documents, slows queries, no CDN, no streaming
```

**Recommended: GridFS** (for files up to any size):
```javascript
// GridFS splits files into 255KB chunks and stores metadata
const bucket = new GridFSBucket(db, { bucketName: "profilePics" })

// Store a file
const uploadStream = bucket.openUploadStream("alice-profile.jpg", {
  metadata: { userId: ObjectId("..."), uploadedAt: new Date() }
})
fs.createReadStream("alice.jpg").pipe(uploadStream)

// Retrieve a file
const downloadStream = bucket.openDownloadStreamByName("alice-profile.jpg")
downloadStream.pipe(res)  // pipe to HTTP response
```

**Best practice: Object storage + metadata reference**
```javascript
// Store in S3/GCS, save URL in MongoDB
db.users.updateOne(
  { _id: userId },
  {
    $set: {
      profilePic: {
        url: "https://cdn.example.com/profiles/alice.jpg",
        s3Key: "profiles/alice-uuid.jpg",
        sizeBytes: 204800,
        contentType: "image/jpeg",
        uploadedAt: new Date()
      }
    }
  }
)
```

Recommendation: Use S3/GCS + CDN for serving, store only the URL and metadata in MongoDB.

---

**S8. A junior developer is writing `db.users.find({ name: "Alice" })` but no results come back, even though the document definitely exists. What could be wrong?**

Common causes:

```javascript
// 1. Case sensitivity — MongoDB string queries are case-sensitive by default
db.users.find({ name: "alice" })   // won't find "Alice"
db.users.find({ name: "ALICE" })   // won't find "Alice"

// Fix: use $regex with case-insensitive flag (slow without text index)
db.users.find({ name: /^alice$/i })
// Better: store name as lowercase, search as lowercase
db.users.find({ nameLower: "alice" })

// 2. Type mismatch
db.users.find({ age: "32" })  // won't find { age: 32 } (number)

// 3. Wrong database
db  // Check current database
use production  // Make sure you're in the right database

// 4. Nested field — need dot notation
db.users.find({ "address.city": "Austin" })  // correct
db.users.find({ address: { city: "Austin" } })  // incorrect — exact subdocument match

// 5. Typo in field name
db.users.find({ Name: "Alice" })  // capital N — field names are case-sensitive
```

---

**S9. Explain how you would migrate data from MySQL to MongoDB without downtime for a live application.**

Strategy: **Dual-write migration pattern**

```
Phase 1: Set up MongoDB, write to both databases simultaneously
Phase 2: Backfill historical data from MySQL to MongoDB
Phase 3: Validate data consistency
Phase 4: Switch reads to MongoDB
Phase 5: Remove MySQL writes
```

```javascript
// Phase 1: Application code — dual write
async function createUser(userData) {
  // Write to MySQL (existing)
  await mysqlDb.query("INSERT INTO users ...", userData)

  // Write to MongoDB (new)
  await mongoDb.users.insertOne({
    _id: ObjectId(),
    mysqlId: userData.id,  // keep reference for reconciliation
    name: userData.name,
    email: userData.email,
    // Transform relational data to document model
    address: {
      street: userData.street,
      city: userData.city,
      zip: userData.zip
    },
    createdAt: new Date()
  })
}

// Phase 2: Backfill script (run in batches)
async function backfillUsers(lastMysqlId = 0) {
  const rows = await mysqlDb.query(
    "SELECT * FROM users WHERE id > ? ORDER BY id LIMIT 1000",
    [lastMysqlId]
  )

  if (rows.length === 0) return

  const docs = rows.map(row => transformUserToDoc(row))
  await mongoDb.users.insertMany(docs, { ordered: false })

  await backfillUsers(rows[rows.length - 1].id)
}
```

---

**S10. You have a MongoDB document that looks like this: `{ _id: "user123", tags: ["admin", "editor", "viewer"] }`. How do you query for all users who have BOTH "admin" and "editor" tags?**

```javascript
// $all operator — document must contain ALL specified array elements
db.users.find({ tags: { $all: ["admin", "editor"] } })
// Matches: { tags: ["admin", "editor", "viewer"] }  ✓
// Matches: { tags: ["editor", "admin", "superuser"] }  ✓
// Does NOT match: { tags: ["admin", "viewer"] }  ✗ (missing "editor")

// Alternative using $and with array element queries
db.users.find({
  $and: [
    { tags: "admin" },
    { tags: "editor" }
  ]
})
// Same result as $all but more verbose

// Find users with EITHER "admin" OR "editor"
db.users.find({ tags: { $in: ["admin", "editor"] } })

// Find users with NONE of these tags
db.users.find({ tags: { $nin: ["admin", "editor"] } })

// Find users with EXACTLY these tags and no others
db.users.find({ tags: ["admin", "editor"] })  // exact array match (order matters!)

// Find users with EXACTLY these tags in any order
db.users.find({ tags: { $all: ["admin", "editor"], $size: 2 } })
```

---

**S11. Describe the behavior of MongoDB when you insert a document without specifying `_id`.**

```javascript
// Insert without _id
db.users.insertOne({ name: "Bob", email: "bob@example.com" })

// MongoDB automatically generates an ObjectId for _id
// Result:
{
  acknowledged: true,
  insertedId: ObjectId("64f2b3c4d5e6f7a8b9c0d1e2")
}

// The driver generates the ObjectId CLIENT-SIDE (not server-side)
// This means:
// 1. No round-trip needed to get the ID
// 2. The ObjectId is available immediately after insert
// 3. Multiple inserts can happen in parallel without ID conflicts
// 4. The timestamp in the ObjectId is the client's time (may differ from server time)
```

The driver generates ObjectIds to:
- Allow offline/disconnected operation
- Enable batch insert optimization
- Avoid server bottleneck for ID generation

---

**S12. A read-heavy application is experiencing high latency. A DBA suggests "use secondary reads." What are the trade-offs?**

```javascript
// Configuring secondary reads
const client = new MongoClient(uri, {
  readPreference: "secondaryPreferred"
})

// Or per-query
db.products.find({ category: "electronics" }).readPref("secondary")
```

Trade-offs:

| Aspect | Primary Reads | Secondary Reads |
|--------|--------------|-----------------|
| Latency | Low (primary) | May be lower (geo) or similar |
| Staleness | Always fresh | May lag (replication lag) |
| Load distribution | All load on primary | Distributed across secondaries |
| Monotonic reads | Guaranteed | Not guaranteed (read from different secondaries) |
| Write-then-read | Consistent | May not see your own write |

Safe for secondary reads:
- Analytics queries on data that can be slightly stale
- Reporting dashboards
- Read-heavy catalogs where stale data is acceptable

Not safe for secondary reads:
- Reading after a write that must be reflected immediately
- Financial balances
- Inventory counts during checkout

Solution for write-then-read consistency:
```javascript
// Use causally consistent session for read-your-writes guarantee
const session = client.startSession({ causalConsistency: true })
await db.users.updateOne({ _id: id }, { $set: { name: "Alice" } }, { session })
const user = await db.users.findOne({ _id: id }, { session, readPreference: "secondary" })
session.endSession()
```

---

**S13. You need to store timestamps for events. What's the best way in MongoDB?**

```javascript
// Best practice: Use ISODate (BSON Date type)
db.events.insertOne({
  event: "user_login",
  userId: ObjectId("..."),
  timestamp: new Date(),              // UTC timestamp, stored as int64 milliseconds
  serverTime: "$$NOW"                 // $$NOW — server-side current timestamp in aggregation
})

// Creating dates
new Date()                            // current UTC time
new Date("2024-01-15T10:30:00Z")     // from ISO string
new Date(1705315800000)              // from Unix milliseconds

// Date comparisons in queries
db.events.find({
  timestamp: {
    $gte: ISODate("2024-01-01T00:00:00Z"),
    $lt: ISODate("2024-02-01T00:00:00Z")
  }
})

// Date aggregation
db.events.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$timestamp" },
        month: { $month: "$timestamp" },
        day: { $dayOfMonth: "$timestamp" }
      },
      count: { $sum: 1 }
    }
  }
])

// BAD: storing as Unix timestamp (integer)
db.events.insertOne({ timestamp: 1705315800 })  // loses precision context, harder to query
// BAD: storing as string
db.events.insertOne({ timestamp: "2024-01-15 10:30:00" })  // can't use $gte/$lt efficiently
```

Always store dates as BSON Date (`ISODate`) — not strings or Unix integers.

---

**S14. How does MongoDB handle the case where two simultaneous inserts have the same `_id`?**

```javascript
// Two concurrent inserts with same _id
// Thread 1:
db.users.insertOne({ _id: "alice", name: "Alice Smith" })

// Thread 2 (concurrent):
db.users.insertOne({ _id: "alice", name: "Alice Jones" })
```

MongoDB's `_id` field has a unique index enforced at the collection level. The second insert will fail with a `DuplicateKeyError`:

```
WriteError({
  code: 11000,
  codeName: "DuplicateKey",
  keyPattern: { _id: 1 },
  keyValue: { _id: "alice" }
})
```

The `_id` index uses WiredTiger's document-level locking, so only one insert wins — there's no data corruption, just an error for the loser.

Handling in application code:
```javascript
try {
  await db.users.insertOne({ _id: "alice", name: "Alice" })
} catch (err) {
  if (err.code === 11000) {
    // Handle duplicate key — update existing or return conflict error
    await db.users.updateOne({ _id: "alice" }, { $setOnInsert: { name: "Alice" } }, { upsert: true })
  } else {
    throw err
  }
}
```

---

**S15. A MongoDB collection has documents with varying structures — some have a `middleName` field, others don't. How do you query for documents where `middleName` exists AND is not null?**

```javascript
// Sample data
// Doc 1: { _id: 1, firstName: "Alice", middleName: "Jane" }
// Doc 2: { _id: 2, firstName: "Bob" }                          -- no middleName field
// Doc 3: { _id: 3, firstName: "Carol", middleName: null }      -- null middleName
// Doc 4: { _id: 4, firstName: "Dave", middleName: "" }         -- empty string

// $exists: true — includes docs where field exists (even if null)
db.users.find({ middleName: { $exists: true } })
// Returns: Doc 1, Doc 3, Doc 4

// $ne: null — includes docs where field exists AND is not null
// Also catches undefined
db.users.find({ middleName: { $ne: null } })
// Returns: Doc 1, Doc 4

// Strictly: field exists AND is not null AND is not empty string
db.users.find({
  middleName: { $exists: true, $ne: null, $ne: "" }
})
// Returns: Doc 1 only
// Note: multiple $ne can't be on same field like this — use $and or $nin

// Correct way:
db.users.find({
  $and: [
    { middleName: { $exists: true } },
    { middleName: { $ne: null } },
    { middleName: { $ne: "" } }
  ]
})

// Using $type to find only string values
db.users.find({ middleName: { $type: "string" } })
// Returns: Doc 1, Doc 4 (both are strings)
```
