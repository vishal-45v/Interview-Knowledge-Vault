# Chapter 04 — Indexes & Performance: Theory Questions

---

## Index Types

**Q1. What is a single-field index and how does it work?**

```javascript
// Create a single-field ascending index
db.users.createIndex({ email: 1 })    // 1 = ascending
db.users.createIndex({ createdAt: -1 }) // -1 = descending

// MongoDB stores the index as a B-tree
// Each leaf node stores: { indexKey, recordId }
// recordId points to the actual document on disk

// With index: O(log n) lookup
db.users.findOne({ email: "alice@example.com" })
// Without index: O(n) scan

// Index also supports range queries
db.products.createIndex({ price: 1 })
db.products.find({ price: { $gte: 10, $lte: 100 } })
// B-tree range scan: jump to 10, scan forward to 100

// Index also supports sort (if sort direction matches index direction)
db.users.find().sort({ createdAt: -1 })
// Descending index on createdAt: forward scan = descending order
// Ascending index on createdAt: backward scan = also works for descending sort
```

---

**Q2. What is a compound index and what is the ESR rule?**

```javascript
// Compound index: index on multiple fields
db.orders.createIndex({ status: 1, customerId: 1, orderDate: -1 })

// ESR RULE: Equality → Sort → Range
// The optimal order for fields in a compound index:
// 1. Equality fields first (fields with exact match: field: "value")
// 2. Sort fields next (fields used in .sort())
// 3. Range fields last (fields with $gt, $lt, $in, etc.)

// Example query:
db.orders.find(
  { status: "active",           // Equality
    orderDate: { $gte: lastWeek } // Range
  }
).sort({ customerId: 1 })       // Sort

// Optimal compound index following ESR:
db.orders.createIndex({ status: 1, customerId: 1, orderDate: 1 })
// E: status (equality)
// S: customerId (sort field)
// R: orderDate (range)

// WHY ESR matters:
// - Equality first: maximally filters documents to a small set
// - Sort next: if sort field is in index between E and R, MongoDB can use index for sort
//   (avoiding in-memory sort)
// - Range last: range conditions "consume" the index — subsequent fields can't be used

// Anti-ESR example (Range before Sort):
db.orders.createIndex({ status: 1, orderDate: 1, customerId: 1 })
// E: status, R: orderDate, then customerId
// MongoDB can't use index for the sort on customerId
// because the range on orderDate breaks the sort key ability
```

---

**Q3. What is a multikey index?**

```javascript
// When you index an array field, MongoDB creates a MULTIKEY INDEX
// One index entry per array element (not one per document)
db.posts.createIndex({ tags: 1 })  // tags is an array → multikey index

// Document: { _id: 1, tags: ["mongodb", "database", "nosql"] }
// Multikey index entries:
// "database" → doc1
// "mongodb"  → doc1
// "nosql"    → doc1

// Querying:
db.posts.find({ tags: "mongodb" })   // uses multikey index — fast
db.posts.find({ tags: { $in: ["mongodb", "nosql"] } })  // also uses multikey index

// MULTIKEY INDEX LIMITATION:
// A compound index CANNOT have TWO array fields
db.collection.createIndex({ arrayField1: 1, arrayField2: 1 })
// ERROR if any document has BOTH fields as arrays simultaneously!
// (Allowed if only one or the other is array per document, but still limited)

// Check if index is multikey:
db.posts.getIndexes()
// { name: "tags_1", multikey: true, ... }

// isMultiKey in explain():
db.posts.find({ tags: "mongodb" }).explain()
// "isMultiKey": true in the winningPlan IXSCAN
```

---

**Q4. What is a text index and how does it differ from a regex search?**

