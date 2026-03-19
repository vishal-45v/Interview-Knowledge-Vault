# Chapter 03 — Querying & Aggregation: Theory Questions

---

## Query Operators

**Q1. What are the comparison query operators in MongoDB?**

```javascript
// $eq — equals (implicit in most queries)
db.users.find({ age: { $eq: 30 } })   // same as: { age: 30 }

// $ne — not equals
db.users.find({ status: { $ne: "inactive" } })

// $gt / $gte — greater than / greater than or equal
db.products.find({ price: { $gt: 100, $lte: 500 } })

// $lt / $lte — less than / less than or equal
db.events.find({ ts: { $lt: ISODate("2024-01-01T00:00:00Z") } })

// $in — value is in the provided array
db.users.find({ role: { $in: ["admin", "moderator"] } })
// Equivalent to: WHERE role = 'admin' OR role = 'moderator'

// $nin — value is NOT in the provided array
db.users.find({ country: { $nin: ["US", "CA", "GB"] } })

// Combining comparisons (implicit $and)
db.products.find({
  price: { $gte: 10, $lte: 100 },
  stock: { $gt: 0 }
})
```

---

**Q2. Explain the logical query operators: $and, $or, $nor, $not.**

```javascript
// $and — all conditions must be true (implicit when using multiple fields)
// Explicit $and needed when applying multiple conditions to the SAME field
db.users.find({
  $and: [
    { age: { $gte: 18 } },
    { age: { $lte: 65 } }
  ]
})
// Shorter implicit version (same result):
db.users.find({ age: { $gte: 18, $lte: 65 } })

// When MUST you use explicit $and? When the same operator appears twice:
db.products.find({
  $and: [
    { price: { $not: { $lt: 10 } } },   // can't use $price: {$not, $not} on same field
    { price: { $not: { $gt: 100 } } }
  ]
})

// $or — at least one condition must be true
db.users.find({
  $or: [
    { age: { $lt: 18 } },
    { age: { $gte: 65 } },
    { isVIP: true }
  ]
})

// $nor — none of the conditions must be true
db.users.find({
  $nor: [
    { status: "banned" },
    { emailVerified: false }
  ]
})
// Returns users who are NOT banned AND have verified email

// $not — inverts the expression
db.products.find({ price: { $not: { $gte: 100 } } })
// Returns products where price < 100 OR price field doesn't exist
// NOT the same as { price: { $lt: 100 } } — $not also includes missing field
```

---

**Q3. What are the element query operators $exists and $type?**

```javascript
// $exists — field exists (true) or doesn't exist (false)
db.users.find({ middleName: { $exists: true } })  // has middleName field
db.users.find({ deletedAt: { $exists: false } })   // soft-deleted users excluded

// $type — field is of specific BSON type
db.collection.find({ price: { $type: "double" } })
db.collection.find({ age: { $type: "int" } })
db.collection.find({ createdAt: { $type: "date" } })

// $type with type number (BSON type codes)
db.collection.find({ _id: { $type: 7 } })    // type 7 = ObjectId
db.collection.find({ name: { $type: 2 } })   // type 2 = string

// $type with array of types (find fields that are any of these types)
db.collection.find({ age: { $type: ["int", "double"] } })  // any numeric

// Common BSON type aliases:
// "double" (1), "string" (2), "object" (3), "array" (4), "binData" (5),
// "objectId" (7), "bool" (8), "date" (9), "null" (10), "regex" (11),
// "int" (16), "timestamp" (17), "long" (18), "decimal" (19)

// Finding mixed-type data (useful for data quality checks)
db.products.aggregate([
  { $group: { _id: { $type: "$price" }, count: { $sum: 1 } } }
])
```

---

**Q4. How do you use $regex for pattern matching queries?**

```javascript
// Basic regex
db.users.find({ name: { $regex: /^alice/i } })  // starts with "alice" (case-insensitive)
db.products.find({ description: { $regex: "mongodb", $options: "i" } })

// $options:
// i — case insensitive
// m — multiline (^ and $ match line beginnings/ends)
// s — dot (.) matches newlines
// x — ignore whitespace in pattern

// Pattern matching examples
db.emails.find({ email: { $regex: /^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$/i } })
db.products.find({ sku: { $regex: /^WIDGET-[0-9]{4}$/ } })

// Anchored regex can use index (starts with "^" uses prefix scan)
db.users.find({ username: { $regex: /^alice/ } })  // can use index on username field

// Non-anchored regex is SLOW (full collection scan)
db.users.find({ bio: { $regex: /mongodb/ } })  // scans ALL bio fields

// Better alternative for full-text search: Atlas Search or $text index
db.users.createIndex({ bio: "text" })
db.users.find({ $text: { $search: "mongodb" } })  // uses text index
```

