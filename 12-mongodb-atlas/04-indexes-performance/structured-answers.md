# Chapter 04 — Indexes & Performance: Structured Answers

---

## Answer 1: "Explain the ESR rule and demonstrate it with a concrete query"

**Point**: The ESR (Equality → Sort → Range) rule determines the optimal field order in a compound index to minimize in-memory sorts and maximize index scan efficiency.

**Reason**: The B-tree structure of an index means that once you use a range scan on a field, MongoDB cannot maintain sorted order for subsequent fields in the index. Placing sort fields BEFORE range fields allows MongoDB to use the index ordering for the sort.

```javascript
// Query: find all completed orders for a specific customer, sorted by date, within the last 30 days
db.orders.find({
  customerId: userId,             // E: equality
  status: "completed",           // E: equality
  createdAt: { $gte: thirtyDaysAgo }  // R: range
}).sort({ createdAt: -1 })      // S: sort

// ESR Analysis:
// E fields: customerId, status (exact matches)
// S fields: createdAt (sort field)
// R fields: createdAt (range — same field as sort)

// When sort field == range field, ESR collapses to ES only:
// createdAt serves as BOTH Sort (S) and Range (R)
// Optimal index: E fields first, then the ES/R field
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 })
//   E:customerId    E:status    S+R:createdAt (-1 matches sort direction)

// Explanation:
// 1. Index scan finds all entries where customerId=X AND status='completed'
// 2. Within that scan, entries are ordered by createdAt DESC (the index direction)
// 3. MongoDB stops scanning when createdAt < thirtyDaysAgo
// 4. Results come out in sorted order — NO in-memory sort stage!

// Verify with explain:
db.orders.find(
  { customerId: userId, status: "completed", createdAt: { $gte: thirtyDaysAgo } }
).sort({ createdAt: -1 }).explain("executionStats")
// winningPlan should have: IXSCAN with NO SORT stage
// sortSpills: 0 (no in-memory sort needed)
```

---

## Answer 2: "What is a covered query and how do you design one?"

**Point**: A covered query is one where the index contains all data needed for the query — filter, sort, AND projection — so MongoDB never reads any documents from storage.

```javascript
// Step 1: Identify the query's required fields
// Query: product price list for a category
// Filter: category = "electronics"
// Sort: price ascending
// Return: name, price (not the full product document)

// Step 2: Create index including ALL referenced fields
db.products.createIndex({ category: 1, price: 1, name: 1 })
// category: in filter (E)
// price: in filter range AND sort (S+R)
// name: in projection

// Step 3: Write query with explicit _id: 0 exclusion
const priceList = await db.products.find(
  { category: "electronics" },
  { name: 1, price: 1, _id: 0 }   // _id: 0 is NON-NEGOTIABLE for covered queries
).sort({ price: 1 }).toArray()

// Step 4: Verify it's covered
db.products.find(
  { category: "electronics" },
  { name: 1, price: 1, _id: 0 }
).sort({ price: 1 }).explain("executionStats")

// Confirm covered query indicators:
{
  winningPlan: {
    stage: "PROJECTION_COVERED",  // ← "COVERED" in stage name
    inputStage: { stage: "IXSCAN" }
    // No "FETCH" stage! Documents never read from storage.
  },
  executionStats: {
    totalDocsExamined: 0,         // ← ZERO = covered query confirmed!
    totalKeysExamined: 45000,     // reads only index
    nReturned: 45000
  }
}

// Performance gain:
// IXSCAN + FETCH (normal): reads index + reads each document (random I/O)
// IXSCAN (covered): reads only index (sequential B-tree scan)
// Typical improvement: 5-50x faster on large collections
```

---

## Answer 3: "Explain all MongoDB index types and their use cases"

```javascript
// 1. SINGLE-FIELD INDEX — most common, B-tree
db.users.createIndex({ email: 1 })
// Use: equality, range, sort on a single field

// 2. COMPOUND INDEX — multiple fields, B-tree
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 })
// Use: queries filtering/sorting by multiple fields

// 3. MULTIKEY INDEX — array fields
db.posts.createIndex({ tags: 1 })  // automatically multikey when tags is an array
// Use: queries that need to find docs by array element value

// 4. TEXT INDEX — full-text search
db.articles.createIndex({ title: "text", content: "text" })
// Use: $text queries, word-based search with stemming

// 5. GEOSPATIAL — 2dsphere for spherical data
db.places.createIndex({ location: "2dsphere" })
// Use: $near, $geoWithin, $geoNear queries

// 6. HASHED — hash of field value
db.users.createIndex({ userId: "hashed" })
// Use: shard key for uniform distribution, equality queries only (no range)

// 7. WILDCARD — all fields or sub-path
db.products.createIndex({ "attributes.$**": 1 })
// Use: querying dynamic/polymorphic schemas where field names vary

// 8. PARTIAL — only documents matching a filter expression
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { isActive: { $eq: true } } }
)
// Use: skip indexing low-value documents (deleted, inactive, etc.)

// 9. SPARSE — only documents where field exists
db.users.createIndex({ phone: 1 }, { sparse: true })
// Use: optional fields that only some documents have

// 10. TTL — time-to-live for automatic document expiry
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })
// Use: session management, event logs, temporary data

// 11. UNIQUE — enforce uniqueness constraint
db.users.createIndex({ email: 1 }, { unique: true })
// Use: natural keys, preventing duplicates

// 12. CLUSTERED (MongoDB 5.3+) — physical document ordering
db.createCollection("events", { clusteredIndex: { key: { _id: 1 }, unique: true } })
// Use: time-series-like sequential access patterns, TTL optimization
```