```javascript
// Create a text index on one or more string fields
db.articles.createIndex({ title: "text", content: "text" })
// Only ONE text index per collection allowed

// Text search query — uses $text operator
db.articles.find({ $text: { $search: "mongodb performance" } })
// Searches for "mongodb" OR "performance" in title and content
// Uses word stemming (e.g., "running" matches "run")

// Exact phrase search (wrap in quotes)
db.articles.find({ $text: { $search: '"mongodb performance"' } })

// Exclude a word with - prefix
db.articles.find({ $text: { $search: "mongodb -nosql" } })

// Get relevance score
db.articles.find(
  { $text: { $search: "mongodb performance" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })

// vs. regex (slow for full-text):
db.articles.find({ content: { $regex: /mongodb/i } })
// regex scans ALL index entries (no word tokenization, no stemming)
// text index tokenizes + stems → much more efficient for text search

// Wildcard text index (all string fields):
db.articles.createIndex({ "$**": "text" })
```

---

**Q5. What is a TTL (Time-To-Live) index?**

```javascript
// TTL index: automatically deletes documents after a specified time
// Creates a background job that removes expired documents every 60 seconds

// Option 1: Expire after N seconds from a Date field
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // delete 1 hour after createdAt
)

// Option 2: Expire AT a specific date (expireAfterSeconds: 0)
db.scheduledJobs.createIndex(
  { expiresAt: 1 },     // field contains the exact expiry Date
  { expireAfterSeconds: 0 }   // expires exactly at the field value
)
db.scheduledJobs.insertOne({
  task: "cleanup",
  expiresAt: new Date(Date.now() + 86400000)  // expires in 24 hours
})

// TTL index requirements:
// 1. MUST be on a Date field (not string, not number)
// 2. Cannot be a compound index (only single-field TTL)
// 3. Cannot be on a capped collection
// 4. The TTL deletion job runs every 60 seconds (not immediate)
//    Documents may live up to 60 seconds past their expiry

// Changing TTL on existing index (MongoDB 4.2+):
db.runCommand({
  collMod: "sessions",
  index: { name: "createdAt_1", expireAfterSeconds: 7200 }  // change to 2 hours
})
```

---

**Q6. What is a partial index and when should you use it?**

```javascript
// Partial index: only indexes documents matching a filter expression
// Smaller index = less memory, faster updates, better cache utilization

// Index only active users (ignore inactive)
db.users.createIndex(
  { email: 1 },
  {
    partialFilterExpression: { isActive: { $eq: true } },
    unique: true  // unique among ACTIVE users only
  }
)

// Queries must include the partition filter or have a more selective filter
// to use this index:
db.users.find({ email: "alice@example.com", isActive: true })  // can use index
db.users.find({ email: "alice@example.com" })  // CANNOT use partial index!
// (MongoDB doesn't know if email belongs to an active user without checking the filter)

// Index only documents with a specific field
db.products.createIndex(
  { category: 1, price: 1 },
  { partialFilterExpression: { category: { $exists: true } } }
)

// Common use cases:
// - "soft deleted" pattern: only index non-deleted documents
db.orders.createIndex(
  { customerId: 1, status: 1 },
  { partialFilterExpression: { deletedAt: { $exists: false } } }
)

// - Only index high-value items
db.orders.createIndex(
  { createdAt: -1 },
  { partialFilterExpression: { total: { $gte: 1000 } } }
)
```

---

**Q7. What is a sparse index?**

```javascript
// Sparse index: only includes documents where the indexed field EXISTS
// Documents without the field are EXCLUDED from the index

db.users.createIndex({ phone: 1 }, { sparse: true })
// Index only contains entries for users WHO HAVE a phone field
// Users without phone field are NOT in the index

// Use case: optional fields that only some documents have
// Without sparse: index has many null entries (wastes space)
// With sparse: index only has entries where data exists

// IMPORTANT TRAP: Sparse index and queries
// Sparse index CANNOT be used for "field doesn't exist" queries:
db.users.find({ phone: { $exists: false } })
// Cannot use sparse index (the non-indexed docs are exactly what you're looking for!)
// Must use COLLSCAN or a partial index instead

// Sparse index for unique constraint on optional field:
db.users.createIndex(
  { phone: 1 },
  { unique: true, sparse: true }
)
// Allows multiple documents with NO phone (all excluded from index)
// But enforces uniqueness among documents THAT HAVE a phone

// Sparse vs Partial index:
// sparse: { sparse: true } — same as partialFilterExpression: { field: { $exists: true } }
// Partial index is more flexible: can use any filter expression
// Sparse index is the legacy simpler option
```

