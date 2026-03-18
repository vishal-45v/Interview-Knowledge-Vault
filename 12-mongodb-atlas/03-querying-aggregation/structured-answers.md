# Chapter 03 — Querying & Aggregation: Structured Answers

---

## Answer 1: "Walk me through how you'd build and optimize a complex aggregation pipeline"

**Point**: Start with `$match` to filter early using indexes, then transform, then aggregate, then project the final shape.

**Methodology**:
1. `explain("executionStats")` before and after optimization
2. Check `totalDocsExamined` vs `nReturned`
3. Verify early `$match` uses an index (`IXSCAN`)
4. Add `allowDiskUse: true` only as last resort

```javascript
// Business question: "Monthly revenue per product category for 2024, top 5"

// Step 1: Naive first draft
db.orders.aggregate([
  { $unwind: "$items" },
  { $match: { orderDate: { $gte: ISODate("2024-01-01") }, status: "completed" } },
  { $group: { _id: { month: { $month: "$orderDate" }, category: "$items.category" }, revenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } } } },
  { $sort: { "_id.month": 1, revenue: -1 } }
])

// Problem: $unwind runs on ALL orders before $match filters by date
// Fix: $match FIRST

// Step 2: Optimized
db.orders.aggregate([
  // $match first — can use index on orderDate + status
  { $match: { orderDate: { $gte: ISODate("2024-01-01"), $lt: ISODate("2025-01-01") }, status: "completed" } },

  // $unwind only on filtered orders
  { $unwind: "$items" },

  // $group by month + category
  {
    $group: {
      _id: { month: { $month: "$orderDate" }, category: "$items.category" },
      revenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } },
      orderCount: { $sum: 1 }
    }
  },

  // Sort within each month by revenue
  { $sort: { "_id.month": 1, revenue: -1 } },

  // Rank within each month (top 5)
  {
    $group: {
      _id: "$_id.month",
      categories: { $push: { category: "$_id.category", revenue: "$revenue", orderCount: "$orderCount" } }
    }
  },
  {
    $project: {
      month: "$_id",
      topCategories: { $slice: ["$categories", 5] }
    }
  },
  { $sort: { month: 1 } }
])
```

---

## Answer 2: "Explain how $lookup works and its performance characteristics"

**Point**: `$lookup` performs a left outer join by executing a subquery for each document in the pipeline, with no query optimizer integration.

```javascript
// Basic $lookup — runs for EACH document in the left side
db.orders.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "customer"
    }
  }
])
// For 100K orders: executes 100K lookups against users collection
// With index on users._id: 100K × O(log n) index lookups = fast
// Without index on foreignField: 100K × O(n) scans = extremely slow!

// ALWAYS index the foreignField:
db.users.createIndex({ _id: 1 })  // _id already indexed by default
db.users.createIndex({ email: 1 })  // if using email as foreignField

// Advanced: limit data transferred with pipeline $lookup
db.orders.aggregate([
  {
    $lookup: {
      from: "products",
      let: { productIds: "$items.productId" },
      pipeline: [
        { $match: { $expr: { $in: ["$_id", "$$productIds"] } } },
        { $project: { name: 1, price: 1, _id: 1 } }  // only needed fields
      ],
      as: "productDetails"
    }
  }
])

// Performance benchmark comparison:
// Embedded approach: db.orders.findOne() = 1ms
// $lookup approach: db.orders.aggregate([$lookup]) = 5-50ms
// Application join: 2 queries, combine in code = 3-8ms

// Recommendation: embed frequently co-accessed data, $lookup for occasional reports
```

---

## Answer 3: "How do you approach query optimization in MongoDB?"

**Step-by-step process**:

