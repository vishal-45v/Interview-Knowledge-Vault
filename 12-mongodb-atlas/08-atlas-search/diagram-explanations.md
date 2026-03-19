# Chapter 08 — Atlas Search: Diagram Explanations

---

## Diagram 1: Atlas Search Architecture — Write and Read Flows

```
ATLAS SEARCH ARCHITECTURE
══════════════════════════════════════════════════════════════════════

WRITE FLOW:
──────────────────────────────────────────────────────────────────────

Application
    │
    │  db.products.insertOne({ name: "Wireless Headphones", price: 79.99 })
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  MongoDB (WiredTiger)                                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Document stored in WiredTiger                             │  │
│  │ { _id: ObjectId("..."), name: "Wireless Headphones" }     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                          │                                       │
│  Oplog record:           │ ← oplog tailed continuously          │
│  { op: "i", ns: "db.products", o: { ... } }                    │
└──────────────────────────┼──────────────────────────────────────┘
                           │
              ~1-5 seconds later (ASYNC)
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│  mongot (Atlas Search sync process)                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Lucene Index:                                             │  │
│  │ Tokens: "wireless" → [doc_abc]                           │  │
│  │         "headphones" → [doc_abc]                         │  │
│  │         "wireless headphones" (phrase) → [doc_abc]       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

READ FLOW:
──────────────────────────────────────────────────────────────────────

Application
    │
    │  db.products.aggregate([{ $search: { text: { query: "wireless" } } }])
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  MongoDB Pipeline Execution                                      │
│  Stage 0: $search (intercepted by mongot)                       │
│               │                                                  │
│               ▼                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Lucene Query: "wireless" → BM25 scoring                 │   │
│  │ Returns: [(doc_abc, score: 9.2), (doc_xyz, score: 7.1)] │   │
│  └─────────────────────────────────────────────────────────┘   │
│               │                                                  │
│               │  Document IDs + scores                          │
│               ▼                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ MongoDB WiredTiger: fetch full documents by RecordId    │   │
│  │ { _id: doc_abc, name: "Wireless Headphones", price: 79.99 } │
│  │ { _id: doc_xyz, name: "Wireless Keyboard", price: 49.99 }   │
│  └─────────────────────────────────────────────────────────┘   │
│  Stage 1: $limit 10                                             │
│  Stage 2: $project { name: 1, price: 1 }                       │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼ Application receives results
```

---

## Diagram 2: `compound` Operator — Scoring Model

```
COMPOUND OPERATOR: SCORING CONTRIBUTION
══════════════════════════════════════════════════════════════════════

Query:
{
  compound: {
    must:    [{ text: { query: "laptop", path: "title" } }],           // required + scored
    should:  [{ range: { path: "rating", gte: 4 } }],                 // optional + scored
    mustNot: [{ equals: { path: "status", value: "deleted" } }],      // exclusion, no score
    filter:  [{ equals: { path: "isActive", value: true } }]           // required, no score
  }
}

Document Evaluation:
─────────────────────────────────────────────────────────────────────

Doc A: { title: "Laptop Computer", rating: 4.5, isActive: true, status: "published" }
  must check:    "laptop" in title? YES → score contribution: 8.5
  should check:  rating >= 4? YES → score contribution: 2.0
  mustNot check: status = "deleted"? NO → not excluded ✓
  filter check:  isActive = true? YES → passes filter ✓
  FINAL SCORE:   8.5 + 2.0 = 10.5 ← included in results

Doc B: { title: "Gaming Laptop Pro", rating: 2.1, isActive: true, status: "published" }
  must check:    "laptop" in title? YES → score contribution: 7.8
  should check:  rating >= 4? NO → score contribution: 0
  mustNot check: status = "deleted"? NO → not excluded ✓
  filter check:  isActive = true? YES → passes filter ✓
  FINAL SCORE:   7.8 + 0 = 7.8 ← included but ranked lower

Doc C: { title: "Laptop Sale", rating: 4.9, isActive: false, status: "published" }
  must check:    "laptop" in title? YES → score contribution: 6.2
  filter check:  isActive = true? NO → EXCLUDED! ← filter violation
  FINAL SCORE:   excluded from results

Doc D: { title: "Laptop Deal", rating: 4.8, isActive: true, status: "deleted" }
  must check:    "laptop" in title? YES
  mustNot check: status = "deleted"? YES → EXCLUDED! ← mustNot violation
  FINAL SCORE:   excluded from results

RESULTS (ranked by score):
  1. Doc A: 10.5 (matched must AND should)
  2. Doc B: 7.8  (matched must, NOT should)
  Doc C: excluded (failed filter)
  Doc D: excluded (failed mustNot)

SCORE CONTRIBUTION TABLE:
──────────────────────────────────────────────────────────────────────
Clause Type  │ Boolean Required?  │ Contributes to Score?
─────────────┼────────────────────┼──────────────────────
must         │ ALL must match     │ YES
should       │ At least 1*        │ YES (boosts ranking)
mustNot      │ NONE must match    │ NO (pure exclusion)
filter       │ ALL must match     │ NO (pure boolean)
*minimumShouldMatch: 1 by default
```

