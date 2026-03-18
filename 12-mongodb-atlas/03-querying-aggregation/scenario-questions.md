# Chapter 03 — Querying & Aggregation: Scenario Questions

---

**S1. Build a monthly revenue report by product category for 2024, showing total revenue, average order value, and order count. Sort by revenue descending.**

```javascript
db.orders.aggregate([
  // Filter to 2024 completed orders
  {
    $match: {
      status: "completed",
      orderDate: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2025-01-01T00:00:00Z")
      }
    }
  },

  // Flatten the items array
  { $unwind: "$items" },

  // Group by month + category
  {
    $group: {
      _id: {
        year: { $year: "$orderDate" },
        month: { $month: "$orderDate" },
        category: "$items.category"
      },
      totalRevenue: {
        $sum: { $multiply: ["$items.price", "$items.qty"] }
      },
      orderCount: { $sum: 1 },
      uniqueOrders: { $addToSet: "$_id" }   // deduplicate (each item unwind creates multiple docs per order)
    }
  },

  // Add average order value
  {
    $addFields: {
      avgOrderValue: { $divide: ["$totalRevenue", { $size: "$uniqueOrders" }] }
    }
  },

  // Sort by revenue desc
  { $sort: { totalRevenue: -1 } },

  // Shape output
  {
    $project: {
      _id: 0,
      year: "$_id.year",
      month: "$_id.month",
      category: "$_id.category",
      totalRevenue: { $round: ["$totalRevenue", 2] },
      orderCount: { $size: "$uniqueOrders" },
      avgOrderValue: { $round: ["$avgOrderValue", 2] }
    }
  }
])
```

---

**S2. You're building an e-commerce search page. Users can filter by category, price range, brand, and rating. Implement faceted search that returns both the filtered products AND the filter counts simultaneously.**

```javascript
// Using Atlas Search (optimal for production)
db.products.aggregate([
  {
    $search: {
      index: "products_search",
      compound: {
        must: [
          { text: { query: userSearchTerm, path: ["name", "description"] } }
        ],
        filter: [
          { text: { query: selectedCategory, path: "category" } }
        ]
      }
    }
  },
  // Facets via $searchMeta (run alongside results in same request)
])

// Using regular aggregation (no Atlas Search)
const pipeline = [
  // Apply user filters
  {
    $match: {
      ...(category && { category }),
      ...(minPrice || maxPrice ? {
        price: {
          ...(minPrice && { $gte: NumberDecimal(minPrice.toString()) }),
          ...(maxPrice && { $lte: NumberDecimal(maxPrice.toString()) })
        }
      } : {}),
      ...(brands?.length && { brand: { $in: brands } }),
      ...(minRating && { rating: { $gte: minRating } }),
      inStock: true
    }
  },

  // $facet: run multiple pipelines on filtered data simultaneously
  {
    $facet: {
      // Paginated results
      products: [
        { $sort: { [sortField]: sortDirection } },
        { $skip: (page - 1) * pageSize },
        { $limit: pageSize },
        { $project: { name: 1, price: 1, brand: 1, rating: 1, thumbnail: 1 } }
      ],

      // Total count for pagination
      totalCount: [{ $count: "count" }],

      // Brand facet counts
      brandFacets: [
        { $group: { _id: "$brand", count: { $sum: 1 } } },
        { $sort: { count: -1 } },
        { $limit: 20 }
      ],

      // Price range distribution
      priceRanges: [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 25, 50, 100, 250, 500],
            default: "500+",
            output: { count: { $sum: 1 } }
          }
        }
      ],

      // Rating distribution
      ratingDistribution: [
        { $group: { _id: { $floor: "$rating" }, count: { $sum: 1 } } },
        { $sort: { _id: -1 } }
      ]
    }
  }
]

const [result] = await db.products.aggregate(pipeline).toArray()
return {
  products: result.products,
  total: result.totalCount[0]?.count || 0,
  facets: {
    brands: result.brandFacets,
    priceRanges: result.priceRanges,
    ratings: result.ratingDistribution
  }
}
```

