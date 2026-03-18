# Chapter 02 — Documents & Collections: Theory Questions

---

## Insert Operations

**Q1. What is the difference between `insertOne()` and `insertMany()`?**

```javascript
// insertOne() — inserts a single document, returns insertedId
const result = await db.users.insertOne({
  name: "Alice",
  email: "alice@example.com",
  createdAt: new Date()
})
console.log(result.insertedId)  // ObjectId("...")
console.log(result.acknowledged)  // true

// insertMany() — inserts multiple documents, returns insertedIds array
const result = await db.users.insertMany([
  { name: "Bob", email: "bob@example.com" },
  { name: "Carol", email: "carol@example.com" },
  { name: "Dave", email: "dave@example.com" }
])
console.log(result.insertedCount)  // 3
console.log(result.insertedIds)    // { '0': ObjectId("..."), '1': ObjectId("..."), '2': ObjectId("...") }
```

Key difference: `insertMany()` by default uses `ordered: true` — inserts stop at the first error. With `ordered: false`, it continues inserting remaining documents even if some fail (reporting all errors at the end).

---

**Q2. What is the `ordered` option in `insertMany()` and when does it matter?**

```javascript
// ordered: true (DEFAULT) — stops at first error
try {
  await db.users.insertMany([
    { _id: 1, name: "Alice" },
    { _id: 1, name: "Duplicate!" },  // DuplicateKey error
    { _id: 2, name: "Bob" }           // NEVER INSERTED (stopped at error)
  ], { ordered: true })
} catch (err) {
  // err.insertedCount = 1 (only first doc inserted)
  // Bob was NOT inserted because processing stopped
}

// ordered: false — continues on error, inserts all non-failing docs
try {
  await db.users.insertMany([
    { _id: 1, name: "Alice" },
    { _id: 1, name: "Duplicate!" },  // DuplicateKey error (reported)
    { _id: 2, name: "Bob" }           // INSERTED despite previous error
  ], { ordered: false })
} catch (err) {
  // err.insertedCount = 2 (Alice and Bob both inserted)
  // err.writeErrors contains the duplicate key error
}
```

Use `ordered: false` for bulk data imports where partial success is acceptable (e.g., re-importing a dataset that may already have some documents).

---

## Update Operations

**Q3. What is the difference between `updateOne()`, `updateMany()`, and `replaceOne()`?**

```javascript
// updateOne() — updates FIRST matching document
db.products.updateOne(
  { category: "electronics" },   // filter
  { $set: { onSale: true } }     // update (uses operators)
)

// updateMany() — updates ALL matching documents
db.products.updateMany(
  { category: "electronics" },
  { $set: { onSale: true } }
)

// replaceOne() — REPLACES the entire document (except _id)
// The entire document is replaced with the new document
db.products.replaceOne(
  { _id: ObjectId("...") },
  {
    // Note: _id is NOT specified here — MongoDB keeps the original _id
    name: "New Widget",
    price: NumberDecimal("29.99"),
    category: "electronics"
    // ALL other fields from the original document are GONE
  }
)
```

Critical: `replaceOne()` removes ALL fields not in the replacement document (except `_id`). Most unintended data loss comes from accidentally using `replaceOne()` or `updateOne()` without update operators.

---

**Q4. What are the core update operators in MongoDB?**

```javascript
// $set — set field value (creates field if it doesn't exist)
db.users.updateOne({ _id: id }, { $set: { email: "new@email.com", updatedAt: new Date() } })

// $unset — remove a field entirely
db.users.updateOne({ _id: id }, { $unset: { temporaryFlag: "" } })
// The value for $unset doesn't matter — "" or 1 or anything works

// $inc — atomically increment/decrement a numeric field
db.products.updateOne({ _id: id }, { $inc: { stock: -1, salesCount: 1 } })
// Safe for concurrent updates — atomic operation

// $mul — multiply a field's value
db.prices.updateOne({ _id: id }, { $mul: { price: 1.1 } })  // 10% price increase

// $min — update field only if new value is LESS than current
db.stats.updateOne({ _id: id }, { $min: { lowestPrice: NumberDecimal("9.99") } })

// $max — update field only if new value is GREATER than current
db.stats.updateOne({ _id: id }, { $max: { highestPrice: NumberDecimal("99.99") } })

// $rename — rename a field
db.users.updateMany({}, { $rename: { "userName": "username" } })

// $currentDate — set field to current date
db.orders.updateOne(
  { _id: id },
  { $currentDate: { lastModified: true, lastModifiedTimestamp: { $type: "timestamp" } } }
)

// $setOnInsert — set fields only when document is inserted (upsert scenario)
db.users.updateOne(
  { email: "alice@example.com" },
  {
    $set: { lastLogin: new Date() },
    $setOnInsert: { createdAt: new Date(), role: "user" }  // only on insert
  },
  { upsert: true }
)
```

