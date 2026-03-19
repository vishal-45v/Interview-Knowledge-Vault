# Chapter 08 — Atlas Search: Theory Questions

---

## Q1: What is Atlas Search? How does it work architecturally?

Atlas Search is a full-text search service embedded in MongoDB Atlas, powered by Apache Lucene. It runs on dedicated search nodes co-located with your Atlas cluster.

```javascript
// Architecture:
// 1. Atlas cluster: stores documents in MongoDB (WiredTiger)
// 2. Atlas Search nodes: separate Lucene index maintained alongside
// 3. Sync process: mongot (mongoDB + Lucene bridge) syncs changes from oplog to Lucene index
// 4. Query execution: $search stage is intercepted by mongot, Lucene executes it,
//    returns document IDs → MongoDB fetches full documents

// Key architectural facts:
// - Asynchronous sync: writes to MongoDB are eventually reflected in Lucene (~seconds)
// - $search is an aggregation pipeline stage (must be FIRST stage)
// - Index separate from MongoDB B-tree indexes (different index, different storage)
// - One Lucene index per Atlas Search index definition (can have multiple per collection)

// Creating a search index (Atlas UI, Atlas CLI, or Atlas Admin API):
// Atlas UI: Collections → Search Indexes → Create Index

// Dynamic vs. static mappings:
// Dynamic mapping: automatically indexes all string/number/boolean fields
{
  "mappings": { "dynamic": true }
}
// Use for: rapid prototyping, unknown schema
// Downside: indexes everything, higher storage, less control

// Static mapping: only index explicitly listed fields
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "title":    { "type": "string",  "analyzer": "lucene.english" },
      "price":    { "type": "number" },
      "category": { "type": "stringFacet" },
      "status":   { "type": "string",  "analyzer": "lucene.keyword" }
    }
  }
}
// Use for: production (controlled, smaller index)

// Query Atlas Search:
db.products.aggregate([
  { $search: { ... } },    // MUST be first stage (with some exceptions)
  { $match: { ... } },     // can filter AFTER search
  { $project: { ... } },
  { $limit: 10 }
])
```

---

## Q2: What analyzers are available in Atlas Search and when do you use each?

An **analyzer** defines how text is processed for indexing and querying — it tokenizes, normalizes, and transforms text into searchable tokens.

```javascript
// Built-in analyzers:

// lucene.standard (default)
// - Tokenizes on whitespace and punctuation
// - Lowercases all tokens
// - Removes English stop words ("the", "is", "and")
// Input: "The Quick Brown Fox"
// Tokens: ["quick", "brown", "fox"]
// Use: general-purpose text search

// lucene.english
// - Standard + English stemming (runs → run, running → run)
// Input: "MongoDB is running on AWS"
// Tokens: ["mongodb", "run", "aws"]  ← "is" removed (stop word), "running" → "run"
// Use: English content, query "runs" matches "running", "ran", "runs"

// lucene.keyword
// - Entire field value = one token (no tokenization!)
// Input: "user@example.com"
// Token: ["user@example.com"]  ← entire value as-is
// Use: exact match fields (email, status, category, enum values)
// WITHOUT keyword analyzer: "user@example.com" → ["user", "example.com"] → partial matches

// lucene.whitespace
// - Tokenizes only on whitespace (keeps punctuation)
// - Lowercases
// Input: "user@example.com test-driven"
// Tokens: ["user@example.com", "test-driven"]
// Use: tags, hashtags, usernames

// lucene.simple
// - Tokenizes on non-letter characters
// - Lowercases
// Input: "Hello World, 123"
// Tokens: ["hello", "world"]  ← "123" dropped (non-letters)

// lucene.stop
// - Same as standard but configurable stop words list

// Custom analyzer (when built-ins aren't enough):
{
  "analyzers": [
    {
      "name": "myCustomAnalyzer",
      "tokenizer": { "type": "standard" },
      "tokenFilters": [
        { "type": "lowercase" },
        { "type": "stopword", "tokens": ["the", "a", "an", "is"] },
        { "type": "snowballStemming", "stemmerName": "english" },
        { "type": "shingle", "minShingleSize": 2, "maxShingleSize": 3 }  // bigrams/trigrams
      ],
      "charFilters": [
        { "type": "htmlStrip" }   // remove HTML tags before tokenizing
      ]
    }
  ]
}

// Important: searchAnalyzer ≠ indexAnalyzer
// By default, the same analyzer is used for both indexing and querying
// For autocomplete: index with edgeGram, query with keyword (exact prefix match)
{
  "fields": {
    "title": [
      { "type": "string",       "analyzer": "lucene.english" },     // for full-text search
      { "type": "autocomplete", "analyzer": "lucene.standard",      // for autocomplete
        "tokenization": "edgeGram", "minGrams": 2, "maxGrams": 10 }
    ]
  }
}
```