---

**S3. Find the top 10 customers by total spend in Q1 2024, including their name, total orders, total spent, and average order value.**

```javascript
db.orders.aggregate([
  // Filter Q1 2024 completed orders
  {
    $match: {
      status: "completed",
      orderDate: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2024-04-01T00:00:00Z")
      }
    }
  },

  // Aggregate by customer
  {
    $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$total" },
      orderCount: { $sum: 1 },
      orders: { $push: { orderId: "$_id", total: "$total", date: "$orderDate" } }
    }
  },

  // Sort by total spent
  { $sort: { totalSpent: -1 } },
  { $limit: 10 },

  // Join with customer data
  {
    $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "customerInfo",
      pipeline: [
        { $project: { name: 1, email: 1, tier: 1 } }
      ]
    }
  },
  { $unwind: "$customerInfo" },

  // Final shape
  {
    $project: {
      _id: 0,
      customerId: "$_id",
      name: "$customerInfo.name",
      email: "$customerInfo.email",
      tier: "$customerInfo.tier",
      totalSpent: { $round: ["$totalSpent", 2] },
      orderCount: 1,
      avgOrderValue: {
        $round: [{ $divide: ["$totalSpent", "$orderCount"] }, 2]
      }
    }
  }
])
```

---

**S4. You have a blog with posts and nested comments. Find all posts that have comments containing the word "spam" in their text (case-insensitive).**

```javascript
// Schema: { _id, title, content, comments: [{ text, author, createdAt }] }

db.posts.find({
  "comments.text": { $regex: /spam/i }
})

// To find the SPECIFIC comments (not just flag the posts):
db.posts.aggregate([
  // Only posts with at least one spam comment
  {
    $match: {
      "comments.text": { $regex: /spam/i }
    }
  },

  // Unwind to work with individual comments
  { $unwind: "$comments" },

  // Keep only spam comments
  {
    $match: {
      "comments.text": { $regex: /spam/i }
    }
  },

  // Group back to see posts with their spam comments
  {
    $group: {
      _id: "$_id",
      postTitle: { $first: "$title" },
      spamComments: {
        $push: {
          text: "$comments.text",
          author: "$comments.author",
          createdAt: "$comments.createdAt"
        }
      },
      spamCount: { $sum: 1 }
    }
  },

  { $sort: { spamCount: -1 } }
])
```

---

**S5. Build a query that finds users who have made purchases in ALL of these categories: "electronics", "clothing", and "books".**

```javascript
// Approach 1: Using $all on a grouped field
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $unwind: "$items" },
  {
    $group: {
      _id: "$customerId",
      purchasedCategories: { $addToSet: "$items.category" }
    }
  },
  {
    $match: {
      purchasedCategories: { $all: ["electronics", "clothing", "books"] }
    }
  },
  // Join with user details
  {
    $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },
  {
    $project: {
      _id: 0,
      userId: "$_id",
      name: "$user.name",
      email: "$user.email",
      purchasedCategories: 1
    }
  }
])

// Approach 2: Using $setIsSubset
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $unwind: "$items" },
  {
    $group: {
      _id: "$customerId",
      purchasedCategories: { $addToSet: "$items.category" }
    }
  },
  {
    $match: {
      $expr: {
        $setIsSubset: [
          ["electronics", "clothing", "books"],
          "$purchasedCategories"
        ]
      }
    }
  }
])
```

---

**S6. Find the 5 most frequently co-purchased products (products that appear together in the same order the most).**

