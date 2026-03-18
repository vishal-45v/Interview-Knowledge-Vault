# Chapter 05 — Schema Design: Diagram Explanations

---

## Diagram 1: Embedding vs Referencing — Side-by-Side Comparison

```
EMBEDDING (document contains related data)
══════════════════════════════════════════════════════════════════════

posts collection:
┌────────────────────────────────────────────────────────────────────┐
│ {                                                                   │
│   _id: ObjectId("abc"),                                            │
│   title: "My Post",                                                │
│   body: "...",                                                      │
│   author: {                                                         │
│     _id: ObjectId("u1"),                                           │
│     name: "Alice",          ← embedded — no join needed            │
│     avatarUrl: "..."                                                │
│   },                                                                │
│   comments: [               ← embedded array (bounded max 20)      │
│     { author: "Bob", body: "Great!" },                             │
│     { author: "Carol", body: "Agreed!" }                           │
│   ],                                                                │
│   commentsCount: 2                                                  │
│ }                                                                   │
└────────────────────────────────────────────────────────────────────┘
         │
         ▼  ONE QUERY to load the entire post with author and comments
       Result: post + author + 20 comments in a single document fetch


REFERENCING (documents link to each other)
══════════════════════════════════════════════════════════════════════

posts collection:             users collection:
┌────────────────────────┐   ┌────────────────────────────────────────┐
│ {                      │   │ {                                      │
│   _id: ObjectId("abc"),│   │   _id: ObjectId("u1"),                │
│   title: "My Post",   │   │   name: "Alice",                      │
│   authorId: ObjectId──┼──▶│   email: "alice@example.com",         │
│            ("u1"),    │   │   bio: "...",                          │
│   commentsCount: 347  │   │   followersCount: 4821                 │
│ }                      │   │ }                                      │
└────────────────────────┘   └────────────────────────────────────────┘
         │                             ▲
         │                   QUERY 2: db.users.findOne({ _id: authorId })
         ▼  QUERY 1: db.posts.findOne({ _id: postId })

comments collection:
┌──────────────────────────────────────────────────────────────────┐
│ { _id: ObjectId("..."), postId: ObjectId("abc"), body: "..." }   │
│ { _id: ObjectId("..."), postId: ObjectId("abc"), body: "..." }   │
│ ... (347 total comments)                                         │
└──────────────────────────────────────────────────────────────────┘
         ▲
QUERY 3: db.comments.find({ postId: ObjectId("abc") }).limit(20)

Total: 3 queries to render one post page with author + first 20 comments
```

---

## Diagram 2: Schema Design Decision Tree

```
SCHEMA DESIGN DECISION TREE
══════════════════════════════════════════════════════════════════════

                       New Relationship to Model
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
         What is the cardinality?
                 │
     ┌───────────┼────────────────┐
     ▼           ▼                ▼
    1:1        1:few          1:many or
               (<100)          unbounded
     │           │                │
     ▼           ▼                ▼
  EMBED        EMBED           REFERENCE
  in parent   in parent       (separate
  document    (bounded        collection)
              array)
     │           │                │
     ▼           ▼                ▼
  Is the      Does the         How often
  child data  child ever       is both
  always      grow beyond      parent AND
  needed?     1000 items?      child needed
     │           │             together?
     ▼           ▼                │
    YES    YES → REFERENCE    ┌───┴───┐
     ▼         NO → EMBED     ▼       ▼
  Keep       (bounded)     ALWAYS  SOMETIMES
  embedded                  │         │
                            ▼         ▼
                       EMBED         REFERENCE
                       subset        + embed
                       (Extended     small subset
                       Reference)    ($lookup or
                                     app join for
                                     full data)

ADDITIONAL QUESTIONS:
──────────────────────────────────────────────────────────────────────

Q: Does the embedded data change frequently?
   YES → Could cause high write amplification if embedded in many docs
         Consider: reference + cache computed values

Q: Do you ever need the child without the parent?
   YES → Must reference (or embed + separate collection)
   NO  → Embedding is fine

Q: Is there a N:M relationship?
   YES → Separate junction collection (like a join table)
   Or   → Embed arrays on each side if sets are bounded

Q: Will the document approach 16MB?
   YES → STOP. Use reference. Check all arrays for upper bounds.
```

---

## Diagram 3: Bucket Pattern — Document Reduction

