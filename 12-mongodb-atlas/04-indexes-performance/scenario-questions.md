# Chapter 04 — Indexes & Performance: Scenario Questions

---

**S1. A query `db.orders.find({ customerId: id, status: "pending" }).sort({ orderDate: -1 })` is taking 5 seconds on a collection with 50 million documents. How do you diagnose and fix it?**

```javascript
// Step 1: Diagnose
db.orders.find(
  { customerId: id, status: "pending" },
  { customerId: 1, status: 1, orderDate: 1 }
).sort({ orderDate: -1 }).explain("executionStats")

// Likely output showing the problem:
{
  winningPlan: { stage: "SORT", inputStage: { stage: "COLLSCAN" } },
  executionStats: {
    executionTimeMillis: 5234,
    totalDocsExamined: 50000000,  // ALL 50M docs examined!
    nReturned: 23,                 // returned only 23
    sortSpills: 1                  // in-memory sort exceeded 100MB, spilled to disk
  }
}

// Step 2: Identify query type
// E (Equality): customerId (exact match)
// E (Equality): status (exact match)
// S (Sort): orderDate (sort field)
// → Apply ESR rule: customerId + status + orderDate

// Step 3: Create optimal compound index
db.orders.createIndex({ customerId: 1, status: 1, orderDate: -1 })
// ESR:
// E: customerId (equality)
// E: status (equality — multiple equalities before sort is fine)
// S: orderDate (sort, matching the -1 sort direction)
// No range fields in this query

// Step 4: Verify improvement
db.orders.find(
  { customerId: id, status: "pending" }
).sort({ orderDate: -1 }).explain("executionStats")

// Expected output after indexing:
{
  winningPlan: { stage: "FETCH", inputStage: { stage: "IXSCAN" } },
  executionStats: {
    executionTimeMillis: 2,       // 5234ms → 2ms!
    totalDocsExamined: 23,        // only 23 docs examined
    totalKeysExamined: 23,        // only 23 index keys
    nReturned: 23,
    sortSpills: 0                  // no in-memory sort (index handles it)
  }
}

// Step 5: Consider covered query if only these fields needed
db.orders.createIndex({ customerId: 1, status: 1, orderDate: -1, total: 1 })
db.orders.find(
  { customerId: id, status: "pending" },
  { customerId: 1, status: 1, orderDate: 1, total: 1, _id: 0 }
).sort({ orderDate: -1 })
// totalDocsExamined: 0 — covered query!
```

---

**S2. You have a products collection with 10 million documents. Some products have a `discountedUntil` date field and need to automatically remove the discount after that date. Implement this with TTL.**

```javascript
// Product document structure:
{
  _id: ObjectId("..."),
  name: "Widget Pro",
  price: NumberDecimal("29.99"),
  originalPrice: NumberDecimal("39.99"),
  isOnSale: true,
  discountedUntil: ISODate("2024-03-15T00:00:00Z")  // TTL field
}

// Approach 1: TTL index + separate sale items collection
// Create a separate "activeSales" collection that auto-expires:
db.createCollection("activeSales")
db.activeSales.createIndex(
  { discountedUntil: 1 },
  { expireAfterSeconds: 0 }   // expires AT the discountedUntil datetime
)

// When adding a sale:
await db.activeSales.insertOne({
  productId: ObjectId("..."),
  discountPercent: 25,
  discountedUntil: new Date("2024-03-15T00:00:00Z")
})
// This document auto-deletes after discountedUntil passes

// Query active sales:
const activeSales = await db.activeSales.find({}).toArray()
const saleProductIds = activeSales.map(s => s.productId)
const saleProducts = await db.products.find({ _id: { $in: saleProductIds } }).toArray()

// Approach 2: Change Streams on activeSales to update product when sale expires
const changeStream = db.activeSales.watch([
  { $match: { operationType: "delete" } }
])
changeStream.on("change", async (change) => {
  // Sale expired — update the product
  const deletedSale = change.documentKey
  // Note: deleted documents don't have fullDocument available
  // Store productId on a separate expiry log OR use a trigger
})

// Approach 3: Atlas Trigger (scheduled) to clean up expired sales
// Use Atlas Scheduled Trigger every hour:
// exports = function() {
//   const activeSales = context.services.get("atlas").db("store").collection("activeSales")
//   activeSales.deleteMany({ discountedUntil: { $lte: new Date() } })
// }

// TTL index limitations to note:
// 1. TTL only works on Date fields
// 2. Background job runs every ~60 seconds (not exact expiry time)
// 3. Cannot use TTL on a capped collection
```

