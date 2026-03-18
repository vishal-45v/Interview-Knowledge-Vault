# Chapter 10 — MongoDB Operations: Theory Questions

---

## Q1: What is the WiredTiger storage engine and how does it work?

WiredTiger is MongoDB's default storage engine (since MongoDB 3.2). It provides document-level concurrency control, compression, and MVCC (Multi-Version Concurrency Control).

**Core mechanics:**

```javascript
// WiredTiger key features:
// 1. Document-level locking (not collection-level like MMAPv1)
// 2. MVCC: readers don't block writers, writers don't block readers
// 3. B-tree data structure for collections and indexes
// 4. Compression: snappy (default), zlib, zstd, none

// Check current storage engine:
db.serverStatus().storageEngine
// { "name": "wiredTiger", "supportsCommittedReads": true, ... }

// WiredTiger cache configuration:
db.serverStatus().wiredTiger.cache
// "maximum bytes configured": 8589934592  ← 8GB (50% of RAM by default)
// "bytes currently in cache": 3221225472
// "bytes read into cache": 45678901234
// "bytes written from cache": 23456789012

// Adjust WiredTiger cache (mongod startup option):
// mongod --wiredTigerCacheSizeGB 4
// Or in mongod.conf:
// storage:
//   wiredTiger:
//     engineConfig:
//       cacheSizeGB: 4

// Document-level concurrency:
// Thread 1: updateOne({ _id: 1 }, { $set: { status: "active" } })
// Thread 2: updateOne({ _id: 2 }, { $set: { status: "inactive" } })
// → These execute CONCURRENTLY (different document locks)
// vs. Thread 1 + Thread 2 updating _id:1 → serialized
```

**Checkpointing:**

```javascript
// WiredTiger writes data to disk via checkpoints (not every write)
// Checkpoint interval: 60 seconds by default
// Between checkpoints: writes go to journal (WAL - Write-Ahead Log)
// If crash between checkpoints: journal replays uncommitted data

// Journal location:
// /data/db/journal/

// Force a checkpoint:
db.adminCommand({ fsync: 1 })
// Flushes all dirty pages to disk, creates a checkpoint

// Journal flush interval (default: 100ms):
// storage:
//   journal:
//     commitIntervalMs: 100
// Lower = less data loss risk, higher I/O
// Higher = better performance, more potential data loss on crash
```

---

## Q2: How does MongoDB indexing work? Describe index types and when to use each.

MongoDB uses B-tree indexes (and specialized structures for text, geo, hashed). Every read operation should ideally use an index — a COLLSCAN (full collection scan) on a large collection is almost always a performance problem.

```javascript
// Single field index:
db.users.createIndex({ email: 1 })
// 1 = ascending, -1 = descending (affects sort direction optimization)

// Compound index (ESR rule: Equality → Sort → Range):
db.orders.createIndex({ customerId: 1, status: 1, createdAt: -1 })
// Equality field first (customerId), sort field second (status),
// range field last (createdAt range queries)

// Unique index:
db.users.createIndex({ email: 1 }, { unique: true })
// Prevents duplicate email values

// Partial index (only index documents matching a filter):
db.orders.createIndex(
  { createdAt: 1 },
  { partialFilterExpression: { status: "active" } }
)
// Index only contains active orders
// Much smaller than full index, query MUST include status: "active"

// Sparse index (skip documents where indexed field is missing):
db.users.createIndex({ phoneNumber: 1 }, { sparse: true })
// Documents without phoneNumber field are not in this index
// Useful for optional fields — avoids indexing null/missing values

// TTL index (auto-delete documents after a time):
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // delete 1 hour after createdAt
)

// Text index (native full-text search, limited vs Atlas Search):
db.articles.createIndex({ title: "text", body: "text" })
db.articles.find({ $text: { $search: "mongodb atlas" } })

// 2dsphere index (geospatial queries):
db.stores.createIndex({ location: "2dsphere" })
db.stores.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-73.97, 40.77] },
      $maxDistance: 5000  // 5km
    }
  }
})

// Hashed index (used for hashed sharding):
db.users.createIndex({ userId: "hashed" })
// Not useful for range queries, only equality

// Wildcard index (index all fields or all sub-fields):
db.products.createIndex({ "attributes.$**": 1 })
// Indexes any attribute regardless of name
// Good for document models with variable attribute names
// (Attribute Pattern use case)

// Check index usage:
db.orders.find({ customerId: "c1" }).explain("executionStats")
// Look for: "stage": "IXSCAN" (good) vs "COLLSCAN" (bad)
```