---

**Q8. What is a wildcard index (MongoDB 4.2+)?**

```javascript
// Wildcard index: indexes ALL fields or all fields within a subpath
// Useful for querying dynamic/polymorphic document schemas

// Index ALL top-level fields
db.products.createIndex({ "$**": 1 })

// Index all fields within a specific subdocument
db.products.createIndex({ "attributes.$**": 1 })

// With a wildcard index:
db.products.find({ "attributes.color": "red" })           // uses index
db.products.find({ "attributes.weight": { $lt: 5 } })     // uses index
db.products.find({ "attributes.material": "wood" })       // uses index
// ANY attribute field is indexed without creating individual indexes!

// Use case: product catalog with different attributes per product type
{
  _id: ObjectId("..."),
  type: "shirt",
  attributes: { color: "blue", size: "M", fabric: "cotton" }
}
{
  _id: ObjectId("..."),
  type: "laptop",
  attributes: { ram: "16GB", storage: "512GB", cpu: "M2" }
}
// One wildcard index covers all attribute queries

// Limitations:
// 1. Cannot use wildcard for sort (must create explicit index for sort fields)
// 2. Cannot be used in compound index with other fields (except compound wildcard in 7.0)
// 3. Less efficient than targeted single-field indexes for known query patterns

// Compound wildcard index (MongoDB 7.0+):
db.products.createIndex({ category: 1, "attributes.$**": 1 })
// Efficient for: { category: "electronics", "attributes.anyField": value }
```

---

**Q9. What is a hashed index and when is it used?**

```javascript
// Hashed index: stores hash of field value, not the actual value
// Used for EQUALITY queries and for HASHED SHARDING

// Create hashed index
db.users.createIndex({ userId: "hashed" })

// Useful for:
// 1. Hashed sharding (uniform distribution)
sh.shardCollection("mydb.users", { _id: "hashed" })
// Random hash distributes documents evenly across shards

// 2. Equality queries on high-cardinality fields
// Hashed index is smaller than B-tree for very long values (UUIDs, emails)

// Limitations:
// - NO range queries (hashes destroy ordering)
// - NO sort using hashed index
// - Cannot use $gt, $lt, $in with hashed index
// - Only for exact equality: { userId: "some-uuid" }

// Use B-tree index (regular) when:
// - Range queries needed
// - Sort needed
// - Compound index needed

// Use hashed index when:
// - Shard key for uniform distribution
// - Only equality queries on the field
// - Field values are very long strings (hash is more compact)
```

---

**Q10. What is a geospatial index (2dsphere)?**

```javascript
// 2dsphere index: for spherical (real-world) geometric data in GeoJSON format
db.places.createIndex({ location: "2dsphere" })

// Document format (GeoJSON):
db.places.insertOne({
  name: "Coffee Shop",
  location: {
    type: "Point",
    coordinates: [-97.7431, 30.2672]   // [longitude, latitude]
  },
  category: "cafe"
})

// Geospatial queries:
// $near: documents near a point, sorted by distance
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-97.7431, 30.2672] },
      $maxDistance: 1000  // meters
    }
  }
})

// $geoWithin: documents within a polygon
db.places.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[[lng1, lat1], [lng2, lat2], [lng3, lat3], [lng1, lat1]]]
      }
    }
  }
})

// $geoNear aggregation stage (also returns distance)
db.places.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-97.7431, 30.2672] },
      distanceField: "distanceMeters",
      maxDistance: 5000,
      spherical: true
    }
  },
  { $limit: 10 }
])

// 2d index (legacy, flat earth model):
db.places.createIndex({ coordinates: "2d" })
// For lat/lon pairs stored as [lat, lng] arrays (old format)
// Not recommended for new applications — use 2dsphere
```

---

## Covered Queries

**Q11. What is a covered query and how do you create one?**