```javascript
// Step 1: Identify the slow query (enable profiler)
db.setProfilingLevel(2, { slowms: 100 })  // log queries taking >100ms
db.system.profile.find({ millis: { $gt: 100 } }).sort({ millis: -1 }).limit(10)

// Step 2: Analyze with explain
const plan = db.products.find({ category: "electronics", price: { $lt: 100 } })
  .explain("executionStats")

// Look for:
const stats = plan.executionStats
console.log("Execution time:", stats.executionTimeMillis, "ms")
console.log("Docs examined:", stats.totalDocsExamined)
console.log("Docs returned:", stats.nReturned)
console.log("Keys examined:", stats.totalKeysExamined)
console.log("Stage:", plan.queryPlanner.winningPlan.stage)
// COLLSCAN = no index, IXSCAN = index used

// Step 3: Create appropriate index
db.products.createIndex({ category: 1, price: 1 })

// Step 4: Verify improvement
const afterPlan = db.products.find({ category: "electronics", price: { $lt: 100 } })
  .explain("executionStats")
// Should now show IXSCAN with much lower totalDocsExamined

// Step 5: Check for covered query opportunity
db.products.find(
  { category: "electronics", price: { $lt: 100 } },
  { name: 1, price: 1, _id: 0 }   // only indexed fields
).explain("executionStats")
// If all projected fields are in the index: "COVERED" — no document fetch!
```

---

## Answer 4: "Explain aggregation expressions for array manipulation: $map, $filter, $reduce"

```javascript
// SCENARIO: Order items array, need to compute per-item stats

// $map — transform every element
db.orders.aggregate([
  {
    $project: {
      itemsWithDiscount: {
        $map: {
          input: "$items",
          as: "item",
          in: {
            sku: "$$item.sku",
            originalPrice: "$$item.price",
            // Apply 10% discount to each item
            finalPrice: { $multiply: ["$$item.price", 0.9] },
            // Add savings amount
            savings: { $multiply: ["$$item.price", 0.1] }
          }
        }
      }
    }
  }
])

// $filter — keep only elements matching a condition
db.orders.aggregate([
  {
    $project: {
      expensiveItems: {
        $filter: {
          input: "$items",
          as: "item",
          cond: { $gte: ["$$item.price", 50] }  // keep items ≥ $50
        }
      }
    }
  }
])

// $reduce — aggregate array to a single value
db.orders.aggregate([
  {
    $project: {
      // Sum line totals
      orderTotal: {
        $reduce: {
          input: "$items",
          initialValue: NumberDecimal("0"),
          in: {
            $add: [
              "$$value",
              { $multiply: ["$$this.price", "$$this.qty"] }
            ]
          }
        }
      },
      // Concatenate item names
      itemSummary: {
        $reduce: {
          input: "$items",
          initialValue: "",
          in: {
            $concat: [
              "$$value",
              { $cond: [{ $eq: ["$$value", ""] }, "", ", "] },
              "$$this.name"
            ]
          }
        }
      }
    }
  }
])

// Combining $filter + $map: transform only matching elements
db.orders.aggregate([
  {
    $project: {
      discountedExpensiveItems: {
        $map: {
          input: {
            $filter: {
              input: "$items",
              as: "item",
              cond: { $gte: ["$$item.price", 100] }  // only expensive items
            }
          },
          as: "item",
          in: {
            sku: "$$item.sku",
            price: "$$item.price",
            discounted: { $multiply: ["$$item.price", 0.8] }
          }
        }
      }
    }
  }
])
```

---

## Answer 5: "Describe the $facet stage and implement faceted search for e-commerce"

**Point**: `$facet` processes multiple aggregation sub-pipelines on the same input documents simultaneously, returning all results in a single pipeline execution — ideal for faceted navigation that needs counts, results, and filter options in one query.

