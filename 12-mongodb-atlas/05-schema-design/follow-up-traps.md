# Chapter 05 — Schema Design: Follow-Up Traps

---

## Trap 1: Embedding Arrays Without an Upper Bound

**The interviewer asks**: "You said you'd embed comments in the post document. How many comments can a post have?"

**Wrong answer**: "It depends on how popular the post is — the array will just grow."

**Why it's wrong**: An unbounded embedded array will eventually hit the 16MB document size limit. At ~200 bytes per comment, you hit the limit around 83,000 comments. Beyond that: writes fail with `BSONObjectTooLarge`. Even before the limit, a 10MB document that gets fetched for every page load wastes RAM and network bandwidth.

**Correct answer**:

```javascript
// WRONG — unbounded array
{
  _id: ObjectId("..."),
  title: "My Post",
  comments: [
    // could grow to 100,000+ entries → document > 16MB → ERROR
    { author: "Alice", body: "Great post!" },
    { author: "Bob",   body: "Agreed!" },
    // ... 99,998 more entries
  ]
}

// RIGHT — Outlier Pattern with bounded embed
{
  _id: ObjectId("..."),
  title: "My Post",
  // Embed ONLY the first 20 comments for fast initial load
  pinnedComments: [
    { author: "Alice", body: "Great post!", likes: 45 }
    // max 20 entries enforced by application layer
  ],
  hasMoreComments: true,        // flag: load overflow from separate collection
  totalCommentsCount: 15420     // pre-computed
}

// Overflow goes to a separate comments collection — unbounded there is fine
// because individual comment docs are tiny

// Rule: always know the maximum size of any embedded array at design time
// If max size is unknown or unbounded → use a reference (separate collection)
```

**The bound rule**: Before embedding any array, answer: "What is the maximum number of elements this array will ever have?" If the answer is "unknown" or "could be millions," use a reference pattern.

---

## Trap 2: "Two-Phase Commit Is How You Handle Multi-Document Atomicity"

**The interviewer asks**: "How do you update two documents atomically in MongoDB?"

**Wrong answer**: "You implement a two-phase commit pattern — mark documents with a 'pending' state, apply changes, then mark as 'committed.' If anything fails, you roll back by checking for pending states."

**Why it's wrong**: Two-phase commit (2PC) was a manual workaround before MongoDB 4.0. Since MongoDB 4.0, multi-document ACID transactions are natively supported. Two-phase commit is now an anti-pattern — it's complex, error-prone, and slower than native transactions.

**Correct answer**:

```javascript
// WRONG — manual two-phase commit (pre-4.0 workaround)
// Phase 1: Apply pending state
db.accounts.updateOne({ _id: aliceId }, { $inc: { balance: -100 }, $set: { pending: txnId } })
db.accounts.updateOne({ _id: bobId },   { $inc: { balance: +100 }, $set: { pending: txnId } })
// Phase 2: Confirm
db.accounts.updateMany({ pending: txnId }, { $unset: { pending: "" } })
// If failure between phases → inconsistent state → recovery job needed
// This is complex, fragile, and NOT recommended

// CORRECT — native ACID transaction (MongoDB 4.0+)
const session = client.startSession()

try {
  await session.withTransaction(async () => {
    const sender = await db.accounts.findOneAndUpdate(
      { _id: aliceId, balance: { $gte: 100 } },
      { $inc: { balance: -100 } },
      { session, returnDocument: "after" }
    )
    if (!sender) throw new Error("Insufficient funds")

    await db.accounts.updateOne(
      { _id: bobId },
      { $inc: { balance: 100 } },
      { session }
    )
    // If ANY operation fails, the entire transaction rolls back automatically
  })
} finally {
  await session.endSession()
}

// Note: transactions have overhead. Single-document operations are ALWAYS atomic
// without a transaction. Use transactions only when you genuinely need multi-document atomicity.
```

**Key fact**: MongoDB 4.0 introduced multi-document transactions for replica sets; MongoDB 4.2 extended them to sharded clusters. Two-phase commit should only appear if you're asked about pre-4.0 MongoDB or legacy codebase maintenance.