---

**S3. A developer created 15 indexes on the orders collection trying to optimize various queries. Performance has actually gotten worse for writes. Diagnose and fix.**

```javascript
// Step 1: Check all indexes and their sizes
db.orders.getIndexes()
// Output: 15 indexes, including many similar compound indexes

db.orders.stats().indexSizes
// {
//   "_id_": 45000000,
//   "status_1": 120000000,
//   "status_1_customerId_1": 200000000,
//   "customerId_1": 180000000,
//   "customerId_1_status_1": 200000000,   // ← duplicate of above (reversed order)
//   "status_1_createdAt_-1": 150000000,
//   "createdAt_-1": 80000000,
//   ...
//   Total index size: ~2GB!
// }

// Step 2: Check index usage statistics
db.orders.aggregate([{ $indexStats: {} }])
// {
//   name: "status_1",            accesses: { ops: 2 }      ← BARELY USED
//   name: "status_1_customerId_1",  accesses: { ops: 1500 }
//   name: "customerId_1_status_1",  accesses: { ops: 18000 }   ← heavily used
//   name: "createdAt_-1",           accesses: { ops: 3 }      ← barely used
//   name: "status_1_createdAt_-1",  accesses: { ops: 8500 }
// }

// Step 3: Identify redundant indexes
// "status_1" is a PREFIX of "status_1_customerId_1"
// Any query served by status_1 alone can also be served by status_1_customerId_1
// "status_1" is redundant!

// "customerId_1" is a PREFIX of "customerId_1_status_1"
// Similarly redundant if all customerId queries also filter by status

// Step 4: Drop unused and redundant indexes
db.orders.dropIndex("status_1")          // redundant — covered by status_1_customerId_1
db.orders.dropIndex("createdAt_-1")      // only 3 uses — probably not worth it
db.orders.dropIndex("customerId_1")      // redundant — covered by customerId_1_status_1

// Step 5: Verify write performance improved
// Measure insert latency before and after:
// Before: 15 indexes × ~5ms each = 75ms per insert
// After: 8 indexes × ~5ms each = 40ms per insert

// Result: fewer indexes = faster writes + less memory usage + better cache efficiency
```

---

**S4. Design the optimal index strategy for a multi-tenant SaaS application where every query always filters by `tenantId` first.**

```javascript
// In a multi-tenant app, EVERY query has tenantId as a filter
// This means tenantId should be the LEFTMOST field in ALL compound indexes

// Common queries:
// 1. Get user by email within tenant
db.users.find({ tenantId: "tenant_A", email: "alice@example.com" })
db.users.createIndex({ tenantId: 1, email: 1 })

// 2. Get user's orders, sorted by date
db.orders.find({ tenantId: "tenant_A", userId: id }).sort({ createdAt: -1 })
db.orders.createIndex({ tenantId: 1, userId: 1, createdAt: -1 })

// 3. Get recent orders by status
db.orders.find({ tenantId: "tenant_A", status: "pending" }).sort({ createdAt: -1 })
db.orders.createIndex({ tenantId: 1, status: 1, createdAt: -1 })

// 4. Get products by category with price filter
db.products.find({ tenantId: "tenant_A", category: "electronics", price: { $lte: 200 } })
db.products.createIndex({ tenantId: 1, category: 1, price: 1 })

// Why tenantId MUST be first:
// Without it, different tenant's data intermingles in the B-tree
// query { email: "alice@example.com" } would scan ALL tenants' alice accounts
// → tenantId first ensures index scan is bounded to one tenant's data

// Alternative: use separate collections per tenant (tenant isolation pattern)
// db["tenant_A_orders"], db["tenant_B_orders"]
// Pros: complete isolation, separate index management
// Cons: tens of thousands of collections at scale is problematic
// Generally: shared collection + tenantId prefix is preferred

// Shard key for multi-tenant sharding:
sh.shardCollection("db.orders", { tenantId: "hashed" })
// Distributes tenants across shards
// Queries for ONE tenant may span shards (scatter-gather)
// Alternative: { tenantId: 1, _id: 1 } ranges — co-locate each tenant on one shard
sh.shardCollection("db.orders", { tenantId: 1, _id: 1 })
// With zone sharding: each tenant's data on a specific shard
```