---

**Q5. What is the $where operator and why should you avoid it?**

```javascript
// $where — executes JavaScript expression in the database
// Allows arbitrary JavaScript that can access document fields as 'this'
db.users.find({
  $where: "this.firstName.length + this.lastName.length > 20"
})

// WHY AVOID $where:
// 1. Cannot use indexes — always does a full collection scan
// 2. JavaScript engine runs PER DOCUMENT — extremely slow
// 3. Security risk: JavaScript injection if user input is not sanitized
// 4. Disabled by default in Atlas shared tier (M0/M2/M5)
// 5. Deprecated in Atlas M10+ as well

// BETTER alternative: Use $expr with aggregation expressions
db.users.find({
  $expr: {
    $gt: [
      { $add: [{ $strLenCP: "$firstName" }, { $strLenCP: "$lastName" }] },
      20
    ]
  }
})
// $expr can use indexes and is faster

// Or store the computed value as a field with a normal query:
db.users.updateMany({}, [
  { $set: { fullNameLength: { $add: [{ $strLenCP: "$firstName" }, { $strLenCP: "$lastName" }] } } }
])
db.users.createIndex({ fullNameLength: 1 })
db.users.find({ fullNameLength: { $gt: 20 } })  // fast index query
```

---

**Q6. How do array query operators work: $all, $elemMatch, $size?**

```javascript
// $all — array must contain ALL specified elements
db.products.find({ tags: { $all: ["mongodb", "database"] } })
// Matches: { tags: ["mongodb", "database", "nosql"] } ✓
// Does NOT match: { tags: ["mongodb", "javascript"] } ✗

// $size — array has EXACTLY the specified number of elements
db.orders.find({ items: { $size: 1 } })    // orders with exactly 1 item
// Limitation: $size doesn't support ranges ($size: {$gt: 2} doesn't work)
// Workaround:
db.orders.find({ "items.2": { $exists: true } })  // has at least 3 elements (index 2 exists)

// $elemMatch — at least one array element matches ALL conditions
db.scores.find({
  results: {
    $elemMatch: {
      subject: "math",
      score: { $gte: 90 }
    }
  }
})
// Finds docs where SAME result entry has subject:math AND score>=90

// Without $elemMatch (incorrect for multi-condition array queries):
db.scores.find({
  "results.subject": "math",
  "results.score": { $gte: 90 }
})
// This could match: subject:math in one entry, score>=90 in DIFFERENT entry
```

---

## Projections

**Q7. How do MongoDB projections work?**

```javascript
// Inclusion projection (specify fields to include, 1 = include)
db.users.find({}, { name: 1, email: 1 })
// Returns: { _id: ObjectId("..."), name: "Alice", email: "alice@example.com" }
// Note: _id is ALWAYS included by default unless explicitly excluded!

// Exclusion projection (specify fields to exclude, 0 = exclude)
db.users.find({}, { password: 0, internalNotes: 0 })
// Returns all fields EXCEPT password and internalNotes

// Exclude _id from inclusion projection
db.users.find({}, { name: 1, email: 1, _id: 0 })

// You CANNOT mix inclusion and exclusion (except for _id)
db.users.find({}, { name: 1, password: 0 })  // ERROR!
// Only _id can be excluded in an inclusion projection:
db.users.find({}, { name: 1, email: 1, _id: 0 })  // valid

// $slice projection — return a subset of an array
db.posts.find({}, { title: 1, comments: { $slice: 5 } })         // first 5 comments
db.posts.find({}, { title: 1, comments: { $slice: -3 } })        // last 3 comments
db.posts.find({}, { title: 1, comments: { $slice: [10, 5] } })   // skip 10, take 5

// $elemMatch in projection — return first matching array element
db.scores.find(
  { student: "Alice" },
  { scores: { $elemMatch: { subject: "math" } } }
)
// Returns only the first math score from the scores array

// Nested field projection with dot notation
db.users.find({}, { "address.city": 1, "address.zip": 1 })
```

---

## Aggregation Pipeline

**Q8. What is the MongoDB aggregation pipeline?**

The aggregation pipeline is a framework for data transformation and analysis. Documents flow through a series of stages, each transforming the documents:

```javascript
db.orders.aggregate([
  // Stage 1: $match — filter documents (like WHERE in SQL)
  {
    $match: {
      status: "completed",
      orderDate: { $gte: ISODate("2024-01-01T00:00:00Z") }
    }
  },

  // Stage 2: $unwind — flatten array field
  { $unwind: "$items" },

  // Stage 3: $group — aggregate (like GROUP BY in SQL)
  {
    $group: {
      _id: "$items.category",
      totalRevenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$total" }
    }
  },

  // Stage 4: $sort — order results
  { $sort: { totalRevenue: -1 } },

  // Stage 5: $limit — return top N
  { $limit: 10 },

  // Stage 6: $project — shape the output
  {
    $project: {
      category: "$_id",
      totalRevenue: { $round: ["$totalRevenue", 2] },
      orderCount: 1,
      avgOrderValue: { $round: ["$avgOrderValue", 2] },
      _id: 0
    }
  }
])
```

Key principle: Put `$match` first to filter documents early, reducing the amount of data subsequent stages process. If `$match` appears first and uses indexed fields, MongoDB can use an index.

---

**Q9. Explain the $group stage and its accumulator operators.**

```javascript
// $group groups documents by _id and applies accumulators
// _id can be a field, expression, null (all docs), or compound

// Example: sales analytics
db.sales.aggregate([
  {
    $group: {
      _id: {                              // group key (can be compound)
        year: { $year: "$saleDate" },
        month: { $month: "$saleDate" },
        category: "$product.category"
      },

      // Accumulator operators:
      totalRevenue: { $sum: "$amount" },          // sum of amounts
      count: { $sum: 1 },                          // count documents
      avgAmount: { $avg: "$amount" },              // average
      maxSale: { $max: "$amount" },                // maximum value
      minSale: { $min: "$amount" },                // minimum value
      products: { $addToSet: "$product.name" },    // unique values array
      allAmounts: { $push: "$amount" },            // all values (with duplicates)
      firstSale: { $first: "$saleDate" },          // first document's value
      lastSale: { $last: "$saleDate" },            // last document's value
      stdDev: { $stdDevPop: "$amount" }            // standard deviation
    }
  }
])

// Grouping ALL documents (null _id = global aggregate)
db.orders.aggregate([
  {
    $group: {
      _id: null,
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$amount" }
    }
  },
  {
    $project: {
      _id: 0,
      totalRevenue: 1,
      orderCount: 1,
      avgOrderValue: { $round: ["$avgOrderValue", 2] }
    }
  }
])
```

---

**Q10. Explain the $lookup stage for joining collections.**

```javascript
// Basic $lookup (like a LEFT OUTER JOIN)
db.orders.aggregate([
  {
    $lookup: {
      from: "users",            // collection to join
      localField: "userId",    // field in orders
      foreignField: "_id",     // field in users
      as: "customer"           // output array field name
    }
  },
  // customer is an array — unwrap it if expecting one match
  { $unwind: { path: "$customer", preserveNullAndEmpty: true } }
])
// Result: each order now has a "customer" field with user document embedded

// $lookup with pipeline (MongoDB 3.6+) — more powerful
db.orders.aggregate([
  {
    $lookup: {
      from: "products",
      let: { itemIds: "$items.productId" },  // pass local variables
      pipeline: [
        {
          $match: {
            $expr: { $in: ["$_id", "$$itemIds"] }  // reference local var with $$
          }
        },
        {
          $project: { name: 1, price: 1, category: 1 }  // only return needed fields
        }
      ],
      as: "productDetails"
    }
  }
])

// Multi-collection lookup chain
db.orders.aggregate([
  { $match: { status: "pending" } },
  // Join customers
  { $lookup: { from: "users", localField: "userId", foreignField: "_id", as: "customer" } },
  { $unwind: "$customer" },
  // Join products for each item
  { $unwind: "$items" },
  { $lookup: { from: "products", localField: "items.productId", foreignField: "_id", as: "items.product" } },
  { $unwind: "$items.product" }
])
```

---

**Q11. What does $unwind do and when would you use it?**