---

## Trap 3: "Normalization Is Always Best for Data Integrity"

**The interviewer asks**: "Doesn't denormalization cause data integrity problems? Isn't normalization safer?"

**Wrong answer**: "Yes, normalization is always better for data integrity — you should normalize MongoDB schemas just like SQL."

**Why it's wrong**: MongoDB is designed for document-oriented access patterns. Strict normalization in MongoDB means frequent `$lookup` operations (which are expensive — no query planner optimization, no hash joins) and multiple round-trips. MongoDB's atomicity at the document level means a single well-designed embedded document can replace a multi-table transaction.

**Correct answer**:

```javascript
// NORMALIZED (wrong for MongoDB in most cases)
// orders collection: { _id, customerId, items: [{ productId, qty }] }
// customers collection: { _id, name, email, address }
// products collection: { _id, name, price }

// To render an order receipt, you need 3 $lookup stages:
db.orders.aggregate([
  { $match: { _id: orderId } },
  { $lookup: { from: "customers", localField: "customerId", foreignField: "_id", as: "customer" } },
  { $lookup: { from: "products", localField: "items.productId", foreignField: "_id", as: "products" } }
])
// Each $lookup is a nested loop join — O(n) per stage — slow for large collections

// DENORMALIZED (correct for MongoDB — Extended Reference Pattern)
{
  _id: ObjectId("..."),
  orderNumber: "ORD-2024-001",

  // Customer snapshot at time of order — immutable after placement
  customer: {
    _id: ObjectId("..."),
    name: "Alice Johnson",          // denormalized — stored at order time
    email: "alice@example.com",
    shippingAddress: { ... }        // address as of order date (may change later)
  },

  items: [
    {
      product: {
        _id: ObjectId("..."),
        name: "Blue Widget",        // denormalized — name at time of purchase
        sku: "WIDGET-A-001"
      },
      price: NumberDecimal("29.99"),  // price at time of purchase (historical record)
      quantity: 2
    }
  ],

  total: NumberDecimal("59.98"),
  createdAt: ISODate("2024-03-01T00:00:00Z")
}
// ONE document fetch = complete order receipt. No joins needed.
// Customer name change doesn't affect historical orders (correct behavior!)

// When to normalize (use references):
// 1. Data changes frequently AND the latest value is always needed
// 2. Data is very large and not always needed
// 3. The relationship is truly N:M and you query from both sides equally
// 4. Strict uniqueness constraints are critical (e.g., a user's email)
```

---

## Trap 4: "$lookup Is Just Like a SQL JOIN — Use It Freely"

**The interviewer asks**: "If I need to join products and categories, I'll just use $lookup — that's MongoDB's JOIN, right?"

**Wrong answer**: "Yes, $lookup is equivalent to SQL JOIN and works just as efficiently."

**Why it's wrong**: `$lookup` is a nested-loop join in MongoDB's aggregation pipeline. There is no hash join, merge join, or optimizer magic. For each document in the left collection, MongoDB executes a query against the right collection. This is O(n × m) in the worst case. Unlike SQL's optimizer, MongoDB cannot reorder `$lookup` stages.

**Correct answer**:

```javascript
// $lookup behavior: for EACH order document, run a query on customers
db.orders.aggregate([
  { $match: { status: "pending" } },    // returns 50,000 orders
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
  }}
  // For each of the 50,000 orders → runs 1 query on customers collection
  // = 50,000 index lookups on customers (acceptable with index)
  // = 50,000 full scans if no index on customers._id (catastrophic — but _id is always indexed)
])

// The real problem: $lookup in sharded collections
// Cross-shard $lookup is NOT supported in transactions
// $lookup is only allowed if the joined collection is unsharded OR on the same shard
db.orders.aggregate([
  { $lookup: { from: "customers", ... } }   // ERROR if customers is on different shard
])

// Better alternatives to $lookup:
// 1. Embed the data you need (Extended Reference Pattern)
{
  _id: ObjectId("..."),
  customer: { _id: ObjectId("..."), name: "Alice", email: "alice@example.com" }
  // No $lookup needed — customer data is right here
}

// 2. Application-level join (for flexibility)
const orders = await db.orders.find({ status: "pending" }).toArray()
const customerIds = [...new Set(orders.map(o => o.customerId.toString()))]
const customers = await db.customers.find({ _id: { $in: customerIds.map(id => ObjectId(id)) } }).toArray()
const customerMap = new Map(customers.map(c => [c._id.toString(), c]))
const enriched = orders.map(o => ({ ...o, customer: customerMap.get(o.customerId.toString()) }))
// Two queries instead of N+1 queries
```