---

## Q3: What is the `$search` aggregation stage? Explain its key operators.

`$search` is an aggregation stage that executes a full-text Lucene query against an Atlas Search index.

```javascript
// Basic structure:
db.products.aggregate([
  {
    $search: {
      index: "default",         // search index name (default: "default")
      <operator>: { ... },      // query operator
      highlight: { path: "description" },   // optional: highlight matches
      count: { type: "total" },             // optional: total result count
      returnStoredSource: true              // optional: use stored source (faster)
    }
  }
])

// KEY OPERATORS:

// 1. text — full-text search (most common)
$search: {
  text: {
    query: "mongodb atlas cloud",
    path: ["title", "description"],      // search in these fields
    fuzzy: { maxEdits: 1 },              // allow 1-char typos
    score: { boost: { path: "rating" } } // boost score by rating field value
  }
}

// 2. compound — combine multiple operators (AND/OR/NOT)
$search: {
  compound: {
    must: [    // ALL must match (AND)
      { text: { query: "mongodb", path: "title" } }
    ],
    should: [  // at least one SHOULD match (boosts relevance score)
      { text: { query: "atlas", path: "body" } }
    ],
    mustNot: [ // NONE must match (NOT)
      { equals: { path: "status", value: "deleted" } }
    ],
    filter: [  // MUST match (like must, but DOESN'T affect score)
      { range: { path: "price", gte: 10, lte: 100 } }
    ]
  }
}

// 3. equals — exact match (for string fields with keyword analyzer)
$search: {
  equals: { path: "status", value: "published" }
}

// 4. range — numeric, date, string range
$search: {
  range: {
    path: "price",
    gte: 10.0,
    lte: 100.0
  }
}
$search: {
  range: {
    path: "createdAt",
    gte: ISODate("2024-01-01"),
    lt:  ISODate("2024-04-01")
  }
}

// 5. wildcard — pattern matching
$search: {
  wildcard: {
    query: "mongo*",    // matches "mongodb", "mongos", "mongosh"
    path: "name",
    allowAnalyzedField: false   // works only with keyword analyzer
  }
}

// 6. regex — regular expression
$search: {
  regex: {
    query: "^[A-Z]{3}-\\d{4}$",   // matches "ABC-1234"
    path: "productCode",
    allowAnalyzedField: false
  }
}

// 7. phrase — exact phrase matching
$search: {
  phrase: {
    query: "compound index",
    path: "title",
    slop: 1   // allow 1 word between "compound" and "index"
  }
}

// 8. near — numeric/date proximity
$search: {
  near: {
    path: "price",
    origin: 50,     // center price
    pivot: 10,      // at 10 away, score halves
    score: { boost: { value: 2 } }
  }
}

// 9. exists — check field exists in Lucene index
$search: {
  exists: { path: "imageUrl" }
}
// Different from MongoDB $exists! This checks the SEARCH INDEX, not the document
```

---

## Q4: How does `$searchMeta` work for faceted search?

`$searchMeta` returns only metadata (counts, facets) without returning any documents. It's used to get facet counts for filtering UIs.