```javascript
// A covered query satisfies filter + sort + projection using ONLY the index
// Zero documents are fetched from collection storage

// Create an index covering all fields needed in the query:
db.products.createIndex({ category: 1, price: 1, name: 1 })

// Covered query — ALL of these apply:
// 1. Filter fields are in the index: category, price
// 2. Projection fields are in the index: name, price, category
// 3. _id is explicitly EXCLUDED (it's not in the index, included by default)
db.products.find(
  { category: "electronics", price: { $lte: 100 } },
  { name: 1, price: 1, category: 1, _id: 0 }  // ← _id: 0 is critical!
)

// Verify with explain:
db.products.find(
  { category: "electronics", price: { $lte: 100 } },
  { name: 1, price: 1, category: 1, _id: 0 }
).explain("executionStats")
// executionStats.totalDocsExamined: 0  ← ZERO document reads!
// Plan: IXSCAN (no FETCH stage)

// Performance benefit:
// With FETCH: reads index + reads each document from disk (random I/O)
// Without FETCH (covered): reads only the index (sequential I/O)
// Typically 10-100x faster for read-heavy workloads
```

---

## Index Selectivity

**Q12. What is index selectivity and why does it matter?**

```javascript
// Selectivity = how many documents an index eliminates
// High selectivity: index narrows down to very few documents (good)
// Low selectivity: index still returns most of the collection (bad)

// HIGH SELECTIVITY — good index
db.users.createIndex({ email: 1 }, { unique: true })
// email is unique: each query returns exactly 1 document
// Selectivity ratio: 1 / N (where N = total users)

// LOW SELECTIVITY — bad index candidate alone
db.orders.createIndex({ status: 1 })
// Assuming 80% of orders are "completed"
// Query: { status: "completed" } — returns 80% of collection
// Index barely helps — near-COLLSCAN performance

// Solution: compound index to increase selectivity
db.orders.createIndex({ status: 1, customerId: 1 })
// { status: "completed", customerId: userId }
// Narrows from 80% to one user's completed orders — high selectivity

// Measuring selectivity:
// Cardinality check:
db.orders.aggregate([
  { $group: { _id: "$status", count: { $sum: 1 } } }
])
// { _id: "completed", count: 800000 }
// { _id: "pending",   count: 150000 }
// { _id: "cancelled", count: 50000 }
// "completed" has 80% of docs — low selectivity

// Rule of thumb: an index is worthwhile when it selects < 20% of documents
// MongoDB may choose COLLSCAN over a low-selectivity index (COLLSCAN can be faster
// due to sequential I/O vs. random index lookups + document fetches)
```

---

**Q13. What does the index stats aggregation stage tell you?**

```javascript
// $indexStats: returns usage statistics for all indexes on a collection
db.orders.aggregate([{ $indexStats: {} }])

// Sample output:
[
  {
    name: "_id_",
    key: { _id: 1 },
    host: "mongo1:27017",
    accesses: {
      ops: 15234,          // number of times this index was used
      since: ISODate("2024-01-01T00:00:00Z")  // when tracking began
    }
  },
  {
    name: "status_1",
    key: { status: 1 },
    accesses: {
      ops: 3,              // ONLY USED 3 TIMES — probably unused!
      since: ISODate("2024-01-01T00:00:00Z")
    }
  },
  {
    name: "customerId_1_orderDate_-1",
    key: { customerId: 1, orderDate: -1 },
    accesses: {
      ops: 2891345,        // heavily used
      since: ISODate("2024-01-01T00:00:00Z")
    }
  }
]

// Use $indexStats to identify:
// 1. Unused indexes (ops = 0 or very low) — these are candidates for removal
//    Unused indexes still consume write overhead and memory!
// 2. Most used indexes — ensure these are optimized
// 3. Index usage distribution

// Also available in Atlas Performance Advisor:
// Suggests new indexes based on slow queries
// Identifies indexes not used in 7 days
```

---

**Q14. Explain the $hint operator and when to use it.**