**Rule**: Use `$lookup` for low-cardinality, infrequent joins. For hot query paths, denormalize instead.

---

## Trap 5: "Schema Validation Protects All My Existing Data"

**The interviewer asks**: "If I add schema validation to my collection, will it catch bad data that's already in there?"

**Wrong answer**: "Yes — MongoDB schema validation checks all documents, including existing ones."

**Why it's wrong**: Schema validation in MongoDB is forward-looking only. It applies to **insertions and updates**. Documents that already exist in the collection when validation is added are **not validated** and are **not rejected**. They remain as-is.

**Correct answer**:

```javascript
// Add validation to an existing collection
db.runCommand({
  collMod: "users",
  validator: {
    $jsonSchema: {
      required: ["email", "username"],
      properties: {
        email:    { bsonType: "string", pattern: "^[^@]+@[^@]+\\.[^@]+$" },
        username: { bsonType: "string", minLength: 3 }
      }
    }
  },
  validationLevel: "moderate",    // ONLY validates new inserts + updates to valid docs
  validationAction: "error"
})

// validationLevel options:
// "strict"   — validates ALL inserts AND updates (even to existing invalid docs)
// "moderate" — validates inserts, and updates ONLY if the document being updated
//              currently passes validation (invalid existing docs can still be updated)
// "off"      — no validation

// Existing documents in the collection:
// { _id: 1, email: "not-an-email", username: "x" }   ← STILL IN COLLECTION after validation added
// This doc will NOT be flagged or removed

// To find and fix invalid existing documents:
db.users.aggregate([
  { $match: {
      $or: [
        { email: { $not: { $regex: "^[^@]+@[^@]+\\.[^@]+" } } },
        { email: { $exists: false } },
        { username: { $exists: false } }
      ]
  }}
]).forEach(doc => {
  console.log(`Invalid doc: ${doc._id}`)
  // Fix or flag these manually
})

// The right approach: when adding validation to a production collection:
// 1. Add with validationLevel: "off" first
// 2. Run a query to find and fix all invalid documents
// 3. Switch to validationLevel: "moderate" then "strict"
```

---

## Trap 6: "The Bucket Pattern Stores Raw Data"

**The interviewer asks**: "For IoT sensor data, you said use the Bucket Pattern. So each bucket document contains all the raw readings for that hour?"

**Wrong answer**: "Yes — all 3,600 readings per hour are stored in a readings array in each bucket document."

**Why it's wrong**: At 50 bytes per reading × 3,600 readings per hour = 180KB per document. With 10,000 sensors, that's 1.8GB per hour of documents. More critically, the document size limit is 16MB, and documents this large are slow to load from disk, strain the WiredTiger cache, and cause significant write amplification on every append.

**Correct answer**:

```javascript
// WRONG — storing all raw readings in the array
{
  sensorId: "SENSOR-001",
  hour: ISODate("2024-03-01T10:00:00Z"),
  readings: [
    { t: 0, temp: 22.5, hum: 65.2 },
    { t: 1, temp: 22.6, hum: 65.1 },
    // ... 3,598 more entries
  ]   // Array with 3,600 elements — very large document
}

// CORRECT — store pre-computed statistics, NOT all raw readings
{
  sensorId: "SENSOR-001",
  hour: ISODate("2024-03-01T10:00:00Z"),
  nReadings: 3600,

  // Pre-aggregated stats (Computed Pattern inside Bucket Pattern)
  stats: {
    tempMin: 21.8,
    tempMax: 23.4,
    tempAvg: 22.525,
    tempSum: 81090.0,    // keep sum for accurate re-averaging if needed
    humAvg: 64.9
  },

  // Only keep anomalies or sampled readings (sparse, not all)
  anomalies: [
    { t: 1840, temp: 41.2, type: "high_temp_alert" }
  ]
}

// For MongoDB Time-Series collections (5.0+): MongoDB handles bucketing internally
// You insert individual readings → MongoDB groups them under the hood
db.createCollection("readings", {
  timeseries: { timeField: "timestamp", metaField: "sensorId", granularity: "seconds" }
})
// MongoDB's internal bucket format is compressed and optimized — you don't manage it
```