---

**S5. Build an efficient pagination system for a product listing with sorting by price, name, or relevance score.**

```javascript
// Index for each sort option
db.products.createIndex({ category: 1, price: 1, _id: 1 })    // sort by price
db.products.createIndex({ category: 1, name: 1, _id: 1 })     // sort by name
db.products.createIndex({ category: 1, _id: -1 })             // sort by newest
// Note: always include _id as last field to ensure unique sort (for stable pagination)

// Keyset pagination (cursor-based) — avoids skip() performance issue
async function getProductPage({ category, sortBy, cursor, pageSize = 20 }) {
  let filter = { category, inStock: true }
  let sort = {}

  // Build sort and filter from cursor
  if (sortBy === "price") {
    sort = { price: 1, _id: 1 }
    if (cursor) {
      // Continue from last seen price+_id
      filter.$or = [
        { price: { $gt: cursor.price } },
        { price: cursor.price, _id: { $gt: cursor._id } }
      ]
    }
  } else if (sortBy === "name") {
    sort = { name: 1, _id: 1 }
    if (cursor) {
      filter.$or = [
        { name: { $gt: cursor.name } },
        { name: cursor.name, _id: { $gt: cursor._id } }
      ]
    }
  } else {  // newest first
    sort = { _id: -1 }
    if (cursor) {
      filter._id = { $lt: cursor._id }
    }
  }

  const products = await db.products.find(filter)
    .sort(sort)
    .limit(pageSize + 1)  // get one extra to detect if there's a next page
    .project({ name: 1, price: 1, category: 1, thumbnail: 1 })
    .toArray()

  const hasMore = products.length > pageSize
  const results = hasMore ? products.slice(0, pageSize) : products
  const lastItem = results[results.length - 1]

  return {
    products: results,
    hasMore,
    nextCursor: hasMore ? {
      _id: lastItem._id,
      price: lastItem.price,
      name: lastItem.name
    } : null
  }
}
```

---

**S6. A MongoDB collection stores social media posts. Users can search by hashtag, and the most common operation is `find({ hashtags: "mongodb" })`. Posts are created 10K/day and the collection grows to 100M docs. What's the optimal index strategy?**

```javascript
// Post document:
{
  _id: ObjectId("..."),
  userId: ObjectId("..."),
  content: "Loving #mongodb and #atlas!",
  hashtags: ["mongodb", "atlas"],     // array field → will be multikey index
  createdAt: ISODate("..."),
  likeCount: 0,
  isPublic: true
}

// Query patterns:
// 1. Find all posts with hashtag #mongodb (most common)
// 2. Find all posts with hashtag #mongodb in last 7 days
// 3. Find popular posts with hashtag #mongodb (sorted by likeCount)

// Multikey index on hashtags:
db.posts.createIndex({ hashtags: 1 })
// One index entry per hashtag per post
// { "mongodb" → post1 }, { "atlas" → post1 }, { "mongodb" → post2 }, ...

// For query 1: simple multikey lookup
db.posts.find({ hashtags: "mongodb", isPublic: true })
// With index: fast O(log n) B-tree lookup

// Compound multikey for query 2:
db.posts.createIndex({ hashtags: 1, createdAt: -1 })
// { tag: "mongodb", createdAt: ... } → post_id

// Query 2:
const lastWeek = new Date(Date.now() - 7 * 86400000)
db.posts.find({
  hashtags: "mongodb",
  createdAt: { $gte: lastWeek }
}).sort({ createdAt: -1 })

// Query 3 (popular posts):
db.posts.createIndex({ hashtags: 1, likeCount: -1 })
db.posts.find({ hashtags: "mongodb" }).sort({ likeCount: -1 }).limit(20)

// For very high-traffic hashtags: pre-computed hashtag counters
// Materialized view of top hashtags:
db.hashtagStats.createIndex({ count: -1 })
// Update on each post insert:
db.hashtagStats.updateMany(
  { hashtag: { $in: post.hashtags } },
  { $inc: { count: 1 }, $set: { lastUsed: new Date() } },
  { upsert: true }
)

// TTL to auto-expire old posts from index scope:
db.posts.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 31536000 }  // 1 year retention
)
```

---

**S7. Implement a system to detect and report "slow queries" in a production MongoDB deployment without Atlas.**

