# Chapter 02 — Documents & Collections: Scenario Questions

---

**S1. You're building a shopping cart system. A user adds an item to their cart. If the item already exists, increment its quantity; if not, add it as a new cart item. Write the MongoDB operation.**

```javascript
// Cart document structure:
{
  _id: ObjectId("..."),
  userId: ObjectId("user_id"),
  items: [
    { sku: "WIDGET-A", name: "Widget A", price: NumberDecimal("9.99"), qty: 2 },
    { sku: "GADGET-B", name: "Gadget B", price: NumberDecimal("24.99"), qty: 1 }
  ],
  updatedAt: ISODate("...")
}

// Add item to cart (increment if exists, add if not)
// Approach 1: Check if item exists and use positional update
const cartItemExists = await db.carts.findOne({
  userId: userId,
  "items.sku": sku
})

if (cartItemExists) {
  // Item exists — increment quantity
  await db.carts.updateOne(
    { userId: userId, "items.sku": sku },
    {
      $inc: { "items.$.qty": qtyToAdd },
      $set: { updatedAt: new Date() }
    }
  )
} else {
  // Item doesn't exist — push new item
  await db.carts.updateOne(
    { userId: userId },
    {
      $push: {
        items: { sku, name, price: NumberDecimal(price.toString()), qty: qtyToAdd }
      },
      $set: { updatedAt: new Date() },
      $setOnInsert: { userId: userId, createdAt: new Date() }
    },
    { upsert: true }  // creates cart if it doesn't exist
  )
}

// Approach 2 (Cleaner, but requires two operations):
// Use arrayFilters for cleaner conditional update
// Note: can't do "push if not exists, increment if exists" in a single atomic op
// The two-step approach (findOne + update) is standard here

// Approach 3: Application-level with transaction (most reliable)
const session = client.startSession()
await session.withTransaction(async () => {
  const cart = await db.carts.findOneAndUpdate(
    { userId: userId },
    { $setOnInsert: { userId, items: [], createdAt: new Date() } },
    { upsert: true, returnDocument: "after", session }
  )

  const itemIndex = cart.items.findIndex(i => i.sku === sku)
  if (itemIndex >= 0) {
    await db.carts.updateOne(
      { userId: userId },
      { $inc: { [`items.${itemIndex}.qty`]: qtyToAdd }, $set: { updatedAt: new Date() } },
      { session }
    )
  } else {
    await db.carts.updateOne(
      { userId: userId },
      { $push: { items: { sku, name, price: NumberDecimal(price.toString()), qty: qtyToAdd } }, $set: { updatedAt: new Date() } },
      { session }
    )
  }
})
```

---

**S2. You need to implement a "like" button for posts. Each user can like a post once. Clicking again unlikes. Design the MongoDB operation.**

```javascript
// Post document structure:
{
  _id: ObjectId("post_id"),
  title: "My Blog Post",
  likedBy: [ObjectId("user1_id"), ObjectId("user2_id")],
  likeCount: 2
}

// Toggle like (like if not liked, unlike if already liked)
// Check if user already liked the post
const post = await db.posts.findOne({
  _id: postId,
  likedBy: userId
})

if (post) {
  // User already liked — remove like
  await db.posts.updateOne(
    { _id: postId },
    {
      $pull: { likedBy: userId },
      $inc: { likeCount: -1 }
    }
  )
} else {
  // User hasn't liked — add like
  await db.posts.updateOne(
    { _id: postId },
    {
      $addToSet: { likedBy: userId },
      $inc: { likeCount: 1 }
    }
  )
}

// CAVEAT: likedBy array grows unbounded for viral posts!
// For posts with millions of likes, use a separate likes collection:
// likes: { _id: ObjectId, postId: ObjectId, userId: ObjectId, likedAt: Date }
// db.likes.createIndex({ postId: 1, userId: 1 }, { unique: true })

// Toggle with separate likes collection
async function toggleLike(postId, userId) {
  try {
    // Try to insert — fails if already liked (unique index)
    await db.likes.insertOne({ postId, userId, likedAt: new Date() })
    await db.posts.updateOne({ _id: postId }, { $inc: { likeCount: 1 } })
    return { liked: true }
  } catch (err) {
    if (err.code === 11000) {
      // Already liked — remove it
      await db.likes.deleteOne({ postId, userId })
      await db.posts.updateOne({ _id: postId }, { $inc: { likeCount: -1 } })
      return { liked: false }
    }
    throw err
  }
}
```