---

## Trap 7: "A Partial Index Automatically Speeds Up Any Query That Touches Those Documents"

**The interviewer asks**: "I created a partial index with `{ isActive: true }`. Will queries like `db.users.find({ email: 'alice@example.com' })` automatically use this faster index?"

**Wrong answer**: "Yes — the partial index only contains active users, so it's smaller and faster for all queries."

**Why it's wrong**: A partial index will only be used by the query planner if the query **includes the filter condition** that matches the `partialFilterExpression`. A query without `isActive: true` may return inactive users, which are not in the partial index — so MongoDB cannot use it safely.

**Correct answer**:

```javascript
// Creating the partial index
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { isActive: true } }
)

// WILL use the partial index (query includes partialFilterExpression condition)
db.users.find({ email: "alice@example.com", isActive: true })    // ✓
db.users.find({ email: "alice@example.com", isActive: { $eq: true } })  // ✓

// WILL NOT use the partial index (query may need inactive users too)
db.users.find({ email: "alice@example.com" })                    // ✗ full scan
db.users.find({ email: "alice@example.com", isActive: false })   // ✗ full scan

// MongoDB's decision logic:
// "Can I guarantee this partial index contains ALL documents matching this query?"
// If query doesn't include isActive: true → inactive users could match → partial index unsafe to use

// Real-world fix: your application layer always sends isActive: true for normal user lookups
// Admin lookups (all users) explicitly fall back to a separate full index:
db.users.createIndex({ email: 1 })   // full index for admin queries
db.users.createIndex(
  { email: 1 },
  { partialFilterExpression: { isActive: true }, name: "email_active_only" }
)  // partial index for 99% of app queries
```

---

## Trap 8: "Use $lookup to Join Referenced Documents in Hot Query Paths"

**The interviewer asks**: "Your product page loads in 2 seconds. You have `$lookup` fetching category and brand details. How would you optimize this?"

**Wrong answer**: "I'd add indexes on the foreign keys to speed up the `$lookup`."

**Why it's wrong**: Even with indexes, `$lookup` executes a query per document. For a product page that's loaded 1,000 times per second, this is 1,000 × (1 + number of $lookup stages) queries per second. This overhead is from the architecture, not from missing indexes.

**Correct answer**:

```javascript
// CURRENT (slow — $lookup on hot path)
db.products.aggregate([
  { $match: { _id: productId } },
  { $lookup: { from: "categories", localField: "categoryId", foreignField: "_id", as: "category" } },
  { $lookup: { from: "brands",     localField: "brandId",    foreignField: "_id", as: "brand" } }
])
// 3 queries for every product page load

// OPTIMIZED — Denormalize category + brand subset into product document
{
  _id: ObjectId("..."),
  name: "Blue Widget",

  // Embedded subsets (Extended Reference Pattern)
  category: {
    _id: ObjectId("..."),
    name: "Electronics",       // what the page needs
    slug: "electronics",
    iconUrl: "..."
  },
  brand: {
    _id: ObjectId("..."),
    name: "Acme Corp",
    logoUrl: "..."
  },

  price: NumberDecimal("29.99")
}
// 1 document fetch = complete page render

// How to handle category/brand name changes?
// Option A: Accept slight staleness — run nightly sync job
// Option B: On category rename, update all products (acceptable if rare)
db.products.updateMany(
  { "category._id": ObjectId(categoryId) },
  { $set: { "category.name": newName, "category.slug": newSlug } }
)
// Fast with index: db.products.createIndex({ "category._id": 1 })
```

---

