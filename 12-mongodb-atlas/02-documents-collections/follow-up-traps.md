# Chapter 02 — Documents & Collections: Follow-Up Traps

---

## Trap 1: updateOne Without $set Replaces the Entire Document!

**Setup**: Candidate explains updateOne for modifying documents.

**Trap question**: "What does `db.users.updateOne({ _id: id }, { name: 'Alice Updated' })` do?"

**Wrong answer**: "It updates the name field of the matching user to 'Alice Updated'."

**Correct answer**: This REPLACES the entire document with `{ name: 'Alice Updated' }` (except `_id`). All other fields are DELETED.

```javascript
// Original document:
{ _id: ObjectId("..."), name: "Alice", email: "alice@example.com", age: 32, role: "admin" }

// BAD — this is a REPLACE, not an update
db.users.updateOne({ _id: id }, { name: "Alice Updated" })

// Result: { _id: ObjectId("..."), name: "Alice Updated" }
// email, age, role are ALL GONE!

// CORRECT — use $set operator
db.users.updateOne({ _id: id }, { $set: { name: "Alice Updated" } })

// Result: { _id: ObjectId("..."), name: "Alice Updated", email: "alice@example.com", age: 32, role: "admin" }
// Only name was changed

// The "no operator = replace" rule applies to updateOne AND updateMany
// ONLY replaceOne intentionally replaces the full document
// The driver will warn you in newer versions, but won't stop you
```

This is one of the most common and destructive MongoDB mistakes. Always verify your updates have `$set` (or another operator) unless you intentionally want a full replacement.

---

## Trap 2: $push vs $addToSet — Duplicates

**Setup**: Candidate explains array update operators.

**Trap question**: "If you run `$push: { tags: 'mongodb' }` five times, how many times does 'mongodb' appear in the array?"

**Wrong answer**: "Once — MongoDB deduplicates array elements."

**Correct answer**: Five times. `$push` always appends, regardless of duplicates.

```javascript
// After 5 x $push:
{ tags: ["mongodb", "mongodb", "mongodb", "mongodb", "mongodb"] }

// $addToSet only adds if not present:
db.users.updateOne({ _id: id }, { $addToSet: { tags: "mongodb" } })
// After 5 x $addToSet: { tags: ["mongodb"] }

// Common bug: using $push in a loop without realizing it creates duplicates
// Example: user clicks "add tag" button 3 times due to slow network
// With $push: 3 copies of the tag
// With $addToSet: 1 copy (idempotent)

// When IS $push appropriate over $addToSet?
// - Order history (you WANT duplicates: user bought same item twice)
// - Event log (each event is a separate entry)
// - Ordered sequences where position matters
// When use $addToSet:
// - Tags, categories, permissions, roles (set semantics — no duplicates)
// - User preferences, subscriptions
```

---

## Trap 3: findOneAndUpdate Returns OLD Document by Default

**Setup**: Candidate explains findOneAndUpdate.

**Trap question**: "After calling `findOneAndUpdate` to increment a counter, the returned document's counter value is lower than expected. Why?"

**Wrong answer**: "That's a bug — findOneAndUpdate should return the updated value."

**Correct answer**: `findOneAndUpdate` returns the document BEFORE the update by default (`returnDocument: "before"` is the default).

```javascript
// Document: { _id: 1, counter: 5 }

// DEFAULT behavior — returns BEFORE update
const result = await db.counters.findOneAndUpdate(
  { _id: 1 },
  { $inc: { counter: 1 } }
)
console.log(result.counter)  // 5 (the value BEFORE increment!)
// The DB now has counter: 6, but you got back 5

// To get the value AFTER the update:
const result = await db.counters.findOneAndUpdate(
  { _id: 1 },
  { $inc: { counter: 1 } },
  { returnDocument: "after" }  // <-- add this!
)
console.log(result.counter)  // 6 (correct)

// The legacy option was called "returnNewDocument: true" (older driver versions)
// Modern drivers use: returnDocument: "after" | "before"
```

---

## Trap 4: deleteMany with Empty Filter `{}` Deletes ALL Documents

**Setup**: Candidate explains delete operations.

**Trap question**: "What does `db.users.deleteMany({})` do?"

**Wrong answer**: "It returns an error because the filter is empty."