---

**S3. A batch job needs to import 500,000 product records from a CSV file into MongoDB. How would you optimize this import?**

```javascript
// Efficient bulk import strategy
const fs = require("fs")
const csv = require("csv-parser")

const BATCH_SIZE = 1000  // optimal batch size for bulkWrite

async function importProducts(filePath) {
  const db = client.db("ecommerce")
  let batch = []
  let totalImported = 0
  let totalErrors = 0

  const stream = fs.createReadStream(filePath).pipe(csv())

  for await (const row of stream) {
    // Transform CSV row to MongoDB document
    const doc = {
      sku: row.sku,
      name: row.name,
      category: row.category,
      price: NumberDecimal(row.price),
      stock: parseInt(row.stock),
      isActive: row.active === "true",
      tags: row.tags ? row.tags.split(",").map(t => t.trim()) : [],
      importedAt: new Date()
    }

    batch.push({
      updateOne: {
        filter: { sku: doc.sku },           // upsert by SKU
        update: { $set: doc },
        upsert: true
      }
    })

    if (batch.length >= BATCH_SIZE) {
      const result = await flushBatch(db, batch)
      totalImported += result.upsertedCount + result.modifiedCount
      totalErrors += result.errors
      batch = []

      console.log(`Progress: ${totalImported} imported, ${totalErrors} errors`)
    }
  }

  // Flush remaining
  if (batch.length > 0) {
    const result = await flushBatch(db, batch)
    totalImported += result.upsertedCount + result.modifiedCount
  }

  console.log(`Import complete: ${totalImported} products, ${totalErrors} errors`)
}

async function flushBatch(db, batch) {
  try {
    const result = await db.collection("products").bulkWrite(batch, { ordered: false })
    return { ...result, errors: 0 }
  } catch (err) {
    // BulkWriteError — some operations succeeded
    return {
      upsertedCount: err.result?.nUpserted || 0,
      modifiedCount: err.result?.nModified || 0,
      errors: err.writeErrors?.length || 0
    }
  }
}
```