---

## Q3: How do you use `explain()` to diagnose query performance?

`explain()` reveals MongoDB's query execution plan — which indexes are used, how many documents were examined, and where time was spent.

```javascript
// Three verbosity levels:
// 1. "queryPlanner" — just the winning plan (default)
// 2. "executionStats" — winning plan + execution metrics
// 3. "allPlansExecution" — all candidate plans + metrics (slowest)

// Basic usage:
db.orders.find({ customerId: "c1", status: "pending" })
  .explain("executionStats")

// Output anatomy:
{
  "queryPlanner": {
    "plannerVersion": 1,
    "namespace": "mydb.orders",
    "indexFilterSet": false,
    "parsedQuery": { ... },
    "winningPlan": {
      "stage": "FETCH",               // ← fetches full document
      "inputStage": {
        "stage": "IXSCAN",            // ← uses an index
        "keyPattern": { "customerId": 1, "status": 1 },
        "indexName": "customerId_1_status_1",
        "isMultiKey": false,
        "direction": "forward",
        "indexBounds": {
          "customerId": [ "[\"c1\", \"c1\"]" ],
          "status": [ "[\"pending\", \"pending\"]" ]
        }
      }
    }
  },
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 5,                   // ← 5 documents returned
    "executionTimeMillis": 2,         // ← 2ms total
    "totalKeysExamined": 5,           // ← 5 index keys examined
    "totalDocsExamined": 5            // ← 5 documents fetched
  }
}

// RED FLAGS in explain output:
// 1. "stage": "COLLSCAN" → no index used, full collection scan
// 2. totalDocsExamined >> nReturned → poor selectivity (index exists but not selective)
//    Example: totalDocsExamined: 50000, nReturned: 5 → 10,000x over-scan
// 3. "memUsage" > 104857600 (100MB) in sort stage → spillover to disk
// 4. "stage": "SORT" without "IXSCAN" → in-memory sort, not using index sort

// Covered query (index covers all projected fields — no FETCH stage):
db.users.createIndex({ email: 1, name: 1, age: 1 })
db.users.find({ email: "user@example.com" }, { email: 1, name: 1, _id: 0 })
  .explain("executionStats")
// "stage": "IXSCAN" with NO "FETCH" stage → covered query!
// totalDocsExamined: 0 (never touches the collection)

// Aggregation pipeline explain:
db.orders.aggregate(
  [
    { $match: { customerId: "c1" } },
    { $group: { _id: "$status", count: { $sum: 1 } } }
  ],
  { explain: true }
)
```

---

## Q4: What is the aggregation pipeline and what are its key stages?

The aggregation pipeline is MongoDB's primary data transformation and analysis framework. Data flows through ordered stages, each transforming the input document stream.