```javascript
// $unwind deconstructs an array field
// For each array element, it creates a SEPARATE document

// Before $unwind:
{ _id: 1, user: "Alice", tags: ["mongodb", "atlas", "nodejs"] }

// After { $unwind: "$tags" }:
{ _id: 1, user: "Alice", tags: "mongodb" }
{ _id: 1, user: "Alice", tags: "atlas" }
{ _id: 1, user: "Alice", tags: "nodejs" }
// One document became THREE documents!

// Use cases:
// 1. Count frequency of tags across all posts
db.posts.aggregate([
  { $unwind: "$tags" },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 10 }
])

// 2. Normalize embedded order items for per-item analytics
db.orders.aggregate([
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.productId",
      totalSold: { $sum: "$items.qty" },
      revenue: { $sum: { $multiply: ["$items.price", "$items.qty"] } }
    }
  }
])

// $unwind options
db.posts.aggregate([
  {
    $unwind: {
      path: "$tags",
      includeArrayIndex: "tagIndex",       // include the array index as a field
      preserveNullAndEmpty: true           // keep docs where array is null/empty
    }
  }
])
// Without preserveNullAndEmpty: docs with empty/null arrays are REMOVED
```

---

**Q12. Explain $project in the aggregation pipeline.**

```javascript
// $project in aggregation — more powerful than find() projection
db.users.aggregate([
  {
    $project: {
      // Include/exclude (same as find projection)
      name: 1,
      email: 1,
      _id: 0,

      // Rename fields
      userName: "$username",

      // Computed fields
      nameLength: { $strLenCP: "$name" },
      nameUpper: { $toUpper: "$name" },
      fullName: { $concat: ["$firstName", " ", "$lastName"] },

      // Conditional fields
      isAdult: { $gte: ["$age", 18] },

      // Nested document computation
      "location.display": {
        $concat: ["$address.city", ", ", "$address.state"]
      },

      // Array operations
      tagCount: { $size: "$tags" },
      firstTag: { $arrayElemAt: ["$tags", 0] },
      lastTag: { $arrayElemAt: ["$tags", -1] },

      // Type conversion
      ageStr: { $toString: "$age" },
      priceNum: { $toDouble: "$price" }
    }
  }
])
```

---

**Q13. What are $addFields and $set stages in aggregation?**

```javascript
// $addFields — adds new fields (KEEPS all existing fields)
// $set is an alias for $addFields (MongoDB 4.2+)
db.products.aggregate([
  {
    $addFields: {
      // New fields added alongside existing ones
      discountedPrice: { $multiply: ["$price", 0.9] },
      inStock: { $gt: ["$stock", 0] },
      totalValue: { $multiply: ["$price", "$stock"] }
    }
  },
  { $match: { inStock: true, discountedPrice: { $lt: 50 } } }
])

// Difference from $project:
// $project: controls WHICH fields appear (can exclude existing ones)
// $addFields/$set: ADDS fields, all existing fields are preserved

// $replaceRoot — replace the entire document with a specified value
db.orders.aggregate([
  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: [
          "$shippingAddress",   // use shippingAddress as the root
          { orderId: "$_id", total: "$total" }  // merge with selected fields
        ]
      }
    }
  }
])

// $replaceWith — alias for $replaceRoot (MongoDB 4.2+)
db.users.aggregate([
  { $replaceWith: { $mergeObjects: ["$$ROOT", { processed: true }] } }
])
```

---

**Q14. Explain the $bucket and $facet stages.**

```javascript
// $bucket — categorize documents into specified ranges
db.products.aggregate([
  {
    $bucket: {
      groupBy: "$price",
      boundaries: [0, 25, 50, 100, 250, 500, 1000],  // must be increasing
      default: "Other",   // bucket for values outside boundaries
      output: {
        count: { $sum: 1 },
        products: { $push: "$name" },
        avgPrice: { $avg: "$price" }
      }
    }
  }
])
// Result:
// { _id: 0, count: 12, ... }    -- prices [0, 25)
// { _id: 25, count: 34, ... }   -- prices [25, 50)
// { _id: "Other", count: 3 }    -- prices >= 1000

// $bucketAuto — automatically create N equal buckets
db.products.aggregate([
  { $bucketAuto: { groupBy: "$price", buckets: 5 } }
])

// $facet — run MULTIPLE aggregation pipelines in parallel on the SAME input
// Essential for faceted search (e-commerce filters)
db.products.aggregate([
  { $match: { category: "electronics", inStock: true } },  // shared filter
  {
    $facet: {
      // Facet 1: price distribution
      priceRanges: [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 50, 100, 250, 500],
            default: "500+"
          }
        }
      ],
      // Facet 2: brands
      brands: [
        { $group: { _id: "$brand", count: { $sum: 1 } } },
        { $sort: { count: -1 } },
        { $limit: 10 }
      ],
      // Facet 3: ratings
      ratings: [
        { $group: { _id: "$rating", count: { $sum: 1 } } },
        { $sort: { _id: 1 } }
      ],
      // Facet 4: total count (for pagination)
      totalCount: [
        { $count: "total" }
      ]
    }
  }
])
// Returns ONE document with all four facets
```

