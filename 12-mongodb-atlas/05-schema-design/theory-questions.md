# Chapter 05 — Schema Design: Theory Questions

---

## Embedding vs Referencing

**Q1. When should you embed vs reference documents?**

The fundamental rule: **data that is accessed together should be stored together.**

```javascript
// EMBED when:
// 1. One-to-one relationship
// 2. One-to-few bounded relationship (comments per post ≤ 100)
// 3. Data accessed together in most queries (user + address)
// 4. Child data doesn't have independent lifecycle
// 5. Write atomicity needed (update user + address in one op)

// One-to-one — almost always embed
{
  _id: ObjectId("..."),
  name: "Alice",
  passport: {                    // embedded — only accessed with user
    number: "A12345678",
    expiresAt: ISODate("2030-01-01"),
    country: "US"
  }
}

// One-to-few — embed (bounded)
{
  _id: ObjectId("..."),
  username: "alice",
  phones: [                      // max 5 phones — bounded, embed
    { type: "mobile", number: "+1-512-555-0100" },
    { type: "work",   number: "+1-512-555-0200" }
  ]
}

// REFERENCE when:
// 1. One-to-many (unbounded): user → orders (could be thousands)
// 2. Many-to-many: products ↔ categories
// 3. Data has independent lifecycle: orders exist after user deletes
// 4. Embedded data would exceed 16MB
// 5. Embedded data duplicated across many docs (update anomalies)

// One-to-many — reference
// users: { _id, name }
// orders: { _id, userId (ref), total, createdAt }
```

---

**Q2. What is the Attribute Pattern and when should you use it?**

```javascript
// Problem: Products have different attributes based on type
// Naive approach creates sparse documents:
{
  _id: 1, type: "book",
  isbn: "978-...", pageCount: 400,
  wattage: null,        // always null for books
  screenSize: null,     // always null for books
  size: null            // always null for books
}

// Attribute Pattern: transform attributes into key-value pairs in an array
{
  _id: 1,
  type: "book",
  name: "MongoDB Guide",
  attributes: [
    { k: "isbn",       v: "978-1491954461" },
    { k: "pageCount",  v: NumberInt(400) },
    { k: "language",   v: "English" }
  ]
}

// For electronics:
{
  _id: 2,
  type: "electronics",
  name: "USB-C Charger",
  attributes: [
    { k: "wattage",   v: NumberInt(65) },
    { k: "inputVolt", v: "100-240V" }
  ]
}

// Index on the attributes array for efficient searching:
db.products.createIndex({ "attributes.k": 1, "attributes.v": 1 })

// Query: find all books with ISBN "978-..."
db.products.find({
  attributes: { $elemMatch: { k: "isbn", v: "978-1491954461" } }
})

// Benefits:
// - No NULL/missing fields waste
// - Single index covers all attribute types
// - Easy to add new attribute types (no schema migration!)
// - Polymorphic documents handled cleanly

// Best for: products, configurations, user preferences, metadata
```

---

**Q3. What is the Bucket Pattern?**