---

**Q5. What are the array update operators?**

```javascript
// $push — appends an element to an array (creates array if doesn't exist)
db.posts.updateOne({ _id: id }, { $push: { comments: { text: "Great post!", author: "Bob" } } })

// $addToSet — adds element to array ONLY IF not already present (set semantics)
db.users.updateOne({ _id: id }, { $addToSet: { tags: "mongodb" } })
// Idempotent — running multiple times has same result

// $pull — removes all array elements matching a condition
db.posts.updateOne({ _id: id }, { $pull: { tags: "deprecated" } })
// With condition:
db.carts.updateOne({ _id: id }, { $pull: { items: { qty: { $lte: 0 } } } })

// $pop — removes first (-1) or last (1) element
db.queue.updateOne({ _id: id }, { $pop: { items: -1 } })  // remove first
db.queue.updateOne({ _id: id }, { $pop: { items: 1 } })   // remove last

// $pullAll — removes all instances of specified values from array
db.users.updateOne({ _id: id }, { $pullAll: { scores: [50, 75] } })

// $each — used with $push or $addToSet to add multiple elements
db.posts.updateOne(
  { _id: id },
  {
    $push: {
      tags: {
        $each: ["mongodb", "database", "nosql"],
        $slice: -10,     // keep only last 10 tags after push
        $sort: 1,        // sort alphabetically before slicing
        $position: 0     // insert at beginning instead of end
      }
    }
  }
)
```

---

**Q6. What is upsert and how does it work?**

```javascript
// upsert: true — update if exists, INSERT if not found
const result = await db.users.updateOne(
  { email: "alice@example.com" },   // filter
  {
    $set: { lastLogin: new Date(), name: "Alice" },
    $setOnInsert: { createdAt: new Date(), role: "user" }
  },
  { upsert: true }                  // upsert option
)

// result tells you what happened:
if (result.upsertedId) {
  console.log("Document was CREATED:", result.upsertedId)
} else if (result.modifiedCount > 0) {
  console.log("Document was UPDATED")
} else {
  console.log("Document matched but was not modified (values already same)")
}

// findOneAndUpdate with upsert — returns the document
const user = await db.users.findOneAndUpdate(
  { email: "alice@example.com" },
  { $set: { lastLogin: new Date() }, $setOnInsert: { createdAt: new Date() } },
  {
    upsert: true,
    returnDocument: "after"  // return the resulting document (not the original)
  }
)
```

Common use case: session tracking, user "get or create" patterns, idempotent state updates.

---

## findOneAndUpdate vs updateOne

**Q7. What is the difference between `findOneAndUpdate()` and `updateOne()`?**

```javascript
// updateOne() — updates the document, returns a write result (NOT the document)
const result = await db.products.updateOne(
  { _id: productId },
  { $inc: { stock: -1 } }
)
// result = { acknowledged: true, matchedCount: 1, modifiedCount: 1 }
// You do NOT get the document back. Must do a separate find() if needed.

// findOneAndUpdate() — updates and RETURNS the document
const product = await db.products.findOneAndUpdate(
  { _id: productId, stock: { $gt: 0 } },  // atomically check stock and decrement
  { $inc: { stock: -1 } },
  {
    returnDocument: "before"   // "before" = original doc, "after" = updated doc
    // default is "before" (returns document BEFORE the update was applied)
  }
)

if (!product) {
  throw new Error("Product out of stock!")
}

// findOneAndUpdate is ATOMIC — the read and write happen in a single operation
// This prevents race conditions where you check stock, then decrement separately

// findOneAndDelete() — deletes and returns the deleted document
const deletedUser = await db.users.findOneAndDelete({ email: "alice@example.com" })
console.log("Deleted:", deletedUser)  // returns the deleted document
// Without findOneAndDelete, you'd need find() + deleteOne() separately
```