```javascript
// This requires generating pairs within each order's items
db.orders.aggregate([
  { $match: { "items.1": { $exists: true } } },  // orders with at least 2 items

  // Get all items as an array
  {
    $project: {
      productIds: "$items.productId"
    }
  },

  // Generate all pairs using $reduce
  {
    $project: {
      pairs: {
        $reduce: {
          input: { $range: [0, { $size: "$productIds" }] },
          initialValue: [],
          in: {
            $concatArrays: [
              "$$value",
              {
                $map: {
                  input: { $range: [{ $add: ["$$this", 1] }, { $size: "$productIds" }] },
                  as: "j",
                  in: {
                    // Sort pair so A+B and B+A are treated the same
                    $cond: {
                      if: { $lt: [{ $arrayElemAt: ["$productIds", "$$this"] }, { $arrayElemAt: ["$productIds", "$$j"] }] },
                      then: {
                        p1: { $arrayElemAt: ["$productIds", "$$this"] },
                        p2: { $arrayElemAt: ["$productIds", "$$j"] }
                      },
                      else: {
                        p1: { $arrayElemAt: ["$productIds", "$$j"] },
                        p2: { $arrayElemAt: ["$productIds", "$$this"] }
                      }
                    }
                  }
                }
              }
            ]
          }
        }
      }
    }
  },

  { $unwind: "$pairs" },
  { $group: { _id: "$pairs", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 5 },

  // Enrich with product names
  {
    $lookup: {
      from: "products",
      localField: "_id.p1",
      foreignField: "_id",
      as: "product1"
    }
  },
  {
    $lookup: {
      from: "products",
      localField: "_id.p2",
      foreignField: "_id",
      as: "product2"
    }
  },
  { $unwind: "$product1" },
  { $unwind: "$product2" },
  {
    $project: {
      _id: 0,
      product1: "$product1.name",
      product2: "$product2.name",
      count: 1
    }
  }
])
```

---

**S7. You have an IoT system with sensor readings. Find the hourly average temperature for each sensor over the last 7 days, filling in missing hours with null.**

```javascript
// Sample document: { sensorId: "S01", temperature: 22.5, ts: ISODate("...") }

const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000)

db.sensorReadings.aggregate([
  {
    $match: {
      ts: { $gte: sevenDaysAgo }
    }
  },

  // Group by sensor + hour
  {
    $group: {
      _id: {
        sensorId: "$sensorId",
        year: { $year: "$ts" },
        month: { $month: "$ts" },
        day: { $dayOfMonth: "$ts" },
        hour: { $hour: "$ts" }
      },
      avgTemp: { $avg: "$temperature" },
      minTemp: { $min: "$temperature" },
      maxTemp: { $max: "$temperature" },
      readingCount: { $sum: 1 }
    }
  },

  // Reconstruct timestamp for the hour slot
  {
    $addFields: {
      hourSlot: {
        $dateFromParts: {
          year: "$_id.year",
          month: "$_id.month",
          day: "$_id.day",
          hour: "$_id.hour"
        }
      }
    }
  },

  // Sort for densification
  { $sort: { "_id.sensorId": 1, hourSlot: 1 } },

  // Densify to fill missing hours (MongoDB 6.0+)
  // For older MongoDB, generate expected hours in app code
  {
    $densify: {
      field: "hourSlot",
      partitionByFields: ["_id.sensorId"],
      range: { step: 1, unit: "hour", bounds: [sevenDaysAgo, new Date()] }
    }
  },

  // Project clean output
  {
    $project: {
      _id: 0,
      sensorId: "$_id.sensorId",
      hourSlot: 1,
      avgTemp: { $round: ["$avgTemp", 2] },
      minTemp: { $round: ["$minTemp", 2] },
      maxTemp: { $round: ["$maxTemp", 2] },
      readingCount: 1
    }
  }
])
```

---

**S8. Implement a "trending posts" query that ranks posts by a score combining recency and engagement (likes + comments weighted by time decay).**