```javascript
// $hint: forces MongoDB to use a specific index, bypassing the query planner
db.users.find({ age: { $gt: 30 } }).hint({ email: 1 })
db.users.find({ age: { $gt: 30 } }).hint("email_1")  // by index name

// When to use $hint:
// 1. Testing: compare performance of different indexes
db.users.find({ status: "active" }).hint({ status: 1 }).explain("executionStats")
db.users.find({ status: "active" }).hint({ status: 1, email: 1 }).explain("executionStats")
// Compare execution times to find optimal index

// 2. Query planner chose wrong index (rare, but happens with stale statistics)
db.orders.find({ customerId: id, status: "pending" }).hint({ customerId: 1 })

// 3. Force COLLSCAN for debugging:
db.users.find({ email: "alice@example.com" }).hint({ $natural: 1 })
// $natural = natural order = COLLSCAN

// WARNING: $hint bypasses the query planner — if you add a better index later,
// $hint will still use the old one. Remove hints when no longer needed.

// Atlas Performance Advisor shows you cases where MongoDB chose a suboptimal
// index (query planner problem), which is when $hint becomes valuable
```

---

**Q15. What are the performance implications of index builds in production?**

```javascript
// MongoDB 4.4+: ONLINE index builds (non-blocking by default)
// Old behavior (<4.2): blocking index builds (held write lock)

// Online index build process:
// 1. Initial scan: reads all documents, builds index in background
//    During this phase: reads/writes proceed, with temporary performance impact
// 2. Lock: briefly takes write lock to finalize
//    Duration: very short (milliseconds)

// Create index in production (4.4+):
db.users.createIndex({ email: 1 }, { unique: true })
// This is non-blocking in 4.4+, but still has performance overhead
// During build: writes are slower (must update in-progress index too)
// Build time: ~minutes to hours for large collections

// Monitor index build progress:
db.currentOp({ "command.createIndexes": { $exists: true } })
// Or in Atlas: monitor in cluster metrics

// Atlas manages index builds safely
// For very large collections: build during off-peak hours

// Dropping an index:
db.users.dropIndex("email_1")        // by name
db.users.dropIndex({ email: 1 })     // by key
db.users.dropIndexes()               // drop ALL non-_id indexes (dangerous!)

// Rolling index build for sharded clusters:
// Build index on secondaries first, then step down primary to become secondary, build there
// Minimizes impact on production reads/writes

// Index building does NOT hold reads during 4.4+ builds
// But writes must update the in-progress index: ~10-30% write throughput reduction
```

---

**Q16. What is index intersection?**

```javascript
// Index intersection: MongoDB can sometimes use TWO separate single-field indexes
// for a single query, combining their results

// Indexes:
db.orders.createIndex({ status: 1 })
db.orders.createIndex({ customerId: 1 })

// Query:
db.orders.find({ status: "active", customerId: userId })

// MongoDB may:
// Option A: Use ONLY { status: 1 } index, then filter by customerId
// Option B: Use ONLY { customerId: 1 } index, then filter by status
// Option C: Use BOTH indexes (AND_SORTED stage) — index intersection

// Check with explain:
db.orders.find({ status: "active", customerId: userId }).explain()
// winningPlan.stage may be "AND_SORTED" (index intersection) or single IXSCAN

// In practice: index intersection is rarely chosen by the planner
// A compound index { status: 1, customerId: 1 } is almost always faster
// Index intersection requires sorting and merging two result sets

// When index intersection IS beneficial:
// - Individual indexes are very selective
// - No compound index exists
// - The two conditions are independently very narrow

// Generally: prefer compound indexes over relying on index intersection
```

---

**Q17. What are background index builds and how do they differ in MongoDB 4.0 vs 4.4+?**