---

## Diagram 3: Text Analyzers Comparison

```
ATLAS SEARCH ANALYZERS: HOW THEY TOKENIZE
══════════════════════════════════════════════════════════════════════

Input: "The Running MongoDB Cluster on AWS us-east-1"

LUCENE.STANDARD:
  Tokenize on whitespace/punctuation → lowercase → remove stop words
  Output: ["running", "mongodb", "cluster", "on", "aws", "us", "east", "1"]
         Note: "The" removed (stop word), "us-east-1" split on hyphen

LUCENE.ENGLISH:
  Standard + English stemming
  Output: ["run", "mongodb", "cluster", "aws", "us", "east", "1"]
         "running" → "run" (stemmed to root form)
         Query "runs" also matches → because "runs" also stems to "run"

LUCENE.KEYWORD:
  Entire field = one token (no tokenization)
  Output: ["The Running MongoDB Cluster on AWS us-east-1"]
         Exact match only: query "MongoDB" does NOT match (different case)
         Query "the running mongodb cluster on aws us-east-1" matches ✓

LUCENE.WHITESPACE:
  Tokenize on whitespace only (keep punctuation)
  → lowercase
  Output: ["the", "running", "mongodb", "cluster", "on", "aws", "us-east-1"]
         "us-east-1" stays intact (not split on hyphen!)

LUCENE.SIMPLE:
  Tokenize on non-letter characters → lowercase
  Output: ["the", "running", "mongodb", "cluster", "on", "aws", "us", "east"]
         "1" removed (non-letter), "us-east-1" split on hyphen and digit

CHOOSING THE RIGHT ANALYZER:
──────────────────────────────────────────────────────────────────────
Use case                          │ Analyzer
──────────────────────────────────┼─────────────────────────────────
Free-text English content          │ lucene.english (stemming helps)
Non-English content                │ lucene.[language] (spanish, french, etc.)
Enum/status/category fields        │ lucene.keyword (exact match)
Email addresses / URLs             │ lucene.keyword (exact) or lucene.whitespace
Product codes (e.g., ABC-123)      │ lucene.keyword or custom
Tags / hashtags                    │ lucene.whitespace or lucene.keyword
Case-insensitive exact match       │ lucene.keyword with lowercase charFilter
General purpose / mixed            │ lucene.standard
```

---

## Diagram 4: Autocomplete EdgeGram Tokenization

```
AUTOCOMPLETE: EDGEGRAM TOKENIZATION
══════════════════════════════════════════════════════════════════════

Product name: "Wireless Headphones"
minGrams: 2, maxGrams: 10, tokenization: "edgeGram"

Tokens generated for INDEX:
  "Wi", "Wir", "Wire", "Wirel", "Wirele", "Wireles", "Wireless"    ← "Wireless" prefixes
  "He", "Hea", "Head", "Headp", "Headph", "Headpho", "Headphon", "Headphone", "Headphones"
                                                                     ← "Headphones" prefixes

USER TYPES:  "wire" (4 chars)
              │
              ▼ $search autocomplete query
Lucene finds all docs with token "wire" in their autocomplete field:
  → "Wireless Headphones" ← matches via "wire" prefix token!
  → "Wire Stripper" ← matches via "wire" prefix token
  → "Wireless Keyboard" ← matches via "wire" prefix token
Results returned to user as suggestions!

USER TYPES: "wirel" (5 chars)
  → "Wireless Headphones" ← still matches ("wirel" token)
  → "Wireless Keyboard" ← still matches
  → "Wire Stripper" ← NO longer matches! ("wire" but not "wirel")
  → Suggestions narrowed!

COMPARISON: edgeGram vs nGram
──────────────────────────────────────────────────────────────────────
edgeGram:    "laptop" → "la", "lap", "lapt", "lapto", "laptop"
             Matches PREFIX only: "lap" → "laptop" ✓, "apt" → "laptop" ✗

nGram:       "laptop" → all substrings: "la","ap","pt","to","op","lap","apt",...
             Matches ANY substring: "apt" → "laptop" ✓ (more flexible, larger index)

For search-as-you-type autocomplete: use edgeGram (most natural, smaller index)
For "contains" search: use nGram (or just use regular text query with fuzzy)

AUTOCOMPLETE QUERY:
──────────────────────────────────────────────────────────────────────
db.products.aggregate([
  { $search: {
      autocomplete: {
        query: "wire",          ← partial input
        path: "name",
        tokenOrder: "sequential"  ← "wi re" must be in order
      }
  }},
  { $limit: 5 }
])
Returns: "Wireless Headphones", "Wire Stripper", "Wireless Keyboard"
```

---

## Diagram 5: Atlas Search Index vs MongoDB B-tree Index

