# Chapter 05 — Schema Design: Structured Answers

---

## Answer 1: How Do You Decide Whether to Embed or Reference?

**PREP Framework**:

**Point**: Embedding and referencing each have distinct performance profiles. The decision depends on access patterns, data volatility, and relationship cardinality — not on mimicking a relational schema.

**Reason**: MongoDB's unit of atomicity is the document. Embedding puts related data in the same document, enabling atomic reads and writes without transactions. Referencing creates normalized data that requires `$lookup` or application-level joins.

**Example**:

```javascript
// Framework: answer these questions before embedding
// 1. Is the data always accessed together?
//    YES → embed   NO → consider reference

// 2. What's the cardinality? (1:1, 1:few, 1:many, 1:squillions)
//    1:1 or 1:few (max ~100) → embed
//    1:many (bounded, up to ~1000) → consider embedding with upper bound
//    1:squillions (unbounded) → ALWAYS reference

// 3. How often does the embedded data change?
//    Rarely → embed (e.g., order's shipping address snapshot)
//    Frequently → consider reference (e.g., a user's current location)

// 4. Will you ever need the embedded data independently?
//    NO → embed    YES → reference or embed + reference

// EMBED: blog post with author subset
{
  _id: ObjectId("..."),
  title: "MongoDB Tips",
  author: {
    _id: ObjectId("..."),
    name: "Alice",           // denormalized — may become stale
    avatarUrl: "..."         // but rarely changes, acceptable
  }
}

// REFERENCE: blog post's comments (unbounded)
{
  _id: ObjectId("..."),
  title: "MongoDB Tips",
  commentsCount: 347          // computed — avoid $count on every load
}
// Comments in separate collection: { postId: ObjectId("..."), body: "..." }
```

**Pitfall**: The most common mistake is embedding unbounded arrays. Always define a maximum element count before embedding any array.

---

## Answer 2: Explain the Attribute Pattern — When and How to Use It

**Point**: The Attribute Pattern replaces a flat structure of sparse fields with a key-value array. It trades schema flexibility for indexability.

**Reason**: When documents in a collection have different sets of attributes (polymorphic products), creating individual indexes for every possible field is inefficient. A `{ k: 1, v: 1 }` compound index on the attributes array covers queries for any attribute.

**Example**:

```javascript
// Before Attribute Pattern — flat sparse fields
// Electronics document
{ _id: ObjectId("..."), ram: "16GB",  cpu: "i7",      wattage: 65,   isbn: null, pages: null }
// Book document
{ _id: ObjectId("..."), ram: null,    cpu: null,       wattage: null, isbn: "978-...", pages: 320 }

// Problems:
// - null fields waste space
// - Need index per attribute: {ram:1}, {isbn:1}, {wattage:1} ...
// - Adding new attribute type = schema change

// After Attribute Pattern — key-value array
{
  _id: ObjectId("..."),
  type: "electronics",
  attributes: [
    { k: "ram",     v: "16GB" },
    { k: "cpu",     v: "Intel i7" },
    { k: "wattage", v: 65,    unit: "W" }
  ]
}

{
  _id: ObjectId("..."),
  type: "book",
  attributes: [
    { k: "isbn",  v: "978-0-06-112008-4" },
    { k: "pages", v: 320 },
    { k: "genre", v: "fantasy" }
  ]
}

// ONE compound index handles all attribute queries
db.products.createIndex({ "attributes.k": 1, "attributes.v": 1 })

// Query: find products with 16GB RAM
db.products.find({ attributes: { $elemMatch: { k: "ram", v: "16GB" } } })

// Tradeoff: you lose native BSON type enforcement on values (all values become mixed)
// Solution: add a type field to the attribute entry
{ k: "wattage", v: 65, vType: "number", unit: "W" }
```

---

## Answer 3: Describe the Bucket Pattern for Time-Series Data

**Point**: The Bucket Pattern groups time-series measurements into time-window documents. This reduces document count, enables pre-aggregation, and dramatically improves query performance on historical data.

