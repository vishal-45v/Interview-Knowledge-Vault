# Chapter 04 — Indexes & Performance: Follow-Up Traps

---

## Trap 1: Compound Index Field Order Matters — Index B vs Index A

**Setup**: Candidate says "I have an index on {a, b, c}."

**Trap question**: "Can you use the index `{a: 1, b: 1}` for the query `{ b: "hello" }`?"

**Wrong answer**: "Yes, b is in the index."

**Correct answer**: No. Compound indexes follow the **leftmost prefix rule**. You can only use the index for queries that start with `a`, optionally include `b`, and optionally include `c`. A query on `b` alone cannot use the `{a, b}` index.

```javascript
// Index: { status: 1, customerId: 1, createdAt: -1 }
// Can use this index:
db.orders.find({ status: "pending" })                    // prefix: status ✓
db.orders.find({ status: "pending", customerId: id })    // prefix: status + customerId ✓
db.orders.find({ status: "pending", customerId: id }).sort({ createdAt: -1 })  // full ✓

// CANNOT use this index:
db.orders.find({ customerId: id })            // no status prefix ✗
db.orders.find({ createdAt: { $gt: date } })  // not a prefix ✗
db.orders.find({ customerId: id, createdAt: { $gt: date } })  // skips status ✗
```

---

## Trap 2: Multikey Index Cannot Be Compound With Another Array Field

**Setup**: Candidate creates a compound index on two fields, one of which is an array.

**Trap question**: "Can you create an index on `{ tags: 1, categories: 1 }` if both fields are arrays?"

**Wrong answer**: "Yes, compound indexes work on any field types."

**Correct answer**: If BOTH fields are arrays in the same document, the compound multikey index CANNOT be created (or will error when that document is inserted):

```javascript
// This index creation succeeds (no documents violate it yet):
db.products.createIndex({ tags: 1, categories: 1 })

// But if you try to insert a document where BOTH tags AND categories are arrays:
db.products.insertOne({
  tags: ["nodejs", "javascript"],
  categories: ["web", "backend"]
})
// ERROR: "cannot index parallel arrays [tags] [categories]"

// This is a hard limitation:
// A multikey compound index can have AT MOST ONE field that is an array PER DOCUMENT
// Different documents can have different fields as arrays — but not the same document

// Solutions:
// 1. Create separate indexes: { tags: 1 } and { categories: 1 }
// 2. Redesign schema: store categories in tags with a prefix
//    { tags: ["tag:nodejs", "cat:web"] } — one array field
// 3. Use wildcard index if schema is dynamic
```

---

## Trap 3: TTL Index Only Works on Date Fields

**Setup**: Candidate adds a TTL index for session expiry.

**Trap question**: "I have `expiresAt: 1705315800` (Unix timestamp as integer). Will a TTL index with `expireAfterSeconds: 0` work?"

**Wrong answer**: "Yes, TTL index works on any numeric timestamp."

**Correct answer**: TTL indexes ONLY work on BSON `Date` type fields. Integer Unix timestamps are NOT recognized:

```javascript
// THIS WILL NOT WORK — integer Unix timestamp
db.sessions.insertOne({ token: "abc", expiresAt: 1705315800 })  // integer
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })
// Index created successfully, but documents NEVER expire!
// The TTL background job only processes ISODate() values

// CORRECT: must use BSON Date
db.sessions.insertOne({
  token: "abc",
  expiresAt: new Date(1705315800 * 1000)  // convert to milliseconds for Date
})
// OR:
db.sessions.insertOne({
  token: "abc",
  expiresAt: ISODate("2024-01-15T10:30:00Z")
})

// Fix existing integer timestamps:
db.sessions.updateMany(
  { expiresAt: { $type: "number" } },
  [{ $set: { expiresAt: { $toDate: { $multiply: ["$expiresAt", 1000] } } } }]
)
```

---

## Trap 4: Sparse Index Excludes Documents From Queries

**Setup**: Candidate uses a sparse index to handle optional fields.

**Trap question**: "You have a sparse unique index on `phone`. A user has no phone number (field missing). Does `db.users.find({ phone: null })` return that user?"

**Wrong answer**: "Yes, null matches missing fields."

**Correct answer**: The sparse index does NOT contain an entry for users without a phone field. MongoDB will NOT use the sparse index for `{ phone: null }` queries. It must fall back to a COLLSCAN to find documents without the phone field:

```javascript
// Sparse index: only indexes documents WITH a phone field
db.users.createIndex({ phone: 1 }, { sparse: true })

// These documents:
{ _id: 1, name: "Alice", phone: "555-1234" }   // IN the index
{ _id: 2, name: "Bob" }                          // NOT in the index (no phone)
{ _id: 3, name: "Carol", phone: null }           // depends on MongoDB version
// In MongoDB 4.4+: null IS included in sparse index (null ≠ missing)

// Query for users without a phone:
db.users.find({ phone: null })    // CANNOT use sparse index!
// Must do COLLSCAN to find users WHERE phone is null OR missing
// sparse index only helps for queries that look for SPECIFIC phone values

// Query for specific phone: CAN use sparse index
db.users.find({ phone: "555-1234" })  // ✓ uses sparse index

// Queries using sparse index for sort:
db.users.find().sort({ phone: 1 })
// Sparse index sort SKIPS documents without phone!
// You'll get results only for documents that HAVE a phone
// Documents without phone are EXCLUDED from sparse index sort results!
```

---

## Trap 5: Index on Array Field Creates One Entry Per Array Element

**Setup**: Candidate creates an index on an array field.

**Trap question**: "A product has 100 tags. How many entries does that one product contribute to the `{ tags: 1 }` multikey index?"

**Wrong answer**: "One entry — the document has one product record."

**Correct answer**: 100 entries — one per array element. This means the multikey index can be MUCH larger than you expect:

```javascript
// Product document with 100 tags:
{ _id: 1, name: "Widget", tags: [...100 tags...] }

// Multikey index { tags: 1 } entries for this ONE document:
// "tag1"   → product1
// "tag2"   → product1
// ...
// "tag100" → product1

// If you have 1M products with avg 50 tags each:
// Regular index: 1M entries (manageable)
// Multikey index: 50M entries (50x larger!)

// Performance implications:
// 1. Index build is much slower (50M writes instead of 1M)
// 2. Index uses 50x more memory
// 3. Each product update that changes tags = up to 100 index operations

// Solutions:
// 1. Limit array size (Bucket Pattern)
// 2. Store frequently queried "primary tags" as a separate flat field
//    { primaryTag: "mongodb", tags: [...] }
//    createIndex({ primaryTag: 1 })  ← regular single-field index, not multikey

// 3. Use a separate collection for the array elements
// products: { _id, name }
// product_tags: { productId, tag } ← createIndex({ tag: 1 })
// One entry per product-tag pair, but in a regular collection
```

---

## Trap 6: One Text Index Per Collection

**Setup**: Candidate tries to create text indexes on multiple fields separately.

**Trap question**: "Can you create a text index on `title` and a separate text index on `description`?"

**Wrong answer**: "Yes, just create two text indexes."

**Correct answer**: MongoDB only allows ONE text index per collection. Creating a second text index throws an error:

```javascript
// First text index:
db.articles.createIndex({ title: "text" })  // ✓ success

// Second text index on different field:
db.articles.createIndex({ description: "text" })
// ERROR: "cannot have two text indexes"
// MongoServerError: only one text index per collection allowed, found existing: title_text

// CORRECT: combine all text fields into a SINGLE text index:
db.articles.createIndex(
  { title: "text", description: "text", content: "text" },
  { weights: { title: 10, description: 5, content: 1 } }
)

// Or index ALL text fields at once:
db.articles.createIndex({ "$**": "text" })  // wildcard text index

// To add a new field to an existing text index:
// You must DROP the existing text index and recreate it
db.articles.dropIndex("title_text")
db.articles.createIndex({ title: "text", description: "text", content: "text" })
```

---

## Trap 7: $regex Case-Insensitive Doesn't Use Index Efficiently

**Setup**: Candidate uses case-insensitive regex for user name lookup.

**Trap question**: "You have an index on `name`. Does `{ name: { $regex: /^alice/i } }` use it efficiently?"

**Wrong answer**: "Yes, the index is on name and it's an anchored regex."

**Correct answer**: Case-insensitive regex with `i` flag cannot efficiently use a standard case-sensitive B-tree index, even if anchored:

```javascript
// Standard index (case-sensitive B-tree):
db.users.createIndex({ name: 1 })

// Case-insensitive regex: scans the entire alphabet range
db.users.find({ name: { $regex: /^alice/i } })
// Must check: "Alice", "alice", "ALICE", "aLiCe", etc.
// Index scan from "a" to "z" for "alice" prefix → wide range scan

// Solutions:

// Option 1: Store lowercase version of field for searching
db.users.createIndex({ nameLower: 1 })
db.users.updateMany({}, [{ $set: { nameLower: { $toLower: "$name" } } }])
// Query:
db.users.find({ nameLower: { $regex: /^alice/ } })  // no 'i' flag needed
// Anchored, no case sensitivity = prefix scan on lowercase index = efficient

// Option 2: Collation index (case-insensitive B-tree sort)
db.users.createIndex(
  { name: 1 },
  { collation: { locale: "en", strength: 2 } }  // strength 2 = case-insensitive
)
// Query WITH collation specified:
db.users.find({ name: "alice" })
  .collation({ locale: "en", strength: 2 })
// Uses the collation index efficiently!
// But: query MUST specify the same collation to use the index

// Option 3: Atlas Search (best for production name search)
// Supports case-insensitive, diacritics, fuzzy — all efficiently
```

---

## Trap 8: Covered Query Requires Projecting Out _id

**Setup**: Candidate explains covered queries.

**Trap question**: "You have an index on `{ category: 1, name: 1, price: 1 }`. You run `find({ category: 'electronics' }, { name: 1, price: 1 })`. Is this a covered query?"

**Wrong answer**: "Yes — all projected fields are in the index."