```javascript
// Core stages and their purposes:

// $match — filter documents (push early to reduce pipeline volume)
{ $match: { status: "active", createdAt: { $gte: ISODate("2026-01-01") } } }
// Uses indexes when first in pipeline

// $project — reshape documents (include/exclude/add fields)
{ $project: {
    customerId: 1,
    amount: 1,
    year: { $year: "$createdAt" },
    _id: 0  // exclude _id
}}

// $group — aggregate/accumulate
{ $group: {
    _id: { customerId: "$customerId", month: { $month: "$createdAt" } },
    totalSpent: { $sum: "$amount" },
    orderCount: { $count: {} },
    avgOrderValue: { $avg: "$amount" },
    maxOrder: { $max: "$amount" }
}}

// $sort — sort documents
{ $sort: { totalSpent: -1, orderCount: -1 } }

// $limit and $skip — pagination
{ $skip: 20 },
{ $limit: 10 }

// $lookup — left outer join with another collection
{ $lookup: {
    from: "products",
    localField: "productId",
    foreignField: "_id",
    as: "productDetails"
}}
// pipeline form (MongoDB 3.6+):
{ $lookup: {
    from: "orders",
    let: { userId: "$_id" },
    pipeline: [
      { $match: { $expr: { $eq: ["$customerId", "$$userId"] } } },
      { $project: { amount: 1, createdAt: 1 } }
    ],
    as: "orders"
}}

// $unwind — deconstruct array field
{ $unwind: "$items" }
// Each element of items array becomes its own document in the stream

// $addFields — add computed fields without removing others
{ $addFields: {
    discountedPrice: { $multiply: ["$price", 0.9] },
    isExpensive: { $gt: ["$price", 1000] }
}}

// $bucket — group into ranges
{ $bucket: {
    groupBy: "$amount",
    boundaries: [0, 50, 100, 500, 1000, Infinity],
    default: "Other",
    output: { count: { $sum: 1 }, totalRevenue: { $sum: "$amount" } }
}}

// $facet — multiple pipelines on same input (for faceted search)
{ $facet: {
    byStatus: [{ $group: { _id: "$status", count: { $sum: 1 } } }],
    byRegion: [{ $group: { _id: "$region", count: { $sum: 1 } } }],
    totals: [{ $group: { _id: null, total: { $sum: "$amount" } } }]
}}

// $out / $merge — write results to a collection
{ $out: "monthly_summary" }  // replaces collection atomically
{ $merge: {                  // upserts per document
    into: "monthly_summary",
    on: ["year", "month"],
    whenMatched: "merge",
    whenNotMatched: "insert"
}}

// Performance optimization: $match and $sort early, $limit before heavy stages
db.orders.aggregate([
  { $match: { status: "completed" } },  // stage 1: filter (uses index)
  { $sort: { createdAt: -1 } },         // stage 2: sort (uses index if available)
  { $limit: 100 },                       // stage 3: limit EARLY (reduces later work)
  { $lookup: { from: "products", ... } } // stage 4: expensive — only on 100 docs
], { allowDiskUse: true })
```

---

## Q5: What is the change stream API and what are its use cases?

Change streams provide a real-time stream of data changes in MongoDB using the oplog under the hood. They are the primary mechanism for event-driven applications.

```javascript
// Basic change stream (collection level):
const changeStream = db.orders.watch()
changeStream.on("change", (change) => {
  console.log("Change detected:", change)
})

// Change event structure:
{
  "_id": { "_data": "826615a1..." },    // resume token
  "operationType": "insert",            // insert | update | replace | delete | invalidate
  "clusterTime": Timestamp(1710000000, 1),
  "ns": { "db": "mydb", "coll": "orders" },
  "documentKey": { "_id": ObjectId("...") },
  "fullDocument": {                     // full document (insert/replace)
    "_id": ObjectId("..."),
    "customerId": "c1",
    "amount": 99.99
  }
}

// Update change event includes only the changed fields:
{
  "operationType": "update",
  "updateDescription": {
    "updatedFields": { "status": "shipped" },
    "removedFields": [],
    "truncatedArrays": []
  }
  // fullDocument NOT included by default for updates (performance)
}

// Get fullDocument on updates too:
const stream = db.orders.watch([], {
  fullDocument: "updateLookup"  // fetches full document for each update
})

// Filter change stream with aggregation pipeline:
const pipeline = [
  { $match: {
      operationType: { $in: ["insert", "update"] },
      "fullDocument.status": "shipped"
  }}
]
const stream = db.orders.watch(pipeline)
// Only emits events for shipped orders being inserted or updated

// Resume token (fault-tolerant streaming):
let resumeToken = null
const stream = db.orders.watch([], {
  resumeAfter: resumeToken  // null = start from now
})
stream.on("change", async (change) => {
  await processChange(change)           // your business logic
  resumeToken = change._id              // save to Redis/DB for restart recovery
  await redis.set("changeStreamToken", JSON.stringify(resumeToken))
})
// On restart:
const savedToken = JSON.parse(await redis.get("changeStreamToken"))
const resumedStream = db.orders.watch([], { resumeAfter: savedToken })
// Picks up exactly where it left off — no missed events!

// Database-level change stream (all collections):
const dbStream = db.watch()

// Cluster-level change stream (all databases):
const clusterStream = client.watch()

// Use cases:
// 1. Real-time notifications (order shipped → push notification)
// 2. Cache invalidation (document updated → invalidate Redis cache)
// 3. Audit logging (any change → write to audit collection)
// 4. Data sync (MongoDB → Elasticsearch index sync)
// 5. Event sourcing (capture all state changes as events)
// 6. Microservice communication (service A writes, service B reacts)
```

---

## Q6: How does MongoDB handle full-text search natively (without Atlas Search)?