**Reason**: One document per sensor reading at 1Hz produces 86,400 documents per sensor per day. With 10,000 sensors, that's 864 million documents per day. Index scans across this volume are slow and memory-intensive.

**Example**:

```javascript
// Without Bucket Pattern — one doc per reading
{ sensorId: "S-001", ts: ISODate("2024-03-01T10:00:01Z"), temp: 22.5 }
{ sensorId: "S-001", ts: ISODate("2024-03-01T10:00:02Z"), temp: 22.6 }
// ... 86,400 docs/day/sensor × 10,000 sensors = 864M docs/day

// With Bucket Pattern — one doc per sensor per hour
{
  _id: ObjectId("..."),
  sensorId: "S-001",
  windowStart: ISODate("2024-03-01T10:00:00Z"),
  windowEnd:   ISODate("2024-03-01T11:00:00Z"),
  nReadings: 3600,

  // Computed stats (Computed Pattern layered inside Bucket Pattern)
  stats: {
    tempMin: 21.8, tempMax: 23.4,
    tempAvg: 22.525, tempSum: 81090.0
  },

  // Store anomalies only (sparse — not all 3600 readings)
  anomalies: [{ t: 1840, temp: 41.2 }]
}
// 10,000 sensors × 24 hours = 240,000 docs/day — 3,600x reduction!

// Append a reading to the current bucket
db.sensorBuckets.updateOne(
  {
    sensorId: "S-001",
    windowStart: currentHourStart,
    nReadings: { $lt: 3600 }        // bucket not full
  },
  {
    $inc: {
      nReadings: 1,
      "stats.tempSum": newTemp
    },
    $min: { "stats.tempMin": newTemp },
    $max: { "stats.tempMax": newTemp },
    $set: {
      "stats.tempAvg": { $divide: [{ $add: ["$stats.tempSum", newTemp] }, { $add: ["$nReadings", 1] }] }
    }
  },
  { upsert: true }
)

// Query: average temperature per sensor for last 24 hours
db.sensorBuckets.aggregate([
  { $match: {
      sensorId: "S-001",
      windowStart: { $gte: yesterday }
  }},
  { $group: {
      _id: null,
      avgTemp: { $avg: "$stats.tempAvg" },
      maxTemp: { $max: "$stats.tempMax" }
  }}
])
```

**Note**: For new projects on MongoDB 5.0+, prefer native time-series collections which implement bucketing automatically with better compression and column-store internals.

---

## Answer 4: What Is the Extended Reference Pattern?

**Point**: The Extended Reference Pattern embeds a frequently-accessed subset of a referenced document's fields directly into the referencing document. This avoids `$lookup` for the common case while keeping a reference for full data access.

**Reason**: A full reference requires a join to get displayable data. A full embed means stale data when the referenced document changes. The Extended Reference Pattern is a targeted compromise: embed only the stable fields needed for rendering, keep the reference for anything that needs to be current.

**Example**:

```javascript
// Problem: order page needs to show customer name and shipping address
// Both are set at order time and must NOT change if the customer later updates their profile

// Full normalization (two queries needed)
{
  _id: ObjectId("..."),
  customerId: ObjectId("..."),    // reference only
  items: [...]
}
// Requires $lookup on customers to render the order

// Full embedding (data becomes stale)
{
  _id: ObjectId("..."),
  customer: {
    _id: ObjectId("..."),
    name: "Alice Johnson",
    email: "alice@example.com",
    currentAddress: { ... }       // problem: if customer moves, order shows wrong address
  }
}

// Extended Reference Pattern — the correct approach for orders
{
  _id: ObjectId("..."),
  customer: {
    _id: ObjectId("..."),           // reference for joining when full data needed

    // Snapshot at order time — these are intentionally FROZEN
    name: "Alice Johnson",          // name as of order date
    shippingAddress: {              // address as of order date
      street: "123 Main St",
      city: "New York",
      state: "NY",
      zipCode: "10001"
    }
  },
  items: [...],
  total: NumberDecimal("59.98"),
  createdAt: ISODate("2024-03-01T00:00:00Z")
}

// Customer updates their address → order document NOT updated (correct!)
// Historical orders must show the address they were shipped to, not the current address

// For fields that SHOULD stay current (like customer tier for loyalty points):
// These belong in a separate lookup, not embedded
```