---

## Delete Operations

**Q8. What are the delete operations and their differences?**

```javascript
// deleteOne() — deletes FIRST matching document
const result = await db.users.deleteOne({ email: "alice@example.com" })
// result.deletedCount = 0 or 1

// deleteMany() — deletes ALL matching documents
const result = await db.users.deleteMany({ status: "inactive", lastLogin: { $lt: cutoffDate } })
// result.deletedCount = number of docs deleted

// findOneAndDelete() — deletes and RETURNS the deleted document
const deleted = await db.sessions.findOneAndDelete(
  { sessionToken: "abc123" },
  { sort: { createdAt: 1 } }  // delete oldest matching session
)

// DANGER: deleteMany with empty filter deletes ALL documents!
db.users.deleteMany({})  // Deletes EVERYTHING in the collection!
// The collection itself is not dropped, but all documents are gone.

// Soft delete pattern (safer than hard delete)
db.users.updateOne(
  { _id: userId },
  { $set: { deletedAt: new Date(), isDeleted: true } }
)
// Don't actually delete — mark as deleted. Useful for audit trails.
```

---

## Bulk Write Operations

**Q9. What is `bulkWrite()` and when should you use it?**

```javascript
// bulkWrite() — sends multiple write operations in a single round trip to the server
const result = await db.inventory.bulkWrite([
  // InsertOne
  { insertOne: { document: { sku: "NEW-001", qty: 100, price: NumberDecimal("9.99") } } },

  // UpdateOne
  {
    updateOne: {
      filter: { sku: "WIDGET-A" },
      update: { $inc: { qty: -5 }, $set: { lastSold: new Date() } },
      upsert: false
    }
  },

  // UpdateMany
  {
    updateMany: {
      filter: { category: "electronics", onSale: true },
      update: { $mul: { price: 0.9 } }  // 10% discount
    }
  },

  // ReplaceOne
  {
    replaceOne: {
      filter: { sku: "OLD-SKU" },
      replacement: { sku: "NEW-SKU", qty: 50 }
    }
  },

  // DeleteOne
  { deleteOne: { filter: { sku: "DISCONTINUED" } } },

  // DeleteMany
  { deleteMany: { filter: { qty: 0, lastSold: { $lt: twoYearsAgo } } } }
], {
  ordered: true  // stop on first error (default)
  // ordered: false  // continue on errors, report all at end
})

console.log(result.insertedCount)   // 1
console.log(result.modifiedCount)   // N
console.log(result.deletedCount)    // N
console.log(result.upsertedCount)   // 0
```

Use bulkWrite for:
- Data imports/migrations
- Batch processing (update 1000 documents per batch)
- Reducing round trips — one network call instead of N

---

## Collation

**Q10. What is collation in MongoDB and why does it matter for string comparisons?**

```javascript
// Without collation — byte-order comparison (case-sensitive, locale-unaware)
db.users.find({ name: "alice" })  // won't find "Alice" or "ALICE"
db.users.find().sort({ name: 1 }) // sorts: "Alice" before "alice" (uppercase comes first in ASCII)

// With collation — locale-aware string comparison
db.users.find({ name: "alice" }).collation({
  locale: "en",
  strength: 2  // 1=primary (a==A==á), 2=secondary (a==A but a!=á), 3=tertiary (a!=A)
})
// Now finds "alice", "Alice", "ALICE"

// Creating a collection with default collation
db.createCollection("users", {
  collation: { locale: "en", strength: 2 }
})

// Creating a case-insensitive index
db.users.createIndex(
  { username: 1 },
  {
    unique: true,
    collation: { locale: "en", strength: 2 }  // unique regardless of case
  }
)
// This index enforces: "alice", "Alice", "ALICE" all map to the same unique slot

// Collation strength levels:
// 1 (primary):   a == A == á == à (ignores case AND accents)
// 2 (secondary): a == A, but a != á (ignores case, respects accents)
// 3 (tertiary):  a != A != á != à (full distinction — default binary behavior)
// 4 (quaternary): considers punctuation
// 5 (identical):  full Unicode canonical equivalence
```

---

## Document Validation