```javascript
// Score formula: (likes * 2 + comments) / (hoursOld + 2)^1.5
// This makes recent posts score higher even with fewer interactions

const now = new Date()

db.posts.aggregate([
  // Only look at posts from last 7 days
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) },
      status: "published"
    }
  },

  // Calculate trend score
  {
    $addFields: {
      hoursOld: {
        $divide: [
          { $subtract: [now, "$createdAt"] },
          3600000    // milliseconds in an hour
        ]
      }
    }
  },
  {
    $addFields: {
      engagementScore: {
        $add: [
          { $multiply: [{ $ifNull: ["$likeCount", 0] }, 2] },
          { $ifNull: ["$commentCount", 0] }
        ]
      }
    }
  },
  {
    $addFields: {
      trendScore: {
        $divide: [
          "$engagementScore",
          {
            $pow: [
              { $add: ["$hoursOld", 2] },
              1.5
            ]
          }
        ]
      }
    }
  },

  { $sort: { trendScore: -1 } },
  { $limit: 20 },

  {
    $project: {
      title: 1,
      author: 1,
      likeCount: 1,
      commentCount: 1,
      createdAt: 1,
      trendScore: { $round: ["$trendScore", 4] },
      hoursOld: { $round: ["$hoursOld", 1] }
    }
  }
])
```

---

**S9. Given a `transactions` collection with deposit and withdrawal events, calculate the running balance for a specific account.**

```javascript
// Transaction document:
// { _id, accountId, type: "deposit"|"withdrawal", amount: Decimal128, ts: Date }

db.transactions.aggregate([
  // Filter for specific account
  { $match: { accountId: ObjectId("account_id") } },

  // Sort chronologically
  { $sort: { ts: 1, _id: 1 } },

  // Calculate signed amount (negative for withdrawals)
  {
    $addFields: {
      signedAmount: {
        $cond: {
          if: { $eq: ["$type", "withdrawal"] },
          then: { $multiply: ["$amount", -1] },
          else: "$amount"
        }
      }
    }
  },

  // Use $setWindowFields for running total (MongoDB 5.0+)
  {
    $setWindowFields: {
      partitionBy: "$accountId",
      sortBy: { ts: 1, _id: 1 },
      output: {
        runningBalance: {
          $sum: "$signedAmount",
          window: { documents: ["unbounded", "current"] }
        },
        transactionNumber: { $documentNumber: {} }
      }
    }
  },

  {
    $project: {
      _id: 0,
      transactionId: "$_id",
      type: 1,
      amount: 1,
      runningBalance: 1,
      transactionNumber: 1,
      ts: 1
    }
  }
])

// For MongoDB < 5.0: use $reduce on sorted array
db.transactions.aggregate([
  { $match: { accountId: ObjectId("account_id") } },
  { $sort: { ts: 1 } },
  { $group: {
    _id: "$accountId",
    transactions: { $push: { type: "$type", amount: "$amount", ts: "$ts" } }
  }},
  {
    $project: {
      history: {
        $reduce: {
          input: "$transactions",
          initialValue: { balance: NumberDecimal("0"), history: [] },
          in: {
            balance: {
              $add: [
                "$$value.balance",
                { $cond: [{ $eq: ["$$this.type", "deposit"] }, "$$this.amount", { $multiply: ["$$this.amount", -1] }] }
              ]
            },
            history: {
              $concatArrays: [
                "$$value.history",
                [{ $mergeObjects: ["$$this", { balance: { $add: ["$$value.balance", "$$this.amount"] } }] }]
              ]
            }
          }
        }
      }
    }
  }
])
```

---

**S10. Write a pipeline that identifies duplicate user accounts based on matching email addresses and phone numbers.**