---

**Q15. What is the $merge stage and how does it differ from $out?**

```javascript
// $out — writes pipeline results to a collection (REPLACES collection)
db.orders.aggregate([
  { $match: { year: 2024 } },
  { $group: { _id: "$customerId", totalSpent: { $sum: "$amount" } } },
  { $out: "customer_totals_2024" }  // REPLACES the entire collection atomically
])
// WARNING: existing data in customer_totals_2024 is GONE

// $merge — writes pipeline results with merging options (MongoDB 4.2+)
db.orders.aggregate([
  { $group: { _id: "$customerId", totalSpent: { $sum: "$amount" } } },
  {
    $merge: {
      into: "customer_totals",     // target collection
      on: "_id",                   // merge key (must be unique indexed)
      whenMatched: "merge",        // options: "replace", "keepExisting", "merge", "fail", "pipeline"
      whenNotMatched: "insert"     // options: "insert", "discard", "fail"
    }
  }
])
// Merges new results into existing collection
// Existing documents not in results are KEPT (unlike $out)

// $merge with pipeline for complex merge logic
db.newSales.aggregate([
  {
    $merge: {
      into: "allSales",
      on: "orderId",
      whenMatched: [
        { $set: { lastUpdated: "$$NOW", mergedFrom: "newSales" } }
      ],
      whenNotMatched: "insert"
    }
  }
])
```

---

**Q16. Explain key aggregation expressions: $map, $filter, $reduce.**

```javascript
// $map — transform each element of an array
db.orders.aggregate([
  {
    $project: {
      // Apply a 10% discount to each item's price
      discountedItems: {
        $map: {
          input: "$items",
          as: "item",
          in: {
            sku: "$$item.sku",
            originalPrice: "$$item.price",
            discountedPrice: { $multiply: ["$$item.price", 0.9] }
          }
        }
      }
    }
  }
])

// $filter — return array elements matching a condition
db.products.aggregate([
  {
    $project: {
      name: 1,
      // Only include items with qty > 0
      availableItems: {
        $filter: {
          input: "$items",
          as: "item",
          cond: { $gt: ["$$item.qty", 0] }
        }
      }
    }
  }
])

// $reduce — accumulate array into a single value
db.orders.aggregate([
  {
    $project: {
      // Sum all item totals
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
      }
    }
  }
])
```

---

**Q17. What is the $lookup with $search (Atlas Search integration)?**

```javascript
// Atlas Search integration in aggregation pipeline
// $search must be the FIRST stage in the pipeline
db.products.aggregate([
  {
    $search: {
      index: "products_search_index",
      text: {
        query: "wireless headphones",
        path: ["name", "description"],
        fuzzy: { maxEdits: 1 }
      }
    }
  },
  // Continue with regular aggregation stages
  { $match: { inStock: true, price: { $lte: 200 } } },
  {
    $project: {
      name: 1,
      price: 1,
      score: { $meta: "searchScore" }   // relevance score from Atlas Search
    }
  },
  { $sort: { score: -1 } },
  { $limit: 20 }
])
```

---

**Q18. How does `explain()` work and what output should you look for?**

```javascript
// explain() modes
db.users.find({ email: "alice@example.com" }).explain()
// "queryPlanner" mode — shows chosen plan, not execution stats

db.users.find({ email: "alice@example.com" }).explain("executionStats")
// Shows actual execution: documents examined, index used, time taken

db.users.find({ email: "alice@example.com" }).explain("allPlansExecution")
// Shows ALL candidate plans and why the chosen plan won

// For aggregation
db.orders.aggregate([
  { $match: { status: "pending" } },
  { $group: { _id: "$userId", count: { $sum: 1 } } }
]).explain("executionStats")

// Key fields to analyze in explain output:
{
  queryPlanner: {
    winningPlan: {
      stage: "IXSCAN",       // GOOD: index scan (vs COLLSCAN = bad)
      indexName: "email_1"
    }
  },
  executionStats: {
    executionTimeMillis: 2,      // actual execution time
    totalDocsExamined: 1,        // documents scanned from storage
    totalKeysExamined: 1,        // index keys examined
    nReturned: 1                 // documents returned to client
  }
}

// RED FLAGS in explain output:
// stage: "COLLSCAN"  → no index used → add an index
// totalDocsExamined >> nReturned  → inefficient (examining many, returning few)
// totalKeysExamined >> nReturned  → index exists but has low selectivity

// Efficiency ratio: nReturned / totalDocsExamined should be close to 1.0
// ratio = 1/10000 means examining 10000 docs to return 1 → bad selectivity
```