**Q11. How do you enforce a schema on a MongoDB collection using JSON Schema validation?**

```javascript
// Create collection with JSON Schema validation
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "age"],
      additionalProperties: false,
      properties: {
        _id: { bsonType: "objectId" },
        name: {
          bsonType: "string",
          minLength: 2,
          maxLength: 100,
          description: "User's full name"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Must be a valid email address"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        role: {
          bsonType: "string",
          enum: ["user", "admin", "moderator"]
        },
        address: {
          bsonType: "object",
          required: ["city"],
          properties: {
            street: { bsonType: "string" },
            city: { bsonType: "string" },
            zip: { bsonType: "string", pattern: "^[0-9]{5}(-[0-9]{4})?$" }
          }
        },
        tags: {
          bsonType: "array",
          items: { bsonType: "string" },
          maxItems: 20
        }
      }
    }
  },
  validationLevel: "strict",    // "strict" (all writes validated) or "moderate" (only matching docs)
  validationAction: "error"     // "error" (reject invalid docs) or "warn" (log but allow)
})

// Add/modify validation on existing collection
db.runCommand({
  collMod: "users",
  validator: { $jsonSchema: { ... } },
  validationLevel: "moderate"
})

// Bypass validation (admin only — use with caution)
db.runCommand({
  insert: "users",
  documents: [{ name: "System", email: "invalid-no-at-sign" }],
  bypassDocumentValidation: true
})
```

---

## Capped Collections

**Q12. What are the limitations of capped collections?**

```javascript
// Create a capped collection
db.createCollection("auditLogs", {
  capped: true,
  size: 52428800,   // 50 MB (required — always specify)
  max: 100000       // optional: max 100K documents
})

// Limitations:
// 1. Cannot delete individual documents (deleteOne/deleteMany not supported)
//    db.auditLogs.deleteOne({ ... })  // throws "can't delete from capped collection"

// 2. Cannot update a document in a way that changes its size
//    Safe: db.auditLogs.updateOne({ _id: id }, { $set: { reviewed: true } })
//    Unsafe (if changes size): { $set: { reviewNotes: "very long text..." } }

// 3. Cannot shard a capped collection (before MongoDB 5.0)
//    (MongoDB 6.0+ allows sharding capped collections with limitations)

// 4. Cannot create a TTL index on a capped collection

// 5. Cannot use $lookup as the "from" collection if it's capped

// What capped collections ARE good for:
// - High-throughput logging (oldest entries auto-removed)
// - Tailable cursors (real-time event notification)

// Tailable cursor example (works only on capped collections)
const cursor = db.auditLogs.find().tailable({ awaitData: true })
while (cursor.hasNext() || cursor.isExhausted() === false) {
  if (cursor.hasNext()) {
    const doc = cursor.next()
    processLogEntry(doc)
  }
}
```

---

## Additional Update Operators

**Q13. What is the `$expr` operator in updates and queries?**

```javascript
// $expr allows using aggregation expressions in query predicates
// Useful for comparing fields within the same document

// Find products where the discountedPrice is less than the originalPrice * 0.5 (50% off+)
db.products.find({
  $expr: {
    $lt: ["$discountedPrice", { $multiply: ["$originalPrice", 0.5] }]
  }
})

// Find orders where quantity * unitPrice !== total (data integrity check)
db.orders.find({
  $expr: {
    $ne: [
      "$total",
      { $multiply: ["$quantity", "$unitPrice"] }
    ]
  }
})

// $expr in aggregation pipeline update
db.products.updateMany(
  {
    $expr: { $lt: ["$currentStock", "$reorderThreshold"] }
  },
  { $set: { needsReorder: true } }
)
```

---

**Q14. How does the aggregation pipeline update syntax work (MongoDB 4.2+)?**

```javascript
// Pipeline update — uses [] instead of {} for the update stage
// Allows referencing other fields in the update

// WITHOUT pipeline update (MongoDB <4.2) — cannot reference other fields
db.users.updateOne(
  { _id: id },
  { $set: { fullName: "hardcoded" } }  // can't concat firstName + lastName
)

// WITH pipeline update (MongoDB 4.2+) — can reference and transform fields
db.users.updateMany(
  {},
  [
    // Stage 1: create fullName from firstName + lastName
    {
      $set: {
        fullName: { $concat: ["$firstName", " ", "$lastName"] },
        updatedAt: "$$NOW"
      }
    }
  ]
)

// More complex example: migrate data structure
db.users.updateMany(
  { address: { $exists: true, $type: "string" } },
  [
    {
      $set: {
        address: {
          full: "$address",  // move old string to address.full
          parsed: false
        }
      }
    }
  ]
)

// Convert string age to integer using pipeline update
db.users.updateMany(
  { age: { $type: "string" } },
  [{ $set: { age: { $toInt: "$age" } } }]
)
```