```javascript
db.users.aggregate([
  // Group by email to find duplicates
  {
    $group: {
      _id: { $toLower: "$email" },     // case-insensitive email comparison
      userIds: { $push: "$_id" },
      count: { $sum: 1 },
      users: {
        $push: {
          _id: "$_id",
          name: "$name",
          email: "$email",
          phone: "$phone",
          createdAt: "$createdAt"
        }
      }
    }
  },

  // Only show groups with duplicates
  { $match: { count: { $gt: 1 } } },

  // Flatten for easier review
  { $unwind: "$users" },

  {
    $group: {
      _id: "$_id",  // email group
      duplicateAccounts: { $push: "$users" },
      count: { $first: "$count" }
    }
  },

  { $sort: { count: -1 } },

  // Also find phone duplicates
]).toArray()

// Combined email + phone duplicate detection
db.users.aggregate([
  {
    $group: {
      _id: null,
      users: { $push: "$$ROOT" }
    }
  },
  { $unwind: { path: "$users", includeArrayIndex: "idx" } },
  // This approach is memory-intensive — better to use a separate query
])

// More practical: two separate queries
const emailDupes = await db.users.aggregate([
  { $group: { _id: { $toLower: "$email" }, ids: { $push: "$_id" }, count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
]).toArray()

const phoneDupes = await db.users.aggregate([
  { $match: { phone: { $exists: true, $ne: null } } },
  { $group: { _id: "$phone", ids: { $push: "$_id" }, count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
]).toArray()
```

---

**S11. Implement a "cohort retention analysis" — for users who signed up in January 2024, what percentage were still active in each subsequent month?**

```javascript
db.users.aggregate([
  // Find January 2024 cohort
  {
    $match: {
      createdAt: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2024-02-01T00:00:00Z")
      }
    }
  },

  // Get all activity months for these users
  {
    $lookup: {
      from: "user_activity",
      let: { userId: "$_id" },
      pipeline: [
        {
          $match: {
            $expr: { $eq: ["$userId", "$$userId"] },
            ts: { $gte: ISODate("2024-01-01T00:00:00Z") }
          }
        },
        {
          $group: {
            _id: {
              year: { $year: "$ts" },
              month: { $month: "$ts" }
            }
          }
        }
      ],
      as: "activeMonths"
    }
  },

  // Count cohort size
  {
    $group: {
      _id: null,
      cohortSize: { $sum: 1 },
      // Collect all activity months
      allActivityMonths: { $push: "$activeMonths" }
    }
  },

  // Flatten activity months
  { $unwind: "$allActivityMonths" },
  { $unwind: "$allActivityMonths" },

  {
    $group: {
      _id: "$allActivityMonths._id",  // group by year+month
      cohortSize: { $first: "$cohortSize" },
      activeUsers: { $sum: 1 }
    }
  },

  // Calculate retention rate
  {
    $project: {
      year: "$_id.year",
      month: "$_id.month",
      cohortSize: 1,
      activeUsers: 1,
      retentionRate: {
        $round: [{ $multiply: [{ $divide: ["$activeUsers", "$cohortSize"] }, 100] }, 1]
      }
    }
  },

  { $sort: { year: 1, month: 1 } }
])
```

---

**S12. You need to find all documents where the `tags` array contains more than 5 elements.**

```javascript
// Option 1: $where (SLOW - avoid)
db.posts.find({ $where: "this.tags.length > 5" })  // bad: no index, JS execution

// Option 2: $expr with $size (better)
db.posts.find({ $expr: { $gt: [{ $size: "$tags" }, 5] } })

// Option 3: Use existence of index N (array index 5 exists = at least 6 elements)
db.posts.find({ "tags.5": { $exists: true } })
// This can use an index on the tags field!

// Performance comparison:
// $where: full collection scan + JS execution per doc = slowest
// $expr + $size: collection scan with server-side computation = medium
// "tags.5" $exists: can use multikey index on tags = fastest

// Create index for this query pattern:
db.posts.createIndex({ "tags.5": 1 })
// Partial index on documents with at least 6 tags:
db.posts.createIndex(
  { likeCount: -1 },
  { partialFilterExpression: { "tags.5": { $exists: true } } }
)

// Count documents with more than 5 tags efficiently
db.posts.countDocuments({ "tags.5": { $exists: true } })
```

---

**S13. You have orders with a "status history" array. Find all orders where the status changed from "processing" to "cancelled" within 24 hours.**