```javascript
// Enable the database profiler
// Level 0: disabled
// Level 1: logs operations slower than slowms threshold
// Level 2: logs ALL operations (never use in production!)

db.setProfilingLevel(1, { slowms: 100 })  // log queries > 100ms
// Or with more control:
db.setProfilingLevel(1, {
  slowms: 100,
  sampleRate: 0.1    // only sample 10% of operations meeting threshold
  // (reduces profiler overhead in high-throughput systems)
})

// Query the profiler:
db.system.profile.find({
  millis: { $gt: 100 },
  ns: { $ne: "admin.system.profile" },   // exclude profiler reads
  op: { $in: ["query", "update", "insert", "remove", "command"] }
}).sort({ millis: -1 }).limit(20)

// Analyze slow queries:
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  {
    $group: {
      _id: {
        ns: "$ns",
        op: "$op",
        // Group by query shape (not values)
        queryShape: { $objectToArray: "$query" }
      },
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" },
      maxMs: { $max: "$millis" },
      examples: { $push: { query: "$query", millis: "$millis", ts: "$ts" } }
    }
  },
  { $sort: { avgMs: -1 } },
  {
    $project: {
      namespace: "$_id.ns",
      operation: "$_id.op",
      count: 1,
      avgMs: { $round: ["$avgMs", 1] },
      maxMs: 1,
      sampleQuery: { $arrayElemAt: ["$examples", 0] }
    }
  }
])

// IMPORTANT: system.profile is a capped collection
// Don't let it grow without bounds in high-traffic systems
// After analysis, turn profiling off:
db.setProfilingLevel(0)
```

---

**S8. You have a large analytics collection that's read-only after initial load. How would you optimize it differently from a transactional collection?**

```javascript
// Read-only analytics collection (loaded in bulk, then read-only)
// Optimization strategies:

// 1. More aggressive indexing than transactional collections
// Since writes are rare, index write overhead is not a concern
db.analyticsEvents.createIndex({ userId: 1, eventType: 1, ts: -1 })
db.analyticsEvents.createIndex({ eventType: 1, ts: -1 })
db.analyticsEvents.createIndex({ ts: -1 })
db.analyticsEvents.createIndex({ sessionId: 1, ts: 1 })
db.analyticsEvents.createIndex({ country: 1, eventType: 1, ts: -1 })

// 2. Use read concern "available" for maximum speed
// (may return stale data, but for analytics this is acceptable)
db.analyticsEvents.find({ ... }).readConcern("available")

// 3. Read from secondaries to offload primary
db.analyticsEvents.find({ ... }).readPref("secondaryPreferred")

// 4. Build aggregated summary collections for common reports
db.analyticsEvents.aggregate([
  { $match: { ts: { $gte: startOfDay } } },
  { $group: { _id: { hour: { $hour: "$ts" }, eventType: "$eventType" }, count: { $sum: 1 } } },
  { $out: "hourly_event_summary" }
])
// Queries against hourly_event_summary are much faster than raw events

// 5. Atlas Online Archive for cold data
// Hot data (last 90 days): in Atlas cluster (fast, expensive)
// Cold data (> 90 days): Atlas Online Archive on S3 (slow, cheap)
// Query both via Atlas Data Federation

// 6. For Parquet/columnar analytics: export to S3 + use Athena or Spark
// MongoDB is excellent for document operations; pure analytics may benefit from
// columnar storage format (Parquet) with separate analytics engine

// 7. Disable document padding (WiredTiger default)
// Read-only collections don't need space for document growth
db.runCommand({ collMod: "analyticsEvents", noPadding: true })
```

---

**S9. How would you design an indexing strategy for a full-text search on a product catalog with 5 million products?**