```javascript
// Create a search index with facet fields
{
  "mappings": {
    "fields": {
      "category": { "type": "stringFacet" },    // for string facets
      "price":    { "type": "numberFacet" },     // for numeric bucket facets
      "rating":   { "type": "numberFacet" }
    }
  }
}

// $searchMeta with facets
db.products.aggregate([
  {
    $searchMeta: {
      index: "productSearch",
      facet: {
        operator: {
          text: { query: "laptop", path: "name" }  // base search
        },
        facets: {
          // String facet: count documents per category value
          categoryBuckets: {
            type: "string",
            path: "category",
            numBuckets: 10     // return top 10 categories
          },
          // Number facet: count documents per price range
          priceBuckets: {
            type: "number",
            path: "price",
            boundaries: [0, 100, 500, 1000, 5000],
            default: "other"   // label for values outside boundaries
          },
          // Date facet:
          dateBuckets: {
            type: "date",
            path: "createdAt",
            boundaries: [
              ISODate("2023-01-01"),
              ISODate("2024-01-01"),
              ISODate("2025-01-01")
            ],
            default: "other"
          }
        }
      }
    }
  }
])

// Result structure:
{
  "count": { "lowerBound": 1247 },   // total matching docs (lower bound for performance)
  "facet": {
    "categoryBuckets": {
      "buckets": [
        { "_id": "electronics", "count": 523 },
        { "_id": "computers",   "count": 312 },
        { "_id": "accessories", "count": 198 },
        // ...
      ]
    },
    "priceBuckets": {
      "buckets": [
        { "_id": 0,    "count": 145 },   // 0-100
        { "_id": 100,  "count": 423 },   // 100-500
        { "_id": 500,  "count": 312 },   // 500-1000
        { "_id": 1000, "count": 234 },   // 1000-5000
        { "_id": "other", "count": 133 }
      ]
    }
  }
}

// Combined: search results + facets in one round-trip using $facet:
db.products.aggregate([
  {
    $searchMeta: {
      index: "productSearch",
      facet: {
        operator: { text: { query: "laptop", path: "name" } },
        facets: { categoryBuckets: { type: "string", path: "category" } }
      }
    }
  }
])
// Returns only metadata. For search results + metadata simultaneously:
// Use two parallel queries OR $facet in aggregation with $search
```

---

## Q5: How do you implement autocomplete with Atlas Search?

```javascript
// 1. Create search index with autocomplete field mapping
{
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": [
        // Regular text search (for full-word queries)
        { "type": "string", "analyzer": "lucene.english" },
        // Autocomplete (for partial word completion)
        {
          "type": "autocomplete",
          "analyzer": "lucene.standard",
          "tokenization": "edgeGram",  // edgeGram | rightEdgeGram | nGram
          "minGrams": 2,
          "maxGrams": 15,
          "foldDiacritics": true       // "café" → "cafe" (accent insensitive)
        }
      ]
    }
  }
}

// EdgeGram tokenization for "laptop":
// minGrams: 2, maxGrams: 15
// Generated tokens: "la", "lap", "lapt", "lapto", "laptop"
// So typing "lap" → matches "laptop", "lapel", "lapboard"

// 2. Autocomplete query:
db.products.aggregate([
  {
    $search: {
      index: "productSearch",
      autocomplete: {
        query: "wirel",          // partial: "wirel" → suggests "wireless"
        path: "name",
        tokenOrder: "sequential", // tokens must appear in order (vs "any")
        fuzzy: {
          maxEdits: 1,
          prefixLength: 1        // first character must match exactly
        }
      }
    }
  },
  { $limit: 10 },
  { $project: { name: 1, _id: 0, score: { $meta: "searchScore" } } }
])

// Autocomplete returns:
// "wireless headphones" (score: 9.2)
// "wireless keyboard"   (score: 8.8)
// "wireless mouse"      (score: 8.1)
// ...

// 3. Phrase autocomplete (multi-word completion):
db.products.aggregate([
  {
    $search: {
      autocomplete: {
        query: "blue wire",          // "blue wire" → "blue wireless headphones"
        path: "name",
        tokenOrder: "sequential"     // "blue" then "wire" in sequence
      }
    }
  },
  { $limit: 5 },
  { $project: { name: 1 } }
])

// Debounce autocomplete queries: don't fire on every keystroke
// Fire after 150-300ms of no typing (client-side debounce)
// Cache common autocomplete results (Redis, CDN)
```