```javascript
// Problem: IoT sensor sends a reading every second = 86,400 docs/day per sensor
// One document per reading = massive collection growth

// Bucket Pattern: group related time-series data into time buckets

// Before (one doc per reading): 86,400 docs/day
{
  _id: ObjectId("..."),
  sensorId: "sensor_A",
  temperature: 22.5,
  ts: ISODate("2024-01-15T10:30:00Z")
}

// After (bucket per hour): 24 docs/day per sensor
{
  _id: ObjectId("..."),
  sensorId: "sensor_A",
  date: ISODate("2024-01-15T10:00:00Z"),  // hour bucket start
  count: 3600,
  sum: { temperature: 81000, humidity: 162000 },
  avg: { temperature: 22.5, humidity: 45.0 },
  min: { temperature: 21.8, humidity: 44.1 },
  max: { temperature: 23.1, humidity: 46.2 },
  measurements: [
    { offset: 0,    temperature: 22.5, humidity: 45.2 },  // 10:30:00
    { offset: 1,    temperature: 22.4, humidity: 45.1 },  // 10:30:01
    // ... 3600 measurements per hour
  ]
}

// Benefits:
// - 3600x fewer documents (documents / 3600)
// - Pre-aggregated stats (avg, min, max) ready for dashboard
// - Faster range queries (fewer docs to scan)
// - More efficient index (fewer index entries)

// Insert a measurement:
db.sensorBuckets.updateOne(
  {
    sensorId: "sensor_A",
    date: new Date("2024-01-15T10:00:00Z"),
    count: { $lt: 3600 }   // don't overflow bucket
  },
  {
    $push: { measurements: { offset: secondsIntoHour, temperature, humidity } },
    $inc: { count: 1, "sum.temperature": temperature },
    $min: { "min.temperature": temperature },
    $max: { "max.temperature": temperature },
    $setOnInsert: { sensorId: "sensor_A", date: bucketStart }
  },
  { upsert: true }
)
```

---

**Q4. What is the Computed Pattern?**

```javascript
// Problem: Expensive calculations run on every read (e.g., total order value)
// If 10,000 users/second read a product's review average, you're calculating it 10K times

// Computed Pattern: pre-calculate and store expensive values
// Update the computed value whenever source data changes

// Without Computed Pattern: calculate on every read
async function getProductWithReviewStats(productId) {
  const [product, reviews] = await Promise.all([
    db.products.findOne({ _id: productId }),
    db.reviews.aggregate([
      { $match: { productId } },
      { $group: {
        _id: null,
        avgRating: { $avg: "$rating" },
        reviewCount: { $sum: 1 }
      }}
    ]).toArray()
  ])
  return { ...product, ...reviews[0] }
}
// Problem: every read triggers an aggregation on reviews

// With Computed Pattern: store computed values on the product
{
  _id: ObjectId("..."),
  name: "Widget Pro",
  price: NumberDecimal("29.99"),
  // Computed fields — updated when reviews change:
  reviewStats: {
    avgRating: 4.3,
    reviewCount: 1247,
    distribution: { 5: 600, 4: 400, 3: 150, 2: 70, 1: 27 },
    lastUpdated: ISODate("...")
  }
}

// Read is now instant — no aggregation needed
db.products.findOne({ _id: productId })

// Update computed stats when a review is added:
async function addReview(productId, rating, comment, userId) {
  await db.reviews.insertOne({ productId, rating, comment, userId, createdAt: new Date() })

  // Update computed stats (accept slight lag: run async)
  const stats = await db.reviews.aggregate([
    { $match: { productId } },
    { $group: {
      _id: null,
      avgRating: { $avg: "$rating" },
      reviewCount: { $sum: 1 }
    }}
  ]).next()

  await db.products.updateOne(
    { _id: productId },
    { $set: {
      "reviewStats.avgRating": Math.round(stats.avgRating * 10) / 10,
      "reviewStats.reviewCount": stats.reviewCount,
      "reviewStats.lastUpdated": new Date()
    }}
  )
}
```

---

**Q5. What is the Extended Reference Pattern?**