```javascript
// Order document:
// { _id, statusHistory: [{ status, changedAt, changedBy }] }

db.orders.aggregate([
  // Filter orders that have both "processing" and "cancelled" statuses
  {
    $match: {
      "statusHistory.status": { $all: ["processing", "cancelled"] }
    }
  },

  // For each order, find the processing→cancelled transition
  {
    $addFields: {
      processingEntry: {
        $arrayElemAt: [
          {
            $filter: {
              input: "$statusHistory",
              as: "entry",
              cond: { $eq: ["$$entry.status", "processing"] }
            }
          },
          0   // first processing entry
        ]
      },
      cancelledEntry: {
        $arrayElemAt: [
          {
            $filter: {
              input: "$statusHistory",
              as: "entry",
              cond: { $eq: ["$$entry.status", "cancelled"] }
            }
          },
          0   // first cancelled entry
        ]
      }
    }
  },

  // Calculate time between processing and cancelled
  {
    $addFields: {
      timeToCancellation: {
        $subtract: ["$cancelledEntry.changedAt", "$processingEntry.changedAt"]
      }
    }
  },

  // Filter: cancellation within 24 hours (86,400,000 ms)
  {
    $match: {
      timeToCancellation: { $gt: 0, $lte: 86400000 }
    }
  },

  {
    $project: {
      orderId: "$_id",
      processingAt: "$processingEntry.changedAt",
      cancelledAt: "$cancelledEntry.changedAt",
      timeToCancellationHours: {
        $round: [{ $divide: ["$timeToCancellation", 3600000] }, 2]
      }
    }
  },

  { $sort: { timeToCancellation: 1 } }
])
```

---

**S14. Implement a geospatial query to find the 10 nearest restaurants to a given location.**

```javascript
// Restaurant document:
// { _id, name, cuisine, location: { type: "Point", coordinates: [longitude, latitude] } }

// Create 2dsphere index for geospatial queries
db.restaurants.createIndex({ location: "2dsphere" })

// Find 10 nearest restaurants within 5km
db.restaurants.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-97.7431, 30.2672] },  // Austin, TX [lng, lat]
      distanceField: "distanceMeters",
      maxDistance: 5000,          // 5km in meters
      spherical: true,
      query: { cuisine: "italian" }  // optional: filter by cuisine
    }
  },
  { $limit: 10 },
  {
    $project: {
      name: 1,
      cuisine: 1,
      distanceMeters: { $round: ["$distanceMeters", 0] },
      distanceKm: { $round: [{ $divide: ["$distanceMeters", 1000] }, 2] },
      location: 1
    }
  }
])

// Alternative: $near in find() (no distance in results)
db.restaurants.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [-97.7431, 30.2672] },
      $maxDistance: 5000
    }
  }
}).limit(10)

// Find restaurants within a polygon (delivery zone)
db.restaurants.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [-97.75, 30.25],
          [-97.73, 30.25],
          [-97.73, 30.28],
          [-97.75, 30.28],
          [-97.75, 30.25]   // close the polygon
        ]]
      }
    }
  }
})
```

---

**S15. Write an aggregation pipeline that detects anomalies in daily sales data — days where sales were more than 2 standard deviations from the mean.**

```javascript
db.sales.aggregate([
  // Group by day
  {
    $group: {
      _id: {
        year: { $year: "$saleDate" },
        month: { $month: "$saleDate" },
        day: { $dayOfMonth: "$saleDate" }
      },
      dailyTotal: { $sum: "$amount" },
      transactionCount: { $sum: 1 }
    }
  },

  // Calculate stats over all days
  {
    $group: {
      _id: null,
      days: {
        $push: {
          date: "$_id",
          total: "$dailyTotal",
          txnCount: "$transactionCount"
        }
      },
      meanSales: { $avg: "$dailyTotal" },
      stdDevSales: { $stdDevPop: "$dailyTotal" }
    }
  },

  // Flatten days back out, keeping mean and stddev
  { $unwind: "$days" },

  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: [
          "$days",
          { mean: "$meanSales", stdDev: "$stdDevSales" }
        ]
      }
    }
  },

  // Calculate z-score for each day
  {
    $addFields: {
      zScore: {
        $divide: [
          { $subtract: ["$total", "$mean"] },
          "$stdDev"
        ]
      }
    }
  },

  // Flag anomalies (|z| > 2)
  {
    $addFields: {
      isAnomaly: { $gt: [{ $abs: "$zScore" }, 2] },
      anomalyType: {
        $cond: {
          if: { $gt: ["$total", "$mean"] },
          then: "spike",
          else: "drop"
        }
      }
    }
  },

  { $match: { isAnomaly: true } },
  { $sort: { "date.year": 1, "date.month": 1, "date.day": 1 } }
])
```