```javascript
// MongoDB < 4.2:
db.users.createIndex({ email: 1 }, { background: true })
// background: true = allowed reads/writes during build, BUT:
// - Slower build time
// - Index not immediately available
// - Still blocked certain operations briefly at completion

db.users.createIndex({ email: 1 })  // without background
// Foreground: blocked ALL reads/writes until build complete
// Faster build, but caused downtime on production

// MongoDB 4.4+: ALL index builds are online (blocking: true no longer relevant)
// The background option is deprecated (ignored)
// Default behavior IS non-blocking
db.users.createIndex({ email: 1 })  // automatically non-blocking in 4.4+

// Build phases in 4.4+:
// Phase 1 (bulk build): scans collection, writes index
//   - Reads: unrestricted
//   - Writes: proceed but must also update in-progress index (some overhead)
// Phase 2 (drain): briefly takes intent lock to finalize
//   - Duration: milliseconds (not seconds/minutes like before)
// Phase 3: index available

// Atlas behavior:
// Atlas builds indexes in "rolling" fashion for dedicated clusters
// Primary is LAST to build — ensures no primary downtime
```

---

**Q18. Explain the ESR rule with a practical example.**

```javascript
// ESR Rule: Equality → Sort → Range
// Determines optimal field ORDER in a compound index

// Query:
db.products.find(
  {
    category: "electronics",                 // E: Equality
    price: { $gte: 10, $lte: 200 }           // R: Range
  }
).sort({ name: 1 })                         // S: Sort

// BAD index order (RSE or ESR without sort):
db.products.createIndex({ category: 1, price: 1, name: 1 })
// This index: E=category, R=price, S=name
// Problem: range on price breaks the index scan for name sort
// MongoDB must sort name in memory (sort stage added in pipeline)

// GOOD index order following ESR:
db.products.createIndex({ category: 1, name: 1, price: 1 })
// E=category, S=name, R=price
// Now: index scan narrows to "electronics", sorted by name, then range on price
// Sort is supported by the index — no in-memory sort needed!

// Verify with explain():
db.products.find({ category: "electronics", price: { $gte: 10, $lte: 200 } })
  .sort({ name: 1 })
  .explain("executionStats")

// BAD (range before sort):
// winningPlan.inputStage: "IXSCAN"
// winningPlan.stage: "SORT" ← in-memory sort added!
// executionStats: sortSpills: 1 ← even spilled to disk

// GOOD (ESR order):
// winningPlan.inputStage: "IXSCAN"
// winningPlan.stage: "FETCH" ← no extra sort stage!
// executionStats: sortSpills: 0

// The ESR rule applies when you have all three types of conditions
// If you only have E and R (no sort): put equality fields first
// If you only have E and S (no range): put equality first, then sort
```

---

**Q19. How do you use the Atlas Performance Advisor?**

The Atlas Performance Advisor analyzes slow queries and recommends indexes:

```javascript
// Atlas Performance Advisor shows:
// 1. Slow queries (> 100ms by default threshold)
// 2. Suggested indexes based on query patterns
// 3. Indexes that haven't been used in 7+ days

// Example recommendation from Atlas Performance Advisor:
// "Create index on: { userId: 1, createdAt: -1 }"
// "Queries analyzed: 1,234 queries over the past 24 hours"
// "Average execution time improvement: 3200ms → 5ms"

// To accept a suggestion from Atlas UI:
// Navigate to: Cluster → Performance Advisor → Create Index

// Programmatically check index suggestions via Atlas Admin API:
// GET /api/atlas/v1.0/groups/{groupId}/processes/{host}/performanceAdvisor/suggestedIndexes

// Schema Advisor (Atlas):
// Identifies schema issues:
// - Documents where field has inconsistent types
// - Collections with high churn rate that might benefit from TTL
// - Documents approaching 16MB limit

// Manual profiler analysis (without Atlas):
db.setProfilingLevel(1, { slowms: 100 })  // log queries > 100ms
db.system.profile.find({
  millis: { $gt: 100 },
  op: { $ne: "command" }
}).sort({ millis: -1 }).limit(20)

// Look for patterns:
// - Same collection repeatedly slow
// - COLLSCAN in planSummary
// - nExamined >> nReturned
```

---

**Q20. What is index bloat and how do you address it?**