**Correct answer**: No. `_id` is included in projections BY DEFAULT and `_id` is NOT in the index (indexes don't include `_id` unless you add it). MongoDB must FETCH each document to get `_id`:

```javascript
// NOT covered: _id is implicitly projected
db.products.find(
  { category: "electronics" },
  { name: 1, price: 1 }   // _id: 1 is IMPLICIT!
).explain()
// Stage: FETCH (must read documents to get _id)

// COVERED: explicitly exclude _id
db.products.find(
  { category: "electronics" },
  { name: 1, price: 1, _id: 0 }  // ← critical!
).explain()
// Stage: IXSCAN with no FETCH (reads only index)
// totalDocsExamined: 0

// Also: if you ADD _id to the index, then you can include it in covered query:
db.products.createIndex({ category: 1, name: 1, price: 1, _id: 1 })
// Wait — _id is already in the index by default? No — the _id field is in the
// _id index, but NOT in a regular compound index unless explicitly added.

// The _id INDEX exists, but the _id FIELD is not part of your custom compound index.
// To include _id in a covered query, you can either:
// A) Exclude it with _id: 0 (most common approach)
// B) Add _id to your compound index (larger index, rarely needed)
```

---

## Trap 9: Index on Nested Array vs Dot Notation

**Setup**: Candidate indexes nested document fields.

**Trap question**: "You have documents like `{ user: { address: { city: 'Austin' } } }`. Does indexing `{ user: 1 }` help with queries on `user.address.city`?"

**Wrong answer**: "Yes — user is indexed so any user field query is covered."

**Correct answer**: Indexing `{ user: 1 }` indexes the ENTIRE subdocument as a value. Queries on nested fields need their OWN indexes using dot notation:

```javascript
// Indexes the entire subdocument as a BSON value:
db.users.createIndex({ user: 1 })
// Helps for: { user: { address: { city: "Austin", zip: "78701" } } }
//            (exact subdocument match — all fields must match exactly)
// Does NOT help for: { "user.address.city": "Austin" }

// Index for nested field queries — use dot notation:
db.users.createIndex({ "user.address.city": 1 })
// Helps for: { "user.address.city": "Austin" }

// Common mistake:
db.users.find({ "address.city": "Austin" })
// Will use the { "user.address.city": 1 } index? NO
// The field path is different: "address.city" vs "user.address.city"

// Create index matching EXACT field path in your queries:
db.users.createIndex({ "address.city": 1 })  // if docs have top-level address
db.users.createIndex({ "user.address.city": 1 })  // if nested under user

// Verify index is being used:
db.users.find({ "address.city": "Austin" }).explain("executionStats")
// Check: stage: "IXSCAN" with correct indexName
```

---

## Trap 10: Index Bloat From Frequent Updates to Indexed Fields

**Setup**: Candidate adds indexes to improve query performance.

**Trap question**: "You index `{ lastLoginAt: 1 }` on a user collection. Users log in 10 times per day on average, and you have 1M active users. What's the performance impact?"

**Wrong answer**: "The index improves login-time queries. Performance is better overall."

**Correct answer**: Each login requires UPDATING the index — deleting the old `lastLoginAt` entry and inserting a new one. 1M users × 10 logins/day = 10M index updates/day just for this one field:

```javascript
// Every login operation:
db.users.updateOne(
  { _id: userId },
  { $set: { lastLoginAt: new Date() } }
)
// This update must also:
// 1. Find the old lastLoginAt index entry
// 2. Delete it from the B-tree
// 3. Insert the new lastLoginAt into the B-tree
// For 10M logins/day: 20M B-tree operations just for this index!

// Problematic indexes (high-churn fields):
// lastLoginAt — changes every login
// lastSeenAt — changes every page view
// viewCount — changes every view
// currentBalance — changes every transaction

// For frequently updated fields, consider:
// 1. Only index if queries frequently filter/sort by this field
// 2. Use a separate statistics collection for rapidly-changing counters
// 3. Use partial index to reduce entries:
db.users.createIndex(
  { lastLoginAt: 1 },
  { partialFilterExpression: { isActive: true } }
)
// Only active users are indexed — inactive users don't add index overhead

// 4. Use "bucket" approach: track activity in daily/hourly buckets
//    { userId, date: "2024-01-15", loginCount: 8 }
//    Index on { date: 1, loginCount: -1 } — rarely updated
```

---

## Trap 11: Sort on Non-Indexed Field After Range Query

**Setup**: Candidate has a compound index and adds a sort.

**Trap question**: "You have index `{ status: 1, price: 1 }`. Query: `find({ status: 'active', price: { $lt: 100 } }).sort({ name: 1 })`. Is the sort covered by the index?"

**Wrong answer**: "Yes — status and price are in the index."

**Correct answer**: The sort is on `name` which is NOT in the index. MongoDB must do an in-memory sort:

```javascript
// Index: { status: 1, price: 1 }
db.products.find(
  { status: "active", price: { $lt: 100 } }
).sort({ name: 1 })

// Plan:
// 1. IXSCAN on { status: 1, price: 1 } — filter uses index
// 2. FETCH — load documents from storage
// 3. SORT — in-memory sort on name field ← not in index!

// Solution: add name to the index, following ESR
// E: status, S: name, R: price
db.products.createIndex({ status: 1, name: 1, price: 1 })
// But now: range on price is AFTER sort on name
// MongoDB scans { status: "active" } → sorted by name → scans each for price < 100

// Alternative: only index sort if sort is more selective than range
// Test both: { status, price, name } vs { status, name, price }
// And compare execution stats

// For this specific query pattern: the ESR optimal index is:
db.products.createIndex({ status: 1, name: 1, price: 1 })
// status (E) → name (S) → price (R)
// The sort is before the range in the index → sort uses index, no in-memory sort
```

---

## Trap 12: WiredTiger Cache and Index Performance

**Setup**: Candidate talks about index performance.

**Trap question**: "Your server has 32GB RAM. The WiredTiger cache is set to 50% (16GB). Your collection's working set is 20GB. What happens?"

**Wrong answer**: "MongoDB will use disk for the overflow — performance stays the same."

**Correct answer**: When working set exceeds WiredTiger cache, MongoDB must read from disk (random I/O) for cache misses. Performance degrades significantly:

```javascript
// Check WiredTiger cache stats:
db.serverStatus().wiredTiger.cache

// Look for:
{
  "bytes currently in the cache": 17179869184,      // 16GB — cache is full!
  "bytes read into cache": 5368709120,              // reads from disk
  "pages read into cache": 1234567,                 // disk page reads
  "pages evicted because of memory pressure": 987654 // evictions = cache pressure!
}

// Symptoms of cache pressure:
// 1. High read latency (random disk reads instead of RAM reads)
// 2. High "pages evicted because of memory pressure" in server stats
// 3. Inconsistent query latency (depends on whether data is cached)

// Solutions:
// 1. Increase WiredTiger cache (if RAM available):
mongod --wiredTigerCacheSizeGB 24  // or in mongod.conf

// 2. Reduce working set (better data modeling/archiving):
// Atlas Online Archive: move cold data to S3, keep hot data in Atlas
// TTL indexes to remove old documents
// Projection in queries to avoid loading large embedded arrays

// 3. Add more RAM to the server / upgrade Atlas tier
// Atlas M30: 8GB RAM | M40: 16GB RAM | M50: 32GB RAM | M60: 64GB RAM

// 4. Index bloat contribution:
// Large indexes that don't fit in cache cause index reads from disk
// Remove unused indexes to shrink the cache footprint
```