---

## Answer 4: "How do you diagnose and resolve a slow MongoDB query step by step?"

```javascript
// STEP 1: Identify slow queries
// Option A: Atlas Performance Advisor (if using Atlas)
// Option B: Enable profiler
db.setProfilingLevel(1, { slowms: 100 })

// Option C: Real-time monitoring
db.currentOp({ active: true, secs_running: { $gt: 5 } })

// STEP 2: Analyze with explain
const explained = db.orders.find({
  customerId: ObjectId("..."),
  status: { $in: ["pending", "processing"] }
}).sort({ createdAt: -1 }).explain("executionStats")

// STEP 3: Interpret the output
// BAD signs:
// - stage: "COLLSCAN" → no index used
// - stage: "SORT" after filter → in-memory sort
// - totalDocsExamined / nReturned ratio > 10 → low selectivity index
// - sortSpills: 1 → sort exceeded 100MB memory

// GOOD signs:
// - stage: "IXSCAN" → index used
// - No "SORT" stage (data sorted by index order) → sort from index
// - totalDocsExamined ≈ nReturned → high selectivity
// - stage: "PROJECTION_COVERED" + totalDocsExamined: 0 → covered query

// STEP 4: Design the fix
// For our query: E: customerId, E: status ($in = range-like), S: createdAt
// Status $in is range-like but comes before sort → test both orders:
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 })
// vs:
db.orders.createIndex({ customerId: 1, createdAt: -1, status: 1 })
// Test both with explain() and compare executionTimeMillis

// STEP 5: Verify improvement
const afterExplain = db.orders.find({
  customerId: ObjectId("..."),
  status: { $in: ["pending", "processing"] }
}).sort({ createdAt: -1 }).explain("executionStats")

console.log("Before:", explained.executionStats.executionTimeMillis, "ms")
console.log("After:", afterExplain.executionStats.executionTimeMillis, "ms")

// STEP 6: Monitor in production
// Atlas: Real-Time Performance Panel, Metrics → Query Targeting
// Query Targeting ratio = docsExamined / docsReturned
// Should be close to 1 for efficient queries
```

---

## Answer 5: "Explain what happens when MongoDB builds an index on a live production collection"

**Point**: MongoDB 4.4+ builds indexes online (non-blocking) using a three-phase process that minimizes impact on production reads/writes.

```javascript
// PHASE 1: Initial scan (bulk build)
// MongoDB performs a forward collection scan, building the index
// Reads: UNRESTRICTED — reads proceed normally
// Writes: ALLOWED but must also update the in-progress index
//         The write happens, AND a "side write buffer" captures the change
// Duration: proportional to collection size (minutes to hours for large collections)

// PHASE 2: Drain side-write buffer
// Applies all writes collected in side buffer to the index
// Brief write pauses (sub-second) to process buffer

// PHASE 3: Commit
// Takes a very brief intent lock to finalize
// Duration: milliseconds (not seconds/minutes)
// After this: index is immediately available for queries

// Monitor active index builds:
db.currentOp({
  "command.createIndexes": { $exists: true }
})
// Shows: % complete, ns, index specification

// Performance impact during build:
// Writes: ~10-30% slower (updating in-progress index + normal index)
// Reads: minimal impact (no lock held)

// Atlas rolling index build (zero primary downtime):
// 1. Build on secondary1 (primary unaffected)
// 2. Build on secondary2 (primary unaffected)
// 3. Step down primary (very brief — seconds)
// 4. Old primary (now secondary) builds index
// 5. Elections may occur (brief ~10s write unavailability)
// Net result: much less production impact than building directly on primary

// Force cancel a running build:
const op = db.currentOp({ "command.createIndexes": { $exists: true } })
db.killOp(op.inprog[0].opid)
// Warning: index will be in an inconsistent state — clean up needed
```

---

## Answer 6: "How would you design an indexing strategy for an e-commerce catalog?"

