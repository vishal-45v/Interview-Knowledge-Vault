# Chapter 04 — Indexes & Performance: Diagram Explanations

---

## Diagram 1: ESR Rule — Compound Index Field Order

```
ESR RULE: EQUALITY → SORT → RANGE
══════════════════════════════════════════════════════════════════════

QUERY:
db.orders.find({
  status: "completed",          // E: Equality
  createdAt: { $gte: lastWeek } // R: Range
}).sort({ customerId: 1 })      // S: Sort

WRONG ORDER (Range before Sort):
Index: { status: 1, createdAt: 1, customerId: 1 }
       E          R              S

B-tree scan:
  [status="completed"] → enters range on createdAt → customerId data fragmented
  MongoDB must add in-memory SORT stage for customerId

  Plan: IXSCAN → SORT (in-memory) → FETCH
  ⚠️ SORT stage = in-memory overhead

CORRECT ORDER (ESR):
Index: { status: 1, customerId: 1, createdAt: 1 }
       E          S               R

B-tree scan:
  [status="completed"] → sorted by customerId → then range on createdAt
  Results come out ALREADY SORTED by customerId
  No in-memory sort needed!

  Plan: IXSCAN → FETCH (no SORT stage)
  ✓ Sort handled by index order

ESR ORDERING RULES:
──────────────────────────────────────────
Pattern                 Index Order
E fields first          { a:1 (E), b:1 (E), ... }
S fields next           { ..., c:1 (S), ... }
R fields last           { ..., d:1 (R) }
S+R same field          Field serves double duty: { a:1 (E), b:1 (S+R) }
```

---

## Diagram 2: COLLSCAN vs IXSCAN Performance

```
COLLECTION SCAN (COLLSCAN) vs INDEX SCAN (IXSCAN)
══════════════════════════════════════════════════════════════════════

Collection: 10,000,000 documents

COLLSCAN (no index):
┌─────────────────────────────────────────────────────────────────┐
│ Storage (disk/cache):                                           │
│ [doc1][doc2][doc3][doc4]...[doc9,999,999][doc10,000,000]       │
│                                                                  │
│ ↑ Read ALL documents one by one                                 │
│   Looking for: { email: "alice@example.com" }                   │
│   Found at: position 4,523,891 (middle of collection)          │
│                                                                  │
│ Documents examined: 10,000,000                                  │
│ Time: ~4,500ms                                                  │
│ RAM used: entire working set scanned                            │
└─────────────────────────────────────────────────────────────────┘

IXSCAN (with index { email: 1 }):
┌─────────────────────────────────────────────────────────────────┐
│ B-Tree Index Structure (sorted, balanced):                      │
│                                                                  │
│                    [M]                  ← root                  │
│                   /   \                                         │
│              [F-L]     [N-Z]            ← internal nodes       │
│             /    \                                               │
│          [A-C]  [D-E]                   ← leaf pages           │
│          / \                                                     │
│       [Ab] [Al-Am] ... "alice@example.com" → RecordId          │
│                          ↑                                      │
│                    FOUND in 3 hops!                             │
│                                                                  │
│ Then: RecordId → fetch doc from storage (one random I/O)       │
│                                                                  │
│ Keys examined: 1                                                │
│ Docs examined: 1                                                │
│ Time: ~2ms                                                      │
└─────────────────────────────────────────────────────────────────┘

PERFORMANCE COMPARISON:
──────────────────────────────────────────
Query                    COLLSCAN    IXSCAN
{ email: exact }          4500ms       2ms   (2250x faster)
{ age: { $gte: 25 } }      800ms       5ms   (160x faster, age not selective)
{ _id: ObjectId }          4500ms      1ms   (default _id index)
```

---

## Diagram 3: Compound Index B-Tree Structure

```
COMPOUND INDEX B-TREE: { category: 1, price: 1 }
══════════════════════════════════════════════════════════════════════

Index entries sorted lexicographically:
(category ASC, then price ASC within same category)

Leaf nodes:
───────────────────────────────────────────────────────────────────
("books",        4.99)  → doc_456
("books",       12.99)  → doc_789
("books",       24.99)  → doc_123
("electronics",  9.99)  → doc_234
("electronics", 49.99)  → doc_567
("electronics", 99.99)  → doc_890
("electronics",149.99)  → doc_111
("fashion",      5.99)  → doc_333
("fashion",     29.99)  → doc_444
───────────────────────────────────────────────────────────────────

QUERY: { category: "electronics", price: { $lte: 100 } }

Step 1: Jump to ("electronics", -infinity) in B-tree
Step 2: Scan forward: 9.99 → 49.99 → 99.99 → STOP (next is 149.99 > 100)
Step 3: Return docs: doc_234, doc_567, doc_890 (3 documents)

RESULT:
  totalKeysExamined: 3  (only the 3 matching entries)
  nReturned: 3
  No wasted scans!

CONTRAST: compound index { price: 1, category: 1 }
(price FIRST, wrong ESR order for this query pattern)

Leaf nodes:
  (4.99, "books") → doc_456
  (5.99, "fashion") → doc_333
  (9.99, "electronics") → doc_234     ← start
  (12.99, "books") → doc_789
  (24.99, "books") → doc_123
  (29.99, "fashion") → doc_444
  (49.99, "electronics") → doc_567
  ...
  (99.99, "electronics") → doc_890    ← stop

Now must scan ALL prices ≤ 100 (multiple categories mixed in!),
then filter for category="electronics" → examines more keys than needed
```

---

## Diagram 4: Covered Query Flow vs. Regular Query Flow

