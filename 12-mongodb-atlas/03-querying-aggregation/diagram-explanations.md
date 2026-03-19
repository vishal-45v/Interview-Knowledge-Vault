# Chapter 03 — Querying & Aggregation: Diagram Explanations

---

## Diagram 1: Aggregation Pipeline Stages Flow

```
AGGREGATION PIPELINE — DOCUMENT FLOW
══════════════════════════════════════════════════════════════════════

Collection: orders (10,000,000 documents)
     │
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 1: $match { status: "completed", year: 2024 }           │
│  INPUT:  10,000,000 docs                                        │
│  OUTPUT:  2,500,000 docs (25% pass filter)                      │
│  Uses index on { status: 1, year: 1 }                          │
└─────────────────────────────────────────────────────────────────┘
     │ 2,500,000 docs
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 2: $unwind "$items"                                      │
│  INPUT:  2,500,000 docs (each with avg 4 items)                │
│  OUTPUT:  10,000,000 item docs                                  │
│  Each array element becomes a separate document                 │
└─────────────────────────────────────────────────────────────────┘
     │ 10,000,000 item docs
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 3: $group { _id: "$items.category" }                    │
│  INPUT:  10,000,000 docs                                        │
│  OUTPUT:  15 docs (one per unique category)                     │
│  $sum accumulates revenue per category                         │
└─────────────────────────────────────────────────────────────────┘
     │ 15 docs
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 4: $sort { revenue: -1 }                                 │
│  INPUT/OUTPUT:  15 docs                                         │
│  Sorts 15 docs in memory — trivial                             │
└─────────────────────────────────────────────────────────────────┘
     │ 15 docs (sorted)
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 5: $limit 5                                              │
│  INPUT:  15 docs                                                │
│  OUTPUT:  5 docs (top 5 categories)                             │
└─────────────────────────────────────────────────────────────────┘
     │ 5 docs
     ▼
┌─────────────────────────────────────────────────────────────────┐
│  Stage 6: $project { category: "$_id", revenue: 1, _id: 0 }   │
│  INPUT/OUTPUT:  5 docs                                          │
│  Reshape output                                                 │
└─────────────────────────────────────────────────────────────────┘
     │ 5 docs (final result)
     ▼
     CLIENT

KEY INSIGHT: $match + index at Stage 1 reduces 10M → 2.5M
             Only 2.5M docs go through expensive $unwind stage
             NOT 10M (as would happen if $match were at the end)
```

---

## Diagram 2: $lookup Join Diagram

```
$LOOKUP: LEFT OUTER JOIN EQUIVALENT
══════════════════════════════════════════════════════════════════════

ORDERS collection (left/source)     USERS collection (right/target)
─────────────────────────────       ─────────────────────────────────
{ _id: O1, userId: U1, amt: 50 }    { _id: U1, name: "Alice" }
{ _id: O2, userId: U2, amt: 30 }    { _id: U2, name: "Bob" }
{ _id: O3, userId: U1, amt: 80 }    { _id: U3, name: "Carol" }
{ _id: O4, userId: U9, amt: 20 }    (U9 doesn't exist)


$lookup: { from: "users", localField: "userId", foreignField: "_id", as: "customer" }


RESULT:
──────────────────────────────────────────────────────────
{ _id: O1, userId: U1, amt: 50, customer: [{ _id: U1, name: "Alice" }] }
{ _id: O2, userId: U2, amt: 30, customer: [{ _id: U2, name: "Bob" }]   }
{ _id: O3, userId: U1, amt: 80, customer: [{ _id: U1, name: "Alice" }] }
{ _id: O4, userId: U9, amt: 20, customer: []  }  ← empty array (no match = LEFT OUTER)


EXECUTION MODEL (important for performance!):
──────────────────────────────────────────────────────────
For each order document:
  1. Take localField value (userId)
  2. Query: db.users.find({ _id: userId })  ← uses index on users._id
  3. Append results as array in "as" field

→ N orders = N separate lookups into users collection
→ With index: N × O(log M) where M = users count
→ Without index on foreignField: N × O(M) = VERY SLOW

PERFORMANCE COMPARISON:
──────────────────────────────────────────────────────────
Approach                            Latency (100K orders, 1M users)
────────────────────────────────    ──────────────────────────────
Embedded userId in order doc         1 query → 5ms
$lookup (with users._id index)       N lookups → 200ms
$lookup (without index on fKey)      N × full scan → MINUTES
App-level join (2 queries)           2 queries → 15ms
```