```javascript
// E-commerce product catalog query patterns:
// 1. Homepage: show all in-stock products by category
// 2. Category page: filter by category, sort by price/popularity
// 3. Search: text search with filters
// 4. Product page: get by _id or SKU
// 5. Admin: find by brand, date range, low stock alerts

// Index design:

// QUERY 1 & 2: Category browse + filters
// filter: category, inStock, optionally price range
// sort: price or rating or createdAt
db.products.createIndex({ category: 1, inStock: 1, price: 1 })
// E: category, E: inStock (partial filter if only showing in-stock)
// Then price for range filter + sort

// Better: partial index (only in-stock items indexed)
db.products.createIndex(
  { category: 1, price: 1 },
  { partialFilterExpression: { inStock: true } }
)
// 50% smaller index if half items are in stock
// All category+price queries MUST include inStock: true to use this

// QUERY 3: Text search with filters
db.products.createIndex(
  { name: "text", description: "text", brand: "text" },
  { weights: { name: 10, brand: 5, description: 1 } }
)
// OR use Atlas Search (much better for production text search)

// QUERY 4: Get by SKU
db.products.createIndex({ sku: 1 }, { unique: true })

// QUERY 5A: Admin - filter by brand + price
db.products.createIndex({ brand: 1, price: 1 })

// QUERY 5B: Admin - low stock alerts
db.products.createIndex(
  { stock: 1 },
  { partialFilterExpression: { stock: { $lt: 10 } } }
)
// Only indexes low-stock items (small index, fast alert queries)

// QUERY 5C: Admin - recently added products
db.products.createIndex({ createdAt: -1 })

// Covered query for product listing page:
db.products.createIndex({ category: 1, price: 1, name: 1, sku: 1, thumbnailUrl: 1 })
db.products.find(
  { category: "electronics", inStock: true },
  { name: 1, price: 1, sku: 1, thumbnailUrl: 1, _id: 0 }
).sort({ price: 1 })
// totalDocsExamined: 0 on product listing page = blazing fast
```

---

## Answer 7: "Explain index intersection and when it's used over compound indexes"

**Point**: Index intersection allows MongoDB to combine results from two separate single-field indexes to satisfy a compound query, but it's rarely more efficient than a dedicated compound index.

```javascript
// Scenario: two separate indexes exist
db.users.createIndex({ country: 1 })
db.users.createIndex({ age: 1 })

// Query using both conditions:
db.users.find({ country: "US", age: { $gt: 30 } }).explain()
// MongoDB may use index intersection (AND_SORTED):
{
  winningPlan: {
    stage: "AND_SORTED",
    inputStages: [
      { stage: "IXSCAN", indexName: "country_1" },  // country = "US"
      { stage: "IXSCAN", indexName: "age_1" }        // age > 30
    ]
  }
}

// HOW index intersection works:
// 1. Scan country index → get set of docIds where country = "US"
// 2. Scan age index → get set of docIds where age > 30
// 3. AND_SORTED: merge the two sets (intersection)
// 4. FETCH: load matched documents

// WHY a compound index is usually better:
db.users.createIndex({ country: 1, age: 1 })
// Single IXSCAN: scan exactly the range where country="US" AND age>30
// Fewer B-tree operations, less memory for merging, no double-scan

// WHEN index intersection can help:
// Two very selective individual indexes on different fields
// Used for different query patterns too (can't combine into one compound)
// Example:
// Index A: { userId: 1 } — used for many user-based queries
// Index B: { status: 1 } — used for many status-based queries
// Intersection query: { userId: X, status: Y } — can use both without new compound index

// In practice: query planner usually chooses compound index over intersection
// if both exist. Intersection is a fallback when only separate indexes exist.
```

---

## Answer 8: "Explain TTL indexes with all edge cases"

```javascript
// Basic TTL: expire N seconds after the indexed date field
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }
)
// Each document expires 3600 seconds (1 hour) after its createdAt value

// Expiry at a specific time: expireAfterSeconds: 0 + Date field = expiry datetime
db.scheduledEmails.createIndex(
  { sendAt: 1 },
  { expireAfterSeconds: 0 }
)
db.scheduledEmails.insertOne({
  email: "alice@example.com",
  template: "welcome",
  sendAt: new Date(Date.now() + 86400000)  // delete exactly 24 hours from now
})

// EDGE CASES:

// 1. TTL background job runs every 60 seconds
//    Documents may live UP TO 60 seconds past their expiry
//    Cannot use TTL for sub-minute precision expiry

// 2. Only works on ISODate() fields (not integers, not strings)
db.sessions.insertOne({ token: "abc", expiresAt: 1705315800 })  // integer — NEVER expires
db.sessions.insertOne({ token: "abc", expiresAt: ISODate("2024-01-15T10:30:00Z") }) // works!

// 3. If the indexed field is an ARRAY of dates:
//    Document expires when EARLIEST date in array has passed expireAfterSeconds
db.events.insertOne({
  timestamps: [
    new Date("2024-01-01"),
    new Date("2024-06-01")
  ]
})
// With expireAfterSeconds: 0 on timestamps:
// Document expires when 2024-01-01 passes (the earliest)

// 4. TTL doesn't work on capped collections
// 5. TTL doesn't work if the field contains non-Date values
//    (field is completely ignored if not a Date or array of Dates)

// 6. TTL in replica sets:
//    Only the PRIMARY runs the TTL background job
//    Secondaries replicate the deletes from primary's oplog
//    Never two TTL jobs running simultaneously

// 7. TTL with partial index — NOT possible
//    You cannot combine TTL and partialFilterExpression on the same index
//    Workaround: add the partial filter as a separate condition in your find queries

// Changing TTL on existing index (without dropping and recreating):
db.runCommand({
  collMod: "sessions",
  index: {
    keyPattern: { createdAt: 1 },
    expireAfterSeconds: 7200    // change from 3600 to 7200
  }
})
```