```javascript
// Problem: Avoiding $lookup for frequently accessed data
// Orders need customer name and email for invoice — requires $lookup every time

// Extended Reference Pattern: store a subset of referenced document's fields

// PURE REFERENCE (normalized — requires $lookup):
{ _id: OrderId, customerId: CustomerId, items: [...] }

// EXTENDED REFERENCE (denormalized subset):
{
  _id: ObjectId("..."),
  orderId: "ORD-001",
  customer: {
    _id: CustomerId,               // reference for full lookup if needed
    name: "Alice Johnson",         // ← copied from customer document
    email: "alice@example.com"     // ← copied from customer document
    // address is NOT copied — we only copy frequently-needed fields
  },
  items: [...],
  total: NumberDecimal("99.99")
}

// Benefits:
// - No $lookup needed for 90% of order reads (name + email on the document)
// - Still have customerId for full customer lookup when needed

// Challenge: data synchronization
// If Alice changes her email, we must update all her orders:
async function updateCustomerEmail(customerId, newEmail) {
  // Update customer document
  await db.customers.updateOne({ _id: customerId }, { $set: { email: newEmail } })

  // Update all orders with extended reference
  await db.orders.updateMany(
    { "customer._id": customerId },
    { $set: { "customer.email": newEmail } }
  )
}
// This is acceptable because: email rarely changes, and slightly stale email
// on old orders is often acceptable (use the archived email for those invoices)
```

---

**Q6. What is the Document Versioning Pattern?**

```javascript
// Problem: Need to track historical versions of a document
// Use case: insurance policies, contracts, pricing changes

// Document Versioning Pattern: store versions in a separate collection
// Main collection: current state only
// History collection: all previous versions

// Current insurance policy:
{
  _id: "POLICY-001",
  version: 5,
  insuredName: "Alice Johnson",
  coverage: { type: "comprehensive", limit: 1000000 },
  premium: NumberDecimal("150.00"),
  effectiveDate: ISODate("2024-06-01")
}

// Policy history collection:
{
  _id: ObjectId("..."),
  policyId: "POLICY-001",
  version: 4,
  snapshot: {
    // Full copy of policy at version 4
    insuredName: "Alice Johnson",
    coverage: { type: "basic", limit: 500000 },
    premium: NumberDecimal("100.00")
  },
  changedAt: ISODate("2024-05-01"),
  changedBy: "agent_bob",
  changeReason: "Coverage upgrade"
}

// When policy is updated:
async function updatePolicy(policyId, changes, changedBy, reason) {
  const session = client.startSession()
  await session.withTransaction(async () => {
    // Get current version
    const current = await db.policies.findOne({ _id: policyId }, { session })

    // Archive current version to history
    await db.policyHistory.insertOne({
      policyId,
      version: current.version,
      snapshot: { ...current },
      changedAt: new Date(),
      changedBy,
      changeReason: reason
    }, { session })

    // Update to new version
    await db.policies.updateOne(
      { _id: policyId },
      { $set: { ...changes, version: current.version + 1 } },
      { session }
    )
  })
}
```

---

**Q7. What is the Outlier Pattern?**

```javascript
// Problem: Most documents are normal size, but a FEW are enormous
// Example: Most blog posts have <100 comments, but viral posts have 100,000 comments

// Outlier Pattern: handle the "celebrity problem"
// Regular posts: embed comments (fast, no $lookup)
// Outlier posts: reference overflow comments

// Regular post document:
{
  _id: ObjectId("..."),
  title: "MongoDB Tips",
  content: "...",
  hasCommentOverflow: false,      // flag for outlier handling
  comments: [
    { author: "Bob", text: "Great!", ts: ISODate("...") },
    // up to ~100 comments embedded
  ],
  commentCount: 3
}

// Viral post with overflow:
{
  _id: ObjectId("..."),
  title: "This Post Went Viral",
  content: "...",
  hasCommentOverflow: true,        // ← flag: some comments are in overflow collection
  comments: [
    // Most recent 100 comments embedded here
  ],
  commentCount: 45000              // actual total
}

// Overflow collection for viral posts:
{
  _id: ObjectId("..."),
  postId: ObjectId("viral_post_id"),
  comments: [
    { author: "Alice", text: "Amazing!", ts: ISODate("...") },
    // batch of overflow comments
  ]
}

// Application logic:
async function getPostComments(postId, page = 1) {
  const post = await db.posts.findOne({ _id: postId })

  if (!post.hasCommentOverflow) {
    return post.comments  // all embedded, fast
  }

  // Has overflow: paginate from overflow collection
  return db.overflowComments.find({ postId })
    .sort({ ts: -1 })
    .skip((page - 1) * 20)
    .limit(20)
    .toArray()
}
```