```javascript
// Complete faceted search implementation
async function facetedSearch({
  query,           // search text
  category,        // selected category filter
  priceMin,        // price range filter
  priceMax,
  brands,          // selected brands (array)
  rating,          // minimum rating
  page = 1,
  pageSize = 20,
  sortBy = "relevance"
}) {
  const filter = {
    ...(category && { category }),
    ...(priceMin !== undefined || priceMax !== undefined ? {
      price: {
        ...(priceMin !== undefined && { $gte: NumberDecimal(priceMin.toString()) }),
        ...(priceMax !== undefined && { $lte: NumberDecimal(priceMax.toString()) })
      }
    } : {}),
    ...(brands?.length && { brand: { $in: brands } }),
    ...(rating && { rating: { $gte: rating } }),
    inStock: true,
    ...(query && { $text: { $search: query } })  // text search if query provided
  }

  const sortStage = {
    relevance: query ? { score: { $meta: "textScore" } } : { _id: -1 },
    priceAsc: { price: 1 },
    priceDesc: { price: -1 },
    rating: { rating: -1, reviewCount: -1 }
  }[sortBy]

  const pipeline = [
    { $match: filter },
    ...(query ? [{ $addFields: { textScore: { $meta: "textScore" } } }] : []),
    {
      $facet: {
        // Paginated results
        results: [
          { $sort: sortStage },
          { $skip: (page - 1) * pageSize },
          { $limit: pageSize },
          { $project: { name: 1, price: 1, brand: 1, rating: 1, image: 1, stock: 1 } }
        ],
        // Total count
        total: [{ $count: "count" }],
        // Category counts (showing available categories with counts)
        categoryFacets: [
          { $group: { _id: "$category", count: { $sum: 1 } } },
          { $sort: { count: -1 } }
        ],
        // Brand counts
        brandFacets: [
          { $group: { _id: "$brand", count: { $sum: 1 } } },
          { $sort: { count: -1 } },
          { $limit: 20 }
        ],
        // Price distribution
        priceFacets: [
          { $bucket: { groupBy: "$price", boundaries: [0, 25, 50, 100, 250, 500], default: "500+", output: { count: { $sum: 1 } } } }
        ],
        // Rating distribution
        ratingFacets: [
          { $group: { _id: { $floor: "$rating" }, count: { $sum: 1 } } },
          { $sort: { _id: 1 } }
        ]
      }
    }
  ]

  const [result] = await db.products.aggregate(pipeline).toArray()

  return {
    results: result.results,
    total: result.total[0]?.count || 0,
    pages: Math.ceil((result.total[0]?.count || 0) / pageSize),
    facets: {
      categories: result.categoryFacets,
      brands: result.brandFacets,
      priceRanges: result.priceFacets,
      ratings: result.ratingFacets
    }
  }
}
```

---

## Answer 6: "How do $merge and $out differ and when would you use each?"

```javascript
// $out — atomically replaces target collection with pipeline results
// Like a "CREATE TABLE AS SELECT" in SQL
db.orders.aggregate([
  { $match: { year: 2024 } },
  { $group: { _id: "$customerId", yearTotal: { $sum: "$amount" } } },
  { $out: "customer_year_totals" }   // REPLACES the whole collection
])
// Use for: cache tables, nightly data refreshes, one-time transformations
// DANGER: if pipeline produces 0 results, target collection becomes empty!

// $merge — merges results into existing collection (MongoDB 4.2+)
// Like a "MERGE/UPSERT" in SQL
db.orders.aggregate([
  { $match: { processedToday: true } },
  { $group: { _id: "$customerId", todaySpend: { $sum: "$amount" } } },
  {
    $merge: {
      into: "customer_stats",       // merge into existing collection
      on: "_id",                    // match key (must be uniquely indexed)
      whenMatched: "merge",         // merge fields (keep existing, add/overwrite new)
      // other whenMatched options:
      // "replace" — replace matched doc with pipeline doc
      // "keepExisting" — keep existing, ignore pipeline result
      // "fail" — throw error on match
      // [{pipeline}] — custom merge pipeline
      whenNotMatched: "insert"      // insert if no match found
      // "discard" — skip if no match
      // "fail" — throw error if no match
    }
  }
])
// Use for: incremental updates, rolling aggregations, enriching existing data

// Comparison:
// $out:   complete replace, all-or-nothing, drops existing indexes then recreates
// $merge: incremental, preserves existing data, index-safe

// Example: real-time dashboard cache (incremental update)
setInterval(async () => {
  await db.orders.aggregate([
    { $match: { updatedAt: { $gte: new Date(Date.now() - 60000) } } },  // last 1 minute
    { $group: { _id: "$status", count: { $sum: 1 }, revenue: { $sum: "$amount" } } },
    {
      $merge: {
        into: "dashboard_stats",
        on: "_id",
        whenMatched: [
          { $set: { count: { $add: ["$count", "$$new.count"] }, revenue: { $add: ["$revenue", "$$new.revenue"] } } }
        ],
        whenNotMatched: "insert"
      }
    }
  ])
}, 60000)
```