```javascript
// Index bloat: index becomes fragmented or larger than necessary
// Causes:
// 1. Many deletes/updates leave gaps in the B-tree (fragmentation)
// 2. Growing arrays in multikey indexes
// 3. Indexes that are no longer aligned with query patterns

// Check index size:
db.collection.stats().indexSizes
// { "_id_": 12345678, "email_1": 98765432, "tags_1": 345678901 }

// Detect fragmented/bloated indexes:
db.runCommand({ validate: "orders", full: true })
// Returns: keysPerIndex, valid: true/false, errors

// Solutions:
// 1. Compact collection (rebuilds collection and all indexes)
db.runCommand({ compact: "orders" })
// WARNING: takes exclusive lock, causes downtime on standalone
// In Atlas: compact happens during maintenance windows automatically

// 2. Drop and recreate problematic index
db.orders.dropIndex("old_index_name")
db.orders.createIndex({ ... })

// 3. Regular maintenance with Atlas:
// Atlas auto-compacts storage during maintenance windows
// No manual compact() needed in most Atlas use cases

// 4. Identify truly unused indexes:
db.orders.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $lt: 10 } } }  // used < 10 times since last reset
])
// Drop unused indexes — they slow down writes without helping reads

// 5. For multikey indexes with growing arrays:
// Switch from embedded arrays to separate collection (reference pattern)
// Prevents unbounded multikey index growth
```

---

**Q21. What is a clustered collection (MongoDB 5.3+)?**

```javascript
// Clustered collection: documents stored in _id order on disk
// In regular collections, the _id index is a separate B-tree structure
// In clustered collections, documents ARE the index (no separate _id index)

// Create a clustered collection
db.createCollection("sensorData", {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true
  }
})

// Benefits:
// 1. Faster range queries on _id (especially for time-based ObjectIds or UUIDs)
// 2. More efficient storage (no separate _id index structure)
// 3. Better cache utilization for sequential _id reads
// 4. TTL index on _id is much faster (deletes sequential ranges efficiently)

// Best for:
// - Time-series-like data with sequential _id values
// - Collections with TTL (use expireAfterSeconds on clustered index)
// - Large collections with frequent range scans on _id

// Creating clustered collection with TTL:
db.createCollection("events", {
  clusteredIndex: { key: { _id: 1 }, unique: true, expireAfterSeconds: 86400 }
})
// Documents expire 24 hours after their ObjectId creation time

// Compare with time-series collection:
// Time-series: optimized for immutable sensor data with bucketing compression
// Clustered: optimized for range scans and TTL, supports updates/deletes
```

---

**Q22. What are the trade-offs of having many indexes?**

```javascript
// BENEFITS of indexes:
// - Faster reads (O(log n) vs O(n))
// - Faster sorts (index sort vs in-memory sort)
// - Enable covered queries (zero document reads)

// COSTS of indexes:
// 1. Write overhead: every insert/update/delete must update ALL indexes
//    10 indexes = 10x more B-tree operations per write

// 2. Memory: indexes must fit in working set (RAM)
//    1M documents × 10 indexes × 30 bytes/entry = ~300MB just for indexes!

// 3. Storage: each index takes disk space
//    Large collections with many indexes can double storage usage

// 4. Slower backup/restore: more data to copy

// Rule of thumb:
// - 5 or fewer indexes per collection for write-heavy collections
// - Keep unused indexes deleted
// - Monitor with $indexStats to find zero-usage indexes

// Atlas Index Advisor shows:
// "7 indexes detected. 3 indexes have not been used in the past 7 days."
// → Drop the unused 3!

// Finding "redundant" indexes (one index is a prefix of another):
db.collection.getIndexes()
// Index A: { userId: 1 }
// Index B: { userId: 1, createdAt: -1 }
// Index A is redundant if ALL queries using A are also served by B!
// (B covers all queries that A would serve, PLUS sort on createdAt)

// Drop Index A if queries are:
// { userId: X }  → Index B handles this (uses userId prefix)
// { userId: X, createdAt: { $gt: date } }  → Index B perfect
// Keep Index A only if there are queries that JUST need userId lookup without sort
// AND the extra field in B causes significant overhead (unlikely)
```

---

**Q23. How do indexes work with the aggregation pipeline?**