---

**Q8. What is the Schema Versioning Pattern?**

```javascript
// Problem: Schema evolves over time, but you can't update 50M documents atomically
// Different documents may have different schema versions simultaneously

// Schema Versioning Pattern: add a schemaVersion field, handle in app code

// Version 1 (original): full name as single string
{
  _id: ObjectId("..."),
  schemaVersion: 1,
  name: "Alice Johnson"
}

// Version 2 (new): split into first/last name
{
  _id: ObjectId("..."),
  schemaVersion: 2,
  firstName: "Alice",
  lastName: "Johnson"
}

// Version 3 (new): add displayName computed field
{
  _id: ObjectId("..."),
  schemaVersion: 3,
  firstName: "Alice",
  lastName: "Johnson",
  displayName: "Alice J."
}

// Application handles all versions:
function getUserName(user) {
  switch (user.schemaVersion) {
    case 1: return user.name
    case 2: return `${user.firstName} ${user.lastName}`
    case 3: return user.displayName
    default: return user.displayName || `${user.firstName} ${user.lastName}` || user.name
  }
}

// Migration: lazy migration (migrate on read/write)
async function getUser(userId) {
  const user = await db.users.findOne({ _id: userId })

  if (user.schemaVersion < 3) {
    // Upgrade this user's document on read
    const migrated = migrateUser(user)
    await db.users.replaceOne({ _id: userId }, migrated)
    return migrated
  }

  return user
}

// Background batch migration (run gradually):
async function batchMigrateUsers() {
  const oldUsers = await db.users.find({ schemaVersion: { $lt: 3 } }).limit(1000).toArray()
  for (const user of oldUsers) {
    await db.users.replaceOne({ _id: user._id }, migrateUser(user))
  }
}
```

---

**Q9. What is the Polymorphic Pattern?**

```javascript
// Problem: Different types of objects share most fields but have unique fields
// One collection for all types (vs. one collection per type)

// Polymorphic Pattern: single collection, discriminator field (type)
db.vehicles.insertMany([
  {
    _id: ObjectId("..."),
    type: "car",
    make: "Toyota",
    model: "Camry",
    year: 2022,
    numDoors: 4,           // car-specific
    transmissionType: "automatic"
  },
  {
    _id: ObjectId("..."),
    type: "motorcycle",
    make: "Harley-Davidson",
    model: "Sportster",
    year: 2021,
    hasSidecar: false,     // motorcycle-specific
    cc: 1200               // engine displacement
  },
  {
    _id: ObjectId("..."),
    type: "truck",
    make: "Ford",
    model: "F-150",
    year: 2023,
    payloadCapacityLbs: 1500,  // truck-specific
    bedLengthFt: 6.5
  }
])

// Query all vehicles by a shared field (works across types):
db.vehicles.find({ make: "Toyota" })
db.vehicles.find({ year: { $gte: 2020 } })

// Query type-specific fields:
db.vehicles.find({ type: "truck", payloadCapacityLbs: { $gte: 1000 } })

// Index shared fields normally:
db.vehicles.createIndex({ make: 1, model: 1 })
db.vehicles.createIndex({ type: 1 })

// Application-layer polymorphism:
function createVehicleInstance(vehicleDoc) {
  switch (vehicleDoc.type) {
    case "car":        return new Car(vehicleDoc)
    case "motorcycle": return new Motorcycle(vehicleDoc)
    case "truck":      return new Truck(vehicleDoc)
    default:           return new Vehicle(vehicleDoc)
  }
}
```

---

**Q10. What are Tree Patterns in MongoDB?**

