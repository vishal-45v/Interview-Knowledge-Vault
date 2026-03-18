# Chapter 02 — Documents & Collections: Structured Answers

---

## Answer 1: "Explain all MongoDB update operators and when to use them"

**Point**: MongoDB has atomic update operators that modify specific fields without replacing the entire document, enabling safe concurrent updates.

**Example — Complete update operator reference**:

```javascript
// Document to update:
{
  _id: ObjectId("..."),
  username: "alice",
  balance: NumberDecimal("100.00"),
  loginCount: 42,
  tags: ["nodejs", "mongodb"],
  scores: [85, 90, 78],
  address: { city: "Austin", zip: "78701" },
  lastLogin: null
}

// All field update operators in one example:
db.users.updateOne({ _id: id }, {
  // $set — create or overwrite field
  $set: { lastLogin: new Date(), "address.city": "Dallas" },

  // $unset — remove field (value doesn't matter)
  $unset: { temporaryToken: "" },

  // $inc — atomic increment/decrement
  $inc: { loginCount: 1, balance: NumberDecimal("-5.00") },

  // $mul — multiply
  $mul: { balance: NumberDecimal("1.05") },  // 5% interest

  // $min — update only if new value is LESS than current
  $min: { lowestScore: 78 },

  // $max — update only if new value is GREATER than current
  $max: { highestScore: 95 },

  // $rename — rename a field
  $rename: { "username": "loginName" },

  // $currentDate — server-side current date
  $currentDate: { updatedAt: true }
})

// Array operators in one example:
db.users.updateOne({ _id: id }, {
  $push: { scores: 95 },                      // append (allows duplicates)
  $addToSet: { tags: "typescript" },           // append only if unique
  $pull: { tags: "deprecated" },              // remove matching elements
  $pop: { scores: -1 },                        // remove first (-1) or last (1)
  $pullAll: { scores: [85, 78] }              // remove all listed values
})
```

---

## Answer 2: "How would you implement an atomic inventory decrement that prevents overselling?"

**Point**: Use `findOneAndUpdate` with a condition in the filter to atomically check and modify in a single operation.

```javascript
// WRONG: Non-atomic read-modify-write
async function deductStockWRONG(productId, quantity) {
  const product = await db.products.findOne({ _id: productId })
  if (product.stock < quantity) throw new Error("Insufficient stock")
  // RACE CONDITION: another request may have deducted stock here!
  await db.products.updateOne({ _id: productId }, { $inc: { stock: -quantity } })
  // Could result in negative stock!
}

// CORRECT: Atomic conditional update
async function deductStock(productId, quantity, orderId) {
  const updatedProduct = await db.products.findOneAndUpdate(
    {
      _id: productId,
      stock: { $gte: quantity }    // Condition: stock must be >= quantity
      // This condition and the $inc happen atomically
    },
    {
      $inc: { stock: -quantity },
      $push: {
        reservations: {
          orderId,
          quantity,
          reservedAt: new Date()
        }
      },
      $set: { lastModified: new Date() }
    },
    { returnDocument: "after" }
  )

  if (!updatedProduct) {
    // Either product doesn't exist OR stock was insufficient
    const product = await db.products.findOne({ _id: productId })
    if (!product) throw new Error("Product not found")
    throw new Error(`Insufficient stock. Available: ${product.stock}, Requested: ${quantity}`)
  }

  return updatedProduct
}
```

---

## Answer 3: "Explain bulk write operations and when they're critical for performance"

**Point**: `bulkWrite()` batches multiple write operations into a single network round trip, dramatically reducing latency for batch processing.

