# Chapter 03 — Querying & Aggregation: Follow-Up Traps

---

## Trap 1: $where Uses the JavaScript Engine — Always Slow, No Index

**Setup**: Candidate mentions using $where for complex queries.

**Trap question**: "Is `$where` faster or slower than `$expr` for comparing two fields in the same document?"

**Wrong answer**: "They're about the same."

**Correct answer**: `$where` is dramatically slower — it runs a JavaScript interpreter for every single document, cannot use any index, and is disabled on Atlas shared tiers:

```javascript
// $where: JS engine runs per document — O(n) always
db.orders.find({ $where: "this.total > this.estimatedTotal" })
// Disabled on Atlas M0/M2/M5
// Deprecated since MongoDB 5.1

// $expr: uses aggregation expression engine — can use indexes
db.orders.find({
  $expr: { $gt: ["$total", "$estimatedTotal"] }
})
// If there's an index on total or estimatedTotal, the query planner can use it
// Server-side expression evaluation (no JS overhead)

// Performance difference on 1M documents:
// $where:    ~8000ms (8 seconds)
// $expr:     ~200ms  (same logic, different engine)
// With index: ~2ms
```

---

## Trap 2: $regex Without Anchors Is Slow

**Setup**: Candidate uses `$regex` for searching text fields.

**Trap question**: "If you create an index on a `name` field and query with `{ name: { $regex: /alice/ } }`, will the index be used?"

**Wrong answer**: "Yes, there's an index on name."

**Correct answer**: Only anchored regex (starting with `^`) can use an index efficiently. Unanchored regex performs a full index scan (reading all keys):

```javascript
// Anchored regex — can use index prefix scan
db.users.find({ name: { $regex: /^alice/i } })
// Index: IXSCAN with index bounds ["alice", "alicf") — efficient prefix scan
// But: $i (case-insensitive) with anchored regex requires a case-insensitive collation index

// Unanchored regex — full index scan
db.users.find({ name: { $regex: /alice/ } })
// Index: still uses the index, but scans ALL index keys looking for "alice"
// For 1M users: reads 1M index keys → nearly as slow as COLLSCAN

// No-anchor full scan:
db.users.find({ bio: { $regex: /loves mongodb/ } })
// bio is likely not indexed anyway, but even with an index: full scan

// Better alternatives for full-text search:
// 1. Text index (exact word matching)
db.users.createIndex({ bio: "text" })
db.users.find({ $text: { $search: "mongodb" } })

// 2. Atlas Search (production-grade full-text)
db.users.aggregate([{
  $search: { text: { query: "loves mongodb", path: "bio" } }
}])
```

---

## Trap 3: $unwind Explodes Cardinality

**Setup**: Candidate uses `$unwind` in an aggregation pipeline.

**Trap question**: "You have 100,000 orders, each with an average of 10 items. After `$unwind: '$items'`, how many documents does the pipeline have?"

**Wrong answer**: "100,000 — one per order."

**Correct answer**: 1,000,000 — one document per item:

```javascript
// 100K orders × 10 items = 1,000,000 documents after $unwind!
// This can cause memory issues or very slow aggregations

// Orders before $unwind: 100,000 documents
// After { $unwind: "$items" }: 1,000,000 documents

// Performance implications:
// 1. Subsequent $sort on this 1M document set is expensive
// 2. $group after $unwind requires grouping 1M docs instead of 100K
// 3. Memory usage explodes — may need allowDiskUse: true

// Optimization: put $match BEFORE $unwind where possible
// BAD:
db.orders.aggregate([
  { $unwind: "$items" },            // 1M docs
  { $match: { "items.category": "electronics" } },  // now filters 1M docs
])

// GOOD:
db.orders.aggregate([
  { $match: { "items.category": "electronics" } },  // filter 100K first (index possible)
  { $unwind: "$items" },            // now maybe only 30K items unwind
  { $match: { "items.category": "electronics" } },  // re-filter post-unwind
])

// Even better: use $filter instead of $unwind if you don't need to group by item
db.orders.aggregate([
  {
    $project: {
      electronicItems: {
        $filter: {
          input: "$items",
          cond: { $eq: ["$$this.category", "electronics"] }
        }
      }
    }
  }
])
// This keeps 100K documents, just filters each document's array
```

---

## Trap 4: $lookup Is NOT a SQL JOIN Substitute

**Setup**: Candidate says `$lookup` replaces SQL JOINs for relational data.