MongoDB's native `$text` index provides basic full-text search capabilities built into the storage engine — significantly more limited than Atlas Search but available without additional services.

```javascript
// Create a text index:
db.articles.createIndex({
  title: "text",
  body: "text",
  tags: "text"
}, {
  weights: { title: 10, tags: 5, body: 1 },  // title matches worth 10x
  name: "article_text_index",
  default_language: "english"
})

// Text query:
db.articles.find(
  { $text: { $search: "mongodb atlas performance" } },
  { score: { $meta: "textScore" } }  // include relevance score
).sort({ score: { $meta: "textScore" } })

// Phrase search:
db.articles.find({ $text: { $search: "\"mongodb atlas\"" } })
// Quote marks = phrase match (words must appear together)

// Negation:
db.articles.find({ $text: { $search: "mongodb -oracle" } })
// Find articles about mongodb that do NOT mention oracle

// $text limitations:
// 1. ONE text index per collection (can't have two)
// 2. No relevance tuning beyond basic weights
// 3. No autocomplete (partial word matching)
// 4. No fuzzy matching (typos not handled)
// 5. No facets
// 6. Only 15 supported languages (no custom analyzers)
// 7. $text must be used in $match stage (not deeply nested)

// When to use $text vs Atlas Search:
// $text:        Simple search, M0/free tier, no Atlas needed
// Atlas Search: Autocomplete, fuzzy, facets, custom analyzers, relevance tuning

// Language configuration:
db.articles.createIndex({ body: "text" }, { default_language: "spanish" })
// Spanish stemming: "correr" matches "corriendo" etc.
db.articles.find({ $text: { $search: "correr", $language: "spanish" } })
```

---

## Q7: What are MongoDB's data types and when do each matter for queries?

MongoDB uses BSON (Binary JSON) which extends JSON with additional types. Type matters for query correctness and index efficiency.

```javascript
// BSON type comparison order (lower number = sorts first):
// 1.  MinKey (internal)
// 2.  Null
// 3.  Numbers (int32, int64, double, decimal128)
// 4.  Symbol, String
// 5.  Object
// 6.  Array
// 7.  BinData (binary)
// 8.  ObjectId
// 9.  Boolean
// 10. Date
// 11. Timestamp (internal)
// 12. Regular Expression
// 14. MaxKey (internal)

// Common type traps:

// 1. Number types: Int32, Int64, Double, Decimal128
db.products.insertOne({ price: 9.99 })  // stored as Double (64-bit IEEE 754)
// Floating point imprecision:
db.products.find({ price: 0.1 + 0.2 })  // 0.30000000000000004 — may not match!
// Use Decimal128 for financial values:
db.products.insertOne({ price: Decimal128("9.99") })
db.products.find({ price: Decimal128("9.99") })  // exact match

// 2. Date vs String
db.events.insertOne({ createdAt: "2026-03-17" })  // WRONG: string, not Date!
db.events.insertOne({ createdAt: new Date("2026-03-17") })  // CORRECT: Date object
// String "2026-03-17" < "2026-04-01" works alphabetically
// But Date comparison uses actual milliseconds — correct for range queries and TTL

// 3. ObjectId contains timestamp:
const id = new ObjectId()
id.getTimestamp()  // ISODate("2026-03-17T09:00:00.000Z")
// Can use ObjectId for range queries to filter by creation time:
const startId = ObjectId.createFromTime(new Date("2026-01-01").getTime() / 1000)
db.events.find({ _id: { $gte: startId } })  // all events from Jan 1 onwards

// 4. Array values in indexes:
db.products.createIndex({ tags: 1 })
db.products.insertOne({ tags: ["mongodb", "nosql", "database"] })
// MongoDB creates one index entry per array element:
// "mongodb" → [doc_1]
// "nosql" → [doc_1]
// "database" → [doc_1]
db.products.find({ tags: "mongodb" })  // finds doc_1 via array element match
// isMultiKey: true in explain output indicates an array index

// 5. $type operator for type checking:
db.users.find({ age: { $type: "string" } })  // find users where age is stored as string (bug!)
db.users.find({ age: { $type: ["int", "double"] } })  // numeric types
// BSON type names: "double", "string", "object", "array", "binData",
//                  "bool", "date", "null", "int", "long", "decimal"
```

---

## Q8: How does MongoDB handle schema validation?

MongoDB allows optional schema validation using JSON Schema validators. This enforces data shape and type constraints at write time.