```
REGULAR QUERY (with FETCH):
══════════════════════════════════════════════════════════════════════

Query: db.products.find({ category: "electronics" }, { name: 1, price: 1 })
Index: { category: 1, price: 1, name: 1 }

Step 1: IXSCAN — scan index for category="electronics"
        Index returns: RecordId → doc_123, doc_456, doc_789...

         INDEX (on disk/cache)
        ┌──────────────────────────────────────┐
        │ ("electronics", 9.99, "Dongle") → 1  │
        │ ("electronics",49.99, "Tablet") → 2  │
        │ ("electronics",99.99, "Laptop") → 3  │
        └──────────────────────────────────────┘
                           │ RecordIds
                           ▼
Step 2: FETCH — load each document from collection storage
        Collection storage (random I/O)
        ┌──────────────────────────────────────┐
        │ Doc 1: { _id:1, cat:"electronics", name:"Dongle", price:9.99, desc:"..." } │
        │ Doc 2: { _id:2, cat:"electronics", name:"Tablet", price:49.99, inStock:true } │
        │ Doc 3: { _id:3, cat:"electronics", name:"Laptop", price:99.99, brand:"..." } │
        └──────────────────────────────────────┘
                           │
                           ▼
Step 3: PROJECTION — extract name and price, include _id
        Returns: { _id: 1, name: "Dongle", price: 9.99 }

totalDocsExamined: 3 (fetched 3 documents from storage)

COVERED QUERY (no FETCH):
══════════════════════════════════════════════════════════════════════

Query: db.products.find({ category: "electronics" }, { name: 1, price: 1, _id: 0 })
Index: { category: 1, price: 1, name: 1 }

Step 1: IXSCAN — scan index for category="electronics"
        Index ALREADY HAS all needed data!

         INDEX (on disk/cache)
        ┌──────────────────────────────────────┐
        │ ("electronics", 9.99, "Dongle") → 1  │  ← name: ✓, price: ✓
        │ ("electronics",49.99, "Tablet") → 2  │  ← name: ✓, price: ✓
        │ ("electronics",99.99, "Laptop") → 3  │  ← name: ✓, price: ✓
        └──────────────────────────────────────┘
                           │
                           ▼ (no FETCH step needed!)
Step 2: Return { name: "Dongle", price: 9.99 } directly from index

totalDocsExamined: 0 — ZERO document reads from storage!
Plan stage: PROJECTION_COVERED (not FETCH)
```

---

## Diagram 5: Multikey Index Entry Expansion

```
MULTIKEY INDEX: ARRAY FIELD EXPANSION
══════════════════════════════════════════════════════════════════════

Documents:
┌───────────────────────────────────────────────────────────┐
│ { _id: 1, name: "Post A", tags: ["mongodb","atlas","db"] }│
│ { _id: 2, name: "Post B", tags: ["redis","cache"]         }│
│ { _id: 3, name: "Post C", tags: ["mongodb","redis"]       }│
└───────────────────────────────────────────────────────────┘

Regular index { name: 1 }: 3 entries (1 per document)
  "Post A" → doc1
  "Post B" → doc2
  "Post C" → doc3

Multikey index { tags: 1 }: 7 entries (1 per array element!)
  "atlas"   → doc1       ← from Post A's tags
  "cache"   → doc2       ← from Post B's tags
  "db"      → doc1       ← from Post A's tags
  "mongodb" → doc1       ← from Post A's tags
  "mongodb" → doc3       ← from Post C's tags (DUPLICATE KEY, different docs)
  "redis"   → doc2       ← from Post B's tags
  "redis"   → doc3       ← from Post C's tags

Query: db.posts.find({ tags: "mongodb" })
  Index scan: find "mongodb" → returns doc1, doc3
  Result: Post A, Post C ✓

SCALING CONCERN:
  1M documents × avg 50 tags = 50M index entries
  vs regular index: 1M entries
  = 50x index size!

COMPOUND MULTIKEY RESTRICTION:
  Index { tags: 1, categories: 1 }
  Doc: { tags: ["a","b"], categories: ["x","y"] }
           ↑ ARRAY          ↑ ARRAY
  ERROR: "cannot index parallel arrays"
  → Both fields are arrays in the SAME document → forbidden
```

---

## Diagram 6: Partial vs Sparse Index

```
PARTIAL INDEX vs SPARSE INDEX COMPARISON
══════════════════════════════════════════════════════════════════════

Collection: 1,000,000 user documents
  - 700,000 users: isActive: true, email: exists
  - 200,000 users: isActive: false, email: exists
  - 100,000 users: isActive: true, email: MISSING

FULL INDEX { email: 1 }:
  Contains: 900,000 entries (everyone with email field)
  Index size: ~90MB

SPARSE INDEX { email: 1 } sparse: true:
  Contains: 900,000 entries (docs WHERE email EXISTS)
  Same as: partialFilterExpression: { email: { $exists: true } }
  Index size: ~90MB (same as full — missing email = null in most cases)
  Note: In MongoDB 4.4+, sparse includes null values too

PARTIAL INDEX { email: 1 } partialFilterExpression: { isActive: true }:
  Contains: 700,000 entries (only ACTIVE users with email)
  Index size: ~70MB (22% smaller!)

  Queries that CAN use partial index:
  ✓ { email: "alice@example.com", isActive: true }   (includes partition filter)
  ✓ { email: "alice@example.com", isActive: { $eq: true } }

  Queries that CANNOT use partial index:
  ✗ { email: "alice@example.com" }    (no isActive filter)
  ✗ { email: "alice@example.com", isActive: false }  (doesn't match partition)

WHEN TO USE EACH:
──────────────────────────────────────────────────
Sparse:  Simple — when field is optional and you want to exclude docs without it
Partial: Flexible — when you want a complex filter condition (any query expression)
         Also: when you want to exclude docs based on a DIFFERENT field's value
               (e.g., exclude soft-deleted: { isDeleted: { $ne: true } })
```