```javascript
// For hierarchical/tree data (categories, org charts, file systems)
// Multiple patterns available — choose based on query needs

// === PATTERN 1: Parent Reference ===
// Simple, but requires multiple queries for subtree
db.categories.insertMany([
  { _id: "sports",     name: "Sports",     parentId: null },
  { _id: "outdoor",    name: "Outdoor",    parentId: "sports" },
  { _id: "camping",    name: "Camping",    parentId: "outdoor" },
  { _id: "tents",      name: "Tents",      parentId: "camping" }
])
// Fast: find children of a node
// Slow: find all ancestors (multiple queries)

// === PATTERN 2: Child Reference ===
// Embed children IDs in parent
{ _id: "sports", children: ["outdoor", "beach", "indoor"] }
// Fast: find children (no join)
// Slow: find path to root (traverse up)

// === PATTERN 3: Array of Ancestors ===
// Store full path from root to each node
{
  _id: "tents",
  name: "Tents",
  ancestors: ["sports", "outdoor", "camping"],  // full path
  parent: "camping"
}
// Fast: find ancestors (already in array)
// Fast: find all descendants with $in on ancestors
db.categories.find({ ancestors: "outdoor" })  // all items under outdoor
// Slow: moving subtrees (must update all descendants)

// === PATTERN 4: Materialized Paths ===
// Store path as a string
{ _id: "tents", path: "sports,outdoor,camping,tents" }
// Index: db.categories.createIndex({ path: 1 })
// Fast: find descendants (regex prefix)
db.categories.find({ path: /^sports,outdoor/ })
// Compact, human-readable

// === PATTERN 5: $graphLookup (Recursive) ===
db.employees.aggregate([
  { $match: { name: "CEO" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "managerId",
      as: "allReports",
      maxDepth: 10
    }
  }
])
// Works with parent reference pattern, handles arbitrary depth
```

---

**Q11. What is the One-to-Squillions Pattern?**

```javascript
// Problem: Unbounded one-to-many (one user → millions of log entries)
// Cannot embed (would exceed 16MB) or even store all references in an array

// One-to-Squillions: reference the parent from the child
// The "child" document has a reference to the "parent", not vice versa

// BAD: storing all log IDs in the user document
{
  _id: userId,
  logIds: [ObjectId("log1"), ObjectId("log2"), ... // millions!]
  // 16MB limit hit with ~500K ObjectIds
}

// GOOD: reference parent from child (reverse reference)
// User document:
{
  _id: userId,
  name: "Alice"
  // No log references here — the other side holds the reference
}

// Log document:
{
  _id: ObjectId("..."),
  userId: userId,          // reference to parent
  level: "ERROR",
  message: "Database connection failed",
  ts: ISODate("...")
}

// Index on userId for fast per-user log retrieval:
db.logs.createIndex({ userId: 1, ts: -1 })

// Queries:
// Get user's recent logs:
db.logs.find({ userId }).sort({ ts: -1 }).limit(50)

// Get user with their recent logs ($lookup with limit):
db.users.aggregate([
  { $match: { _id: userId } },
  {
    $lookup: {
      from: "logs",
      let: { uid: "$_id" },
      pipeline: [
        { $match: { $expr: { $eq: ["$userId", "$$uid"] } } },
        { $sort: { ts: -1 } },
        { $limit: 10 }
      ],
      as: "recentLogs"
    }
  }
])
```

---

**Q12. What is the Normalization vs Denormalization trade-off in MongoDB?**