**Correct answer**: An empty filter `{}` matches ALL documents. `deleteMany({})` deletes every document in the collection.

```javascript
// This deletes EVERY document in the users collection:
db.users.deleteMany({})

// The collection still exists (not dropped), but is now empty
// db.users.countDocuments() === 0

// Similarly dangerous:
db.users.updateMany({}, { $set: { role: "guest" } })
// Makes EVERY user a guest!

// Protection: require explicit confirmation in application code
async function deleteUsersByStatus(status) {
  if (status === undefined || status === null) {
    throw new Error("Status filter required — cannot delete all users")
  }
  return db.users.deleteMany({ status })
}

// MongoDB Atlas has "Data Explorer" guardrails
// For CLI: always double-check your filter before deleteMany
// For production scripts: log the count before deletion
const count = await db.users.countDocuments(filter)
console.log(`About to delete ${count} documents with filter:`, filter)
if (count > 1000) throw new Error("Too many documents — please confirm deletion")
```

---

## Trap 5: insertMany `ordered: true` Stops at First Error

**Setup**: Candidate explains bulk insert operations.

**Trap question**: "You run `insertMany` with 1000 documents where document 500 has a duplicate key error. How many documents get inserted?"

**Wrong answer (with ordered:true)**: "999 — MongoDB skips the duplicate and continues."
**Wrong answer (with ordered:false)**: "0 — the error causes a rollback."

**Correct answer**:
- With `ordered: true` (default): 499 documents inserted (first 499 succeed, stops at 500)
- With `ordered: false`: 999 documents inserted (skips the duplicate, continues with 501-1000)

```javascript
// ordered: true (DEFAULT) — stops at first error
try {
  await db.users.insertMany([
    { _id: 1, name: "Alice" },      // inserted ✓
    { _id: 2, name: "Bob" },        // inserted ✓
    { _id: 1, name: "Dup!" },       // DuplicateKey error! Stops here
    { _id: 3, name: "Carol" }       // NEVER reached
  ], { ordered: true })
} catch (err) {
  console.log(err.result.nInserted)  // 2
  // Carol was NOT inserted
}

// ordered: false — continues after errors
try {
  await db.users.insertMany([
    { _id: 1, name: "Alice" },      // inserted ✓
    { _id: 2, name: "Bob" },        // inserted ✓
    { _id: 1, name: "Dup!" },       // DuplicateKey error! But continues
    { _id: 3, name: "Carol" }       // inserted ✓
  ], { ordered: false })
} catch (err) {
  console.log(err.result.nInserted)  // 3
  console.log(err.writeErrors)       // Array of errors for the duplicate
}
```

---

## Trap 6: ObjectId Timestamp Extraction

**Setup**: Candidate explains ObjectId structure.

**Trap question**: "Can you use ObjectId to query for documents created in a specific date range WITHOUT having a separate `createdAt` field?"

**Correct answer**: Yes, but with caveats:

```javascript
// Create ObjectIds representing the start and end of the date range
function objectIdFromDate(date) {
  return ObjectId.createFromTime(date.getTime() / 1000)
}

const startId = objectIdFromDate(new Date("2024-01-01T00:00:00Z"))
const endId = objectIdFromDate(new Date("2024-02-01T00:00:00Z"))

// Query using ObjectId range
db.users.find({
  _id: {
    $gte: startId,
    $lt: endId
  }
})
// This uses the _id index — very efficient!

// CAVEATS:
// 1. ObjectId timestamp is second-precision only (not millisecond)
// 2. ObjectId is generated CLIENT-SIDE — client clock drift means
//    the timestamp may not exactly match server insertion time
// 3. Custom _id values (strings, numbers) don't have embedded timestamps
// 4. Documents with pre-set ObjectId values (not auto-generated) may not
//    have accurate timestamps

// Best practice: still add createdAt field for clarity and flexibility
// Use ObjectId range queries only when createdAt is unavailable
```

---

## Trap 7: $pull with Objects vs Primitives

**Setup**: Candidate explains $pull for removing array elements.

**Trap question**: "You have `tags: ['mongodb', 'atlas', 'mongodb']`. After `$pull: { tags: 'mongodb' }`, what does the array look like?"