---

## Answer 5: Explain the Computed Pattern

**Point**: The Computed Pattern pre-calculates and stores derived values (averages, counts, sums) in the document itself. It trades extra write operations for dramatically faster reads.

**Reason**: If a value is read 1,000 times per second but updated once per minute, recomputing it on every read wastes CPU. Computing it once on write and caching it in the document means 1,000 cheap field reads instead of 1,000 aggregation operations.

**Example**:

```javascript
// Without Computed Pattern — calculate average rating on every product page load
db.reviews.aggregate([
  { $match: { productId: productId } },
  { $group: {
      _id: null,
      avgRating: { $avg: "$rating" },
      totalReviews: { $sum: 1 },
      distribution: {
        $push: "$rating"   // then post-process to get distribution
      }
  }}
])
// 1,000 page views/sec = 1,000 aggregation scans of reviews collection

// With Computed Pattern — stats pre-stored on product
{
  _id: ObjectId("..."),
  name: "Blue Widget",
  reviewStats: {
    totalReviews: 1247,
    averageRating: NumberDecimal("4.32"),
    distribution: { "1": 45, "2": 63, "3": 189, "4": 398, "5": 552 },
    lastUpdatedAt: ISODate("2024-03-01T09:00:00Z")
  }
}
// Read: single field access — O(1), no aggregation

// Update computed stats when new review added
db.products.updateOne(
  { _id: productId },
  {
    $inc: {
      "reviewStats.totalReviews": 1,
      [`reviewStats.distribution.${rating}`]: 1
    }
  }
)
// Then recalculate average (use background job or pipeline update)

// When is staleness acceptable?
// "Total orders count" on a dashboard: 1-minute staleness is fine
// "Current inventory level" on checkout: must be accurate (use real-time, not computed)
// "Article read count": eventual consistency is fine — use $inc and let it lag slightly
```

---

## Answer 6: Describe the Schema Versioning Pattern

**Point**: The Schema Versioning Pattern adds a `schemaVersion` field to documents, allowing the application to handle multiple document formats simultaneously and migrate data lazily over time.

**Reason**: In production, you cannot stop the database to run migrations on millions of documents. The schema versioning pattern enables zero-downtime migrations by deploying code that supports both old and new formats, then migrating documents lazily when they're next accessed.

**Example**:

```javascript
// v1 schema (original)
{ _id: ObjectId("..."), fullName: "Alice Johnson", email: "alice@example.com" }
// Note: no schemaVersion field = implicitly version 1

// v2 schema (new requirement: separate firstName/lastName)
{
  _id: ObjectId("..."),
  firstName: "Alice",
  lastName: "Johnson",
  email: "alice@example.com",
  schemaVersion: 2
}

// Application code handles both
function getDisplayName(user) {
  if ((user.schemaVersion || 1) >= 2) {
    return `${user.firstName} ${user.lastName}`
  }
  return user.fullName
}

// Lazy migration — upgrade on access
async function fetchAndMigrateUser(userId) {
  const user = await db.users.findOne({ _id: ObjectId(userId) })

  if (!user.schemaVersion || user.schemaVersion < 2) {
    const [firstName, ...rest] = (user.fullName || "").split(" ")
    const lastName = rest.join(" ")

    await db.users.updateOne(
      { _id: user._id, schemaVersion: { $exists: false } },   // idempotent guard
      {
        $set: { firstName, lastName, schemaVersion: 2 },
        $unset: { fullName: "" }
      }
    )

    user.firstName = firstName
    user.lastName = lastName
    user.schemaVersion = 2
  }

  return user
}

// Track migration progress
db.users.aggregate([
  { $group: {
      _id: { $ifNull: ["$schemaVersion", 1] },
      count: { $sum: 1 }
  }}
])
// After months of traffic, most users are on v2 naturally
// Run a batch job for the long tail of inactive users
```

---

## Answer 7: What Is the Outlier Pattern?

