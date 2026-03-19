# Chapter 01 — MongoDB Fundamentals: Follow-Up Traps

These are questions interviewers ask AFTER you give a correct initial answer — designed to catch candidates who memorized surface facts without deep understanding.

---

## Trap 1: ObjectId Is NOT a String

**Setup**: Candidate says "the `_id` is an ObjectId which looks like a 24-character hex string."

**Trap question**: "If I stored an `_id` as the string `'64f1a2b3c4d5e6f7a8b9c0d1'`, is that the same as `ObjectId('64f1a2b3c4d5e6f7a8b9c0d1')`?"

**Wrong answer**: "Yes, they represent the same value."

**Correct answer**: No — they are completely different BSON types:

```javascript
// These are NOT equal
db.users.findOne({ _id: "64f1a2b3c4d5e6f7a8b9c0d1" })    // looks for String
db.users.findOne({ _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1") }) // looks for ObjectId

// Most common bug: receiving an ID from a REST API (as a string),
// then querying without wrapping in ObjectId()
const userIdFromRequest = req.params.id  // "64f1a2b3c4d5e6f7a8b9c0d1" (string)

// BUG: this will never find the document
db.users.findOne({ _id: userIdFromRequest })

// CORRECT:
db.users.findOne({ _id: new ObjectId(userIdFromRequest) })
```

MongoDB stores type information alongside values in BSON. Type `objectId` (BSON type 7) and type `string` (BSON type 2) are never equal, even with identical byte content.

---

## Trap 2: _id Immutability

**Setup**: Candidate explains `_id` is a primary key.

**Trap question**: "Can you rename a user from a username-based `_id` like `_id: 'alice_123'` to `_id: 'alice_johnson'`?"

**Wrong answer**: "Sure, just do `$set: { _id: 'alice_johnson' }`."

**Correct answer**: No. `_id` is immutable. The only way to "change" it is:

```javascript
// Step 1: Read the document
const doc = db.users.findOne({ _id: "alice_123" })

// Step 2: Insert with new _id
const newDoc = { ...doc, _id: "alice_johnson" }
db.users.insertOne(newDoc)

// Step 3: Delete the old document
db.users.deleteOne({ _id: "alice_123" })

// These must be wrapped in a transaction to be atomic:
const session = client.startSession()
session.withTransaction(async () => {
  const doc = await db.users.findOne({ _id: "alice_123" }, { session })
  await db.users.insertOne({ ...doc, _id: "alice_johnson" }, { session })
  await db.users.deleteOne({ _id: "alice_123" }, { session })
})
```

This is why using natural/mutable values (usernames, email addresses) as `_id` is risky.

---

## Trap 3: BSON 16MB Document Size Limit

**Setup**: Candidate describes the document model and nested arrays.

**Trap question**: "What happens when a user has 100,000 comments embedded in their document?"

**Wrong answer**: "MongoDB handles it — documents are flexible."

**Correct answer**: Documents have a hard limit of **16MB**. A document with 100,000 embedded comments would likely exceed this:

```javascript
// 100,000 comments at ~200 bytes each = ~20MB > 16MB limit
// MongoDB will throw: BSONObjectTooLarge

// The fix: don't embed unbounded arrays — use a separate collection
// BAD pattern (unbounded embedding):
{
  _id: userId,
  comments: [
    { text: "...", date: ISODate("...") },
    // ... potentially unlimited entries
  ]
}

// GOOD pattern (reference):
// users collection
{ _id: userId, name: "Alice" }

// comments collection
{ _id: commentId, userId: userId, text: "...", date: ISODate("...") }

// Query with $lookup if needed
db.users.aggregate([
  { $match: { _id: userId } },
  { $lookup: {
    from: "comments",
    localField: "_id",
    foreignField: "userId",
    as: "comments"
  }}
])
```

---

## Trap 4: Write Concern w:0 (Fire and Forget)

**Setup**: Candidate says w:0 is the fastest write concern.

**Trap question**: "When would w:0 be appropriate? What are the risks?"

**Wrong answer**: "w:0 is fine for most cases when you need speed."

**Correct answer**: w:0 means the driver sends the write operation and IMMEDIATELY returns without waiting for any server acknowledgment. Risks:

```javascript
// w:0 — driver doesn't wait for server to receive the write
db.metrics.insertOne({ event: "click", ts: new Date() }, { writeConcern: { w: 0 } })

// If the server is unreachable, you won't know
// If the write fails (e.g., disk full), you won't know
// If there's a DuplicateKey error, you won't know
```

Appropriate uses: non-critical telemetry, high-frequency metrics where some loss is acceptable.

NEVER use w:0 for:
- Financial transactions
- User account creation
- Any data you cannot afford to lose
- Any operation where you need to confirm success

---

## Trap 5: Embedding Unbounded Arrays

**Setup**: Candidate recommends embedding related data.

**Trap question**: "You're designing a social network. Would you embed all of a user's followers in their document?"