---

**Q15. What is the `$elemMatch` operator in queries and projections?**

```javascript
// In a query — find documents where AT LEAST ONE array element matches ALL conditions
// Sample data:
db.scores.insertMany([
  { student: "Alice", scores: [{ subject: "math", score: 90 }, { subject: "science", score: 75 }] },
  { student: "Bob", scores: [{ subject: "math", score: 65 }, { subject: "science", score: 85 }] }
])

// WITHOUT $elemMatch — finds docs where ANY element has score > 70
// AND ANY element has subject: "math" (can be different elements!)
db.scores.find({
  "scores.score": { $gt: 70 },
  "scores.subject": "math"
})
// Returns: Alice AND Bob
// Bob: math=65 (not >70), science=85 (>70) — but query still matches!

// WITH $elemMatch — finds docs where SAME element has score > 70 AND subject: "math"
db.scores.find({
  scores: {
    $elemMatch: { subject: "math", score: { $gt: 70 } }
  }
})
// Returns: Alice only (math=90 > 70)
// Bob: math=65 — not >70, so doesn't match

// In a projection — return only the FIRST matching array element
db.scores.find(
  { "scores.subject": "math" },
  { "scores.$": 1 }             // returns only the first matching element
)
// OR use $elemMatch in projection:
db.scores.find(
  { student: "Alice" },
  { scores: { $elemMatch: { subject: "math" } } }
)
// Returns: { scores: [{ subject: "math", score: 90 }] }
```

---

**Q16. What is the difference between `$push` and `$addToSet` for arrays?**

```javascript
// $push — always appends (allows duplicates)
db.users.updateOne({ _id: id }, { $push: { tags: "nodejs" } })
db.users.updateOne({ _id: id }, { $push: { tags: "nodejs" } })
// Result: { tags: ["nodejs", "nodejs"] }  ← duplicates!

// $addToSet — only adds if element NOT already in array (set semantics)
db.users.updateOne({ _id: id }, { $addToSet: { tags: "nodejs" } })
db.users.updateOne({ _id: id }, { $addToSet: { tags: "nodejs" } })
// Result: { tags: ["nodejs"] }  ← no duplicate

// $addToSet comparison uses BSON equality
// Works for primitives: strings, numbers, booleans
db.users.updateOne({ _id: id }, { $addToSet: { roles: "admin" } })

// For objects in arrays, full document equality is used
db.users.updateOne({ _id: id }, {
  $addToSet: {
    preferences: { type: "email", frequency: "weekly" }
  }
})
// Only adds if EXACT object doesn't exist (all fields must match)

// $addToSet with $each for multiple elements
db.users.updateOne({ _id: id }, {
  $addToSet: {
    tags: { $each: ["mongodb", "database", "nosql"] }
  }
})
// Adds each element only if not already present
```

---

**Q17. Explain the `$positional` operator `$` for updating array elements.**

```javascript
// $ (positional operator) — updates the first element that matches the query filter
// Sample document:
{
  _id: 1,
  items: [
    { sku: "WIDGET-A", qty: 5, status: "ordered" },
    { sku: "GADGET-B", qty: 2, status: "shipped" },
    { sku: "DOOHICKEY-C", qty: 8, status: "ordered" }
  ]
}

// Update the STATUS of the first item with sku "WIDGET-A" to "shipped"
db.orders.updateOne(
  { _id: 1, "items.sku": "WIDGET-A" },   // filter MUST include the array field
  { $set: { "items.$.status": "shipped" } }  // $ refers to the matched element
)

// $[<identifier>] — filtered positional operator (update all matching elements)
// Update ALL items with status "ordered" to status "processing"
db.orders.updateOne(
  { _id: 1 },
  { $set: { "items.$[item].status": "processing" } },
  {
    arrayFilters: [
      { "item.status": "ordered" }  // apply to elements where status is "ordered"
    ]
  }
)

// $[] — all positional operator (update ALL elements in array)
// Apply 10% discount to ALL items in order
db.orders.updateOne(
  { _id: 1 },
  { $mul: { "items.$[].price": 0.9 } }
)
```