---

## Answer 9: "How does a multikey index work and what are its limitations?"

```javascript
// Multikey index: created when you index an array field
// MongoDB automatically detects array fields and creates a multikey index

// Example document:
{
  _id: 1,
  title: "MongoDB Guide",
  authors: ["Alice", "Bob"],          // multikey on authors
  tags: ["mongodb", "database"]       // multikey on tags
}

// Creating multikey index (same syntax as regular index):
db.books.createIndex({ authors: 1 })
// MongoDB detects authors is an array → multikey: true automatically

// Index entries created:
// ("Alice", doc1), ("Bob", doc1) — TWO entries for ONE document!

// Querying:
db.books.find({ authors: "Alice" })   // uses multikey index, efficient
db.books.find({ authors: { $in: ["Alice", "Bob"] } })  // also uses multikey

// LIMITATIONS:

// 1. Compound multikey: only ONE array field per compound index
// If a document has BOTH authors[] AND tags[], the compound index
// { authors: 1, tags: 1 } CANNOT be used (parallel array prohibition)
// Solution: create two separate indexes { authors: 1 } and { tags: 1 }

// 2. Sharding: cannot use a multikey field as a shard key
// (array field values vary across documents, would need complex routing)

// 3. No covered queries with multikey indexes
// The multikey expansion means you can't guarantee document=index entry

// 4. Sort on multikey: cannot sort on a multikey field
db.books.find({}).sort({ authors: 1 })
// ERROR or no sort guarantee: authors is an array, how to sort?

// 5. Unique constraint: unusual behavior with arrays
db.books.createIndex({ tags: 1 }, { unique: true })
// Two documents with the same tag in their array → DuplicateKeyError!
// { tags: ["mongodb", "db"] } and { tags: ["mongodb", "atlas"] }
// → ERROR: both have "mongodb" in tags, violates unique constraint

// Performance note:
// Multikey indexes with large arrays = much larger index than expected
// 1M docs × avg 50 tags = 50M index entries!
// Consider using a separate normalized collection for high-cardinality arrays
```

---

## Answer 10: "Describe the lifecycle of an index build and its impact on Atlas"

```javascript
// Atlas rolling index build lifecycle:

// STEP 1: Build triggered (via Atlas UI, Compass, or mongosh)
db.orders.createIndex({ customerId: 1, createdAt: -1 })

// STEP 2: Atlas identifies all replica set members
// For an M30 3-node replica set: P, S1, S2

// STEP 3: Build on SECONDARY first (S1)
// - S1 performs index build (online, non-blocking)
// - S1 reads and writes proceed
// - S1 may fall slightly behind on replication (reading its oplog)
// - Duration: proportional to collection size

// STEP 4: Build on SECONDARY second (S2)
// - Same as S1

// STEP 5: Step down PRIMARY
// - MongoDB triggers rs.stepDown()
// - Brief election (~10-30s) — write unavailability window
// - S1 or S2 becomes new primary (already has the index!)
// - Old primary becomes secondary

// STEP 6: Build on OLD PRIMARY (now secondary)
// - Builds index in the background
// - Once complete: index is on all members

// STEP 7: Done
// - All 3 members have the index
// - Write unavailability was only the brief election in Step 5

// Monitoring in Atlas:
// Cluster → Metrics → Current Operations
// Or: db.currentOp({ "command.createIndexes": { $exists: true } })

// For very large collections:
// - Build during off-peak hours (lower write volume = faster build)
// - Monitor: cluster metrics → Read/Write Latency during build
// - Writes are 10-30% slower during build (extra index maintenance)
// - Plan for 1-2 hours per 100M documents on typical hardware

// Index build failures:
// - If build fails (e.g., OOM, unique constraint violation during build)
// - Atlas will retry automatically once
// - If fails again: index creation fails, no partial index left
// - No cleanup needed from operator
```