**Point**: The Outlier Pattern handles the "celebrity problem" — a small number of documents that vastly exceed the typical document size. These outlier documents are handled differently, while the main pattern works optimally for the majority.

**Reason**: If 99.9% of blog posts have <100 comments and 0.1% have >100,000 comments, designing your schema for the 0.1% case (no embedded comments) penalizes the 99.9% (extra queries). The Outlier Pattern flags exceptional documents and routes them to a different storage strategy.

**Example**:

```javascript
// Normal post (99.9% of posts) — embed top comments
{
  _id: ObjectId("..."),
  title: "My MongoDB Journey",
  content: "...",
  commentsCount: 23,
  pinnedComments: [
    { author: "Alice", body: "Great!" },
    // up to 20 entries
  ],
  hasOverflowComments: false    // NO overflow needed for normal posts
}

// Viral post (0.1% of posts) — Outlier Pattern
{
  _id: ObjectId("..."),
  title: "MongoDB Just Became Free!",
  content: "...",
  commentsCount: 87432,
  pinnedComments: [
    // only top 20 comments embedded (same structure as normal posts)
  ],
  hasOverflowComments: true,    // FLAG: this is an outlier
  overflowCommentsBuckets: [    // references to overflow buckets
    ObjectId("bucket_1"),
    ObjectId("bucket_2"),
    // ...
  ]
}

// Application logic
async function loadPost(postId) {
  const post = await db.posts.findOne({ _id: ObjectId(postId) })

  // Load overflow only if flagged — no overhead for normal posts
  if (post.hasOverflowComments) {
    const page = 1   // pagination
    const overflow = await db.comments.find(
      { postId: ObjectId(postId) },
      { sort: { createdAt: -1 }, limit: 20, skip: (page - 1) * 20 }
    ).toArray()
    post.additionalComments = overflow
  }

  return post
}
```

---

## Answer 8: Compare Tree Patterns in MongoDB

**Point**: MongoDB offers four tree storage patterns, each optimized for different tree operations. The choice depends on which traversal operations are most frequent.

**Example**:

```javascript
// Scenario: product category hierarchy
// Electronics → Computers → Laptops → Gaming Laptops

// PATTERN 1: Parent Reference — simple, supports upward traversal
{ _id: "gaming-laptops", name: "Gaming Laptops", parentId: "laptops" }
// Fast: find parent (one lookup), find children (one query)
// Slow: find all ancestors (N queries = N levels deep)
db.categories.find({ parentId: "laptops" })  // direct children

// PATTERN 2: Array of Ancestors — best for breadcrumbs and ancestor queries
{ _id: "gaming-laptops", name: "Gaming Laptops", parentId: "laptops",
  ancestors: ["electronics", "computers", "laptops"] }
// Fast: find ancestors (inline array, no queries), find descendants ($in on ancestors)
// Slow: inserting a new node mid-tree (must update all descendant ancestor arrays)
db.categories.find({ ancestors: "computers" })  // all descendants of computers

// PATTERN 3: Materialized Path — good for regex subtree queries
{ _id: "gaming-laptops", path: "/electronics/computers/laptops/gaming-laptops" }
// Fast: find subtree (anchored regex on path)
db.categories.find({ path: { $regex: "^/electronics/computers" } })
// Caveat: unanchored regex is slow — MUST anchor with ^ for index use

// PATTERN 4: $graphLookup — recursive traversal without redundant storage
// No ancestor arrays or paths stored — computed on query
db.categories.aggregate([
  { $match: { _id: "electronics" } },
  { $graphLookup: {
      from: "categories",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "parentId",
      as: "allDescendants",
      maxDepth: 10
  }}
])
// Fast for ad-hoc traversals, slower than array patterns for frequent queries
// Best when the tree structure changes frequently (no redundant data to maintain)

// Summary table:
// Pattern           | Find ancestors | Find descendants | Insert | Update
// Parent Reference  | Slow (N queries) | Fast           | Fast   | Fast
// Array of Ancestors| Fast (inline)    | Fast           | Fast   | Slow (update all)
// Materialized Path | Fast (regex)     | Fast           | Fast   | Slow (update all)
// $graphLookup      | Fast (computed)  | Fast (computed) | Fast   | Fast
```