---

**Q18. What is the purpose of `db.collection.aggregate()` vs `db.collection.find()` for updates?**

```javascript
// find() with projection — read-only, returns filtered/projected documents
const docs = db.users.find(
  { active: true },
  { name: 1, email: 1 }
)

// aggregate() for reads — more powerful transformations, multi-stage pipeline
db.users.aggregate([
  { $match: { active: true } },
  { $project: { name: 1, email: 1, nameLength: { $strLenCP: "$name" } } },
  { $sort: { nameLength: -1 } }
])

// aggregate() for writes — $merge and $out stages write results
db.users.aggregate([
  { $match: { active: true } },
  { $group: { _id: "$country", count: { $sum: 1 } } },
  { $out: "usersByCountry" }          // WRITE results to new collection
  // or $merge to update existing collection:
  // { $merge: { into: "usersByCountry", on: "_id", whenMatched: "replace" } }
])

// Pipeline syntax in updateOne/updateMany — referenced in Q14
// Allows field references and expressions in updates
db.products.updateMany(
  {},
  [{ $set: { slug: { $toLower: { $replaceAll: { input: "$name", find: " ", replacement: "-" } } } } }]
)
```

---

**Q19. What is a "covered update" and how does it differ from a regular update?**

A regular update requires:
1. Find matching documents (may use index)
2. Fetch full document from storage
3. Apply modification
4. Write updated document back

An efficient update using atomic operators (`$set`, `$inc`, `$push`, etc.) minimizes work by:
1. Using an index to find the document quickly
2. Applying only the specified field changes (not rewriting the whole document)
3. WiredTiger updates only the changed portions in the storage engine

```javascript
// Efficient: only updates the 'stock' field
db.products.updateOne(
  { _id: productId },   // index lookup
  { $inc: { stock: -1 } }   // atomic field-level update
)

// Inefficient: replaceOne reads and rewrites entire document
const doc = await db.products.findOne({ _id: productId })
doc.stock -= 1
await db.products.replaceOne({ _id: productId }, doc)
// Problem: race condition AND full document rewrite

// Most efficient for hot fields (atomicity + index):
db.products.findOneAndUpdate(
  { _id: productId, stock: { $gt: 0 } },  // atomic check-and-update
  { $inc: { stock: -1 } },
  { returnDocument: "after" }
)
```

---

**Q20. What are the MongoDB time-series collections and how do they differ from regular collections?**

```javascript
// Time-series collections (MongoDB 5.0+)
// Optimized for sequences of measurements over time (IoT, metrics, events)
db.createCollection("sensorReadings", {
  timeseries: {
    timeField: "timestamp",      // required: field containing the time
    metaField: "metadata",       // optional: field containing labels/tags (device ID, etc.)
    granularity: "seconds"       // "seconds", "minutes", "hours" — affects internal bucketing
  },
  expireAfterSeconds: 2592000    // optional: TTL for time-series documents
})

// Insert a time-series document
db.sensorReadings.insertOne({
  timestamp: new Date(),
  metadata: { deviceId: "sensor-001", location: "warehouse-A" },
  temperature: 22.5,
  humidity: 45.2,
  pressure: 1013.25
})

// Query time-series (same as regular find)
db.sensorReadings.find({
  "metadata.deviceId": "sensor-001",
  timestamp: { $gte: ISODate("2024-01-15T00:00:00Z"), $lt: ISODate("2024-01-16T00:00:00Z") }
})

// Internal optimization: MongoDB groups measurements into "buckets" for compression
// One bucket = many measurements from same metadata/timeframe
// Results in ~10x compression vs regular collection for time-series data

// Limitations vs regular collections:
// - Cannot create unique indexes (except on timeField + metaField combinations)
// - Cannot use transactions on time-series collections
// - Cannot update or delete individual measurements (MongoDB 6.1+ allows limited deletes)
// - Cannot be used as a $lookup target (as of MongoDB 7.0)
```