---

## Q6: How do you implement relevance scoring and boosting in Atlas Search?

```javascript
// Atlas Search uses BM25 (Best Match 25) algorithm by default
// BM25 factors:
// 1. Term frequency (TF): how often does the term appear in the document?
// 2. Inverse document frequency (IDF): how rare is the term across all documents?
// 3. Field length normalization: shorter fields score higher for same term match

// ACCESSING THE SCORE:
db.products.aggregate([
  { $search: { text: { query: "mongodb atlas", path: "name" } } },
  { $project: { name: 1, score: { $meta: "searchScore" } } },
  { $sort: { score: { $meta: "searchScore" } } }
])

// BOOSTING STRATEGIES:

// 1. Field boost (title matches worth 3x more than body matches)
$search: {
  compound: {
    should: [
      { text: { query: "mongodb", path: "title",
          score: { boost: { value: 3 } } } },    // 3x boost
      { text: { query: "mongodb", path: "body",
          score: { boost: { value: 1 } } } }     // baseline
    ]
  }
}

// 2. Document field boost (boost by document's rating field)
$search: {
  text: {
    query: "laptop",
    path: "description",
    score: {
      boost: {
        path: "rating",        // use document's rating field value as multiplier
        undefined: 1           // default if rating field is missing
      }
    }
  }
}
// A laptop with rating: 4.5 scores 4.5x higher than one with rating: 1.0

// 3. Decay function (near operator — score decreases as value moves away from target)
$search: {
  near: {
    path: "publishedAt",
    origin: new Date(),         // today
    pivot: 1000 * 60 * 60 * 24 * 7,  // pivot = 7 days in milliseconds
    // score halves for every 7 days from today
    // → very recent docs score highest, older docs gradually lower
  }
}

// 4. Constant score (don't use document relevance, fixed score)
$search: {
  compound: {
    must: [
      { text: { query: "mongodb", path: "name",
          score: { constant: { value: 5 } } } }  // always score = 5 regardless of match quality
    ]
  }
}

// 5. Function score (complex custom scoring with $function)
// Available in Atlas Search: arithmetic operators in score expression
$search: {
  text: {
    query: "laptop",
    path: "name",
    score: {
      function: {
        multiply: [
          { score: "relevance" },      // BM25 base score
          { path: { value: "rating", undefined: 1 } },    // × document.rating
          { gaussian: {
              path: "price",
              origin: 500,             // ideal price = $500
              scale: 200,              // falloff: $200 away = 0.5× score
              offset: 50,             // within $50 of $500: no penalty
              decay: 0.5             // at scale away from origin: score × 0.5
          }}
        ]
      }
    }
  }
}
```

---

## Q7: What is a stored source in Atlas Search? When should you use it?

```javascript
// By default, Atlas Search returns document IDs → MongoDB fetches full documents
// This requires a round-trip from Lucene to MongoDB storage

// With storedSource:
// Specific fields are stored INSIDE the Lucene index
// Atlas Search returns these fields directly from Lucene (no MongoDB fetch needed)

// Configure stored source fields:
{
  "storedSource": {
    "include": ["name", "price", "category", "imageUrl"]
    // Only these fields are stored in Lucene (not the full document)
  }
}
// OR store everything (expensive, large Lucene index):
{
  "storedSource": true
}

// Querying with returnStoredSource:
db.products.aggregate([
  {
    $search: {
      text: { query: "laptop", path: "name" },
      returnStoredSource: true    // use Lucene-stored data, skip MongoDB fetch
    }
  },
  {
    $project: {
      name: 1, price: 1, category: 1, imageUrl: 1  // only storedSource fields!
    }
  }
])

// Benefits:
// - Faster: no round-trip to MongoDB storage
// - Reduces I/O: MongoDB doesn't need to load full documents
// - Better for search result lists (you usually only show name, price, image in list)

// Limitations:
// - Stored fields = additional storage in Lucene index (~duplicate of selected fields)
// - Stored source is read-only (you can't update it separately from the document)
// - Does NOT support $project exclusions — only include what's in storedSource
// - If a storedSource field is updated in MongoDB: sync delay applies

// When to use:
// ✓ Search result listing pages (name, price, thumbnail — not full description)
// ✓ High-traffic search endpoints where latency matters
// ✓ Large documents where you only need a small projection from search results

// When NOT to use:
// ✗ You need all document fields (full document fetch is fine)
// ✗ Storage cost is a concern (stored source doubles the indexed field storage)
```