---

**Q19. What is the $count and $sort stage in aggregation?**

```javascript
// $count — returns total count of documents as a field
db.users.aggregate([
  { $match: { active: true } },
  { $count: "activeUserCount" }  // returns: { activeUserCount: 12345 }
])

// $sort — sort documents by one or more fields
db.products.aggregate([
  { $sort: { price: -1, name: 1 } }  // price descending, name ascending
])

// $sort with computed fields (must be projected first)
db.products.aggregate([
  { $addFields: { totalValue: { $multiply: ["$price", "$stock"] } } },
  { $sort: { totalValue: -1 } }
])

// $sortByCount — group by field and sort by count (shorthand)
db.orders.aggregate([
  { $sortByCount: "$status" }
])
// Equivalent to:
// [{ $group: { _id: "$status", count: { $sum: 1 } } }, { $sort: { count: -1 } }]

// $skip and $limit for pagination
db.products.aggregate([
  { $match: { category: "electronics" } },
  { $sort: { price: 1 } },
  { $skip: 40 },       // skip first 40 (page 3 with pageSize 20)
  { $limit: 20 }       // return 20
])
```

---

**Q20. Explain $lookup with a correlated subquery (pipeline version).**

```javascript
// Use case: for each user, get their last 3 orders
db.users.aggregate([
  {
    $lookup: {
      from: "orders",
      let: { userId: "$_id" },      // local variable from outer doc
      pipeline: [
        {
          $match: {
            $expr: { $eq: ["$userId", "$$userId"] }  // correlated condition
          }
        },
        { $sort: { orderDate: -1 } },
        { $limit: 3 },              // limit INSIDE the lookup — crucial for performance!
        { $project: { orderId: 1, amount: 1, orderDate: 1, _id: 0 } }
      ],
      as: "recentOrders"
    }
  },
  {
    $project: {
      name: 1,
      email: 1,
      recentOrders: 1
    }
  }
])

// $lookup with $unwind optimization:
// When combined, MongoDB can optimize: instead of materializing the array
// and then unwinding it, it processes each document individually
db.orders.aggregate([
  {
    $lookup: {
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product"
    }
  },
  { $unwind: "$product" }   // MongoDB may optimize the lookup+unwind together
])
```

---

**Q21. What are aggregation pipeline optimization techniques?**

```javascript
// 1. Put $match FIRST — filter early, reduce document count for later stages
// BAD:
db.orders.aggregate([
  { $project: { ... } },  // projects ALL docs before filtering
  { $match: { status: "active" } }
])
// GOOD:
db.orders.aggregate([
  { $match: { status: "active" } },  // filter first (can use index!)
  { $project: { ... } }
])

// 2. Put $sort before $limit (and the optimizer will use the sortKey for limit)
db.orders.aggregate([
  { $sort: { orderDate: -1 } },
  { $limit: 10 }
])
// MongoDB optimizer: uses index for sort+limit → only scans 10 docs

// 3. Limit $lookup results with pipeline
db.users.aggregate([
  { $lookup: { from: "orders", ..., pipeline: [{ $limit: 5 }] } }
])

// 4. Use $project to reduce document size before expensive stages
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $project: { items: 1, userId: 1 } },  // drop large fields before $unwind
  { $unwind: "$items" }
])

// 5. allowDiskUse for large pipelines
db.bigCollection.aggregate([...], { allowDiskUse: true })
// Allows intermediary stages to spill to disk when 100MB memory limit is exceeded

// 6. $merge/$out — avoid repeatedly loading data; persist intermediate results
// Run heavy aggregation once, cache results in a separate collection
```

---

**Q22. What are the $limit and $skip pipeline stages and their performance implications?**

```javascript
// $limit — returns only the first N documents
db.products.aggregate([
  { $match: { category: "electronics" } },
  { $sort: { price: 1 } },
  { $limit: 10 }
])
// Performance: if $limit follows $sort and an index covers the sort,
// MongoDB stops scanning after N documents → very efficient

// $skip — skips the first N documents
db.products.aggregate([
  { $skip: 100 },
  { $limit: 10 }
])
// Performance: $skip + $limit works but $skip is SLOW at high values
// Must scan and discard all skipped documents
// Page 1000 with pageSize 10 → skip 9990 documents → examines 10000 docs

// Prefer keyset/cursor pagination for large offsets:
// Instead of skip(9990), use _id or sort key from last document:
db.products.aggregate([
  { $match: { _id: { $gt: lastSeenId } } },  // start from last seen
  { $sort: { _id: 1 } },
  { $limit: 10 }
])
// This is O(log n) regardless of page number
```