**Wrong answer**: "It removes the first occurrence, leaving `['atlas', 'mongodb']`."

**Correct answer**: `$pull` removes ALL matching elements:

```javascript
// Before: { tags: ["mongodb", "atlas", "mongodb"] }
db.users.updateOne({ _id: id }, { $pull: { tags: "mongodb" } })
// After: { tags: ["atlas"] }
// BOTH occurrences of "mongodb" are removed

// $pull with objects — uses EXACT match by default
// Before: { scores: [{ game: "chess", score: 100 }, { game: "chess", score: 200 }] }
db.users.updateOne(
  { _id: id },
  { $pull: { scores: { game: "chess", score: 100 } } }
)
// Removes ONLY the exact matching object
// After: { scores: [{ game: "chess", score: 200 }] }

// $pull with condition — removes all elements matching the condition
db.orders.updateOne(
  { _id: id },
  { $pull: { items: { qty: { $lte: 0 } } } }
)
// Removes all items with qty <= 0

// If you want to remove just ONE occurrence, use $unset + $pull in two steps
// Or use a manual approach with JavaScript (not atomic)
```

---

## Trap 8: Capped Collections Cannot Be Sharded (Pre-MongoDB 6.0)

**Setup**: Candidate recommends capped collections for logging at scale.

**Trap question**: "You recommend capped collections for a high-throughput logging system that will handle 10TB of data. What's the catch?"

**Wrong answer**: "Just make the capped collection large enough."

**Correct answer**: Before MongoDB 6.0, capped collections cannot be sharded. A single capped collection is limited to one shard's storage capacity. At 10TB, this is a significant limitation.

```javascript
// Alternatives to capped collections at scale:

// Option 1: TTL index on a regular (shardable) collection
db.logs.createIndex({ ts: 1 }, { expireAfterSeconds: 2592000 })  // 30 days
sh.shardCollection("logs", { ts: "hashed" })  // shard for scale

// Option 2: Time-series collection (MongoDB 5.0+)
// Better compression, auto-expiration, can be sharded
db.createCollection("logs", {
  timeseries: {
    timeField: "ts",
    metaField: "metadata",
    granularity: "seconds"
  },
  expireAfterSeconds: 2592000
})

// Option 3: Manual capping with Atlas Online Archive
// Keep 30 days in MongoDB, older data automatically moved to S3
// Query both via Data Federation

// Capped collections ARE appropriate when:
// - Small, fixed-size log buffer (< single shard capacity)
// - Tailable cursor is required
// - Natural order is important
// - Data is in a single-shard or non-sharded deployment
```

---

## Trap 9: Document Validation Only Applies to New/Updated Documents

**Setup**: Candidate adds JSON Schema validation to a collection.

**Trap question**: "You add strict validation to an existing collection that already has 10 million documents. Some existing documents don't conform to the new schema. What happens to them?"

**Wrong answer**: "They'll be rejected/flagged immediately."

**Correct answer**: Validation only applies to **write operations** (inserts and updates), not to existing data. Existing non-conforming documents remain in the collection unchanged.

```javascript
// Existing document (doesn't have required "email" field):
{ _id: 1, name: "Alice" }  // missing email

// Add validation requiring email
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      required: ["name", "email"],
      properties: {
        email: { bsonType: "string" }
      }
    }
  },
  validationLevel: "strict"  // only validates NEW and UPDATED documents
})

// Existing document { _id: 1, name: "Alice" } is STILL IN THE COLLECTION
// Reads of this document work fine
// But if you try to UPDATE it without email, the update fails:
db.users.updateOne(
  { _id: 1 },
  { $set: { name: "Alice Updated" } }  // fails — document still missing email
)

// validationLevel options:
// "strict" — validates all inserts and all updates
// "moderate" — validates inserts and updates to documents that ALREADY pass validation
//              (existing invalid docs can still be updated without conforming!)

// To validate existing data, run a manual audit:
db.users.find({ email: { $exists: false } }).count()
// Then fix or migrate non-conforming documents
```

---

## Trap 10: bulkWrite `ordered: false` Error Handling

**Setup**: Candidate recommends bulkWrite for efficiency.

**Trap question**: "If you run `bulkWrite` with `ordered: false` and 5 out of 100 operations fail, does it throw an exception?"