---

## Q8: How do you combine Atlas Search with regular MongoDB queries?

```javascript
// $search must be the FIRST stage in an aggregation pipeline
// Subsequent stages can use any MongoDB operators

// Pattern 1: $search + $match for post-filtering
db.orders.aggregate([
  {
    $search: {
      text: { query: "laptop", path: "productName" }
    }
  },
  // Post-filter: only show active orders (Lucene doesn't have this data)
  {
    $match: { status: "active", quantity: { $gt: 0 } }
  },
  { $limit: 10 }
])
// Note: $match AFTER $search is less efficient than using compound.filter in $search
// Prefer: use filter in compound operator (doesn't affect score, applied in Lucene)

// BETTER: use compound.filter (applied in Lucene — more efficient)
db.orders.aggregate([
  {
    $search: {
      compound: {
        must: [
          { text: { query: "laptop", path: "productName" } }
        ],
        filter: [
          { equals: { path: "status", value: "active" } },
          { range: { path: "quantity", gte: 1 } }
        ]
      }
    }
  },
  { $limit: 10 }
])

// Pattern 2: $search + $lookup (search then join)
db.products.aggregate([
  { $search: { text: { query: "laptop", path: "name" } } },
  { $limit: 10 },
  {
    $lookup: {
      from: "inventory",
      localField: "_id",
      foreignField: "productId",
      as: "stock"
    }
  }
])

// Pattern 3: $search + $group (search then aggregate)
db.reviews.aggregate([
  { $search: { text: { query: "great battery", path: "body" } } },
  {
    $group: {
      _id: "$productId",
      avgRating: { $avg: "$rating" },
      matchCount: { $sum: 1 }
    }
  },
  { $sort: { matchCount: -1 } }
])

// Pattern 4: using $search with $facet for results + facets simultaneously
db.products.aggregate([
  {
    $search: {
      text: { query: "laptop", path: "name" },
      count: { type: "total" }
    }
  },
  {
    $facet: {
      results: [
        { $limit: 20 },
        { $project: { name: 1, price: 1, score: { $meta: "searchScore" } } }
      ],
      facets: [
        { $limit: 1000 },   // load all results for client-side faceting (small sets only)
        { $group: { _id: "$category", count: { $sum: 1 } } }
      ]
    }
  }
])
// Note: $facet materializes ALL results from $search → use $searchMeta for large result sets
```

---

## Q9: How do Atlas Search indexes sync with MongoDB? What is the sync delay?