```javascript
// Product document:
{
  _id: ObjectId("..."),
  name: "Apple MacBook Pro 14-inch M3 Chip",
  description: "The most powerful MacBook Pro ever...",
  brand: "Apple",
  category: "laptops",
  tags: ["laptop", "apple", "macbook", "m3"],
  price: NumberDecimal("1999.00"),
  specs: { ram: "18GB", storage: "512GB", cpu: "M3" },
  inStock: true
}

// Option 1: Native text index (simple, built-in)
db.products.createIndex({
  name: "text",
  description: "text",
  tags: "text"
}, {
  weights: {
    name: 10,        // name matches are most important
    tags: 5,         // tags are moderately important
    description: 1   // description least important
  },
  default_language: "english"
})

// Text search query:
db.products.find(
  { $text: { $search: "macbook pro", $language: "english" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } }).limit(20)

// Combine with other filters:
db.products.find({
  $text: { $search: "macbook" },
  category: "laptops",
  price: { $lte: NumberDecimal("2500.00") },
  inStock: true
})

// Limitations of text index:
// - Only one text index per collection
// - No fuzzy matching
// - No synonym support
// - No autocomplete
// - Language-specific stemming only

// Option 2: Atlas Search (production-grade)
// Create Atlas Search index (JSON definition):
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": [
        { "type": "string", "analyzer": "lucene.english" },
        { "type": "autocomplete", "analyzer": "lucene.standard" }
      ],
      "description": { "type": "string", "analyzer": "lucene.english" },
      "brand": { "type": "stringFacet" },
      "category": { "type": "stringFacet" },
      "price": { "type": "number" },
      "tags": { "type": "string" }
    }
  }
}

// Atlas Search query:
db.products.aggregate([
  {
    $search: {
      index: "products_idx",
      compound: {
        must: [{ text: { query: "macbook pro", path: ["name", "description"], fuzzy: { maxEdits: 1 } } }],
        filter: [{ range: { path: "price", lte: 2500 } }]
      }
    }
  },
  { $project: { name: 1, price: 1, score: { $meta: "searchScore" } } },
  { $sort: { score: -1 } },
  { $limit: 20 }
])
```

---

**S10. A compound index `{ a: 1, b: 1, c: 1 }` exists. Which of the following queries can use this index? Explain why.**

```javascript
// Index: { a: 1, b: 1, c: 1 }
// Compound index can be used by queries that use a PREFIX of the index key pattern

// Query 1: { a: 5 }
db.collection.find({ a: 5 })
// ✓ CAN USE INDEX — a is the leftmost prefix
// Uses: a index entries only

// Query 2: { a: 5, b: "hello" }
db.collection.find({ a: 5, b: "hello" })
// ✓ CAN USE INDEX — uses a and b (consecutive prefix)

// Query 3: { a: 5, b: "hello", c: true }
db.collection.find({ a: 5, b: "hello", c: true })
// ✓ CAN USE INDEX — uses the full index

// Query 4: { b: "hello" }
db.collection.find({ b: "hello" })
// ✗ CANNOT USE INDEX — b is not the leftmost prefix
// Must create a separate { b: 1 } index

// Query 5: { b: "hello", c: true }
db.collection.find({ b: "hello", c: true })
// ✗ CANNOT USE INDEX — b is not leftmost

// Query 6: { a: 5, c: true }
db.collection.find({ a: 5, c: true })
// ✓ PARTIAL USE — uses the index for the a field filter
// Then scans all "a: 5" entries checking c
// Better than COLLSCAN, but not as good as if { a: 1, c: 1 } existed
// Called an "index scan with bound overlap"

// Query 7: sort only — db.collection.find({}).sort({ a: 1 })
// ✓ CAN USE INDEX FOR SORT — a is the leftmost field
// index provides sorted results for a

// Query 8: sort on non-leftmost — db.collection.find({}).sort({ b: 1 })
// ✗ CANNOT USE INDEX FOR SORT — b is not leftmost
// Must do in-memory sort

// Summary: the "leftmost prefix rule" governs compound index usage
// a         → uses index
// a + b     → uses index
// a + b + c → uses index
// b         → no
// b + c     → no
// a + c     → partial (uses a only, then filter on c)
```

---

**S11. You have an e-commerce platform where 90% of queries are for "in-stock" products. Should you create a partial index? When does it help vs hurt?**