---

**Q23. What is the $sample stage and when would you use it?**

```javascript
// $sample — randomly selects N documents from the collection or pipeline
// Two behaviors depending on whether N > 5% of collection:

// Small sample (N <= 5% of total): uses pseudo-random cursor
db.products.aggregate([{ $sample: { size: 10 } }])

// Large sample (N > 5%): full collection scan then random sort
// (Use sparingly on large collections)

// Use cases:
// A/B testing — randomly assign users to control/test groups
db.users.aggregate([
  { $match: { isActive: true } },
  { $sample: { size: 10000 } },     // random 10k users for testing
  { $set: { testGroup: "B" } }
])

// Data sampling for ML training sets
db.events.aggregate([
  { $match: { eventType: "purchase" } },
  { $sample: { size: 50000 } }
])

// Random survey selection
db.customers.aggregate([
  { $match: { totalPurchases: { $gte: 3 } } },
  { $sample: { size: 100 } }
])
```

---

**Q24. How does the $graphLookup stage work for recursive queries?**

```javascript
// $graphLookup — recursive lookup for graph/tree data
// Use case: organizational hierarchy, category trees, friend-of-friends

// Employee hierarchy document:
{
  _id: ObjectId("..."),
  name: "Alice",
  reportsTo: ObjectId("manager_id")  // self-reference
}

// Get all direct and indirect reports for a manager
db.employees.aggregate([
  { $match: { name: "CEO" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "reportsTo",
      as: "directAndIndirectReports",
      maxDepth: 5,                      // limit recursion depth
      depthField: "level",              // add depth level to each result
      restrictSearchWithMatch: {        // optional filter on results
        isActive: true
      }
    }
  }
])

// Category tree: get all ancestors of a category
db.categories.aggregate([
  { $match: { _id: leafCategoryId } },
  {
    $graphLookup: {
      from: "categories",
      startWith: "$parentId",
      connectFromField: "parentId",
      connectToField: "_id",
      as: "ancestors"
    }
  }
])
```

---

**Q25. What are conditional aggregation expressions: $cond, $ifNull, $switch?**

```javascript
// $cond — ternary if-then-else
db.orders.aggregate([
  {
    $project: {
      orderId: 1,
      total: 1,
      // Add discount tier based on total
      discountTier: {
        $cond: {
          if: { $gte: ["$total", 1000] },
          then: "gold",
          else: {
            $cond: {
              if: { $gte: ["$total", 500] },
              then: "silver",
              else: "standard"
            }
          }
        }
      }
    }
  }
])

// $switch — cleaner multi-case conditional
db.products.aggregate([
  {
    $project: {
      name: 1,
      priceCategory: {
        $switch: {
          branches: [
            { case: { $lt: ["$price", 25] }, then: "budget" },
            { case: { $lt: ["$price", 100] }, then: "mid-range" },
            { case: { $lt: ["$price", 500] }, then: "premium" }
          ],
          default: "luxury"
        }
      }
    }
  }
])

// $ifNull — return alternative value if field is null or doesn't exist
db.users.aggregate([
  {
    $project: {
      displayName: { $ifNull: ["$nickname", "$firstName"] },
      country: { $ifNull: ["$address.country", "Unknown"] }
    }
  }
])

// $coalesce — similar to $ifNull but for multiple alternatives
db.users.aggregate([
  {
    $project: {
      contactEmail: {
        $ifNull: ["$workEmail", { $ifNull: ["$personalEmail", "$email"] }]
      }
    }
  }
])
```

---

**Q26. What is the $setWindowFields stage (MongoDB 5.0+)?**

```javascript
// $setWindowFields — window functions (like SQL window functions)
// Calculate running totals, moving averages, rankings without grouping

db.sales.aggregate([
  {
    $setWindowFields: {
      partitionBy: "$salesRep",          // group (like PARTITION BY in SQL)
      sortBy: { saleDate: 1 },           // order within partition
      output: {
        // Running total
        runningTotal: {
          $sum: "$amount",
          window: { documents: ["unbounded", "current"] }  // from start to current
        },
        // Moving average (last 7 days)
        movingAvg7d: {
          $avg: "$amount",
          window: { range: [-7, 0], unit: "day" }
        },
        // Rank within partition
        rank: { $rank: {} },
        denseRank: { $denseRank: {} },
        rowNumber: { $documentNumber: {} }
      }
    }
  }
])
```