## Trap 9: "The Schema Versioning Pattern Requires a Migration Job"

**The interviewer asks**: "If I add a new required field with Schema Versioning, I need to run a migration on all 100M documents before deploying, right?"

**Wrong answer**: "Yes — you need to update every document before the new code goes live."

**Why it's wrong**: The entire point of the Schema Versioning Pattern is **lazy migration** — you update documents only when they are naturally accessed (read/updated), not upfront. This avoids a massive, blocking migration job that could take days and put the database under extreme write pressure.

**Correct answer**:

```javascript
// Schema Versioning Pattern — lazy migration approach

// Step 1: Add schemaVersion to new documents going forward
db.users.insertOne({
  email: "alice@example.com",
  username: "alice",
  preferences: { theme: "dark", notifications: true },   // new field in v2
  schemaVersion: 2
})

// Step 2: Application handles BOTH versions
function getTheme(user) {
  if (user.schemaVersion >= 2) {
    return user.preferences.theme
  } else {
    // v1 users don't have preferences — return default
    return "light"
  }
}

// Step 3: Upgrade on natural access (lazy migration)
async function getUser(userId) {
  const user = await db.users.findOne({ _id: ObjectId(userId) })

  if (!user.schemaVersion || user.schemaVersion < 2) {
    // Upgrade this document in the background
    await db.users.updateOne(
      { _id: user._id, schemaVersion: { $exists: false } },  // idempotent
      {
        $set: {
          preferences: { theme: "light", notifications: true },  // safe defaults
          schemaVersion: 2
        }
      }
    )
    user.schemaVersion = 2
    user.preferences = { theme: "light", notifications: true }
  }

  return user
}

// Step 4: Track migration progress
db.users.aggregate([
  { $group: { _id: "$schemaVersion", count: { $sum: 1 } } },
  { $sort: { _id: 1 } }
])
// { _id: null, count: 87000000 }   ← still v1 (schemaVersion missing)
// { _id: 2,    count: 13000000 }   ← already upgraded
// Gradually shifts as users log in — no big bang migration needed

// Step 5 (optional): Background migration to clean up the long tail
// Run ONLY after the new code has been deployed everywhere
// Target only old-format documents during low-traffic windows
```

---

## Trap 10: "Polymorphic Collections Are Bad Because They Have Null Fields"

**The interviewer asks**: "Storing books, shoes, and electronics in one collection is messy — every document has null fields from the other types. Isn't that wasteful?"

**Wrong answer**: "Yes — each product type should have its own collection to avoid null fields."

**Why it's wrong**: In MongoDB (unlike relational databases), documents in a collection don't need to share the same structure. A book document simply doesn't have `shoeSize` — the field is absent, not stored as null. The storage overhead for absent fields is zero. Multiple collections add complexity without benefit when the query patterns are the same ("get product by ID", "search products by category").

**Correct answer**:

```javascript
// WRONG — separate collection per type (over-normalized for MongoDB)
// books collection, shoes collection, electronics collection
// Problem: "get product by ID" requires knowing which collection to query
// Problem: "search all products" requires a union query across 3 collections
// Problem: shared logic (pricing, inventory) must be applied to each collection

// CORRECT — Polymorphic Pattern — one collection, type discriminator field
// Book document
{
  _id: ObjectId("..."),
  type: "book",               // discriminator field
  name: "The Hobbit",
  price: NumberDecimal("14.99"),
  isbn: "978-0-06-112008-4", // book-specific
  author: "J.R.R. Tolkien",  // book-specific
  pages: 310,                 // book-specific
  genre: "fantasy"            // book-specific
  // NO shoeSize, NO wattage — these fields simply don't exist in this document
}

// Shoe document — absent fields cost NOTHING
{
  _id: ObjectId("..."),
  type: "shoe",
  name: "Running Pro X3",
  price: NumberDecimal("89.99"),
  shoeSize: "10",             // shoe-specific
  color: "blue",              // shoe-specific
  material: "mesh"            // shoe-specific
  // NO isbn, NO pages — simply absent
}

// Shared queries work across all types
db.products.find({ type: "book",   price: { $lt: 20 } })   // just books
db.products.find({                 price: { $lt: 20 } })   // all cheap products
db.products.find({ _id: productId })                        // any product by ID

// Schema validation per type (MongoDB 4.0+)
// Use $switch inside $jsonSchema to validate type-specific required fields
// Or: use separate validators for each type checked at application layer
```