---

**S16. Implement a "recommendations" query that finds products frequently viewed by users who also viewed a specific product.**

```javascript
// View event: { userId, productId, viewedAt }

async function getProductRecommendations(targetProductId, limit = 10) {
  return db.viewEvents.aggregate([
    // Find users who viewed the target product
    { $match: { productId: targetProductId } },

    // Get all other products those users viewed
    {
      $lookup: {
        from: "viewEvents",
        let: { viewerIds: "$userId" },
        pipeline: [
          {
            $match: {
              $expr: { $eq: ["$userId", "$$viewerIds"] },
              productId: { $ne: targetProductId }  // exclude the target product
            }
          },
          { $project: { productId: 1, _id: 0 } }
        ],
        as: "otherViews"
      }
    },

    // Count how many of our target viewers also viewed each other product
    { $unwind: "$otherViews" },
    {
      $group: {
        _id: "$otherViews.productId",
        viewerOverlap: { $sum: 1 }
      }
    },
    { $sort: { viewerOverlap: -1 } },
    { $limit: limit },

    // Enrich with product details
    {
      $lookup: {
        from: "products",
        localField: "_id",
        foreignField: "_id",
        as: "product"
      }
    },
    { $unwind: "$product" },
    {
      $project: {
        productId: "$_id",
        name: "$product.name",
        price: "$product.price",
        viewerOverlap: 1,
        _id: 0
      }
    }
  ]).toArray()
}
```

---

**S17. Using `explain()`, demonstrate the performance difference between a query with and without an index.**

```javascript
// Collection: 10 million user documents
// { _id, email, country, age, createdAt }

// WITHOUT index:
db.users.find({ email: "alice@example.com" }).explain("executionStats")
// Output:
{
  queryPlanner: {
    winningPlan: {
      stage: "COLLSCAN"   // ← Collection scan: reads ALL 10M documents
    }
  },
  executionStats: {
    executionTimeMillis: 4523,        // 4.5 SECONDS
    totalDocsExamined: 10000000,      // examined ALL 10M docs
    totalKeysExamined: 0,             // no index keys used
    nReturned: 1,                     // found 1 document
    // Ratio: 1 returned / 10M examined = terrible
  }
}

// Create index:
db.users.createIndex({ email: 1 }, { unique: true })

// WITH index:
db.users.find({ email: "alice@example.com" }).explain("executionStats")
// Output:
{
  queryPlanner: {
    winningPlan: {
      stage: "IXSCAN",         // ← Index scan
      indexName: "email_1"
    }
  },
  executionStats: {
    executionTimeMillis: 1,           // 1ms!
    totalDocsExamined: 1,             // examined exactly 1 doc
    totalKeysExamined: 1,             // used 1 index key
    nReturned: 1,
    // Ratio: 1 returned / 1 examined = perfect
  }
}

// Performance analysis:
// Without index: O(n) — scans every document
// With index: O(log n) — B-tree traversal
// For 10M documents: 4523ms → 1ms = 4,523x faster

// Checking aggregation explain:
db.orders.aggregate([
  { $match: { customerId: ObjectId("...") } }
]).explain("executionStats")
```

---

**S18. Write a query using $lookup to find all products in a user's wishlist along with their current availability and price.**