---

## Answer 9: When Should You Use Separate Collections vs Embedding?

**Decision framework as a checklist**:

```javascript
// USE SEPARATE COLLECTION when:

// 1. The embedded data is unbounded
// BAD: { postId: "...", comments: [ ...potentially millions... ] }
// GOOD: separate comments collection with postId reference

// 2. The embedded data is needed independently (queried without the parent)
// BAD: embed addresses in users — if you query "all users in New York" frequently
db.users.find({ "address.city": "New York" })   // forces loading entire user document
// GOOD: separate addresses with userId reference + index on city

// 3. The embedded data is large and not always needed
// BAD: embed full product description (10KB HTML) in order line item
// GOOD: embed productId + name + price (snapshot) in order

// 4. Multiple documents reference the same data AND updates must propagate
// BAD: embed product details in 10,000 order line items — when product is discontinued,
//      must update 10,000 documents
// GOOD: reference product by ID; look up current status when needed

// USE EMBEDDING when:

// 1. Data is always accessed together (single query, single network round-trip)
{
  _id: ObjectId("..."),
  title: "Order #1234",
  items: [
    { productId: ObjectId("..."), name: "Widget", price: NumberDecimal("29.99"), qty: 2 }
  ],
  shippingAddress: { street: "123 Main St", city: "New York" },   // always needed for order display
  total: NumberDecimal("59.98")
}

// 2. Data is bounded and small (< 20 items, < 500 bytes each)
// 3. Data doesn't need to be queried independently
// 4. Atomicity of the combined data matters (order + line items must be consistent)
```

---

## Answer 10: How Do You Handle Schema Anti-Patterns in Production?

**The five major anti-patterns and their fixes**:

```javascript
// ANTI-PATTERN 1: Unbounded Arrays
// DOC: { userId: "...", followers: [id1, id2, ..., id10000000] }  // can hit 16MB
// FIX: Separate follows collection
{ followerId: ObjectId("..."), followeeId: ObjectId("..."), createdAt: new Date() }
db.follows.createIndex({ followerId: 1, followeeId: 1 }, { unique: true })
db.follows.createIndex({ followeeId: 1, createdAt: -1 })

// ANTI-PATTERN 2: Over-indexing
// 20 indexes on a collection means EVERY write (insert/update/delete)
// must update 20 B-trees — write throughput collapses
// FIX: identify unused indexes with $indexStats
db.orders.aggregate([{ $indexStats: {} }])
// Drop indexes with accesses: { ops: 0 }  (never used since server start)
db.orders.dropIndex("unused_index_name")
// General rule: max 5-10 indexes per collection, only for your hottest query patterns

// ANTI-PATTERN 3: Massive Documents (the "God Document")
// Single document with 500 fields and 5MB of data
// FIX: decompose into document + extensions
{
  _id: ObjectId("..."),
  // Core profile (always loaded)
  name: "Alice Johnson",
  email: "alice@example.com",
  tier: "premium"
}
// Extended profile in separate doc (loaded only when needed)
{
  _id: ObjectId("alice_extended"),   // predictable ID
  userId: ObjectId("alice_id"),
  preferences: { ... },             // large nested object
  fullBioHtml: "...",               // 50KB bio
  galleryImages: [ ... ]            // 100 image URLs
}

// ANTI-PATTERN 4: Unnecessary $lookup on Hot Paths
// FIX: denormalize with Extended Reference Pattern (see Answer 4)

// ANTI-PATTERN 5: Schema Designed for Data, Not Access Patterns
// Schema modeled like a relational database, normalized to 3NF
// FIX: redesign around "what queries does the application run?" not "how is data related?"
// Ask:
// 1. What are the 5 most frequent queries? Design indexes for these.
// 2. What data is ALWAYS fetched together? Embed these.
// 3. What data is RARELY needed? Reference or separate collection.
// 4. What is the read:write ratio? High read → pre-compute (Computed Pattern).
// 5. What is the largest this document will ever grow? Set explicit bounds.
```