**Wrong answer**: "Yes, embedding is better for performance since it's one read."

**Correct answer**: No — followers can grow without bound (a celebrity might have millions). Embedding would:
- Exceed the 16MB document limit
- Make every read of the user document expensive (loading all followers)
- Make "who follows whom" updates require rewriting a huge document

```javascript
// WRONG: embedding unbounded followers
{
  _id: userId,
  name: "Taylor Swift",
  followers: [ObjectId("user1"), ObjectId("user2"), ... /* millions */]
}

// RIGHT: separate edge collection (graph pattern)
// follows collection
{
  _id: ObjectId(),
  followerId: ObjectId("fan_user_id"),
  followingId: ObjectId("celebrity_user_id"),
  followedAt: ISODate("...")
}

db.follows.createIndex({ followerId: 1, followingId: 1 }, { unique: true })
db.follows.createIndex({ followingId: 1 })

// Get follower count (doesn't load all followers)
db.follows.countDocuments({ followingId: userId })
```

---

## Trap 6: String vs Number Comparison in Queries

**Setup**: Candidate explains BSON type system.

**Trap question**: "If I store age as a string in some documents and a number in others, what happens when I query `{ age: { $gt: 25 } }`?"

**Wrong answer**: "MongoDB will compare them automatically."

**Correct answer**: MongoDB compares values of the SAME type. The BSON comparison order across types is: MinKey < Null < Numbers < Symbol < String < Object < Array < BinData < ObjectId < Boolean < Date < Timestamp < Regular Expression < MaxKey

```javascript
// Mixed types — queries only return type-matching documents
db.demo.insertMany([
  { name: "Alice", age: 30 },        // number
  { name: "Bob", age: "30" },        // string
  { name: "Carol", age: 28 }         // number
])

db.demo.find({ age: { $gt: 25 } })
// Returns: Alice (30 > 25 as numbers), Carol (28 > 25)
// Does NOT return Bob — "30" (string) is NOT > 25 (number)

// The $type operator helps identify type issues
db.demo.find({ age: { $type: "string" } })  // finds Bob
db.demo.find({ age: { $type: "number" } })  // finds Alice, Carol

// Fix mixed types with an update
db.demo.updateMany(
  { age: { $type: "string" } },
  [{ $set: { age: { $toInt: "$age" } } }]  // aggregation pipeline update
)
```

---

## Trap 7: Replica Set Read Your Own Writes

**Setup**: Candidate explains secondary reads for scalability.

**Trap question**: "You update a user's email address and immediately redirect them to their profile page which reads from a secondary. What could go wrong?"

**Wrong answer**: "Nothing — the read happens after the write."

**Correct answer**: Replication is asynchronous. The secondary might not have applied the oplog entry yet when the read happens:

```javascript
// Timeline:
// t=0ms: Write to primary (email updated to new@email.com)
// t=5ms: User redirect triggers read from secondary
// t=50ms: Secondary applies oplog entry (replication lag)

// Result: User sees their OLD email address (stale read)

// Solutions:

// Option 1: Read from primary after write
db.users.findOne({ _id: userId }, { readPreference: "primary" })

// Option 2: Use causally consistent session
const session = client.startSession({ causalConsistency: true })
await db.users.updateOne({ _id: userId }, { $set: { email: "new@email.com" } }, { session })
// This read is guaranteed to see the update above, even from a secondary
const user = await db.users.findOne({ _id: userId }, { session })

// Option 3: w:majority + r:majority ensures linearizable behavior
await db.users.updateOne(
  { _id: userId },
  { $set: { email: "new@email.com" } },
  { writeConcern: { w: "majority" } }
)
const user = await db.users.findOne(
  { _id: userId },
  { readConcern: { level: "majority" } }
)
```

---

## Trap 8: BSON Size Limit Is on the Final Document

**Setup**: Candidate knows the 16MB limit.

**Trap question**: "If I run `db.collection.aggregate([...pipeline that produces a 20MB intermediate document...])`, does it fail at 16MB?"

**Wrong answer**: "Yes, 16MB limit applies to all documents."

**Correct answer**: The 16MB limit applies to **documents stored in a collection** and to **individual pipeline stage output documents**. However:

```javascript
// Aggregation pipeline stages can produce intermediate results up to 100MB in memory
// (or unlimited with allowDiskUse: true)

// BUT: if $merge or $out write results to a collection,
// each output document must be <= 16MB

// Also: $lookup result with "as" array can make document exceed 16MB
// This will throw BSONObjectTooLarge
db.orders.aggregate([
  { $lookup: {
    from: "orderItems",
    localField: "_id",
    foreignField: "orderId",
    as: "items"  // if there are 10,000 items, this could exceed 16MB
  }}
])

// Fix: use $lookup with pipeline + $limit
db.orders.aggregate([
  { $lookup: {
    from: "orderItems",
    let: { orderId: "$_id" },
    pipeline: [
      { $match: { $expr: { $eq: ["$orderId", "$$orderId"] } } },
      { $limit: 100 },  // prevent unbounded join results
      { $project: { sku: 1, qty: 1, price: 1 } }
    ],
    as: "items"
  }}
])
```