---

## Answer 7: "What is a covered query and how do you achieve it?"

**Point**: A covered query is one where MongoDB can satisfy the entire query — filter, sort, AND projection — using only the index, without reading any documents from disk.

```javascript
// Index: { category: 1, price: 1, name: 1 }
db.products.createIndex({ category: 1, price: 1, name: 1 })

// Covered query: all referenced fields are in the index
const result = db.products.find(
  { category: "electronics", price: { $lte: 100 } },  // filter on indexed fields
  { name: 1, price: 1, category: 1, _id: 0 }          // project ONLY indexed fields
).explain("executionStats")

// executionStats for covered query:
{
  totalDocsExamined: 0,   // ZERO documents read from storage!
  totalKeysExamined: 150, // only reads index
  nReturned: 150
}
// Stage: IXSCAN with no FETCH stage = pure covered query

// REQUIREMENTS for covered query:
// 1. Filter fields must be in the index
// 2. Projection fields must be in the index
// 3. _id must be EXPLICITLY excluded (it's not in the index, included by default!)
//    { _id: 0 } is critical for covered queries

// Most common mistake: forgetting to exclude _id
db.products.find(
  { category: "electronics" },
  { name: 1, price: 1 }   // _id is implicitly included — breaks covered query!
)
// Fix:
db.products.find(
  { category: "electronics" },
  { name: 1, price: 1, _id: 0 }   // now covered query is possible
)

// When are covered queries most valuable?
// - High-frequency reads on large collections
// - Read-heavy APIs (product listings, autocomplete)
// - Reporting queries on summary fields
```

---

## Answer 8: "Explain the aggregation pipeline memory limit and how to work around it"

```javascript
// Default: each stage can use up to 100MB of memory
// If exceeded: "QueryExceededMemoryLimitNoDiskUseAllowed" error

// Scenarios that trigger memory limits:
// 1. Large $sort without an index
db.orders.aggregate([
  { $sort: { customField: -1 } }  // custom field not indexed → memory sort
])

// 2. $group on high-cardinality field with many accumulators
db.events.aggregate([
  { $group: { _id: "$sessionId", events: { $push: "$$ROOT" } } }
])
// $push: "$$ROOT" = each group entry holds ALL documents from that session

// 3. $facet with large input
db.products.aggregate([
  { $facet: { ... } }  // materializes all input in memory
])

// Fix 1: allowDiskUse: true
db.orders.aggregate([...], { allowDiskUse: true })
// Spills intermediate results to disk when memory limit hit
// Slower but won't error

// Fix 2: Reduce data earlier in pipeline
db.events.aggregate([
  { $match: { date: lastMonth } },   // filter first
  { $project: { sessionId: 1, type: 1 } },  // project to small subset
  { $group: { _id: "$sessionId", types: { $addToSet: "$type" } } }
])

// Fix 3: Paginate heavy operations
// Instead of grouping ALL users: paginate the upstream
let lastId = null
while (true) {
  const batch = await db.orders.aggregate([
    ...(lastId ? [{ $match: { _id: { $gt: lastId } } }] : []),
    { $sort: { _id: 1 } },
    { $limit: 10000 },
    { $group: { ... } }
  ]).toArray()
  if (batch.length === 0) break
  lastId = batch[batch.length - 1]._id
}

// Fix 4: Precompute and store expensive aggregations
// Run nightly, store results in summary collection
// Dashboard reads from summary collection (fast) instead of raw data (slow)
```