```javascript
// NORMALIZATION (like RDBMS): minimize data duplication, references only
// Pros: updates to shared data affect one place
// Cons: requires $lookup for related data access

// DENORMALIZATION: embed duplicate data
// Pros: faster reads (all data in one doc)
// Cons: update anomalies (must update copies everywhere)

// MongoDB approach: denormalize by DEFAULT, normalize exceptions

// Denormalization example: order with embedded product snapshot
{
  _id: OrderId,
  orderDate: ISODate("..."),
  items: [
    {
      // Snapshot of product at time of order
      productId: ObjectId("..."),
      name: "Widget Pro",           // denormalized — product name at order time
      price: NumberDecimal("29.99"), // denormalized — price at order time
      qty: 2
    }
  ]
}
// IMPORTANT: Product price/name can change, but the ORDER should preserve
// the price the customer actually paid. Denormalization is CORRECT here!
// This is a case where you WANT the data to be "stale" relative to current product

// Normalization case: blog posts → categories
// Many posts can belong to the same category
// Category name might change
// Embed category name on every post = update all posts when category renamed
// Reference: update one category document, all posts automatically reflect change

db.posts.createIndex({ categoryId: 1 })
// Reference is better here: low query overhead (one $lookup), easy updates
```

---

**Q13. What is schema validation and what are its limitations?**

```javascript
// JSON Schema validation enforces document structure at write time
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email"],
      properties: {
        name: { bsonType: "string", minLength: 1 },
        email: {
          bsonType: "string",
          pattern: "^[\\w._%+-]+@[\\w.-]+\\.[a-zA-Z]{2,}$"
        },
        age: { bsonType: ["int", "null"], minimum: 0, maximum: 150 },
        role: { enum: ["user", "admin", "moderator"] }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})

// LIMITATIONS:
// 1. Validation only runs on INSERT and UPDATE — NOT on existing documents
//    Old documents that violate the schema stay in the collection

// 2. validationLevel: "moderate" — doesn't validate updates to documents
//    that ALREADY violate the schema

// 3. bypassDocumentValidation: true — admin operations can bypass validation
//    db.runCommand({ insert: "users", documents: [...], bypassDocumentValidation: true })

// 4. Performance overhead — each write must validate against the schema
//    For complex schemas with many fields: small but measurable overhead

// 5. Cannot enforce relational constraints (e.g., "userId must exist in users collection")
//    Only validates the document structure itself

// 6. Regex in patterns uses PCRE2 syntax — slightly different from JS regex
//    Test all patterns thoroughly before deploying

// 7. Dynamic/polymorphic schemas require careful design:
//    Use discriminator + anyOf to handle type-specific validation
```

---

**Q14. What are anti-patterns in MongoDB schema design?**

```javascript
// ANTI-PATTERN 1: Unbounded array growth
// BAD: storing all user activity in an embedded array
{
  userId: ObjectId("..."),
  activityLog: [
    // This grows indefinitely — will hit 16MB!
    { action: "login", ts: ISODate("...") },
    { action: "view_product", ts: ISODate("...") },
    // ... millions of entries
  ]
}
// FIX: separate collection with reference

// ANTI-PATTERN 2: Massive number of collections
// BAD: one collection per user/tenant at scale
// db["user_alice_orders"], db["user_bob_orders"], ...
// 1M users = 1M collections → terrible performance, hard to manage
// FIX: shared collection with tenantId/userId field + index

// ANTI-PATTERN 3: Large, growing documents
// BAD: appending to an array indefinitely
db.users.updateOne({ _id: id }, { $push: { sessionTokens: newToken } })
// After 1 year: thousands of tokens in the array
// FIX: separate sessions collection + TTL index

// ANTI-PATTERN 4: Unnecessary $lookup (denormalize more)
// BAD: normalizing every related entity and using $lookup for every read
db.orders.aggregate([
  { $lookup: { from: "customers", ... } },
  { $lookup: { from: "products", ... } },
  { $lookup: { from: "shippers", ... } },
  // 3 $lookups for every order read
])
// FIX: embed frequently co-accessed data (customer name, product name at order time)

// ANTI-PATTERN 5: Non-indexed sort fields
// BAD: db.posts.find({}).sort({ likeCount: -1 })
// Without index, sorts 10M posts in memory (if it doesn't fail)
// FIX: db.posts.createIndex({ likeCount: -1 })

// ANTI-PATTERN 6: Over-indexing
// BAD: creating an index for every possible query
// 20 indexes on a write-heavy collection → writes are 20x slower
// FIX: index only the top 5 most common queries

// ANTI-PATTERN 7: Using _id as a "foreign key" without understanding type
// BAD: storing ObjectId as a string in references
{ orderId: "64f1a2b3..." }  // string, not ObjectId
// FIX: always store references as ObjectId type
{ orderId: ObjectId("64f1a2b3...") }
```