```javascript
// Naive approach: N round trips
const products = await fetchProductsToUpdate()
for (const product of products) {
  await db.products.updateOne(
    { _id: product._id },
    { $set: { updatedPrice: product.newPrice } }
  )
}
// For 10,000 products: 10,000 network round trips = very slow

// Optimized: bulkWrite in batches
const BATCH_SIZE = 500
const chunks = chunkArray(products, BATCH_SIZE)

for (const chunk of chunks) {
  const operations = chunk.map(product => ({
    updateOne: {
      filter: { _id: product._id },
      update: { $set: { price: product.newPrice, updatedAt: new Date() } }
    }
  }))

  const result = await db.products.bulkWrite(operations, { ordered: false })
  console.log(`Batch: ${result.modifiedCount} updated, ${result.writeErrors?.length || 0} errors`)
}
// For 10,000 products in batches of 500: 20 round trips

// Mixed operation example (e.g., price adjustment script):
await db.products.bulkWrite([
  // New product
  {
    insertOne: {
      document: { sku: "NEW-001", price: NumberDecimal("19.99"), stock: 100 }
    }
  },
  // Price update
  {
    updateMany: {
      filter: { category: "clearance" },
      update: { $mul: { price: 0.7 } }  // 30% off clearance
    }
  },
  // Remove discontinued
  {
    deleteMany: {
      filter: { status: "discontinued", stock: 0 }
    }
  }
], { ordered: true })  // ordered: true ensures operations run in sequence
```

---

## Answer 4: "How does document validation work and how would you add it to an existing collection?"

**Point**: MongoDB JSON Schema validation enforces document structure at the database level, independent of application code.

```javascript
// Step 1: Audit existing data for issues
db.users.aggregate([
  {
    $facet: {
      missingEmail: [{ $match: { email: { $exists: false } } }, { $count: "count" }],
      wrongAgeType: [{ $match: { age: { $not: { $type: "number" } } } }, { $count: "count" }],
      invalidRole: [{ $match: { role: { $nin: ["user", "admin", "moderator"] } } }, { $count: "count" }]
    }
  }
])

// Step 2: Fix data issues before adding strict validation
db.users.updateMany(
  { email: { $exists: false } },
  { $set: { email: null } }
)

// Step 3: Add validation (use "moderate" first to not break existing data)
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "createdAt"],
      properties: {
        name: {
          bsonType: "string",
          minLength: 1,
          maxLength: 100
        },
        email: {
          bsonType: ["string", "null"],  // allow null for legacy data
          pattern: "^[\\w._%+-]+@[\\w.-]+\\.[a-zA-Z]{2,}$"
        },
        age: {
          bsonType: ["int", "null"],
          minimum: 0,
          maximum: 150
        },
        role: {
          enum: ["user", "admin", "moderator", null]
        }
      }
    }
  },
  validationLevel: "moderate",  // don't reject updates to existing invalid docs
  validationAction: "warn"      // log violations instead of rejecting (dev phase)
})

// Step 4: After validating no new violations, switch to strict enforcement
db.runCommand({
  collMod: "users",
  validationLevel: "strict",
  validationAction: "error"
})
```

---

## Answer 5: "Explain the difference between find+sort+limit vs aggregation pipeline for queries"

```javascript
// find() + sort + limit — simple and efficient for most use cases
db.products.find(
  { category: "electronics", inStock: true },
  { name: 1, price: 1, brand: 1, _id: 0 }
)
.sort({ price: 1 })
.limit(20)

// Advantages:
// - Simpler syntax
// - MongoDB optimizer can use indexes for filter + sort
// - Less memory overhead

// Aggregation pipeline — needed for:
// - Computed fields, transformations
// - Grouping and aggregation
// - Multiple collections ($lookup)
// - Complex filtering on computed values
db.products.aggregate([
  // Stage 1: Filter (can use index if first stage is $match)
  { $match: { category: "electronics", inStock: true } },

  // Stage 2: Add computed field
  {
    $addFields: {
      discountedPrice: {
        $multiply: ["$price", { $subtract: [1, { $divide: ["$discountPercent", 100] }] }]
      }
    }
  },

  // Stage 3: Filter on computed field (can't do this with find())
  { $match: { discountedPrice: { $lt: 100 } } },

  // Stage 4: Shape the output
  {
    $project: {
      name: 1,
      price: 1,
      discountedPrice: { $round: ["$discountedPrice", 2] },
      brand: 1,
      _id: 0
    }
  },

  // Stage 5: Sort by computed field
  { $sort: { discountedPrice: 1 } },

  // Stage 6: Paginate
  { $skip: 0 },
  { $limit: 20 }
])
```

---

## Answer 6: "How would you implement a 'mark as read' feature for a messaging system?"