**Wrong answer**: "No — it silently ignores errors."
**Wrong answer**: "Yes — it throws and none of the 100 operations complete."

**Correct answer**: It throws a `BulkWriteError`, but the 95 successful operations ARE committed. The error object contains details about the failures.

```javascript
try {
  const result = await db.collection.bulkWrite([...100 operations...], { ordered: false })
} catch (err) {
  if (err.name === "MongoBulkWriteError") {
    // err.result contains counts for ALL operations (including successful ones)
    console.log("Inserted:", err.result.nInserted)
    console.log("Updated:", err.result.nModified)
    console.log("Deleted:", err.result.nRemoved)

    // err.writeErrors contains the individual failures
    err.writeErrors.forEach(writeError => {
      console.log("Failed op index:", writeError.index)
      console.log("Error code:", writeError.code)
      console.log("Error message:", writeError.errmsg)
    })
  }
}

// The 95 successful operations are committed to the database.
// This is NOT a transaction — there's no rollback.
// If you need all-or-nothing semantics, use a transaction.
```

---

## Trap 11: findOneAndUpdate vs updateOne Performance

**Setup**: Candidate uses findOneAndUpdate everywhere.

**Trap question**: "Is `findOneAndUpdate` more or less efficient than `updateOne`? When should you use each?"

**Wrong answer**: "They're the same — findOneAndUpdate just returns the document."

**Correct answer**: `findOneAndUpdate` has overhead because it must return the full document:

- `findOneAndUpdate` fetches the full document from storage (to return it)
- `updateOne` only applies the modification — no document fetch needed
- `findOneAndUpdate` is also slightly more expensive in terms of server-side processing

```javascript
// Use updateOne when you don't need the resulting document
// (simpler, lower overhead, especially for bulk updates)
await db.products.updateOne(
  { _id: id },
  { $inc: { views: 1 } }
)

// Use findOneAndUpdate when you need the document atomically with the update:
// 1. Atomic check-and-modify (need to validate the found document)
const product = await db.products.findOneAndUpdate(
  { _id: id, stock: { $gt: 0 } },  // need to know if stock existed
  { $inc: { stock: -1 } },
  { returnDocument: "after" }
)
if (!product) throw new Error("Out of stock")

// 2. Get the value BEFORE the update (for audit or notification)
const oldUser = await db.users.findOneAndUpdate(
  { _id: userId },
  { $set: { email: newEmail } }
  // returnDocument: "before" is default
)
await sendEmailChangeNotification(oldUser.email, newEmail)

// 3. Queue/workflow pattern — atomically claim a task and get its details
const task = await db.tasks.findOneAndUpdate(
  { status: "pending" },
  { $set: { status: "processing", workerId: id } },
  { returnDocument: "after", sort: { priority: -1 } }
)
```

---

## Trap 12: $currentDate vs new Date() in Updates

**Setup**: Candidate uses `$set: { updatedAt: new Date() }` for timestamps.

**Trap question**: "What's the difference between `$set: { updatedAt: new Date() }` and `$currentDate: { updatedAt: true }`?"

**Correct answer**: Both set the field to the current datetime, BUT:

```javascript
// $set: { updatedAt: new Date() }
// - new Date() is evaluated CLIENT-SIDE (in the application driver)
// - The time is the application server's clock
// - Time may differ from MongoDB server's clock (clock skew in distributed systems)
// - The timestamp is included in the network request

// $currentDate: { updatedAt: true }
// - Evaluated SERVER-SIDE on the MongoDB server
// - Uses MongoDB server's clock (consistent across all writes to same replica set)
// - Slightly more compact wire protocol (no timestamp in payload)

db.users.updateOne(
  { _id: id },
  {
    $set: { name: "Alice" },
    $currentDate: {
      updatedAt: true,                            // sets to current date (ISODate)
      internalTimestamp: { $type: "timestamp" }   // sets to BSON Timestamp (oplog format)
    }
  }
)

// In practice, for most applications the difference is negligible
// Server-side $currentDate is slightly preferred for audit fields
// where server time accuracy matters over client-observed time

// For MongoDB Atlas multi-region deployments, using server-side time
// ensures all writes to the primary use the same clock reference
```