```javascript
// User document: { _id, name, wishlist: [ObjectId, ObjectId, ...] }
// Product document: { _id, name, price, stock, category }

db.users.aggregate([
  // Find the specific user
  { $match: { _id: userId } },

  // Look up wishlist products
  {
    $lookup: {
      from: "products",
      localField: "wishlist",       // array of ObjectIds
      foreignField: "_id",          // matches product _id
      as: "wishlistProducts",
      pipeline: [
        {
          $project: {
            name: 1,
            price: 1,
            category: 1,
            inStock: { $gt: ["$stock", 0] },
            stock: 1,
            thumbnail: 1
          }
        }
      ]
    }
  },

  // Add availability metadata
  {
    $addFields: {
      wishlistProductCount: { $size: "$wishlist" },
      inStockCount: {
        $size: {
          $filter: {
            input: "$wishlistProducts",
            as: "p",
            cond: "$$p.inStock"
          }
        }
      }
    }
  },

  {
    $project: {
      name: 1,
      wishlistProducts: 1,
      wishlistProductCount: 1,
      inStockCount: 1,
      outOfStockCount: { $subtract: ["$wishlistProductCount", "$inStockCount"] }
    }
  }
]).next()

// Find users whose wishlist contains out-of-stock items that just came back in stock
db.users.aggregate([
  {
    $lookup: {
      from: "products",
      localField: "wishlist",
      foreignField: "_id",
      as: "wishlistProducts"
    }
  },
  {
    $match: {
      "wishlistProducts": {
        $elemMatch: {
          stock: { $gt: 0 },
          wasOutOfStock: true  // flag set when stock went to 0
        }
      }
    }
  },
  { $project: { name: 1, email: 1 } }
  // → send restock notifications
])
```

---

**S19. Find all users who have been inactive for more than 90 days but have a positive balance.**

```javascript
const ninetyDaysAgo = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000)

db.users.find({
  $and: [
    // Inactive for 90+ days
    {
      $or: [
        { lastLoginAt: { $lt: ninetyDaysAgo } },
        { lastLoginAt: { $exists: false } }   // never logged in
      ]
    },
    // Positive balance
    { balance: { $gt: 0 } },
    // Not already flagged
    { flaggedForDormancy: { $ne: true } }
  ]
})

// More efficient with a partial index:
db.users.createIndex(
  { lastLoginAt: 1, balance: 1 },
  { partialFilterExpression: { balance: { $gt: 0 }, flaggedForDormancy: { $ne: true } } }
)

// Aggregation version with additional stats:
db.users.aggregate([
  {
    $match: {
      $or: [
        { lastLoginAt: { $lt: ninetyDaysAgo } },
        { lastLoginAt: { $exists: false } }
      ],
      balance: { $gt: 0 },
      flaggedForDormancy: { $ne: true }
    }
  },
  {
    $addFields: {
      daysSinceLogin: {
        $divide: [
          { $subtract: [new Date(), { $ifNull: ["$lastLoginAt", "$createdAt"] }] },
          86400000   // milliseconds per day
        ]
      }
    }
  },
  { $sort: { balance: -1 } },  // highest balance first
  { $limit: 1000 }
])
```

---

**S20. Build a query to compute a 7-day moving average of daily sign-ups.**

```javascript
db.users.aggregate([
  // Group by day
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%d", date: "$createdAt" }
      },
      dailySignups: { $sum: 1 }
    }
  },

  { $sort: { _id: 1 } },

  // Add date field for window function
  { $addFields: { date: { $dateFromString: { dateString: "$_id" } } } },

  // 7-day moving average using $setWindowFields
  {
    $setWindowFields: {
      sortBy: { date: 1 },
      output: {
        movingAvg7d: {
          $avg: "$dailySignups",
          window: { range: [-6, 0], unit: "day" }  // current day + 6 days back = 7 days
        },
        movingTotal7d: {
          $sum: "$dailySignups",
          window: { range: [-6, 0], unit: "day" }
        }
      }
    }
  },

  {
    $project: {
      _id: 0,
      date: "$_id",
      dailySignups: 1,
      movingAvg7d: { $round: ["$movingAvg7d", 1] },
      movingTotal7d: 1
    }
  }
])
```