```javascript
// Scenario: products collection, 10M docs, 9M are inStock: true

// Option 1: Regular index
db.products.createIndex({ category: 1, price: 1 })
// Index size: 10M entries (includes all products)

// Option 2: Partial index (only inStock products)
db.products.createIndex(
  { category: 1, price: 1 },
  { partialFilterExpression: { inStock: true } }
)
// Index size: 9M entries (90% smaller in this extreme case)

// Query that BENEFITS from partial index:
db.products.find({ category: "electronics", price: { $lt: 200 }, inStock: true })
// MongoDB can use partial index: filter includes the partial expression
// Smaller index → better cache utilization → faster queries

// Query that CANNOT use partial index:
db.products.find({ category: "electronics", price: { $lt: 200 } })
// Missing inStock: true in filter → partial index excluded from consideration
// Falls back to COLLSCAN or other indexes

// Query that finds out-of-stock items:
db.products.find({ category: "electronics", inStock: false })
// Partial index doesn't include inStock: false docs → cannot use
// Need a separate regular index for out-of-stock queries (rare)

// Decision: when to use partial index?
// ✓ Use when:
// - Majority of queries include the partition condition (90%+ in our case)
// - Partition condition eliminates significant portion of collection (>30%)
// - Memory is constrained and full index doesn't fit in RAM
// ✓ Avoid when:
// - Many queries don't include the partition condition
// - Partition eliminates small % of collection (10% out of stock)
// - You need the full index for sorts on the excluded documents too
```

---

**S12. Explain how to fix a "SORT exceeded memory limit" error in production.**

```javascript
// Error: "Sort exceeded memory limit of 104857600 bytes.
//         Add allowDiskUse:true to opt in to writing temporary files to disk."

// Approach 1 (Quick fix): allowDiskUse
db.orders.find({}).sort({ createdAt: -1 }).allowDiskUse(true)
// Spills to disk when sort exceeds 100MB in memory
// Slower but doesn't error out

db.orders.aggregate([
  { $sort: { createdAt: -1 } }
], { allowDiskUse: true })

// Approach 2 (Better): Create index to support the sort
// Instead of in-memory sort, use index order
db.orders.createIndex({ status: 1, createdAt: -1 })

// Query with this index (sort is free — data comes back in index order):
db.orders.find({ status: "pending" }).sort({ createdAt: -1 })
// No SORT stage needed in plan → no memory issue

// Approach 3: Reduce result set BEFORE sorting
// Avoid sorting 10M documents → sort only the subset you need
db.orders.aggregate([
  { $match: { status: "pending", createdAt: { $gte: lastWeek } } },  // filter first
  { $sort: { createdAt: -1 } },    // sort small subset
  { $limit: 100 }                   // take top 100
])
// With index on { status: 1, createdAt: -1 }: no in-memory sort at all

// Approach 4: Project smaller documents before sort
db.orders.aggregate([
  { $project: { customerId: 1, total: 1, createdAt: 1 } },  // tiny docs
  { $sort: { createdAt: -1 } }
  // Smaller documents = less memory for sort buffer
])

// Atlas note: M0/M2/M5 shared tiers have lower memory limits
// allowDiskUse is not available on M0 shared tier
// Upgrade to dedicated M10+ or optimize the query
```

---

**S13. Design indexes for a message inbox where users query "all messages in a thread" and "all unread messages".**

```javascript
// Message document:
{
  _id: ObjectId("..."),
  threadId: ObjectId("..."),
  senderId: ObjectId("..."),
  recipientId: ObjectId("..."),
  subject: "Re: Project Update",
  body: "...",
  sentAt: ISODate("..."),
  readAt: null,               // null = unread, Date = when read
  isDeleted: false,
  folder: "inbox"
}

// Query 1: All messages in a thread (ordered by time)
db.messages.find({ threadId: threadId }).sort({ sentAt: 1 })
db.messages.createIndex({ threadId: 1, sentAt: 1 })

// Query 2: All unread messages for a user
db.messages.find({
  recipientId: userId,
  readAt: null,
  isDeleted: false
}).sort({ sentAt: -1 })
// ESR: recipientId (E), sentAt (S), readAt and isDeleted (range/partial)
// Best index: partial index excluding deleted + read messages
db.messages.createIndex(
  { recipientId: 1, sentAt: -1 },
  {
    partialFilterExpression: {
      readAt: null,
      isDeleted: false
    }
  }
)
// Queries MUST include readAt: null and isDeleted: false to use this index

// Query 3: Inbox page (all non-deleted messages for a user, paginated)
db.messages.find({
  recipientId: userId,
  folder: "inbox",
  isDeleted: false
}).sort({ sentAt: -1 })
// ESR: recipientId (E), folder (E), sentAt (S), isDeleted (filter/partial)
db.messages.createIndex(
  { recipientId: 1, folder: 1, sentAt: -1 },
  { partialFilterExpression: { isDeleted: false } }
)

// Query 4: Unread count per folder
db.messages.aggregate([
  { $match: { recipientId: userId, isDeleted: false, readAt: null } },
  { $group: { _id: "$folder", unreadCount: { $sum: 1 } } }
])
// The partial index on { recipientId, sentAt } covers this

// Covered query for unread count (fast):
db.messages.createIndex(
  { recipientId: 1, folder: 1, readAt: 1 },
  { partialFilterExpression: { isDeleted: false } }
)
// { recipientId: 1, folder: 1, readAt: 1 } — with projection of same fields
// totalDocsExamined: 0 for unread count query
```