```javascript
// Create collection with schema validation:
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "name", "createdAt"],
      additionalProperties: false,  // reject unknown fields
      properties: {
        _id: { bsonType: "objectId" },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "must be a valid email"
        },
        name: {
          bsonType: "string",
          minLength: 1,
          maxLength: 200
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150
        },
        createdAt: { bsonType: "date" },
        roles: {
          bsonType: "array",
          items: {
            bsonType: "string",
            enum: ["admin", "user", "moderator"]
          },
          uniqueItems: true
        },
        address: {
          bsonType: "object",
          required: ["city", "country"],
          properties: {
            city: { bsonType: "string" },
            country: { bsonType: "string" },
            zip: { bsonType: "string" }
          }
        }
      }
    }
  },
  validationLevel: "strict",    // "strict" = validate all writes (default)
                                 // "moderate" = validate only non-empty documents
  validationAction: "error"     // "error" = reject invalid docs (default)
                                 // "warn" = log warning, allow write
})

// Add validation to existing collection:
db.runCommand({
  collMod: "orders",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["customerId", "amount", "status"],
      properties: {
        amount: { bsonType: "double", minimum: 0 },
        status: {
          bsonType: "string",
          enum: ["pending", "processing", "shipped", "delivered", "cancelled"]
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})

// IMPORTANT LIMITATION: Validation only applies to NEW documents and updates
// Existing documents that violate the schema are NOT retroactively validated
// Use validationLevel: "moderate" to skip validation on existing violating docs
// when adding new stricter validation to a collection with legacy data
```

---

## Q9: What are MongoDB's key performance tuning parameters and techniques?

Performance tuning involves: correct indexing, query optimization, WiredTiger cache sizing, connection pool tuning, and hardware sizing.

```javascript
// 1. Index strategy
// ESR rule for compound indexes:
db.orders.createIndex({
  customerId: 1,    // E: Equality
  status: 1,        // S: Sort
  createdAt: -1     // R: Range
})

// Identify missing indexes:
db.setProfilingLevel(2, { slowms: 100 })  // log queries > 100ms
db.system.profile.find({ millis: { $gt: 100 } }).sort({ millis: -1 })

// 2. WiredTiger cache sizing
// Default: 50% of (RAM - 1GB), minimum 256MB
// For a dedicated database server with 32GB RAM:
// cacheSizeGB: (32 - 1) × 0.5 = 15.5GB
// mongod.conf:
// storage:
//   wiredTiger:
//     engineConfig:
//       cacheSizeGB: 15

// Monitor cache effectiveness:
db.serverStatus().wiredTiger.cache
// "pages read into cache" should be LOW compared to total operations
// High "pages read" = working set exceeds cache = disk I/O bottleneck

// 3. Connection pool tuning
// Default maxPoolSize: 100 connections per MongoClient instance
// For high-concurrency apps:
const client = new MongoClient(uri, {
  maxPoolSize: 200,          // max connections in pool
  minPoolSize: 10,           // keep 10 connections open always
  maxIdleTimeMS: 30000,      // close idle connections after 30s
  waitQueueTimeoutMS: 5000   // fail fast if no connection available in 5s
})

// 4. Read preference for analytics isolation
// Route expensive aggregations to secondaries:
db.analytics.aggregate(pipeline, { readPreference: "secondary" })

// 5. allowDiskUse for large aggregations
db.events.aggregate(
  [/* pipeline with large sorts */],
  { allowDiskUse: true }  // spill to disk if in-memory sort exceeds 100MB
)

// 6. Query hints (force specific index):
db.orders.find({ customerId: "c1" }).hint({ customerId: 1, status: 1 })
// Useful when query planner chooses suboptimal plan

// 7. Index background building (MongoDB 4.2+: all index builds are non-blocking)
// MongoDB 4.4+: Simultaneous index builds across all members
db.orders.createIndex(
  { customerId: 1 },
  { background: true }  // legacy option, all builds are non-blocking in 4.2+
)

// 8. Projection to limit data transfer:
// BAD: returns entire document
db.users.find({ status: "active" })
// GOOD: returns only needed fields
db.users.find({ status: "active" }, { email: 1, name: 1, _id: 0 })
```

---

## Q10: What is the MongoDB profiler and how do you use it for query optimization?

The MongoDB profiler records slow operations to the `system.profile` collection, enabling diagnosis of performance issues.