```
BUCKET PATTERN: FROM MILLIONS TO THOUSANDS
══════════════════════════════════════════════════════════════════════

BEFORE (one document per reading):

Time:   10:00:00  10:00:01  10:00:02  ...  10:59:59
        ┌───────┐ ┌───────┐ ┌───────┐     ┌───────┐
        │S1 22.5│ │S1 22.6│ │S1 22.4│ ... │S1 22.8│
        └───────┘ └───────┘ └───────┘     └───────┘
         doc_1     doc_2     doc_3          doc_3600

Per sensor per hour: 3,600 documents
10,000 sensors × 24 hours = 240,000 documents/day
10,000 sensors × 24 hours × 365 days = 87,600,000 documents/year PER SENSOR TYPE


AFTER (Bucket Pattern — one document per sensor per hour):

                           ┌───────────────────────────────────────┐
10:00:00 - 10:59:59  →    │ {                                     │
                           │   sensorId: "S-001",                  │
                           │   windowStart: ISODate("2024-03..."), │
                           │   nReadings: 3600,                    │
                           │   stats: {                            │
                           │     tempMin: 21.8,   ← pre-computed   │
                           │     tempMax: 23.4,   ← pre-computed   │
                           │     tempAvg: 22.525  ← pre-computed   │
                           │   },                                  │
                           │   anomalies: [                        │
                           │     { t: 1840, temp: 41.2 }          │
                           │   ]         ← only outliers stored    │
                           │ }                                     │
                           └───────────────────────────────────────┘
                                           1 document

Per sensor per hour: 1 document (was 3,600)
10,000 sensors × 24 hours = 240,000 documents/day (was 864,000,000)
REDUCTION: 3,600x fewer documents

QUERY COMPARISON:
──────────────────────────────────────────────────────────────────────

"What was the average temp for sensor S-001 last week?"

Without Bucket Pattern:
  Match: 7 × 24 × 3600 = 604,800 documents scanned
  Aggregate: $avg over 604,800 readings

With Bucket Pattern:
  Match: 7 × 24 = 168 bucket documents scanned
  Aggregate: $avg over 168 pre-computed stats
  3,600x fewer documents processed!
```

---

## Diagram 4: Extended Reference Pattern

```
EXTENDED REFERENCE PATTERN
══════════════════════════════════════════════════════════════════════

PROBLEM: Order page needs customer name + address for display.

OPTION A — Full Normalization (reference only):
┌──────────────────────────────┐    ┌──────────────────────────────────┐
│ orders                       │    │ customers                        │
│ { _id, customerId → ─────────┼───▶│ { _id, name, email, address,     │
│   items: [...],             │    │   preferences, history, ... }    │
│   total: 59.98 }             │    └──────────────────────────────────┘
└──────────────────────────────┘
Problem: EVERY order page view = 2 queries
         If customer address changes → must handle "which address was used?"

OPTION B — Full Embedding (copy all fields):
┌──────────────────────────────────────────────────────────────────┐
│ orders                                                            │
│ { _id,                                                            │
│   customer: {  ← FULL COPY                                       │
│     _id, name, email, address, preferences, history...           │
│   },                                                              │
│   items: [...], total: 59.98 }                                   │
└──────────────────────────────────────────────────────────────────┘
Problem: Large docs, stale data if customer preferences change,
         stores data that's never used on order page

OPTION C — Extended Reference Pattern (CORRECT):
┌───────────────────────────────────────────────────────────────────┐
│ orders                                                             │
│ {                                                                  │
│   _id: ObjectId("..."),                                           │
│   customer: {                                                      │
│     _id: ObjectId("u1"),        ← reference (for full lookup)    │
│     name: "Alice Johnson",      ← SNAPSHOT: frozen at order time │
│     shippingAddress: {          ← SNAPSHOT: address used         │
│       street: "123 Main St",                                      │
│       city: "New York"                                            │
│     }                           ← only the subset we need        │
│   },                                                              │
│   items: [...],                                                   │
│   total: NumberDecimal("59.98")                                   │
│ }                                                                  │
└───────────────────────────────────────────────────────────────────┘
         │
         ▼
1 query = complete order page render
Customer moves to LA next month → old order still shows "New York" (correct!)
Need full customer profile? → look up by customer._id
```

---

## Diagram 5: Schema Versioning — Migration Without Downtime