---

## Trap 9: ObjectId Timestamp Is Client-Generated

**Setup**: Candidate explains ObjectId encodes creation time.

**Trap question**: "Can you rely on ObjectId timestamps for precise event ordering in a distributed system?"

**Wrong answer**: "Yes, ObjectIds are sortable and their timestamp gives you insertion order."

**Correct answer**: ObjectId timestamps are accurate to the **second** (not millisecond), and they are generated **client-side**:

```javascript
const id = ObjectId()
id.getTimestamp()  // accurate to SECOND, not millisecond

// Problems:
// 1. Two inserts in the same second may be ordered by the counter component,
//    but the "correct" order depends on which client generated the ObjectId first

// 2. If client clocks are not synchronized (NTP drift), ObjectId timestamps
//    may not reflect actual operation order

// 3. For high-frequency event ordering, use a dedicated timestamp field:
db.events.insertOne({
  _id: ObjectId(),
  ts: new Date(),        // millisecond precision
  tsNano: process.hrtime.bigint().toString(),  // nanosecond for same-millisecond ordering
  sequenceId: await getNextSequenceId()  // application-level sequence
})

// 4. ObjectId ordering within the same second uses the random component,
//    NOT the counter! (The counter increments per process, not globally)
```

For strict ordering, use a dedicated `ts` field with millisecond precision.

---

## Trap 10: Replica Set Requires Odd Number of Voting Members

**Setup**: Candidate explains replica set elections.

**Trap question**: "Can I have a replica set with 2 members? Why or why not?"

**Wrong answer**: "Yes, one primary and one secondary."

**Correct answer**: You CAN have 2 members technically, but it's not recommended because majority of 2 = 2. If the primary goes down, the secondary cannot elect itself (needs 2 votes, only has 1):

```javascript
// rs.status() on a 2-member set where primary is down:
// Secondary goes into: SECONDARY with "not primary and secondaryDelay unset"
// No new primary is elected — you have NO write availability

// Solution: Always have an odd number of voting members
// Option 1: 3-member PSS (Primary + Secondary + Secondary) — preferred
// Option 2: 2-member + 1 arbiter (PSA) — arbiter has no data, just votes

// Add an arbiter to a 2-member set
rs.addArb("mongodb-arbiter.example.com:27017")

// BUT: PSA has its own risks:
// - If secondary goes down, you still need the arbiter to vote
// - Arbiter + primary = 2 votes = majority of 3, so writes proceed
// - But you have NO data redundancy (primary is the only full copy)

// Recommended minimum for production: PSS (3 full data members)
```

---

## Trap 11: mongosh vs mongo (Legacy Shell)

**Setup**: Candidate mentions using the MongoDB shell.

**Trap question**: "What's the difference between `mongo` and `mongosh`?"

**Wrong answer**: "They're the same thing, just different names."

**Correct answer**:
- `mongo` is the **legacy shell** — deprecated since MongoDB 5.0, removed in MongoDB 6.0
- `mongosh` is the **modern MongoDB Shell** — Node.js-based, uses the same JavaScript engine as modern drivers, supports async/await, better autocomplete, `.mongoshrc.js` config

```javascript
// mongosh supports async/await natively
const result = await db.users.findOne({ email: "alice@example.com" })

// mongosh supports modern JavaScript
const users = await db.users.find().toArray()
users.filter(u => u.age > 30).forEach(u => printjson(u))

// mongosh uses the unified driver under the hood
// Commands are closer to what production drivers do

// Legacy mongo shell: synchronous API, older JS engine, no async/await
```

---

## Trap 12: db.collection.find() vs db.collection.find().toArray()

**Setup**: Candidate explains cursor behavior.

**Trap question**: "Is there a memory concern with `find({}).toArray()` on a large collection?"

**Wrong answer**: "It returns an array which is fine."

**Correct answer**: `.toArray()` **loads ALL documents into memory at once**. On a 10M document collection, this will exhaust memory and crash:

```javascript
// DANGEROUS: loads entire collection into memory
const allUsers = await db.users.find({}).toArray()

// SAFE: iterate with a cursor (streams documents in batches)
const cursor = db.users.find({})
for await (const user of cursor) {
  // processes one document at a time (batches of 101 initially)
  processUser(user)
}

// SAFE: use batchSize to control memory
const cursor = db.users.find({}).batchSize(500)

// For large exports, use streams
const pipeline = [{ $project: { name: 1, email: 1 } }]
db.users.aggregate(pipeline, { allowDiskUse: true }).stream()
  .pipe(csvTransformer)
  .pipe(outputFile)
```

Cursors batch documents from the server (first batch: 101 docs or 16MB, subsequent: 4MB or cursor batch size). Always use cursor iteration for large result sets.