```javascript
// Profiling levels:
// 0 = Off (default in production)
// 1 = Log slow operations (above slowms threshold)
// 2 = Log ALL operations (use only briefly, very I/O intensive)

// Set profiling level:
db.setProfilingLevel(1, { slowms: 100 })  // log queries > 100ms
// Or with filter:
db.setProfilingLevel(1, {
  slowms: 50,
  filter: { op: { $in: ["query", "update", "insert"] } }  // only these op types
})

// Check current profiling settings:
db.getProfilingStatus()
// { "was": 1, "slowms": 100, "sampleRate": 1.0 }

// Query the profiler output:
db.system.profile.find({
  millis: { $gt: 1000 },       // took longer than 1 second
  ns: { $regex: "mydb" }       // in mydb database
}).sort({ millis: -1 }).limit(20)

// Profile document structure:
{
  "op": "query",
  "ns": "mydb.orders",
  "command": {
    "find": "orders",
    "filter": { "status": "pending" },
    "projection": {}
  },
  "keysExamined": 0,       // 0 = no index used!
  "docsExamined": 50000,   // examined 50k docs
  "nreturned": 12,         // returned 12
  "millis": 2345,          // took 2.3 seconds
  "planSummary": "COLLSCAN",  // ← the problem
  "ts": ISODate("2026-03-17T09:00:00.000Z")
}

// Identify top slow queries:
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  { $group: {
      _id: { op: "$op", ns: "$ns" },
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" },
      maxMillis: { $max: "$millis" },
      totalMillis: { $sum: "$millis" }
  }},
  { $sort: { totalMillis: -1 } },
  { $limit: 10 }
])

// MongoDB Atlas: Performance Advisor shows this automatically
// with recommended indexes for slow queries
```

---

## Q11: How does MongoDB handle backup and restore?

MongoDB provides several backup strategies with different RPO (Recovery Point Objective) and RTO (Recovery Time Objective) tradeoffs.

```javascript
// Method 1: mongodump / mongorestore (logical backup)
// Export BSON + metadata files
// Suitable for small-medium deployments, targeted collection backups

// Full database backup:
// mongodump --host localhost:27017 --out /backup/2026-03-17/
// Restores to:  /backup/2026-03-17/mydb/collection.bson

// Single collection:
// mongodump --db mydb --collection orders --out /backup/

// Compressed:
// mongodump --host localhost:27017 --gzip --archive=/backup/dump.gz

// Restore:
// mongorestore --host localhost:27017 /backup/2026-03-17/
// mongorestore --nsInclude="mydb.orders" /backup/
// mongorestore --gzip --archive=/backup/dump.gz

// Method 2: mongoexport / mongoimport (JSON/CSV — not for full backup)
// mongoexport --db mydb --collection products --out products.json --type json
// mongoexport --db mydb --collection products --out products.csv --type csv --fields name,price,category

// Method 3: Filesystem snapshot (fastest for large databases)
// Stop writes, flush to disk, take LVM/EBS snapshot, resume writes
db.adminCommand({ fsync: 1, lock: true })
// ... take snapshot ...
db.adminCommand({ fsyncUnlock: 1 })

// Method 4: Atlas Cloud Backup (production standard)
// Automated snapshots: every 6 hours (M10), every 1 hour (M30+)
// Point-in-time restore: oplog-based, restore to any second within retention window
// Continuous oplog backup: constantly ships oplog to Atlas backup storage

// RPO/RTO comparison:
// mongodump:       RPO = time since last backup, RTO = hours (large DB)
// FS snapshot:     RPO = seconds (with journal), RTO = minutes
// Atlas Backup:    RPO = 1 second (PITR), RTO = depends on DB size
```

---

## Q12: How does MongoDB handle concurrent writes and the writeConcern at the application level?

Write concern controls the level of acknowledgment requested from MongoDB. Application-level write concern strategy balances performance and durability.