```javascript
// Message document structure:
{
  _id: ObjectId("..."),
  conversationId: ObjectId("..."),
  senderId: ObjectId("..."),
  recipientId: ObjectId("..."),
  text: "Hey, how are you?",
  sentAt: ISODate("..."),
  readAt: null      // null = unread, Date = when read
}

// Index for efficient unread queries
db.messages.createIndex({ recipientId: 1, readAt: 1, sentAt: -1 })
db.messages.createIndex({ conversationId: 1, sentAt: 1 })

// Mark single message as read
async function markMessageRead(messageId, userId) {
  return db.messages.updateOne(
    {
      _id: messageId,
      recipientId: userId,
      readAt: null  // only update if not already read
    },
    {
      $set: { readAt: new Date() }
    }
  )
}

// Mark all messages in a conversation as read
async function markConversationRead(conversationId, userId) {
  const result = await db.messages.updateMany(
    {
      conversationId: conversationId,
      recipientId: userId,
      readAt: null
    },
    {
      $set: { readAt: new Date() }
    }
  )
  return result.modifiedCount
}

// Get unread message count per conversation
async function getUnreadCounts(userId) {
  return db.messages.aggregate([
    {
      $match: {
        recipientId: userId,
        readAt: null
      }
    },
    {
      $group: {
        _id: "$conversationId",
        unreadCount: { $sum: 1 },
        latestMessage: { $max: "$sentAt" }
      }
    },
    { $sort: { latestMessage: -1 } }
  ]).toArray()
}
```

---

## Answer 7: "Explain how to safely migrate a field name across all documents in production"

```javascript
// Step 1: Add new field alongside old field (zero-downtime approach)
// Phase 1: Deploy code that writes BOTH old and new fields
// New inserts/updates write to both "userName" and "username"

// Step 2: Backfill — add new field to existing documents
// Run in batches to avoid memory pressure and long-running operations
async function migrateFieldName(oldField, newField) {
  let lastId = null
  let migrated = 0

  while (true) {
    const filter = {
      [oldField]: { $exists: true },    // has old field
      [newField]: { $exists: false },   // doesn't have new field yet
      ...(lastId && { _id: { $gt: lastId } })  // cursor-based pagination
    }

    const docs = await db.users.find(filter)
      .sort({ _id: 1 })
      .limit(1000)
      .toArray()

    if (docs.length === 0) break

    const bulkOps = docs.map(doc => ({
      updateOne: {
        filter: { _id: doc._id },
        update: {
          $set: { [newField]: doc[oldField] },
          // Don't remove old field yet — keep for rollback
        }
      }
    }))

    await db.users.bulkWrite(bulkOps, { ordered: false })
    migrated += docs.length
    lastId = docs[docs.length - 1]._id
    console.log(`Migrated ${migrated} documents`)
  }
}

// Step 3: Deploy code that reads new field with fallback
function getUserName(user) {
  return user.username ?? user.userName  // prefer new, fall back to old
}

// Step 4: Verify 100% of documents have new field
const remaining = await db.users.countDocuments({ username: { $exists: false } })
console.log(`Remaining documents without 'username': ${remaining}`)

// Step 5: Remove old field
if (remaining === 0) {
  await db.users.updateMany({}, { $unset: { userName: "" } })
}

// Alternatively, use $rename in a single operation (SLOWER for large collections):
await db.users.updateMany(
  { userName: { $exists: true } },
  { $rename: { userName: "username" } }
)
```

---

## Answer 8: "What is a capped collection and implement a tailable cursor consumer"