---

**Q15. What are the key data access pattern questions to ask before designing a schema?**

Before designing a MongoDB schema, ask:

1. **What are the primary entities?** (Users, Orders, Products)
2. **What are the relationships?** (1:1, 1:N, N:M)
3. **What are the most common read queries?** (Product page, user feed, order history)
4. **What is the read:write ratio?** (Read-heavy → optimize for reads → embed more)
5. **How much data volume?** (Unbounded arrays need referencing)
6. **Do entities have independent lifecycles?** (Orders outlive users → reference)
7. **What consistency is required?** (Financial: transactional → be careful with denormalization)
8. **What are the performance requirements?** (<10ms read = may need covered queries)
9. **How frequently do referenced entities change?** (Frequently changing = consider reference to avoid update anomalies)
10. **Will data be accessed from multiple contexts?** (Orders accessed from order page AND user profile AND admin = may need to normalize)

```javascript
// Design worksheet for a new schema:
const designDecisions = {
  entity: "Order",
  relationships: {
    customer: "1:N (reference — customers are shared, orders outlive accounts)",
    items: "1:few (embed — accessed together with order, bounded by ~50)",
    shipping: "1:1 (embed — shipping address is order-specific snapshot)"
  },
  primaryQueries: [
    "Get order by ID: O(log n) with _id index",
    "Get all orders for customer: need { customerId: 1, createdAt: -1 } index",
    "Get pending orders: need { status: 1, createdAt: -1 } index"
  ],
  readWriteRatio: "80:20 — read-heavy → embed more",
  volumeConsiderations: "Average 10 items per order — bounded, embed is safe"
}
```

---

**Q16. What is the difference between the schema patterns for 1:1, 1:N, and N:M relationships?**

```javascript
// 1:1 Relationship: User → Passport
// Always embed (shared lifecycle, always accessed together)
{
  _id: userId,
  name: "Alice",
  passport: { number: "A12345678", expiresAt: ISODate("2030-01-01") }
}

// 1:N (few): User → Addresses (max 5-10)
// Embed bounded array
{
  _id: userId,
  name: "Alice",
  addresses: [
    { type: "home", street: "123 Main", city: "Austin" },
    { type: "work", street: "456 Oak", city: "Austin" }
  ]
}

// 1:N (many): User → Orders
// Reference (orders are separate entities, many per user, independent lifecycle)
// users: { _id, name, email }
// orders: { _id, userId, total, createdAt }
// db.orders.createIndex({ userId: 1, createdAt: -1 })

// 1:N (squillions): User → Events
// Child-side reference only (parent doesn't know about children)
// users: { _id, name }
// events: { _id, userId, type, ts }  // userId is the reference

// N:M: Products ↔ Categories
// Option A: store array of references on ONE side
{
  _id: productId,
  name: "Widget",
  categoryIds: [ObjectId("electronics"), ObjectId("gadgets")]
}
db.products.createIndex({ categoryIds: 1 })

// Option B: junction collection (when relationship has its own attributes)
// product_categories: { productId, categoryId, addedAt, addedBy }
db.productCategories.createIndex({ productId: 1 })
db.productCategories.createIndex({ categoryId: 1 })

// When to use junction collection:
// - The relationship itself has attributes (addedAt, weight, etc.)
// - You need efficient queries from BOTH sides
// - Very high fan-out (many categories, many products each)
```