**Trap question**: "If you normalize MongoDB data and use $lookup for all reads, will performance be comparable to a SQL JOIN?"

**Wrong answer**: "Yes, $lookup is MongoDB's JOIN equivalent."

**Correct answer**: `$lookup` has fundamental performance differences from SQL JOINs:

```javascript
// SQL JOIN: query optimizer can push predicates, use indexes on both sides simultaneously
// SELECT * FROM orders o JOIN users u ON o.userId = u._id WHERE u.country = 'US'
// SQL may use index on users.country AND orders.userId simultaneously

// $lookup: MongoDB first evaluates the left side (orders), THEN for each
// order document, executes a subquery against users
// The $lookup happens AFTER $match stages before it
// There is NO optimizer that pushes conditions from after $lookup to inside $lookup

// This means:
db.orders.aggregate([
  // Match on orders side — OK, uses index
  { $match: { status: "pending" } },

  // $lookup fetches ALL orders' customers
  { $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "customer" } },

  // This match runs AFTER the lookup — too late!
  // All customers are already fetched, now we're filtering
  { $match: { "customer.country": "US" } }
])

// Better: use pipeline $lookup with conditions INSIDE the pipeline
db.orders.aggregate([
  { $match: { status: "pending" } },
  {
    $lookup: {
      from: "users",
      let: { userId: "$userId" },
      pipeline: [
        {
          $match: {
            $expr: { $eq: ["$_id", "$$userId"] },
            country: "US"  // filter INSIDE the lookup — less data transferred
          }
        }
      ],
      as: "customer"
    }
  },
  { $match: { customer: { $ne: [] } } }  // only orders with US customers
])

// For high-frequency reads: embed the data instead of using $lookup
```

---

## Trap 5: $group _id Must Be a Field Path, Expression, or null

**Setup**: Candidate writes a `$group` stage.

**Trap question**: "What does `$group: { _id: 'category' }` group by?"

**Wrong answer**: "It groups by the category field."

**Correct answer**: `'category'` (a string literal without `$`) groups ALL documents into ONE group with `_id: "category"`. The field reference requires `$`:

```javascript
// WRONG: groups all documents into one group with _id = "category" (a string)
db.products.aggregate([
  { $group: { _id: "category", count: { $sum: 1 } } }
])
// Result: [{ _id: "category", count: 10000 }]  — one group for everything!

// CORRECT: use $ to reference the field
db.products.aggregate([
  { $group: { _id: "$category", count: { $sum: 1 } } }
])
// Result: [{ _id: "electronics", count: 500 }, { _id: "clothing", count: 300 }, ...]

// Compound group key:
db.orders.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$orderDate" },  // expression
        status: "$status"               // field reference with $
      },
      count: { $sum: 1 }
    }
  }
])

// Group all (global aggregate):
db.orders.aggregate([
  { $group: { _id: null, totalRevenue: { $sum: "$amount" } } }
])
// _id: null = group all documents into one
```

---

## Trap 6: Projection Includes _id by Default

**Setup**: Candidate writes a projection to return specific fields.

**Trap question**: "Does `{ name: 1, email: 1 }` in a find() projection return `_id`?"

**Wrong answer**: "No, I only specified name and email."

**Correct answer**: Yes, `_id` is included by default in ALL projections unless explicitly set to `0`:

```javascript
// This returns: { _id: ObjectId("..."), name: "Alice", email: "alice@example.com" }
db.users.find({}, { name: 1, email: 1 })

// To exclude _id:
db.users.find({}, { name: 1, email: 1, _id: 0 })
// Returns: { name: "Alice", email: "alice@example.com" }

// Why does this matter?
// 1. API responses may expose internal ObjectIds to clients (security concern)
// 2. Data pipeline outputs include unnecessary _id field
// 3. Covered queries (no document fetch) require projecting out _id if it's not in the index

// In aggregation $project: same rule applies
db.users.aggregate([
  { $project: { name: 1, email: 1 } }         // _id included!
  { $project: { name: 1, email: 1, _id: 0 } }  // _id excluded
])
```

---

## Trap 7: .explain("executionStats") Required for Real Analysis

**Setup**: Candidate says they use `explain()` to check performance.

**Trap question**: "You run `db.users.find({...}).explain()` and see `IXSCAN`. Is this enough to confirm the query is performing well?"

**Wrong answer**: "Yes, IXSCAN means it's using an index."