```javascript
// Create a capped collection for real-time event streaming
db.createCollection("events", {
  capped: true,
  size: 104857600,  // 100MB circular buffer
  max: 1000000      // max 1M events
})

// Producer — inserts events (fast, no delete overhead)
async function publishEvent(type, payload) {
  await db.events.insertOne({
    type,
    payload,
    publishedAt: new Date(),
    // Note: no _id needed — ObjectId is auto-generated
  })
}

// Consumer — tailable cursor (like tail -f on a file)
async function consumeEvents(fromDate) {
  const query = fromDate ? { publishedAt: { $gte: fromDate } } : {}

  while (true) {
    try {
      const cursor = db.events.find(query).tailable({ awaitData: true })

      while (true) {
        if (await cursor.hasNext()) {
          const event = await cursor.next()
          await processEvent(event)
          // Update query to start from last processed event
          query.publishedAt = { $gt: event.publishedAt }
        } else {
          // No new events — cursor waits (awaitData: true means it waits server-side)
          await new Promise(resolve => setTimeout(resolve, 100))
        }
      }
    } catch (err) {
      if (err.code === 'CursorNotFound' || err.message.includes('tailable')) {
        // Cursor was killed (e.g., server restarted) — restart
        console.log("Cursor invalidated, restarting...")
        await new Promise(resolve => setTimeout(resolve, 1000))
        continue
      }
      throw err
    }
  }
}

// Note: For production streaming, prefer Change Streams over tailable cursors.
// Tailable cursors don't support resuming from a specific position across restarts.
// Change Streams have resume tokens for exactly-once processing.
```

---

## Answer 9: "Explain the aggregation pipeline update and give a real use case"

```javascript
// Use case: Normalize a field that has inconsistent data types

// Problem: 'age' field is stored as string in some docs, number in others
// Caused by a bug in an older version of the API

// Check the extent of the problem
db.users.aggregate([
  {
    $group: {
      _id: { $type: "$age" },
      count: { $sum: 1 }
    }
  }
])
// Result: [{ _id: "double", count: 45000 }, { _id: "string", count: 3200 }]

// Fix: Convert string ages to numbers using pipeline update
db.users.updateMany(
  { age: { $type: "string" } },  // only update documents with string age
  [
    {
      $set: {
        age: {
          $cond: {
            if: { $regexMatch: { input: "$age", regex: "^[0-9]+$" } },
            then: { $toInt: "$age" },
            else: null  // invalid value — set to null for manual review
          }
        },
        ageMigrated: true  // flag for audit
      }
    }
  ]
)

// Another use case: Add a computed denormalized field
db.products.updateMany(
  { slug: { $exists: false } },
  [
    {
      $set: {
        slug: {
          $toLower: {
            $replaceAll: {
              input: {
                $replaceAll: {
                  input: "$name",
                  find: " ",
                  replacement: "-"
                }
              },
              find: "/",
              replacement: ""
            }
          }
        }
      }
    }
  ]
)
```

---

## Answer 10: "Describe document versioning patterns in MongoDB"

```javascript
// Pattern 1: Schema Versioning (lightweight)
// Add a 'schemaVersion' field, handle in application code
{
  _id: ObjectId("..."),
  schemaVersion: 2,     // v1 had 'name', v2 has 'firstName' + 'lastName'
  firstName: "Alice",
  lastName: "Johnson"
}

// Application handles multiple versions
function getUserDisplayName(user) {
  if (user.schemaVersion >= 2) {
    return `${user.firstName} ${user.lastName}`
  } else {
    return user.name  // v1 field
  }
}

// Pattern 2: Full Document Versioning (history tracking)
// Current state in main collection, history in separate collection
// (full implementation in S12 of scenario-questions.md)

// Pattern 3: Delta Versioning (store only changes)
// Compact but requires replaying deltas to reconstruct state
{
  _id: ObjectId("..."),
  documentId: ObjectId("user_id"),
  version: 3,
  delta: {                          // only the fields that changed
    updatedFields: { email: "new@email.com" },
    removedFields: ["legacyField"]
  },
  changedAt: ISODate("..."),
  changedBy: ObjectId("admin_id")
}

// Reconstruct version N:
async function getDocumentAtVersion(docId, targetVersion) {
  const doc = await db.users.findOne({ _id: docId })
  if (doc.currentVersion <= targetVersion) return doc

  // Get all deltas from targetVersion to currentVersion
  const deltas = await db.userDeltas.find({
    documentId: docId,
    version: { $gt: targetVersion, $lte: doc.currentVersion }
  }).sort({ version: -1 }).toArray()

  // Apply deltas in reverse
  let reconstructed = { ...doc }
  for (const delta of deltas) {
    for (const [field, value] of Object.entries(delta.delta.updatedFields || {})) {
      // This would require storing the previous value too for full reversal
      // Simplified: use the 'before' value from the delta
    }
  }
  return reconstructed
}
```