```javascript
// Sync architecture:
// 1. MongoDB write: db.products.updateOne({ _id: id }, { $set: { name: "New Name" } })
// 2. Oplog records the change
// 3. Atlas Search's mongot process tails the oplog
// 4. mongot applies the change to the Lucene index
// 5. New document name is now searchable

// Sync delay:
// - Typical: < 1 second for small document changes
// - Under heavy write load: a few seconds
// - Cluster maintenance or failover: could be minutes
// - Atlas Search index pause/resume: could be hours (full reindex)

// Monitoring sync status:
db.products.aggregate([{ $searchIndexes: {} }])
// Returns index metadata including sync status:
// { name: "default", status: "READY" }   ← fully in sync
// { name: "default", status: "BUILDING" } ← initial build in progress
// { name: "default", status: "STALE" }    ← temporarily behind (catch-up in progress)

// Practical implications:
// 1. Don't search immediately after write and expect the new data:
const result = await db.users.insertOne({ name: "Alice", bio: "MongoDB expert" })
const searchResult = await db.users.aggregate([
  { $search: { text: { query: "MongoDB expert", path: "bio" } } }
]).toArray()
// searchResult might be empty if sync hasn't completed yet!
// If you need immediate searchability: use findOne by _id, not $search

// 2. For testing: add a small delay or poll for status
await new Promise(r => setTimeout(r, 500))  // 500ms delay in tests

// 3. For the last-write-wins problem (user updates profile, wants to see it in search):
// Show the direct MongoDB read (findOne by _id) for their own profile
// Use $search for searching OTHER users

// Index replication lag monitoring:
db.adminCommand({ "atlasSearchIndexes": { "getStatus": "default" } })
// Not a real command — use Atlas UI: Search → Index Status tab
// Shows: "Status: READY, Documents Indexed: 1,247,832, Number Outdated: 0"
```

---

## Q10: What are the key Atlas Search performance considerations?

```javascript
// 1. Index size vs. search performance
// Larger Lucene index = more RAM needed = more Atlas Search nodes needed
// Optimize: use static mappings instead of dynamic
{
  "mappings": {
    "dynamic": false,          // only index what you specify
    "fields": {
      "name": { "type": "string" },
      "category": { "type": "stringFacet" }
      // Don't index: long body text if you only search by name
    }
  }
}

// 2. numCandidates in $vectorSearch
// Higher numCandidates = more accurate results, slower query
// Lower numCandidates = faster, might miss some relevant docs
db.docs.aggregate([
  {
    $vectorSearch: {
      numCandidates: 100,   // scan 100 candidates, return top 10
      limit: 10
    }
  }
])
// Rule of thumb: numCandidates = limit × 10

// 3. Highlight performance
// Highlighting requires loading the original field text from Lucene
// Avoid highlighting on very long fields (thousands of words)
// Limit highlight characters:
$search: {
  text: { ... },
  highlight: {
    path: "description",
    maxCharsToExamine: 500000,  // only examine first 500K chars
    maxNumPassages: 3           // return at most 3 highlighted passages
  }
}

// 4. count: "total" vs count: "lowerBound"
// count: "total" — exact count (requires scanning all results) — SLOW
// count: "lowerBound" — approximate count (faster, accurate enough for UI)
$search: {
  text: { query: "laptop", path: "name" },
  count: { type: "lowerBound", threshold: 1000 }  // exact up to 1000, then approximate
}

// 5. Use returnStoredSource for frequently accessed fields
// Avoids round-trip to MongoDB storage for common projections

// 6. Search node sizing
// Atlas Search nodes are separate from MongoDB nodes
// M10 Atlas Search = 2GB RAM for Lucene
// Index RAM usage ≈ index size on disk (indexes must fit in RAM for best performance)
// For 10GB search index: need ~10GB RAM on search nodes = M30+ Atlas Search nodes

// 7. Avoid $match before $search
// db.products.aggregate([ { $match: { category: "electronics" } }, { $search: {...} } ])
// WRONG: $match runs first (COLLSCAN!), then $search on subset
// RIGHT: include the filter in $search compound.filter clause
// $search MUST be the first pipeline stage for index to be used
```

---

## Q11: How do you use Atlas Search synonyms?