```javascript
// Index usage in aggregation: only early stages can use indexes

// $match (first stage) can use an index:
db.orders.aggregate([
  { $match: { status: "pending", customerId: userId } }  // uses index
])

// $sort (if immediately after $match) can use index:
db.orders.aggregate([
  { $match: { customerId: userId } },
  { $sort: { orderDate: -1 } }     // can use { customerId: 1, orderDate: -1 } index
])

// $limit + $sort gets special optimization:
db.orders.aggregate([
  { $sort: { _id: -1 } },
  { $limit: 10 }
])
// Optimizer combines sort+limit → reads only first 10 from index
// Instead of sorting ALL docs then taking 10

// After $unwind, $group, $project — index can no longer be used:
db.orders.aggregate([
  { $match: { status: "pending" } },   // uses index
  { $unwind: "$items" },
  { $match: { "items.category": "electronics" } },  // CANNOT use index on items.category
  { $sort: { "items.price": 1 } }     // CANNOT use index
])

// Check aggregation index usage:
db.orders.aggregate([...]).explain()
// winningPlan in the explain output shows whether indexes were used

// $lookup doesn't benefit from aggregation pipeline indexes,
// but the "from" collection lookup DOES use indexes on the foreignField:
db.orders.aggregate([
  { $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "user" } }
])
// db.users._id index is used for each lookup — critical for performance!
```

---

**Q24. What is a unique index and how does it enforce uniqueness?**

```javascript
// Unique index: enforces uniqueness of the indexed field(s)
db.users.createIndex({ email: 1 }, { unique: true })

// Attempting to insert a duplicate throws DuplicateKeyError:
db.users.insertOne({ email: "alice@example.com" })  // success
db.users.insertOne({ email: "alice@example.com" })  // Error 11000

// Compound unique index: combination of fields must be unique
db.orders.createIndex(
  { customerId: 1, orderNumber: 1 },
  { unique: true }
)
// Two different customers can have orderNumber: "001"
// But the same customer cannot have two orders with orderNumber: "001"

// Unique index with null values:
// By default, null/missing values DO count toward uniqueness
// Only ONE document can have null/missing value for the unique field!
db.users.insertOne({ name: "Alice" })  // email missing = null, OK
db.users.insertOne({ name: "Bob" })    // ERROR: second null violates uniqueness

// Fix: sparse + unique allows MULTIPLE null/missing values:
db.users.createIndex(
  { email: 1 },
  { unique: true, sparse: true }
)
// Now: multiple docs without email are allowed
// But: docs WITH email must have unique email values

// Creating unique index on existing collection with duplicates:
// MongoDB will FAIL to create the index if duplicates exist
db.users.aggregate([
  { $group: { _id: "$email", count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
])
// Fix duplicates first, then create unique index
```

---

**Q25. Describe the MongoDB index build process and how to monitor it.**

```javascript
// MongoDB 4.4+ index build process:
// Phase 1: Collection scan — reads all documents, builds index in memory/disk
// Phase 2: Index flush — writes all accumulated changes
// Phase 3: Commit — briefly takes write lock to finalize

// Monitor active index builds:
db.currentOp({
  "command.createIndexes": { $exists: true }
})
// Output:
{
  op: "command",
  ns: "mydb.orders",
  command: { createIndexes: "orders", indexes: [{ key: { customerId: 1 } }] },
  msg: "Index Build: scanning collection (10% complete)",
  progress: { done: 1000000, total: 10000000 }
}

// Monitor via Atlas:
// Cluster → Metrics → Index Build Throughput
// Or: Atlas UI shows index build status with percentage complete

// Force kill a running index build:
db.killOp(opId)

// Check if index build completed:
db.orders.getIndexes()
// If the index appears in getIndexes(), build is complete

// Atlas-specific behavior:
// Atlas clusters build indexes in a "rolling" fashion:
// 1. Build on secondary 1
// 2. Build on secondary 2
// 3. Step down primary → becomes secondary → build on it
// 4. Old primary (now secondary) gets re-elected as primary
// Minimizes primary impact

// After index build, MongoDB runs an index validation:
db.orders.validate({ full: true })
// Checks index integrity — run after major operations
```