```
SCHEMA VERSIONING PATTERN
══════════════════════════════════════════════════════════════════════

Initial state: all documents are v1 (no schemaVersion field)

users collection:
┌─────────────────────────────────────────────────┐
│ { _id: 1, fullName: "Alice Johnson",            │
│   email: "alice@example.com" }         ← v1     │
│ { _id: 2, fullName: "Bob Smith",               │
│   email: "bob@example.com" }           ← v1     │
│ { _id: 3, fullName: "Carol Davis",             │
│   email: "carol@example.com" }         ← v1     │
└─────────────────────────────────────────────────┘

STEP 1: Deploy new app code (handles both v1 and v2)
        No DB changes yet — zero downtime

STEP 2: New inserts use v2 format
┌─────────────────────────────────────────────────┐
│ { _id: 1, fullName: "Alice",   email: "..." }   │ ← still v1
│ { _id: 2, fullName: "Bob",     email: "..." }   │ ← still v1
│ { _id: 3, fullName: "Carol",   email: "..." }   │ ← still v1
│ { _id: 4, firstName: "Dave",   lastName: "Lee", │
│           email: "...", schemaVersion: 2 }      │ ← NEW v2
└─────────────────────────────────────────────────┘

STEP 3: Lazy migration — upgrade on access
        User 1 (Alice) logs in → app reads → detects v1 → upgrades:

┌─────────────────────────────────────────────────┐
│ { _id: 1, firstName: "Alice", lastName: "Johnson",│
│   email: "...", schemaVersion: 2 }    ← UPGRADED │ ← now v2
│ { _id: 2, fullName: "Bob",   email: "..." }      │ ← still v1
│ { _id: 3, fullName: "Carol", email: "..." }      │ ← still v1
│ { _id: 4, firstName: "Dave", lastName: "Lee",    │
│           email: "...", schemaVersion: 2 }        │ ← v2
└─────────────────────────────────────────────────┘

MONTHS LATER: Most active users are v2, inactive users remain v1

VERSION DISTRIBUTION OVER TIME:
Week 0:   [████████████████████] 100% v1
Week 4:   [████████████████░░░░]  80% v1, 20% v2  (active users upgraded)
Week 12:  [████████░░░░░░░░░░░░]  40% v1, 60% v2
Week 26:  [████░░░░░░░░░░░░░░░░]  20% v1, 80% v2
Week 52:  [██░░░░░░░░░░░░░░░░░░]  10% v1, 90% v2  (mostly inactive accounts)

STEP 4 (optional): Run batch migration for the 10% long tail
                   Target only: { schemaVersion: { $exists: false } }
                   Run during off-peak hours in batches of 1,000
```

---

## Diagram 6: Tree Patterns Comparison

```
HIERARCHY: Electronics → Computers → Laptops → Gaming Laptops
══════════════════════════════════════════════════════════════════════

THE HIERARCHY:
                     [Electronics]
                    /             \
             [Computers]       [Phones]
            /          \
        [Laptops]   [Desktops]
           /
    [Gaming Laptops]

PATTERN 1: Parent Reference
──────────────────────────────────────────────────────────────────────
{ _id: "electronics",    parentId: null }
{ _id: "computers",      parentId: "electronics" }
{ _id: "laptops",        parentId: "computers" }
{ _id: "gaming-laptops", parentId: "laptops" }

Find parent:          db.cats.findOne({ _id: "laptops" }) → parentId = "computers"
Find children:        db.cats.find({ parentId: "laptops" })
Find all ancestors:   4 queries (hop up the tree)    ← SLOW

PATTERN 2: Array of Ancestors
──────────────────────────────────────────────────────────────────────
{ _id: "gaming-laptops", parentId: "laptops",
  ancestors: ["electronics", "computers", "laptops"] }

Find all ancestors:   Read ancestors array directly    ← FAST (inline)
Breadcrumb:           "Electronics > Computers > Laptops > Gaming Laptops"
                      → one document read!
Find all descendants: db.cats.find({ ancestors: "computers" })  ← FAST
Insert new node:      FAST — just set parent + ancestors
Move node mid-tree:   Update ALL descendants' ancestor arrays   ← SLOW

PATTERN 3: Materialized Path
──────────────────────────────────────────────────────────────────────
{ _id: "gaming-laptops", path: "/electronics/computers/laptops/gaming-laptops" }

Find subtree:     db.cats.find({ path: { $regex: "^/electronics/computers" } })
                  FAST if anchored with ^ (uses index)
                  SLOW if unanchored (no index)
Depth:            path.split("/").length - 1

PATTERN 4: $graphLookup (computed, no redundant storage)
──────────────────────────────────────────────────────────────────────
All docs have only: { _id: "gaming-laptops", parentId: "laptops" }

db.categories.aggregate([
  { $match: { _id: "electronics" } },
  { $graphLookup: {
      from: "categories",
      startWith: "$_id",
      connectFromField: "_id",
      connectToField: "parentId",
      as: "allDescendants",
      maxDepth: 5
  }}
])

No redundant data → always consistent
Computed on every query → no storage overhead, higher compute cost

COMPARISON TABLE:
──────────────────────────────────────────────────────────────────────
Pattern            Find Parent  Find Children  Find Ancestors  Find Subtree  Move Node
Parent Reference       O(1)         O(1)          O(depth)        O(n)        O(1)
Array of Ancestors     O(1)         O(1)            O(1)          O(1)     O(descendants)
Materialized Path      O(1)         O(1)            O(1)          O(1)     O(descendants)
$graphLookup           O(1)         O(1)         O(depth)       O(depth)      O(1)
```

---

## Diagram 7: Outlier Pattern — Normal vs Viral Document Routing