```javascript
// Synonyms allow queries to match documents that use different words for the same concept
// "laptop" should also match "notebook computer"

// Step 1: Create a synonyms collection
db.synonyms.insertMany([
  // Equivalence synonyms: all terms are interchangeable
  {
    mappingType: "equivalent",
    synonyms: ["laptop", "notebook", "portable computer"]
  },
  // Explicit synonyms: from terms map to explicit terms
  {
    mappingType: "explicit",
    input: ["cellphone", "cell"],
    synonyms: ["mobile phone", "smartphone"]
  }
])

// Step 2: Add synonyms mapping to Atlas Search index
{
  "synonyms": [
    {
      "name": "productSynonyms",
      "analyzer": "lucene.english",
      "source": {
        "collection": "synonyms"   // reference to your synonyms collection
      }
    }
  ]
}

// Step 3: Use synonyms in $search query
db.products.aggregate([
  {
    $search: {
      text: {
        query: "laptop",
        path: "name",
        synonyms: "productSynonyms"   // reference the synonym mapping name
      }
    }
  }
])
// Returns: products matching "laptop" OR "notebook" OR "portable computer"

// Dynamic synonyms: if you update the synonyms collection, Atlas Search syncs automatically
// No index rebuild needed for synonym updates (unlike analyzer changes which require rebuild)

// Use cases:
// Product catalog: "headphones" = "earphones" = "earbuds"
// Medical: "myocardial infarction" = "heart attack"
// Legal: "agreement" = "contract" = "deed"
// Tech: "k8s" = "kubernetes", "ml" = "machine learning"

// Limitations:
// - Synonyms expand queries (more terms to match) → may reduce precision
// - Large synonym lists can slow queries
// - Synonyms are applied at query time (not index time for equivalence)
// - Explicit synonyms are unidirectional by default
//   "cell" → "mobile phone" but "mobile phone" does NOT → "cell"
```

---

## Q12: What is the `$search` `compound` operator? Explain `must`, `should`, `mustNot`, and `filter`.

```javascript
// compound operator combines multiple query clauses
// Each clause affects scoring differently

// MUST: document MUST match ALL must clauses
// Contributes to relevance score
$search: {
  compound: {
    must: [
      { text: { query: "mongodb", path: "title" } },
      { text: { query: "atlas",   path: "title" } }
    ]
    // Document must contain BOTH "mongodb" AND "atlas" in title
    // Score: combines scores from both matches
  }
}

// SHOULD: document doesn't HAVE to match, but matching increases score
// If ALL clauses are "should", at least ONE must match (minimumShouldMatch: 1 by default)
$search: {
  compound: {
    should: [
      { text: { query: "mongodb", path: "title",       score: { boost: { value: 3 } } } },
      { text: { query: "mongodb", path: "description", score: { boost: { value: 1 } } } }
    ],
    minimumShouldMatch: 1   // at least 1 should clause must match
  }
}
// Title matches score 3x higher than description matches

// MUST NOT: document must NOT match any mustNot clause
// Does NOT contribute to score (exclusion only)
$search: {
  compound: {
    must: [
      { text: { query: "database", path: "body" } }
    ],
    mustNot: [
      { equals: { path: "status", value: "deleted" } },
      { equals: { path: "status", value: "archived" } }
    ]
  }
}

// FILTER: like must, but DOES NOT affect relevance score
// Used for "hard constraints" that must be true but aren't relevance factors
$search: {
  compound: {
    must: [
      { text: { query: "laptop", path: "name" } }  // affects score
    ],
    filter: [
      { equals: { path: "isActive", value: true } },    // must be true, no score impact
      { range: { path: "price", gte: 100, lte: 2000 } } // price range, no score impact
    ]
  }
}
// Filtering in compound.filter is more efficient than post-$search $match:
// → Applied in Lucene (before fetching documents from MongoDB)
// → Reduces the number of documents passed to subsequent pipeline stages

// Combining all four:
$search: {
  compound: {
    must: [
      { text: { query: "wireless headphones", path: "name" } }  // must contain these words
    ],
    should: [
      { range: { path: "rating", gte: 4 } },                    // boost: high-rated products
      { equals: { path: "inStock", value: true } }               // boost: in-stock products
    ],
    mustNot: [
      { equals: { path: "category", value: "refurbished" } }    // exclude refurbished
    ],
    filter: [
      { range: { path: "price", lte: 500 } },                   // under $500 (hard filter)
      { equals: { path: "isActive", value: true } }              // active listings only
    ]
  }
}
```