---

## Answer 9: "How do window functions in $setWindowFields solve running total problems?"

```javascript
// WITHOUT window functions (MongoDB < 5.0): requires $reduce + sorting in app
db.transactions.aggregate([
  { $match: { accountId } },
  { $sort: { ts: 1 } },
  {
    $group: {
      _id: null,
      transactions: { $push: { amount: "$amount", type: "$type", ts: "$ts" } }
    }
  },
  // Manual running total via $reduce
  {
    $project: {
      withBalance: {
        $reduce: {
          input: "$transactions",
          initialValue: { balance: 0, records: [] },
          in: { /* complex accumulator logic */ }
        }
      }
    }
  }
])
// Complex, hard to maintain, works for small datasets

// WITH $setWindowFields (MongoDB 5.0+): clean, efficient
db.transactions.aggregate([
  { $match: { accountId } },
  { $sort: { ts: 1 } },
  {
    $setWindowFields: {
      partitionBy: "$accountId",   // separate running total per account
      sortBy: { ts: 1 },
      output: {
        runningBalance: {
          $sum: "$signedAmount",
          window: { documents: ["unbounded", "current"] }  // from start to now
        },
        dailyTotal: {
          $sum: "$signedAmount",
          window: { range: [0, 0], unit: "day" }  // same day
        },
        movingAvg30d: {
          $avg: "$signedAmount",
          window: { range: [-30, 0], unit: "day" }
        },
        rank: { $rank: {} },
        rowNum: { $documentNumber: {} }
      }
    }
  }
])

// More window function examples:
db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$region",
      sortBy: { saleDate: 1 },
      output: {
        // Cumulative sum
        cumulativeRevenue: { $sum: "$amount", window: { documents: ["unbounded", "current"] } },
        // Previous row's value
        prevDayRevenue: { $shift: { output: "$amount", by: -1 } },
        // Next row's value
        nextDayRevenue: { $shift: { output: "$amount", by: 1 } },
        // Running average
        runningAvg: { $avg: "$amount", window: { documents: ["unbounded", "current"] } }
      }
    }
  }
])
```

---

## Answer 10: "Compare the performance of $in vs multiple $or conditions"

```javascript
// Query: find users in US, CA, or GB
// Option 1: $in
db.users.find({ country: { $in: ["US", "CA", "GB"] } })

// Option 2: $or
db.users.find({
  $or: [
    { country: "US" },
    { country: "CA" },
    { country: "GB" }
  ]
})

// For simple single-field conditions: $in and $or are equivalent
// MongoDB optimizer converts both to the same execution plan

// $in with index: performs N index lookups (one per value)
// $or with index: also performs N index lookups (one per condition)

// HOWEVER: for different fields, $or can be slower
// $or on multiple fields = separate index scans merged
db.users.find({
  $or: [
    { country: "US" },
    { role: "admin" }
  ]
})
// Two separate IXSCAN stages + result merge (requires two indexes)

// $in with multiple fields (not possible — $in is single-field only)
// Must use $or for different fields

// When $or is much slower:
// 1. When not all conditions have indexes (one COLLSCAN ruins the plan)
db.users.find({
  $or: [
    { country: "US" },        // indexed field — fast
    { bio: { $regex: /admin/ } }  // non-indexed regex — full scan
  ]
})
// MongoDB can't use index for the regex part → may do a COLLSCAN for entire OR

// 2. Large $in lists with unindexed field
db.products.find({ tag: { $in: [...500 values...] } })
// Scans index 500 times → may be slower than COLLSCAN at some threshold
```