```javascript
// Write concern levels:
// w:0   → Fire and forget (no ack) — max performance, zero durability
// w:1   → Primary only acknowledges — fast, loses data on primary failure before replication
// w:majority → Majority of nodes acknowledge — durable, standard production choice
// w:N   → Exactly N nodes acknowledge — specific durability guarantee

// j:false (default for w:majority in some versions) → memory acknowledgment
// j:true → journal flush on all acknowledging nodes before responding

// Application-level strategy by operation type:
const db = client.db("mydb")

// User-visible action (profile update, checkout):
await db.users.updateOne(
  { _id: userId },
  { $set: { lastLogin: new Date() } },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
// Durable, must succeed, fail fast if cluster is unhealthy

// Analytics/metrics write (can tolerate some loss):
await db.analytics.insertOne(
  { event: "page_view", page: "/home", ts: new Date() },
  { writeConcern: { w: 1, j: false } }
)
// Fast, minimal overhead, acceptable to lose a few page views on crash

// Financial transaction (maximum durability):
const session = client.startSession({
  defaultTransactionOptions: {
    writeConcern: { w: "majority", j: true }
  }
})
await session.withTransaction(async () => {
  await db.accounts.updateOne(
    { userId: "user1" },
    { $inc: { balance: -100 } },
    { session }
  )
  await db.accounts.updateOne(
    { userId: "user2" },
    { $inc: { balance: 100 } },
    { session }
  )
})

// Batch import (performance-sensitive):
await db.products.insertMany(
  products,
  {
    writeConcern: { w: 1 },  // primary only for speed
    ordered: false            // continue on error, maximize parallelism
  }
)

// wtimeout: fail fast if majority not available
// Prevents application from hanging indefinitely during cluster issues:
await db.orders.insertOne(
  orderDoc,
  { writeConcern: { w: "majority", wtimeout: 10000 } }
)
// Throws MongoWriteConcernError if majority not achieved in 10 seconds
```

---

## Q13: How does MongoDB handle connection management and what is the driver connection lifecycle?

Understanding connection lifecycle is critical for avoiding connection exhaustion — one of the most common production issues.

```javascript
// MongoClient is a connection pool, NOT a single connection
// One MongoClient per process is the recommended pattern

// BAD: creating new MongoClient per request (common mistake):
app.get("/api/orders", async (req, res) => {
  const client = new MongoClient(uri)  // ← opens new pool on every request!
  await client.connect()
  const orders = await client.db("mydb").collection("orders").find().toArray()
  await client.close()  // closes pool
  res.json(orders)
})

// GOOD: singleton MongoClient:
// db.js (module initialized once):
const client = new MongoClient(uri, {
  maxPoolSize: 100,
  minPoolSize: 5,
  connectTimeoutMS: 10000,
  socketTimeoutMS: 45000,
  serverSelectionTimeoutMS: 30000
})
await client.connect()
export const db = client.db("mydb")

// routes.js:
import { db } from "./db.js"
app.get("/api/orders", async (req, res) => {
  const orders = await db.collection("orders").find().toArray()
  res.json(orders)
})

// Connection pool lifecycle:
// 1. Pool created with minPoolSize connections established immediately
// 2. On operation: lease a connection from pool
// 3. If no connection available and pool < maxPoolSize: create new connection
// 4. If pool is full: wait (waitQueueTimeoutMS) then throw TimeoutError
// 5. After operation: return connection to pool
// 6. Idle connections closed after maxIdleTimeMS

// Monitor pool health:
db.adminCommand({ serverStatus: 1 }).connections
// {
//   "current": 45,      ← active connections
//   "available": 55,    ← available slot in pool
//   "totalCreated": 120 ← total connections ever created (includes closed)
// }

// Connection exhaustion symptoms:
// - Timeout errors increasing
// - "MongoServerSelectionError: Server selection timed out"
// - P99 latency spike with concurrent requests

// Atlas connection limits by tier:
// M0 (Free): 500 connections
// M10: 1500 connections
// M30: 3000 connections
// M50: 6000 connections
// → Match maxPoolSize × app_instances to cluster connection limit
```

---

## Q14: What is MongoDB's security model? Describe authentication and authorization.

MongoDB uses role-based access control (RBAC) with fine-grained privileges. Security has multiple layers: network, authentication, and authorization.