---

**S14. Walk through using explain() to verify that a covered query is working correctly.**

```javascript
// Goal: build a covered query for product price list (name + price only)
// Collection: 10M products

// Step 1: Start with basic query — check if covered
db.products.find(
  { category: "electronics" },
  { name: 1, price: 1, _id: 0 }   // project only name and price
).explain("executionStats")

// Output (without right index):
{
  winningPlan: {
    stage: "PROJECTION_DEFAULT",
    inputStage: {
      stage: "COLLSCAN"   // full collection scan
    }
  },
  executionStats: {
    totalDocsExamined: 10000000,
    totalKeysExamined: 0,
    nReturned: 500000
  }
}
// Not covered (COLLSCAN), reads all 10M docs

// Step 2: Create index including all projected fields + filter field
db.products.createIndex({ category: 1, name: 1, price: 1 })

// Step 3: Re-run explain
db.products.find(
  { category: "electronics" },
  { name: 1, price: 1, _id: 0 }
).explain("executionStats")

// Output:
{
  winningPlan: {
    stage: "PROJECTION_COVERED",    // ← "COVERED" in the stage name
    inputStage: {
      stage: "IXSCAN",
      indexName: "category_1_name_1_price_1"
    }
  },
  executionStats: {
    totalDocsExamined: 0,    // ← ZERO document reads!
    totalKeysExamined: 500000,  // reads only index keys
    nReturned: 500000
  }
}
// COVERED QUERY: PROJECTION_COVERED + totalDocsExamined: 0

// Step 4: Verify what breaks the covered query
db.products.find(
  { category: "electronics" },
  { name: 1, price: 1 }   // forgot _id: 0!
).explain("executionStats")
// Output: stage: "PROJECTION_DEFAULT", totalDocsExamined: 500000
// Adding _id to projection forces document fetch (FETCH stage added)
// Always include _id: 0 for covered queries!
```

---

**S15. A time-series collection has a compound index on `{ sensorId: 1, timestamp: -1 }`. A query filters by sensorId AND timestamp range. Walk through the index scan mechanics.**

```javascript
// Index: { sensorId: 1, timestamp: -1 }
// Query:
db.sensorData.find({
  sensorId: "sensor_A",
  timestamp: {
    $gte: ISODate("2024-01-15T00:00:00Z"),
    $lte: ISODate("2024-01-15T23:59:59Z")
  }
}).sort({ timestamp: -1 })

// How the B-tree index scan works:
// 1. Index entries sorted as: (sensorId ASC, timestamp DESC)
//
// B-tree leaf structure:
// ...
// ("sensor_A", 2024-01-15T23:59:59) → docId_1
// ("sensor_A", 2024-01-15T23:55:00) → docId_2
// ("sensor_A", 2024-01-15T12:00:00) → docId_3
// ("sensor_A", 2024-01-15T00:01:00) → docId_4
// ("sensor_A", 2024-01-14T23:59:59) → docId_5  ← out of range, stop here
// ("sensor_B", ...)
// ...
//
// 2. MongoDB walks the B-tree to find the start of ("sensor_A", 2024-01-15T23:59:59)
// 3. Scans forward (in B-tree order = descending timestamp) until timestamp < 2024-01-15T00:00
// 4. Results are already in descending timestamp order (matches .sort({ timestamp: -1 }))
//    → No in-memory sort needed!

// explain output shows this as:
{
  stage: "IXSCAN",
  direction: "forward",
  indexBounds: {
    sensorId: ["[\"sensor_A\", \"sensor_A\"]"],
    timestamp: ["[new Date(1705276799999), new Date(1705276800000)]"]
  }
}
// indexBounds shows exactly which portion of the index was scanned
// Narrow bounds = efficient scan (only the relevant sensor and day)
```