```
OUTLIER PATTERN: HANDLING THE CELEBRITY PROBLEM
══════════════════════════════════════════════════════════════════════

DISTRIBUTION: Post popularity follows power law
  99.9% of posts: < 100 comments
  0.1% of posts:  > 10,000 comments (viral posts)

WITHOUT Outlier Pattern (designed for worst case):
All posts → separate comments collection → ALWAYS two queries
┌─────────────────────────────────────┐
│ Post A (3 comments)                 │ ← still needs 2nd query for 3 comments
│ Post B (7 comments)                 │ ← still needs 2nd query for 7 comments
│ Post C (87,432 comments) ← OUTLIER  │ ← 2nd query returns 87,432 docs
└─────────────────────────────────────┘
All posts penalized for the 0.1% case

WITH Outlier Pattern:
┌─────────────────────────────────────────────────────────────────┐
│ Normal Post (99.9%):                                             │
│ {                                                                │
│   title: "My Post",                                             │
│   pinnedComments: [                                             │
│     { author: "Alice", body: "Great!" },  ← embedded (fast!)   │
│     { author: "Bob",   body: "Agreed!" }                        │
│   ],                                                            │
│   hasOverflowComments: false    ← no extra query needed         │
│ }                                                                │
│ → 1 query to render post with comments                          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Viral Post (0.1%):                                              │
│ {                                                                │
│   title: "MongoDB is Free Now!",                                │
│   pinnedComments: [                                             │
│     { author: "Alice", body: "OMG!", likes: 5021 },  ← top 20  │
│     // ...19 more top comments                                  │
│   ],                                                            │
│   hasOverflowComments: true,    ← FLAG: load more from comments │
│   commentsCount: 87432                                          │
│ }                                                                │
│ → 2 queries: post doc + paginated comments from separate coll   │
└─────────────────────────────────────────────────────────────────┘

                     comments collection
                     ┌──────────────────────────────────────────┐
                     │ { postId: "viral-post", author: "...",   │
                     │   body: "...", createdAt: ISODate("...") }│
                     │ × 87,432 entries (unbounded, that's fine) │
                     └──────────────────────────────────────────┘

RESULT:
  Normal posts (99.9%): 1 query  (fast path)
  Viral posts  (0.1%):  2 queries (heavy path, but acceptable for rare case)
  Average query count:  1.001 queries per post view (vs 2 without Outlier Pattern)
```

---

## Diagram 8: Schema Anti-Patterns Reference Card

```
SCHEMA ANTI-PATTERNS QUICK REFERENCE
══════════════════════════════════════════════════════════════════════

ANTI-PATTERN 1: Unbounded Array
──────────────────────────────────────────────────────────────────────
BAD:  { userId: "alice", followers: [id1, id2, ..., id10_000_000] }
       ↑ Will hit 16MB when followers > ~83,000 (at ~200 bytes/entry)
       ↑ Every write to add a follower rewrites massive array
GOOD: Separate follows collection with index on followeeId

ANTI-PATTERN 2: Over-Indexing
──────────────────────────────────────────────────────────────────────
BAD:  20 indexes on orders collection
       INSERT one order → update 20 B-trees (20x write overhead)
       QUERY planner evaluates all 20 candidates → slower planning
GOOD: 5-8 targeted indexes covering your actual query patterns
      Use db.collection.aggregate([{$indexStats:{}}]) to find unused

ANTI-PATTERN 3: Unnecessary $lookup on Hot Path
──────────────────────────────────────────────────────────────────────
BAD:
  Product page (1000 req/sec) → $lookup categories → $lookup brand
  = 1000 × 3 queries/sec = 3000 queries/sec just for product pages
GOOD: Denormalize category name + brand name into product doc
  Update on category/brand rename (rare operation)

ANTI-PATTERN 4: Massive Documents
──────────────────────────────────────────────────────────────────────
BAD:  1 user document with ALL user data: profile + history + prefs + files
       Every user operation loads entire 2MB document
GOOD: Core document (name, email, status) + extension documents per concern
      { _id: "user_pref_alice", userId: ObjectId("alice"), preferences: {...} }

ANTI-PATTERN 5: Schema Not Designed for Access Patterns
──────────────────────────────────────────────────────────────────────
BAD:  Normalize to 3NF like SQL:
       users → orders → order_items → products → categories
       (5 $lookup stages to render one order page)
GOOD: Ask first → then design:
  "Show me your top 5 most frequent queries"
  "What data is always fetched together?"
  "What is the max size of any array in this document?"
  THEN design the schema around those answers

DOCUMENT SIZE BUDGET:
──────────────────────────────────────────────────────────────────────
Hard limit:     16MB  (MongoDB BSON limit)
Practical max:  ~1MB  (above this, network/RAM pressure increases)
Sweet spot:     < 100KB for documents on hot query paths
Embedded array: define a maximum; enforce at application layer
```