**Correct answer**: `explain()` without arguments only shows the **winning plan** (what MongoDB chose to run). You need `"executionStats"` to see ACTUAL performance numbers:

```javascript
// explain() — only shows the plan, no execution data
db.users.find({ age: { $gt: 30 } }).explain()
// winningPlan: IXSCAN — looks good!
// But: how many documents were actually examined?

// explain("executionStats") — shows what actually happened
db.users.find({ age: { $gt: 30 } }).explain("executionStats")
{
  executionStats: {
    executionTimeMillis: 3200,         // 3.2 SECONDS — terrible for "IXSCAN"!
    totalDocsExamined: 950000,         // examined 950K docs from storage
    totalKeysExamined: 980000,         // scanned 980K index keys
    nReturned: 850000                  // returned 850K docs
  }
}
// The query IS using an index, but age: { $gt: 30 } has LOW SELECTIVITY
// 850K out of 1M users are over 30 → index barely helps vs COLLSCAN
// Solution: age is a poor index field if 85% of docs match

// "allPlansExecution" — why the optimizer chose this plan
db.users.find({...}).explain("allPlansExecution")
// Shows all candidate plans and their execution stats

// Key efficiency ratio:
// nReturned / totalDocsExamined should be close to 1.0
// 850000 / 950000 = 0.89 — not terrible (most examined were returned)
// But 1 / 950000 = 0.000001 — terrible (examined 950K to return 1)
```

---

## Trap 8: $lookup in a Transaction Across Shards

**Setup**: Candidate discusses transactions and $lookup together.

**Trap question**: "Can you use $lookup in a multi-document transaction in a sharded cluster?"

**Wrong answer**: "Yes, transactions support all aggregation stages including $lookup."

**Correct answer**: In a sharded cluster, `$lookup` inside a transaction has restrictions:

```javascript
// $lookup within transactions — sharded cluster restrictions:
// 1. The "from" collection must be on the SAME shard as the source collection
//    OR the "from" collection must be UNSHARDED (lives on a single shard/primary)
// 2. Cross-shard $lookup within a transaction is NOT supported

// Safe: $lookup to an unsharded collection
const session = client.startSession()
await session.withTransaction(async () => {
  const result = await db.orders.aggregate([
    { $match: { _id: orderId } },
    {
      $lookup: {
        from: "configs",   // unsharded collection — OK in transaction
        localField: "configId",
        foreignField: "_id",
        as: "config"
      }
    }
  ], { session }).toArray()
})

// Problematic: $lookup to another sharded collection within a transaction
// May throw: "Transaction number X has been aborted" or cross-shard error

// Workaround: do the lookup OUTSIDE the transaction, then start transaction
// with the fetched data
const config = await db.configs.findOne({ _id: configId })
const session = client.startSession()
await session.withTransaction(async () => {
  // Use config data we already have, no $lookup needed inside transaction
  await db.orders.updateOne({ _id: orderId }, { $set: { config: config } }, { session })
})
```

---

## Trap 9: $facet Memory Limit

**Setup**: Candidate recommends `$facet` for all faceted search scenarios.

**Trap question**: "What happens if the input to `$facet` is 500MB of data?"

**Wrong answer**: "$facet handles it automatically."

**Correct answer**: The `$facet` stage materializes ALL input documents in memory before running the sub-pipelines. If input exceeds 100MB (or disk limit), it fails:

```javascript
// $facet materializes input ONCE and passes it to each sub-pipeline
// If input docs are 500MB and you have 4 facets:
// MongoDB holds 500MB in memory (not 4 × 500MB — shared)
// BUT: 500MB > default 100MB in-memory aggregation limit

db.products.aggregate([
  { $match: { category: "electronics" } },  // MUST filter BEFORE $facet!
  {
    $facet: { ... }
  }
])
// If "electronics" category has 10M products: 10M docs * ~1KB = 10GB → fails

// Solutions:
// 1. Add strong $match BEFORE $facet to reduce input
// 2. Use allowDiskUse: true to allow spilling to disk
db.products.aggregate([...], { allowDiskUse: true })

// 3. For very large datasets: Atlas Search $facet is MUCH more efficient
// Atlas Search runs facet queries at the Lucene level (inverted index)
// Not limited by MongoDB document memory constraints

// 4. Paginate results separately from facet counts
// Run $facet only for the aggregate counts, separate query for paginated results
```

---

## Trap 10: Array Query Without $elemMatch Returns Unexpected Results