```
ATLAS SEARCH INDEX vs MONGODB B-TREE INDEX
══════════════════════════════════════════════════════════════════════

MONGODB B-TREE INDEX:
──────────────────────────────────────────────────────────────────────
Sorted tree structure:
              [500]
             /     \
          [100]    [800]
          /   \    /   \
       [50] [200] [700] [900]

Query: { price: { $gte: 400, $lte: 700 } }
→ B-tree scan from 400 to 700
→ IXSCAN stage
→ Fast for range queries, exact matches, sort

What B-tree is GOOD AT:
✓ Range queries: { price: { $gte: 100 } }
✓ Sort operations: .sort({ price: 1 })
✓ Equality: { status: "active" }
✓ Covered queries: no document fetch needed

What B-tree is BAD AT:
✗ Full-text search ("find all docs containing 'mongodb'")
✗ Relevance ranking (scoring by match quality)
✗ Fuzzy matching (typo tolerance)
✗ Autocomplete (partial word matching)

ATLAS SEARCH (LUCENE) INDEX:
──────────────────────────────────────────────────────────────────────
Inverted index structure:
"atlas"       → [doc_1, doc_3, doc_7] with positions and scores
"mongodb"     → [doc_1, doc_2, doc_4, doc_5, doc_8]
"performance" → [doc_2, doc_5, doc_9]

Query: "mongodb performance"
→ Lucene finds intersection: [doc_2, doc_5]
→ BM25 scores: doc_5 (score: 9.2), doc_2 (score: 7.1)
→ Returns ranked by relevance

What Lucene is GOOD AT:
✓ Full-text search with ranking
✓ Fuzzy matching (typo tolerance)
✓ Autocomplete (edgeGram tokenization)
✓ Phrase queries
✓ Faceted search (stringFacet, numberFacet)
✓ Custom analyzers (stemming, synonyms)

What Lucene is BAD AT:
✗ Exact equality (use B-tree index or compound.filter)
✗ Sort by non-score field (post-search $sort uses MongoDB)
✗ Transactions (use MongoDB queries)
✗ Aggregation (post-search $group uses MongoDB)

COMBINED USAGE (best of both):
──────────────────────────────────────────────────────────────────────
$search:          Lucene handles relevance + fuzzy + autocomplete
compound.filter:  Lucene applies boolean filters (fast, no score impact)
post-search $sort:  MongoDB sorts result set by a non-score field
post-search $lookup: MongoDB joins with another collection
→ Lucene for FINDING, MongoDB for REFINING
```

---

## Diagram 6: `$search` vs `$searchMeta` Pipeline Stages

```
$search vs $searchMeta: WHAT THEY RETURN
══════════════════════════════════════════════════════════════════════

$search: Returns DOCUMENTS (ranked by relevance)
──────────────────────────────────────────────────────────────────────

Pipeline:
db.products.aggregate([
  { $search: { text: { query: "laptop", path: "name" } } },
  { $limit: 5 }
])

What flows through the pipeline:
  Input to $search: entire collection
  Output of $search: documents matching "laptop", ranked by score

  ┌──────────────────────────────────────────────────────────┐
  │ { _id: ..., name: "Gaming Laptop Pro",  score: 9.2 }     │
  │ { _id: ..., name: "Business Laptop 15", score: 8.7 }     │
  │ { _id: ..., name: "Laptop Stand",       score: 5.1 }     │
  │ { _id: ..., name: "Laptop Backpack",    score: 4.3 }     │
  │ { _id: ..., name: "Wireless Laptop",    score: 3.8 }     │
  └──────────────────────────────────────────────────────────┘
  Documents with scores (score via { $meta: "searchScore" })

$searchMeta: Returns METADATA ONLY (no documents)
──────────────────────────────────────────────────────────────────────

Pipeline:
db.products.aggregate([
  { $searchMeta: {
      facet: {
        operator: { text: { query: "laptop", path: "name" } },
        facets: { categoryBuckets: { type: "string", path: "category" } }
      }
  }}
])

What flows through the pipeline:
  Output of $searchMeta: ONE metadata document

  ┌──────────────────────────────────────────────────────────┐
  │ {                                                        │
  │   "count": { "lowerBound": 1247 },                      │
  │   "facet": {                                             │
  │     "categoryBuckets": {                                 │
  │       "buckets": [                                       │
  │         { "_id": "electronics",  "count": 523 },        │
  │         { "_id": "computers",    "count": 312 },        │
  │         { "_id": "accessories",  "count": 198 }         │
  │       ]                                                  │
  │     }                                                    │
  │   }                                                      │
  │ }                                                        │
  └──────────────────────────────────────────────────────────┘
  ZERO documents! Only counts and facets.

COMBINED APPROACH (parallel queries):
──────────────────────────────────────────────────────────────────────

const [results, meta] = await Promise.all([
  // Query 1: $search → documents
  db.products.aggregate([
    { $search: { text: { query: "laptop", path: "name" } } },
    { $skip: 0 }, { $limit: 20 }
  ]).toArray(),

  // Query 2: $searchMeta → facets
  db.products.aggregate([
    { $searchMeta: { facet: { operator: ..., facets: ... } } }
  ]).toArray()
])
// Two queries running in PARALLEL → ~same latency as one query alone
// products page renders: 20 results + facet counts (sidebar)
```