---

## Diagram 3: $unwind Behavior Diagram

```
$UNWIND BEHAVIOR
══════════════════════════════════════════════════════════════════════

BEFORE $unwind:
┌─────────────────────────────────────────────────┐
│ { _id: 1, title: "Post A",                      │
│   tags: ["mongodb", "database", "nosql"] }      │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│ { _id: 2, title: "Post B", tags: ["redis"] }    │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│ { _id: 3, title: "Post C", tags: [] }           │
└─────────────────────────────────────────────────┘

After { $unwind: "$tags" }:
┌──────────────────────────────────────┐
│ { _id: 1, title: "Post A", tags: "mongodb" }   │
├──────────────────────────────────────┤
│ { _id: 1, title: "Post A", tags: "database" }  │
├──────────────────────────────────────┤
│ { _id: 1, title: "Post A", tags: "nosql" }     │
├──────────────────────────────────────┤
│ { _id: 2, title: "Post B", tags: "redis" }     │
└──────────────────────────────────────┘
// Note: Post C (empty array) is REMOVED by default!

After { $unwind: { path: "$tags", preserveNullAndEmpty: true } }:
┌──────────────────────────────────────┐
│ { _id: 1, title: "Post A", tags: "mongodb" }   │
│ { _id: 1, title: "Post A", tags: "database" }  │
│ { _id: 1, title: "Post A", tags: "nosql" }     │
│ { _id: 2, title: "Post B", tags: "redis" }     │
│ { _id: 3, title: "Post C" }                    │ ← preserved
└──────────────────────────────────────┘

With includeArrayIndex: "tagPosition":
{ _id: 1, title: "Post A", tags: "mongodb",  tagPosition: 0 }
{ _id: 1, title: "Post A", tags: "database", tagPosition: 1 }
{ _id: 1, title: "Post A", tags: "nosql",    tagPosition: 2 }

CARDINALITY EXPLOSION:
────────────────────────────────────
Documents before $unwind:  2
Average array size:         3 tags
Documents after $unwind:   6  (2 × 3)

For production data:
100,000 orders × avg 5 items = 500,000 docs after $unwind
500,000 × avg 3 events = 1,500,000 docs after second $unwind

→ ALWAYS put $match BEFORE $unwind to minimize docs entering the stage
```

---

## Diagram 4: $facet Parallel Sub-Pipelines

```
$FACET: MULTIPLE PIPELINES, SINGLE PASS
══════════════════════════════════════════════════════════════════════

Input (after $match): 50,000 electronics products
         │
         │  $facet materializes these 50K docs once
         │
         ├──────────────────────────────────────────────┐
         │                                              │
         ▼                                              ▼
  Sub-pipeline "results":                    Sub-pipeline "brandCounts":
  ┌─────────────────────────┐               ┌─────────────────────────┐
  │ $sort: { price: 1 }     │               │ $group: { _id: "$brand" │
  │ $skip: 0                │               │   count: {$sum: 1} }    │
  │ $limit: 20              │               │ $sort: { count: -1 }    │
  └─────────────────────────┘               │ $limit: 20              │
  Output: 20 products                       └─────────────────────────┘
                                            Output: 20 brand counts
         │                                              │
         └──────────────────────┬───────────────────────┘
                                │
                                ▼
                    Single output document:
                    {
                      results: [20 products],
                      brandCounts: [20 brand objects],
                      priceRanges: [6 price bucket objects],
                      total: [{ count: 50000 }]
                    }

vs WITHOUT $facet: 4 separate queries
  Query 1: paginated products   → 5ms
  Query 2: brand counts         → 80ms
  Query 3: price buckets        → 70ms
  Query 4: total count          → 50ms
  Total: 205ms (sequential) or ~80ms (parallel)

WITH $facet: 1 query
  All 4 computations in one pipeline: ~100ms
  (+ less overhead: one network round trip, consistent snapshot)
```