**Setup**: Candidate writes a query to filter documents by array element conditions.

**Trap question**: "You want products that have a variant with color='red' AND size='L'. You write `{ 'variants.color': 'red', 'variants.size': 'L' }`. Is this correct?"

**Wrong answer**: "Yes, that's how you query nested array objects."

**Correct answer**: This query can match a document where ONE variant is red and a DIFFERENT variant is size L:

```javascript
// Documents:
// Doc A: { variants: [{ color: "red", size: "S" }, { color: "blue", size: "L" }] }
// Doc B: { variants: [{ color: "red", size: "L" }] }

// WRONG query: can match Doc A (red in one element, L in another)
db.products.find({
  "variants.color": "red",
  "variants.size": "L"
})
// Returns: Doc A AND Doc B — Doc A doesn't have a red-L variant!

// CORRECT: use $elemMatch for same-element conditions
db.products.find({
  variants: {
    $elemMatch: { color: "red", size: "L" }
  }
})
// Returns: Doc B only — only documents with a variant that IS BOTH red AND L

// This is one of the most common query bugs in MongoDB
// Always use $elemMatch when multiple conditions must apply to the same array element
```

---

## Trap 11: $sort Memory Limit Without Index

**Setup**: Candidate adds a `$sort` stage to a large aggregation.

**Trap question**: "If you sort 50 million documents without an index, what error will you get?"

**Wrong answer**: "The sort just takes longer."

**Correct answer**: Without `allowDiskUse: true`, the aggregation will fail with a "exceeded memory limit" error:

```javascript
// ERROR: sort stage exceeded 100MB memory limit
db.orders.aggregate([
  { $match: { year: 2024 } },   // returns 10M docs
  { $sort: { total: -1 } }      // sort 10M docs in memory → fails!
])
// Error: "Sort exceeded memory limit of 104857600 bytes..."

// Fix 1: Add allowDiskUse for large sorts
db.orders.aggregate([
  { $match: { year: 2024 } },
  { $sort: { total: -1 } }
], { allowDiskUse: true })
// Spills to disk — slower but won't error

// Fix 2: Create an index to support the sort
db.orders.createIndex({ year: 1, total: -1 })
// Now $match { year: 2024 } + $sort { total: -1 } uses the index
// No in-memory sort needed — data comes back in sorted order

// Fix 3: Filter more aggressively before sorting
db.orders.aggregate([
  { $match: { year: 2024, total: { $gte: 1000 } } },  // reduce document count
  { $sort: { total: -1 } },
  { $limit: 100 }  // sort + limit = very efficient with an index
])

// allowDiskUse tradeoffs:
// Pros: doesn't fail on large sorts
// Cons: slower (disk I/O), uses disk space, not available on Atlas free tier (M0)
```

---

## Trap 12: $lookup Result Array Can Exceed 16MB Limit

**Setup**: Candidate uses `$lookup` to join collections.

**Trap question**: "You do `$lookup` from orders to order_items. If an order has 100,000 line items, what could go wrong?"

**Wrong answer**: "The join might be slow but will work."

**Correct answer**: The resulting document with 100,000 items embedded in the `as` field may exceed the 16MB BSON document size limit:

```javascript
// Order with 100,000 items at ~200 bytes each = ~20MB > 16MB limit
db.orders.aggregate([
  { $match: { _id: orderId } },
  {
    $lookup: {
      from: "orderItems",
      localField: "_id",
      foreignField: "orderId",
      as: "items"   // if 100K items: document becomes 20MB → ERROR
    }
  }
])
// Error: "BSONObjectTooLarge: object too large"

// Fixes:

// Fix 1: Limit inside $lookup pipeline
db.orders.aggregate([
  { $match: { _id: orderId } },
  {
    $lookup: {
      from: "orderItems",
      let: { orderId: "$_id" },
      pipeline: [
        { $match: { $expr: { $eq: ["$orderId", "$$orderId"] } } },
        { $limit: 1000 },        // hard limit
        { $project: { sku: 1, qty: 1, price: 1 } }  // project only needed fields
      ],
      as: "items"
    }
  }
])

// Fix 2: Paginate items separately (two queries)
const order = await db.orders.findOne({ _id: orderId })
const items = await db.orderItems.find({ orderId }, { sku: 1, qty: 1 }).limit(50).toArray()

// Fix 3: Don't use $lookup for this pattern — use a separate collection query
// This is WHY the data is in a separate collection in the first place!
```