```javascript
// Authentication mechanisms:
// 1. SCRAM-SHA-256 (default): username/password
// 2. X.509 certificates: TLS client certificates
// 3. LDAP: enterprise feature, integrates with corporate directory
// 4. Kerberos: enterprise feature

// Creating users:
// Application user (least privilege):
db.createUser({
  user: "appUser",
  pwd: "securePassword123!",
  roles: [
    { role: "readWrite", db: "mydb" }        // read+write on mydb only
  ]
})

// Read-only analytics user:
db.createUser({
  user: "analyticsUser",
  pwd: "analyticsPass!",
  roles: [
    { role: "read", db: "mydb" },            // read-only
    { role: "read", db: "reporting" }        // read reporting too
  ]
})

// Admin user (DBA):
db.getSiblingDB("admin").createUser({
  user: "dbAdmin",
  pwd: "adminPass!",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" }
  ]
})

// Built-in roles:
// read              → read any collection in DB
// readWrite         → read + write (no index management)
// dbAdmin           → db administration (no user management)
// userAdmin         → user management (no data access)
// dbOwner           → dbAdmin + userAdmin + readWrite combined
// clusterAdmin      → cluster-wide operations (rs, sh management)
// backup            → mongodump permissions
// restore           → mongorestore permissions

// Custom role (fine-grained):
db.createRole({
  role: "orderProcessor",
  privileges: [
    {
      resource: { db: "mydb", collection: "orders" },
      actions: ["find", "update"]  // only find and update on orders
    },
    {
      resource: { db: "mydb", collection: "inventory" },
      actions: ["find", "update"]  // read+update inventory
    }
  ],
  roles: []  // no inherited roles
})

// Enable authentication in mongod.conf:
// security:
//   authorization: enabled

// TLS/SSL configuration:
// net:
//   tls:
//     mode: requireTLS
//     certificateKeyFile: /etc/ssl/mongodb.pem
//     CAFile: /etc/ssl/ca.pem
```

---

## Q15: How does MongoDB handle large-scale data exports and migrations?

Data export and migration operations require careful planning to avoid impacting production systems.

```javascript
// Approach 1: mongodump for logical backup/export
// Safe for production: reads from secondary to avoid primary I/O
// mongodump --host secondary1:27017 --readPreference secondary
//           --db mydb --out /backup/ --numParallelCollections 4

// Approach 2: Atlas Live Migration
// For moving from self-managed to Atlas with minimal downtime:
// 1. Configure Atlas as migration target
// 2. Live Migration pulls data from source (no changes to source app)
// 3. Source keeps running; Atlas stays in sync via oplog
// 4. Cutover: update connection string, verify, point DNS

// Approach 3: Change streams + bulk write for online migration
async function migrateCollection(sourceDb, targetDb, collName) {
  // Phase 1: Initial copy (bulk, doesn't block source)
  const cursor = sourceDb.collection(collName).find({}, {
    readPreference: "secondary"  // don't impact primary
  })

  const BATCH_SIZE = 1000
  let batch = []
  for await (const doc of cursor) {
    batch.push(doc)
    if (batch.length >= BATCH_SIZE) {
      await targetDb.collection(collName).insertMany(batch, {
        ordered: false,
        writeConcern: { w: 1 }  // fast writes during migration
      })
      batch = []
    }
  }
  if (batch.length) await targetDb.collection(collName).insertMany(batch)

  // Phase 2: Tail change stream for changes during migration
  const changeStream = sourceDb.collection(collName).watch([], {
    resumeAfter: startToken  // from when we started Phase 1
  })
  for await (const change of changeStream) {
    if (change.operationType === "insert") {
      await targetDb.collection(collName).insertOne(change.fullDocument)
    } else if (change.operationType === "update") {
      await targetDb.collection(collName).updateOne(
        { _id: change.documentKey._id },
        [{ $set: change.updateDescription.updatedFields }]
      )
    } else if (change.operationType === "delete") {
      await targetDb.collection(collName).deleteOne({ _id: change.documentKey._id })
    }
    // Stop when target is caught up (lag < 1s)
  }
}

// Approach 4: $merge aggregation for collection transformation/migration
db.legacyOrders.aggregate([
  { $project: {
      customerId: 1,
      amount: "$total",           // rename field
      status: {
        $switch: {
          branches: [
            { case: { $eq: ["$state", 1] }, then: "pending" },
            { case: { $eq: ["$state", 2] }, then: "processing" },
            { case: { $eq: ["$state", 3] }, then: "completed" }
          ],
          default: "unknown"
        }
      },
      createdAt: { $toDate: "$created_timestamp" }  // type conversion
  }},
  { $merge: {
      into: { db: "newdb", coll: "orders" },
      on: "_id",
      whenMatched: "replace",
      whenNotMatched: "insert"
  }}
])
// Runs on the server, no data transfer to application
// Can be re-run to re-sync (idempotent with whenMatched: "replace")
```