---

**Q27. Explain `$expr` in the aggregation context and its use in queries.**

```javascript
// $expr — use aggregation expressions in query stage ($match, find())

// Compare two fields in the same document
db.orders.find({
  $expr: { $gt: ["$total", "$expectedTotal"] }
})

// Complex expressions as query conditions
db.products.find({
  $expr: {
    $and: [
      { $gt: ["$stock", 0] },
      { $lt: ["$price", { $multiply: ["$costPrice", 2] }] }  // price < 2x cost
    ]
  }
})

// Using $expr in aggregation $match
db.inventory.aggregate([
  {
    $match: {
      $expr: {
        $lt: ["$currentStock", "$reorderPoint"]
      }
    }
  },
  { $project: { sku: 1, currentStock: 1, reorderPoint: 1 } }
])

// $expr can use variables and let bindings in $lookup pipelines
db.orders.aggregate([
  {
    $lookup: {
      from: "discounts",
      let: { orderTotal: "$total", userId: "$userId" },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$userId", "$$userId"] },
                { $gte: ["$$orderTotal", "$minimumOrderValue"] }
              ]
            }
          }
        }
      ],
      as: "applicableDiscounts"
    }
  }
])
```

---

**Q28. What are $densify and $fill stages (MongoDB 6.0+)?**

```javascript
// $densify — fill gaps in a sequence (numeric or date)
// Use case: time series with missing data points
db.sensorReadings.aggregate([
  {
    $densify: {
      field: "timestamp",
      range: {
        step: 1,
        unit: "hour",
        bounds: [ISODate("2024-01-01T00:00:00Z"), ISODate("2024-01-02T00:00:00Z")]
      },
      partitionByFields: ["sensorId"]   // densify within each partition
    }
  },
  // After densify, some docs have no 'temperature' value — fill it
  {
    $fill: {
      partitionByFields: ["sensorId"],
      sortBy: { timestamp: 1 },
      output: {
        temperature: {
          method: "linear"    // linear interpolation between known values
          // OR: method: "locf" — Last Observation Carried Forward
        }
      }
    }
  }
])
```

---

**Q29. What are $lookup limitations and when should you NOT use it?**

```javascript
// $lookup limitations:
// 1. Performance: $lookup is NOT like a SQL JOIN with query optimizer
//    The pipeline is executed for EACH document in the left collection

// 2. The "from" collection must be in the same database
//    (unless using Atlas Data Federation)

// 3. Cannot use $lookup in a transaction with more than one shard
//    (must be on the same shard or use unsharded target collection)

// 4. $lookup target with array field creates a large result document
//    Can exceed 16MB document size limit

// When NOT to use $lookup:
// - High-frequency, low-latency queries (the overhead is significant)
// - When the "join" result would exceed 16MB
// - Real-time dashboards querying millions of documents

// When to use $lookup:
// - Batch processing and reporting
// - Data export/migration scripts
// - Analytical queries that run periodically
// - Admin dashboards with moderate traffic

// Better alternatives for relational data at scale:
// 1. Embed related data (denormalize) — no $lookup needed
// 2. Application-level join (two separate queries in app code)
// 3. Atlas Data Federation for cross-database queries
// 4. Precompute joined results in a summary collection ($merge)

// Example: application-level join (often faster for hot paths)
async function getOrderWithCustomer(orderId) {
  const order = await db.orders.findOne({ _id: orderId })
  const customer = await db.users.findOne({ _id: order.userId })
  return { ...order, customer }
}
```

---

**Q30. What are $searchMeta and $search stages for analytics on search results?**

```javascript
// $search (detailed in Chapter 8) — full-text search
// $searchMeta — returns metadata about Atlas Search results (facets, counts)
// without returning the actual documents

db.products.aggregate([
  {
    $searchMeta: {
      index: "products_index",
      facet: {
        operator: {
          text: { query: "laptop", path: "description" }
        },
        facets: {
          categoryFacet: {
            type: "string",
            path: "category",
            numBuckets: 5
          },
          priceFacet: {
            type: "number",
            path: "price",
            boundaries: [0, 100, 500, 1000],
            default: "other"
          }
        }
      }
    }
  }
])
// Returns: { count: { lowerBound: 1432 }, facet: { categoryFacet: [...], priceFacet: [...] } }
// Does NOT return actual product documents — only counts/facets
// Use this for building sidebar filters without loading all products
```