Key optimizations:
- Batch size of 500–2000 documents (test for your document size)
- `ordered: false` for parallel error handling
- `upsert: true` for idempotent imports (safe to re-run)
- Stream the file (don't load all 500K rows into memory)
- Drop indexes before import, recreate after (for very large imports)

---

**S4. You have a `notifications` collection. When a user dismisses all notifications, you need to mark all their unread notifications as read in a single operation. The collection has 50 million documents.**

```javascript
// Documents:
{
  _id: ObjectId("..."),
  userId: ObjectId("user_id"),
  type: "comment",
  message: "Bob replied to your post",
  read: false,
  createdAt: ISODate("...")
}

// Index needed for this operation to be efficient
db.notifications.createIndex({ userId: 1, read: 1 })

// Mark all unread notifications as read for a specific user
const result = await db.notifications.updateMany(
  { userId: userId, read: false },
  {
    $set: {
      read: true,
      readAt: new Date()
    }
  }
)
console.log(`Marked ${result.modifiedCount} notifications as read`)

// This is efficient because:
// 1. Index { userId: 1, read: 1 } covers the filter
// 2. Only updates documents matching the filter
// 3. Atomic per document (no transaction needed for this use case)

// For the dashboard "unread count" without counting all documents:
// Use a counter field on the user document and $inc it
db.users.updateOne(
  { _id: userId },
  { $set: { unreadNotifications: 0 } }
)
// Increment counter when creating notification:
db.users.updateOne({ _id: userId }, { $inc: { unreadNotifications: 1 } })
```

---

**S5. You're building a leaderboard. When a user's score changes, you need to update their entry if it exists, create it if not, and ONLY update if the new score is higher than the current one.**

```javascript
// Leaderboard document:
{
  _id: ObjectId("..."),
  userId: ObjectId("user_id"),
  username: "alice_gamer",
  score: 15000,
  achievedAt: ISODate("...")
}

// Upsert with $max to only update if new score is higher
const result = await db.leaderboard.updateOne(
  { userId: userId },
  {
    $max: { score: newScore },          // only updates if newScore > current score
    $setOnInsert: {                      // these fields set ONLY on insert
      userId: userId,
      username: username,
      createdAt: new Date()
    },
    $set: {
      // This runs on every match — use conditional update pattern instead
      // for achievedAt to only update when score improves
    }
  },
  { upsert: true }
)

// More precise: update achievedAt ONLY when score improves
// Requires two operations or a transaction:
const currentEntry = await db.leaderboard.findOne({ userId })
if (!currentEntry || newScore > currentEntry.score) {
  await db.leaderboard.updateOne(
    { userId },
    {
      $max: { score: newScore },
      $set: { achievedAt: new Date(), username },
      $setOnInsert: { userId, createdAt: new Date() }
    },
    { upsert: true }
  )
}

// For real-time leaderboard queries:
db.leaderboard.createIndex({ score: -1 })  // for rank queries
db.leaderboard.find().sort({ score: -1 }).limit(100)  // top 100
```

---

**S6. A user management system stores audit history. Every time a user's role changes, you need to append to an audit log array in the user document. But you're worried the array will grow too large over time.**

```javascript
// User document with bounded audit log (Bucket Pattern for fixed window)
{
  _id: ObjectId("..."),
  username: "alice",
  currentRole: "admin",
  auditLog: [
    { role: "user", changedAt: ISODate("2024-01-01"), changedBy: "system" },
    { role: "moderator", changedAt: ISODate("2024-03-15"), changedBy: "admin_bob" },
    { role: "admin", changedAt: ISODate("2024-06-20"), changedBy: "admin_carol" }
    // Keep only last N entries in the user document
  ]
}

// Keep only last 10 audit entries using $push with $slice
await db.users.updateOne(
  { _id: userId },
  {
    $set: { currentRole: newRole },
    $push: {
      auditLog: {
        $each: [{ role: newRole, changedAt: new Date(), changedBy: changedByUserId }],
        $slice: -10,   // keep only the LAST 10 entries (trims oldest)
        $sort: { changedAt: 1 }  // sort by date before slicing
      }
    }
  }
)

// For full audit history: write to a separate audit_log collection
await db.auditLog.insertOne({
  entityType: "user",
  entityId: userId,
  field: "role",
  oldValue: oldRole,
  newValue: newRole,
  changedAt: new Date(),
  changedBy: changedByUserId,
  ipAddress: req.ip
})

// Index for audit queries
db.auditLog.createIndex({ entityId: 1, changedAt: -1 })
db.auditLog.createIndex({ changedAt: 1 }, { expireAfterSeconds: 31536000 }) // 1 year TTL
```

---

**S7. You need to implement a "soft delete" system for user accounts — deleted accounts should be invisible to normal queries but retained for compliance for 7 years.**

```javascript
// Add soft delete support to user documents
// Instead of deleting, mark as deleted
await db.users.updateOne(
  { _id: userId },
  {
    $set: {
      isDeleted: true,
      deletedAt: new Date(),
      deletedBy: adminUserId,
      // Anonymize PII while retaining account for compliance
      email: `deleted_${userId}@anonymized.example`,
      name: "Deleted User"
    },
    $unset: {
      // Remove sensitive fields immediately
      phoneNumber: "",
      address: "",
      paymentMethods: ""
    }
  }
)

// Create a partial index to efficiently query non-deleted users
// This index ONLY includes documents where isDeleted is NOT true
db.users.createIndex(
  { email: 1 },
  {
    partialFilterExpression: { isDeleted: { $ne: true } },
    unique: true  // email unique among active users
  }
)

// Regular application queries automatically exclude deleted users
// Add to all user queries:
db.users.find({
  $and: [
    { isDeleted: { $ne: true } },   // exclude deleted
    { ...yourFilter }
  ]
})

// OR: create a view that filters deleted users
db.createView("activeUsers", "users", [
  { $match: { isDeleted: { $ne: true } } }
])
// Now queries against activeUsers collection automatically exclude deleted
db.activeUsers.find({ country: "US" })

// Compliance archive after 7 years
db.users.createIndex(
  { deletedAt: 1 },
  { expireAfterSeconds: 220752000 }  // 7 years = 220,752,000 seconds
  // Only affects documents where deletedAt field exists
)
```

---

**S8. You're storing product inventory. Multiple warehouse workers simultaneously try to reduce the stock of the same product. How do you prevent overselling (stock going below 0)?**

```javascript
// Anti-pattern: read-then-update (RACE CONDITION)
const product = await db.products.findOne({ _id: productId })
if (product.stock >= requestedQty) {
  await db.products.updateOne(
    { _id: productId },
    { $inc: { stock: -requestedQty } }
  )
  // PROBLEM: between findOne and updateOne, another thread may have
  // reduced stock below 0!
}

// CORRECT: Atomic conditional update
const result = await db.products.findOneAndUpdate(
  {
    _id: productId,
    stock: { $gte: requestedQty }   // condition checked atomically with update
  },
  {
    $inc: { stock: -requestedQty },
    $push: {
      reservations: {
        orderId: orderId,
        qty: requestedQty,
        reservedAt: new Date()
      }
    }
  },
  { returnDocument: "after" }
)

if (!result) {
  throw new Error("Insufficient stock — product may have been sold by another request")
}

// This is ATOMIC: the filter condition AND the update happen as one operation
// WiredTiger document-level locking ensures no other write can interleave

// For very high concurrency: optimistic locking with version field
{
  _id: productId,
  stock: 50,
  version: 42       // increment on every update
}

const product = await db.products.findOne({ _id: productId })
const result = await db.products.updateOne(
  {
    _id: productId,
    version: product.version,          // must match the version we read
    stock: { $gte: requestedQty }
  },
  {
    $inc: { stock: -requestedQty, version: 1 }
  }
)

if (result.matchedCount === 0) {
  // Another update happened between our read and write — retry
  await retryWithBackoff(() => processOrder(orderId, productId, requestedQty))
}
```

---

**S9. Describe how you would implement a pagination system for a product catalog with 10 million products using MongoDB.**

```javascript
// Option 1: Skip/Limit pagination (simple but slow at high offset)
// Page 1:
db.products.find({ category: "electronics" })
  .sort({ _id: 1 })
  .skip(0)
  .limit(20)

// Page 500:
db.products.find({ category: "electronics" })
  .sort({ _id: 1 })
  .skip(9980)  // SLOW: MongoDB must scan and discard 9980 documents!
  .limit(20)

// Problem with skip/limit: O(n) performance at high page numbers

// Option 2: Cursor-based pagination (keyset pagination) — RECOMMENDED
// Use the last document's _id (or sort field) as the "cursor"

// Page 1:
const page1 = await db.products.find({ category: "electronics" })
  .sort({ _id: 1 })
  .limit(20)
  .toArray()

const lastId = page1[page1.length - 1]._id

// Page 2 (using last seen _id as cursor):
const page2 = await db.products.find({
  category: "electronics",
  _id: { $gt: lastId }   // start AFTER the last seen document
})
.sort({ _id: 1 })
.limit(20)
.toArray()

// This is O(log n) — uses the index directly, no scanning!

// Option 3: Cursor-based with compound sort field (e.g., price + _id)
// When sorting by price, use (price, _id) as composite cursor
let lastDoc = page1[page1.length - 1]

const nextPage = await db.products.find({
  category: "electronics",
  $or: [
    { price: { $gt: lastDoc.price } },
    { price: lastDoc.price, _id: { $gt: lastDoc._id } }  // tie-break with _id
  ]
})
.sort({ price: 1, _id: 1 })
.limit(20)
.toArray()

// Return pagination metadata to client:
return {
  data: page2,
  nextCursor: page2.length === 20 ? page2[page2.length - 1]._id : null,
  hasMore: page2.length === 20
}
```

---

**S10. A content management system stores articles with draft/published states. Multiple editors might be editing the same article simultaneously. How do you handle concurrent edits?**

```javascript
// Optimistic locking with version number
// Article document:
{
  _id: ObjectId("..."),
  title: "My Article",
  content: "...",
  status: "draft",
  version: 5,
  lastEditedBy: ObjectId("editor_id"),
  lastEditedAt: ISODate("...")
}

// Editor reads the article (including version)
const article = await db.articles.findOne({ _id: articleId })
// Editor sees version: 5

// Editor makes changes, then saves with version check
const saveResult = await db.articles.updateOne(
  {
    _id: articleId,
    version: article.version  // only update if version hasn't changed
  },
  {
    $set: {
      title: updatedTitle,
      content: updatedContent,
      lastEditedBy: currentEditorId,
      lastEditedAt: new Date()
    },
    $inc: { version: 1 }  // increment version
  }
)

if (saveResult.matchedCount === 0) {
  // Another editor saved first — inform user of conflict
  const currentArticle = await db.articles.findOne({ _id: articleId })
  throw new ConflictError("Article was modified by another user", {
    yourVersion: article.version,
    currentVersion: currentArticle.version,
    lastEditedBy: currentArticle.lastEditedBy
  })
}

// Conflict resolution strategies:
// 1. Last-write-wins (force save, overwrite other editor's changes)
// 2. Merge (show diff, let user merge manually)
// 3. Lock (acquire distributed lock before editing — prevents conflict entirely)
```

---

**S11. You need to copy all documents from a `staging` collection to a `production` collection with transformation (add `migratedAt` timestamp, rename a field).**

```javascript
// Option 1: Aggregation with $out (atomic replace of output collection)
db.getSiblingDB("staging").products.aggregate([
  // Transform documents
  {
    $addFields: {
      migratedAt: new Date(),
      productName: "$name"   // add renamed field
    }
  },
  {
    $unset: ["name"]         // remove old field name
  },
  // Write to production
  {
    $out: {
      db: "production",
      coll: "products"
    }
  }
])
// WARNING: $out REPLACES the target collection atomically — existing data gone!

// Option 2: $merge (update existing collection — safer for production)
db.getSiblingDB("staging").products.aggregate([
  {
    $addFields: {
      migratedAt: new Date(),
      productName: "$name"
    }
  },
  { $unset: ["name"] },
  {
    $merge: {
      into: { db: "production", coll: "products" },
      on: "_id",                          // merge key
      whenMatched: "replace",             // replace existing docs
      whenNotMatched: "insert"            // insert new docs
    }
  }
])

// Option 3: Manual batch migration (most control)
const batchSize = 500
let processed = 0

const stagingCursor = db.getSiblingDB("staging").products.find({})

while (await stagingCursor.hasNext()) {
  const batch = []
  for (let i = 0; i < batchSize && await stagingCursor.hasNext(); i++) {
    const doc = await stagingCursor.next()
    const transformed = {
      ...doc,
      productName: doc.name,
      migratedAt: new Date()
    }
    delete transformed.name

    batch.push({ replaceOne: { filter: { _id: doc._id }, replacement: transformed, upsert: true } })
  }

  await db.getSiblingDB("production").products.bulkWrite(batch, { ordered: false })
  processed += batch.length
  console.log(`Migrated ${processed} documents`)
}
```

---

**S12. Implement a document versioning system where every update to a document creates a new version, and the previous version is preserved.**

```javascript
// Current document lives in main collection
// Versions live in a versions collection

// Users collection (current state):
{
  _id: ObjectId("user_id"),
  name: "Alice",
  email: "alice@example.com",
  currentVersion: 3
}

// User versions collection (historical states):
{
  _id: ObjectId("..."),
  documentId: ObjectId("user_id"),    // reference to the original document
  version: 2,
  snapshot: {                          // complete document state at this version
    name: "Alice Smith",
    email: "alice.smith@example.com",
    currentVersion: 2
  },
  createdAt: ISODate("2024-01-10T..."),
  changedBy: ObjectId("admin_id"),
  changeDescription: "Name correction"
}

// Update with versioning
async function updateWithVersion(collection, docId, updates, changedBy, description) {
  const session = client.startSession()
  await session.withTransaction(async () => {
    // Get current document
    const current = await db[collection].findOne({ _id: docId }, { session })
    if (!current) throw new Error("Document not found")

    // Save current version to history
    await db[`${collection}_versions`].insertOne({
      documentId: docId,
      version: current.currentVersion,
      snapshot: { ...current },
      createdAt: new Date(),
      changedBy,
      changeDescription: description
    }, { session })

    // Apply update and increment version
    await db[collection].updateOne(
      { _id: docId },
      { $set: { ...updates, currentVersion: current.currentVersion + 1 } },
      { session }
    )
  })
}

// Retrieve a specific version
async function getVersion(collection, docId, version) {
  if (version === "current") {
    return db[collection].findOne({ _id: docId })
  }
  const versionDoc = await db[`${collection}_versions`].findOne({
    documentId: docId,
    version: version
  })
  return versionDoc?.snapshot
}
```

---

**S13. You have a MongoDB collection where documents represent tasks in a queue. Implement a "claim" operation where a worker atomically picks the next available task.**

```javascript
// Task document:
{
  _id: ObjectId("..."),
  type: "email_send",
  payload: { to: "user@example.com", subject: "Welcome", templateId: "welcome" },
  status: "pending",   // pending | processing | completed | failed
  priority: 5,
  availableAt: ISODate("..."),  // not available until this time (for retries)
  attempts: 0,
  claimedBy: null,
  claimedAt: null,
  createdAt: ISODate("...")
}

// Index for efficient task claiming
db.taskQueue.createIndex({ status: 1, priority: -1, availableAt: 1 })

// Atomically claim the next available task
async function claimNextTask(workerId) {
  const task = await db.taskQueue.findOneAndUpdate(
    {
      status: "pending",
      availableAt: { $lte: new Date() }
    },
    {
      $set: {
        status: "processing",
        claimedBy: workerId,
        claimedAt: new Date()
      },
      $inc: { attempts: 1 }
    },
    {
      sort: { priority: -1, createdAt: 1 },  // highest priority first, then oldest
      returnDocument: "after"
    }
  )
  return task  // null if no tasks available
}

// Complete a task
async function completeTask(taskId, result) {
  await db.taskQueue.updateOne(
    { _id: taskId },
    { $set: { status: "completed", completedAt: new Date(), result } }
  )
}

// Fail a task with retry logic
async function failTask(taskId, error, maxAttempts = 3) {
  const task = await db.taskQueue.findOneAndUpdate(
    { _id: taskId },
    {
      $set: {
        status: "pending",  // back to pending for retry
        claimedBy: null,
        lastError: error.message,
        // Exponential backoff: retry after 2^attempts minutes
        availableAt: new Date(Date.now() + Math.pow(2, task.attempts) * 60000)
      }
    },
    { returnDocument: "after" }
  )

  if (task.attempts >= maxAttempts) {
    await db.taskQueue.updateOne({ _id: taskId }, { $set: { status: "failed" } })
  }
}
```

---

**S14. You have a social platform where users can follow topics. Each topic can have millions of followers. Design and implement efficient "follow/unfollow" and "get followed topics for a user" operations.**

```javascript
// Follow relationship collection (NOT embedded in topic or user):
{
  _id: ObjectId("..."),
  userId: ObjectId("user_id"),
  topicId: ObjectId("topic_id"),
  followedAt: ISODate("...")
}

// Indexes
db.follows.createIndex({ userId: 1, topicId: 1 }, { unique: true })
db.follows.createIndex({ topicId: 1 })  // for follower counts and topic feed

// Follow a topic
async function followTopic(userId, topicId) {
  try {
    await db.follows.insertOne({ userId, topicId, followedAt: new Date() })
    await db.topics.updateOne({ _id: topicId }, { $inc: { followerCount: 1 } })
  } catch (err) {
    if (err.code === 11000) {
      throw new Error("Already following this topic")
    }
    throw err
  }
}

// Unfollow a topic
async function unfollowTopic(userId, topicId) {
  const result = await db.follows.deleteOne({ userId, topicId })
  if (result.deletedCount > 0) {
    await db.topics.updateOne({ _id: topicId }, { $inc: { followerCount: -1 } })
  }
}

// Get all topics a user follows (with topic details)
async function getFollowedTopics(userId, page = 1, limit = 20) {
  const skip = (page - 1) * limit
  return db.follows.aggregate([
    { $match: { userId } },
    { $sort: { followedAt: -1 } },
    { $skip: skip },
    { $limit: limit },
    {
      $lookup: {
        from: "topics",
        localField: "topicId",
        foreignField: "_id",
        as: "topic"
      }
    },
    { $unwind: "$topic" },
    { $replaceRoot: { newRoot: { $mergeObjects: ["$topic", { followedAt: "$followedAt" }] } } }
  ]).toArray()
}

// Check if user follows a specific topic
async function isFollowing(userId, topicId) {
  const follow = await db.follows.findOne({ userId, topicId })
  return !!follow
}
```

---

**S15. A document has a nested array of comments, each with sub-arrays of replies. You need to add a reply to a specific comment. Write the MongoDB operation.**

```javascript
// Document structure:
{
  _id: ObjectId("post_id"),
  title: "My Post",
  comments: [
    {
      _id: ObjectId("comment1_id"),
      text: "Great post!",
      author: "Bob",
      replies: []
    },
    {
      _id: ObjectId("comment2_id"),
      text: "I agree!",
      author: "Carol",
      replies: [
        { _id: ObjectId("reply1_id"), text: "Thanks Carol!", author: "Alice" }
      ]
    }
  ]
}

// Add a reply to comment2
const commentId = ObjectId("comment2_id")
const newReply = {
  _id: new ObjectId(),
  text: "Welcome!",
  author: "Dave",
  createdAt: new Date()
}

await db.posts.updateOne(
  {
    _id: postId,
    "comments._id": commentId    // find the comment in the array
  },
  {
    $push: {
      "comments.$.replies": newReply   // $ refers to matched comment
    },
    $set: { updatedAt: new Date() }
  }
)

// NOTE: this only works for ONE level of nesting!
// The $ positional operator only matches the FIRST level of nesting.
// For deeply nested arrays, consider flattening the schema instead.

// Better schema for deep nesting: separate comments collection
// posts: { _id, title, commentCount }
// comments: { _id, postId, parentId, text, author, createdAt }
// (parentId = null for top-level comments, = commentId for replies)

// Find all replies to a comment:
db.comments.find({
  postId: postId,
  parentId: commentId
}).sort({ createdAt: 1 })
```