---

## Trap 11: "Always Use ObjectId for _id"

**The interviewer asks**: "What type should you use for `_id`?"

**Wrong answer**: "Always use ObjectId — that's MongoDB's _id type."

**Why it's wrong**: ObjectId is the default and convenient, but custom `_id` values are perfectly valid and often superior. For entities that already have a natural unique identifier (like a product SKU, a username, an ISO country code, or a UUID from another system), using that as `_id` eliminates the need for a separate unique index and simplifies lookups.

**Correct answer**:

```javascript
// Custom _id examples — perfectly valid and often better than ObjectId

// Country lookup table — ISO code is naturally unique
{ _id: "US", name: "United States", currency: "USD", continent: "NA" }
{ _id: "DE", name: "Germany",       currency: "EUR", continent: "EU" }
db.countries.findOne({ _id: "US" })   // O(1) with _id — no separate index needed

// Configuration document — singleton, use a meaningful key
{ _id: "app_config", maintenanceMode: false, maxUploadMB: 10, featureFlags: { ... } }
db.config.findOne({ _id: "app_config" })

// User with natural unique username
{ _id: "alice_johnson", email: "alice@example.com", ... }
db.users.findOne({ _id: "alice_johnson" })
// No need for a separate unique index on username

// UUID from another system (for portability / cross-system joins)
{ _id: "550e8400-e29b-41d4-a716-446655440000", externalSystem: "Salesforce", ... }

// When to stick with ObjectId:
// 1. You need to infer creation time from _id (ObjectId has timestamp)
// 2. You need globally unique IDs across distributed systems
// 3. You have no natural unique identifier for the entity
// 4. The entity is ephemeral (log entries, events) — auto-generated is fine

// WARNING: Don't try to change _id after document creation
// _id is immutable — requires delete + insert (or with a transaction):
await db.users.insertOne({ ...oldDoc, _id: newId })
await db.users.deleteOne({ _id: oldId })
```

---

## Trap 12: "Aggregation $out Writes Atomically to the Target Collection"

**The interviewer asks**: "I use `$out` at the end of my aggregation pipeline to write results to a `reports` collection. Is this safe if multiple aggregations run at once?"

**Wrong answer**: "`$out` writes documents one by one, so concurrent writes can intermix results from different runs."

**Why it's actually nuanced**:

```javascript
// $out behavior — atomic at collection level
db.orders.aggregate([
  { $match: { status: "completed", month: "2024-03" } },
  { $group: { _id: "$customerId", totalSpent: { $sum: "$total" } } },
  { $out: "monthlyReport" }    // ← atomic collection replacement
])

// $out REPLACES the entire target collection atomically.
// It:
// 1. Writes results to a TEMP collection
// 2. Atomically renames the temp collection to the target name
// 3. If the pipeline fails mid-way, the original collection is untouched
// → Safe for concurrent reads during the aggregation (readers see old data until swap)
// → NOT safe for concurrent WRITES (two $out pipelines writing to same target = race condition)

// $merge (MongoDB 4.2+) is more flexible but NOT atomic at collection level
db.orders.aggregate([
  { $group: { _id: "$customerId", total: { $sum: "$total" } } },
  { $merge: {
      into: "customerStats",
      on: "_id",
      whenMatched: "merge",
      whenNotMatched: "insert"
  }}
  // $merge: upserts individual documents — NOT a collection-level replacement
  // Two concurrent $merge pipelines CAN intermix (non-atomic at document level unless on same _id)
])

// Correct use:
// $out  → "Rebuild this entire report from scratch" (snapshot replacement)
// $merge → "Update/upsert individual records in an existing collection"

// For scheduled reports: $out is safer because it gives you a clean swap
// Schedule via Atlas Triggers or a cron job, ensuring only one run at a time
```