---

## Diagram 5: Query Operator Decision Tree

```
CHOOSING THE RIGHT QUERY OPERATOR
══════════════════════════════════════════════════════════════════════

Is the field a simple scalar?
    ├── YES: Compare a value?
    │       ├── Equal:          { field: value }  or { $eq: value }
    │       ├── Not equal:      { $ne: value }
    │       ├── Range:          { $gt: x, $lte: y }
    │       ├── In a list:      { $in: [v1, v2, v3] }
    │       ├── Not in a list:  { $nin: [v1, v2] }
    │       ├── Exists:         { $exists: true/false }
    │       ├── Type check:     { $type: "string" }
    │       └── Pattern:        { $regex: /pattern/ }
    │
    └── NO: Is the field an array?
            ├── Contains ALL values:   { $all: [v1, v2] }
            ├── Array length:          { "arr.N": { $exists: true } }  ← "has N+1 elements"
            └── Element conditions:
                    ├── Single condition:  { "arr.field": value }
                    └── MULTIPLE conditions on SAME element: { arr: { $elemMatch: { f1: v1, f2: v2 } } }

Combining conditions?
    ├── All must match:    Implicit (multiple fields) or $and: [...]
    ├── Any must match:    $or: [...]
    ├── None must match:   $nor: [...]
    └── Invert condition:  { $not: { ... } }

Comparing fields to each other?
    └── Use $expr: { $gt: ["$field1", "$field2"] }

Complex computation as filter?
    ├── Avoid: $where (JS, no index, slow)
    └── Use: $expr with aggregation expressions
```

---

## Diagram 6: explain() Output Analysis

```
HOW TO READ EXPLAIN("executionStats") OUTPUT
══════════════════════════════════════════════════════════════════════

{
  queryPlanner: {
    namespace: "mydb.users",
    winningPlan: {
      stage: "FETCH",             ← Fetch documents from storage
      inputStage: {
        stage: "IXSCAN",          ← Index scan (GOOD)
        // vs: stage: "COLLSCAN" (BAD — no index used)
        indexName: "email_1",
        isMultiKey: false,        ← Multikey = index on array field
        direction: "forward"
      }
    },
    rejectedPlans: [...]         ← Other plans MongoDB considered
  },
  executionStats: {
    executionSuccess: true,
    nReturned: 1,                 ← Documents returned to client
    executionTimeMillis: 2,       ← Total time in milliseconds
    totalKeysExamined: 1,         ← Index keys read
    totalDocsExamined: 1,         ← Documents fetched from storage

    // DIAGNOSIS:
    // totalKeysExamined == nReturned:  Perfect selectivity (index is exact)
    // totalKeysExamined >> nReturned:  Index has low selectivity
    // totalDocsExamined >> nReturned:  Post-filter needed after index scan
    // totalDocsExamined == 0:          COVERED QUERY (all data from index alone)
  }
}

INTERPRETING KEY RATIOS:
──────────────────────────────────────────────────────
nReturned / totalDocsExamined:
  ≈ 1.0   → Excellent: almost everything examined was returned
  0.1     → OK: examining 10x more than needed (consider better index)
  0.001   → Poor: examining 1000x more than needed (bad filter selectivity or missing index)
  0.0001  → Critical: COLLSCAN on large collection

totalDocsExamined = 0 → Covered query! (no document reads, just index reads)

PIPELINE STAGES (BAD → GOOD):
COLLSCAN → IXSCAN → FETCH → PROJECTION
COLLSCAN alone:     full collection scan (worst)
IXSCAN + FETCH:     uses index, then fetches documents (normal)
IXSCAN + no FETCH:  covered query (best — no document I/O)
```
