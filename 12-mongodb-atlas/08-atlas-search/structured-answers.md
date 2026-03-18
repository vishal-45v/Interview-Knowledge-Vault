# Chapter 08 — Atlas Search: Structured Answers

---

## Answer 1: How Does Atlas Search Work Under the Hood?

**Point**: Atlas Search runs Apache Lucene on dedicated nodes alongside your Atlas cluster. A sync process (mongot) tails the MongoDB oplog and keeps the Lucene index updated asynchronously.

**Example**:

```javascript
// Write flow:
// 1. db.products.insertOne({ name: "Wireless Headphones", price: 79.99 })
//    → MongoDB stores in WiredTiger, writes to oplog
// 2. mongot (Atlas Search daemon) tails the oplog (~1-5 seconds lag)
// 3. Lucene index updated: "wireless" + "headphones" tokens added

// Read flow ($search query):
db.products.aggregate([
  { $search: { text: { query: "wireless headphones", path: "name" } } }
])
// 1. Pipeline reaches $search stage
// 2. MongoDB forwards query to mongot (Lucene)
// 3. Lucene executes BM25 scoring, returns ranked document IDs
// 4. MongoDB fetches full documents by those IDs from WiredTiger
// 5. Returns to caller

// Key characteristics:
// - $search MUST be the first pipeline stage
// - Asynchronous sync: new documents searchable after ~1-5 seconds
// - Stored source: skip step 4 (field values stored in Lucene directly)
// - Score: Lucene's BM25 ranking — not MongoDB's default sort

// In MongoDB explain():
db.products.explain().aggregate([
  { $search: { text: { query: "headphones", path: "name" } } }
])
// Shows: "SEARCH" stage → handled by mongot, not by WiredTiger
```

---

## Answer 2: Explain the `compound` Operator — must/should/mustNot/filter

**Point**: The `compound` operator combines multiple search clauses with different semantic meanings. `must`/`should` affect relevance scoring while `mustNot`/`filter` are pure boolean constraints.

**Example**:

```javascript
db.articles.aggregate([
  {
    $search: {
      compound: {
        // MUST match all (boolean AND + affects score)
        must: [
          { text: { query: "mongodb", path: "title" } }
        ],

        // SHOULD match (boosts score but not required)
        should: [
          { text: { query: "atlas", path: "title", score: { boost: { value: 2 } } } },
          { range: { path: "rating", gte: 4 } }  // high-rated articles get score boost
        ],

        // MUST NOT match (exclusion, no score impact)
        mustNot: [
          { equals: { path: "status", value: "deleted" } }
        ],

        // FILTER (boolean AND, no score impact — most efficient for hard constraints)
        filter: [
          { equals: { path: "isPublished", value: true } },
          { range: { path: "publishedAt", gte: ISODate("2024-01-01") } }
        ]
      }
    }
  }
])

// Score composition:
// Document A: matches must (5.0) + matches should[0] (2.0 × boost=2 = 4.0) = 9.0
// Document B: matches must (5.0) + matches should[1] (1.0) = 6.0
// → Document A ranks higher even though both match the required criteria

// Practical pattern:
// must   → the core query (what users typed)
// should → quality signals (recency, ratings, in-stock)
// mustNot → exclusions (deleted, archived, adult content)
// filter → access control, tenant ID, date ranges (required but not "searchable")
```

---

## Answer 3: How Do You Implement Faceted Search With Atlas Search?

**Point**: Use `$searchMeta` with the `facet` operator to get facet counts separately from search results. Run both queries in parallel for efficiency.

**Example**:

```javascript
// Index configuration (stringFacet and numberFacet types required):
{
  "mappings": {
    "fields": {
      "category": { "type": "stringFacet" },
      "price":    { "type": "numberFacet" }
    }
  }
}

// Parallel queries: results + facets
const [results, meta] = await Promise.all([
  // Search results
  db.products.aggregate([
    { $search: {
        compound: {
          should: [{ text: { query: "laptop", path: "name" } }],
          filter: [{ equals: { path: "isActive", value: true } }]
        }
    }},
    { $limit: 20 },
    { $project: { name: 1, price: 1, category: 1 } }
  ]).toArray(),

  // Facet counts
  db.products.aggregate([
    { $searchMeta: {
        facet: {
          operator: {
            compound: {
              should: [{ text: { query: "laptop", path: "name" } }],
              filter: [{ equals: { path: "isActive", value: true } }]
            }
          },
          facets: {
            categoryBuckets: { type: "string",  path: "category", numBuckets: 10 },
            priceBuckets:    { type: "number",  path: "price",
              boundaries: [0, 100, 250, 500, 1000], default: "1000+" }
          }
        }
    }}
  ]).toArray()
])

// UI renders:
// Results: 20 products
// Category filters: Electronics (523), Computers (312), Accessories (198)
// Price filters: $0-100 (145), $100-250 (423), $250-500 (312), $500-1000 (234), $1000+ (133)
```

---

## Answer 4: How Do You Handle Synonyms in Atlas Search?

**Point**: Atlas Search synonyms are defined in a MongoDB collection, referenced from the search index, and applied at query time. They come in two flavors: equivalent (bidirectional) and explicit (unidirectional).

**Example**:

```javascript
// 1. Create synonyms collection
db.synonyms.insertMany([
  // Equivalent: any term matches any other
  { mappingType: "equivalent", synonyms: ["laptop", "notebook computer", "portable computer"] },
  // Explicit: input terms mapped to synonym terms (unidirectional)
  { mappingType: "explicit", input: ["k8s", "kube"], synonyms: ["kubernetes"] }
])

// 2. Reference in index configuration
{
  "synonyms": [{
    "name": "mySynonyms",
    "analyzer": "lucene.english",
    "source": { "collection": "synonyms" }
  }]
}

// 3. Use in query
db.products.aggregate([
  { $search: {
      text: {
        query: "notebook",
        path: "name",
        synonyms: "mySynonyms"     // "notebook" also searches for "laptop"
      }
  }}
])

// Key advantages vs fuzzy:
// Synonyms handle SEMANTIC equivalence (k8s → kubernetes: different words, same meaning)
// Fuzzy handles TYPOGRAPHIC similarity (mongdb → mongodb: same word, typo)

// Dynamic synonym updates:
// Update the synonyms collection → Atlas Search syncs automatically (no index rebuild)
// Great for evolving vocabularies (new product names, industry jargon)
```

---

## Answer 5: When Do You Use `returnStoredSource`?

**Point**: `returnStoredSource` tells Atlas Search to return field values directly from the Lucene stored source, bypassing the round-trip to MongoDB storage. Use for high-traffic search endpoints where you only need a small projection.

**Example**:

```javascript
// Configure stored source in the index:
{
  "storedSource": {
    "include": ["name", "price", "category", "thumbnailUrl", "rating"]
  }
}

// Query WITHOUT returnStoredSource (default):
// 1. Lucene finds matching doc IDs
// 2. MongoDB fetches full documents from WiredTiger (random I/O)
// 3. Apply projection
// Latency: +5-20ms for storage fetch per batch

// Query WITH returnStoredSource:
db.products.aggregate([
  { $search: {
      text: { query: "laptop", path: "name" },
      returnStoredSource: true    // get fields from Lucene directly
  }},
  { $project: { name: 1, price: 1, category: 1, thumbnailUrl: 1 } }
])
// 1. Lucene finds matching doc IDs + returns storedSource fields
// 2. No MongoDB storage fetch needed
// Latency: -5-20ms (Lucene already has the data)

// When the trade-off makes sense:
// ✓ Search results page (name, price, thumbnail: all in stored source)
// ✓ Autocomplete (just need name and category)
// ✓ High-throughput endpoints (1000 req/sec → 20ms savings × 1000 = significant)

// When NOT worth it:
// ✗ You need fields not in storedSource (falls back to MongoDB fetch anyway)
// ✗ Small collection (<1M docs): MongoDB storage fetch is fast enough
// ✗ Low traffic: premature optimization

// Storage cost: ~20-50% extra per stored field in Lucene index
// Worth it for: top 5-10 projection fields on your main search page
```

---

## Answer 6: How Do You Tune Atlas Search Relevance?

**Point**: Relevance tuning in Atlas Search combines BM25 defaults with explicit boosts, decay functions, and field weighting to match your business requirements.

**Example**:

```javascript
// Start with what matters most:
// For a job board:
// 1. Title match is critical (10x boost)
// 2. Skills match is important (3x boost)
// 3. Recent jobs preferred (decay by age)
// 4. Jobs with salary are more engaging (small boost)

db.jobs.aggregate([
  {
    $search: {
      compound: {
        must: [{
          compound: {
            should: [
              { text: { query: userQuery, path: "title",
                  score: { boost: { value: 10 } } } },
              { text: { query: userQuery, path: "skills",
                  score: { boost: { value: 3 } } } },
              { text: { query: userQuery, path: "description" } }
            ],
            minimumShouldMatch: 1
          }
        }],
        should: [
          // Recency boost (decay function)
          { near: { path: "postedAt", origin: new Date(),
              pivot: 7 * 24 * 60 * 60 * 1000 } },  // halves every 7 days
          // Salary boost
          { equals: { path: "hasSalary", value: true } }
        ]
      }
    }
  }
])

// A/B test your scoring:
// Track: click-through rate, time-to-apply for each scoring variant
// High click-through on top results → good relevance
// Low click-through → relevance needs tuning

// scoreDetails for debugging relevance:
db.jobs.aggregate([
  { $search: {
      text: { query: "software engineer", path: "title" },
      tracking: { searchTerms: "software engineer" },   // for Atlas Search analytics
      scoreDetails: true    // not a standard option, but Atlas Search provides score details in explain
  }},
  { $project: {
      title: 1,
      score: { $meta: "searchScore" },
      scoreDetails: { $meta: "searchScoreDetails" }  // if available
  }}
])
```

---

## Answer 7: Explain the Atlas Search Index Build Process

**Point**: Creating an Atlas Search index is an online operation that doesn't block reads or writes. The build goes through several phases and takes time proportional to collection size.

**Example**:

```javascript
// Create index (via Atlas UI, CLI, or Admin API):
// Atlas CLI:
// atlas clusters search indexes create --clusterName MyCluster --file searchIndex.json

// Index build phases:
// 1. BUILDING: Lucene processes all existing documents (initial scan)
//    - Reads all documents from MongoDB
//    - Builds Lucene index segments
//    - Duration: minutes to hours (depends on collection size)
//    - During build: $search queries may return partial results or error
//    - MongoDB reads/writes: NOT blocked

// 2. BUILDING (syncing): processing oplog changes that happened during step 1
//    - Catches up with writes that occurred during the initial scan

// 3. READY: index fully built and in sync
//    - All $search queries return accurate results
//    - Ongoing sync: 1-5 second lag after writes

// Monitor index status:
db.products.aggregate([{ $searchIndexes: {} }])
// Returns:
// { id: "...", name: "default", status: "BUILDING", queryable: false }
// { id: "...", name: "default", status: "READY",    queryable: true }

// Queryable: false means $search will error or return empty
// Best practice: check status before using $search in application startup

// Dropping an index:
// Atlas UI → Search Indexes → Drop
// CLI: atlas clusters search indexes delete <indexId> --clusterName MyCluster
// NOTE: dropping an index does NOT delete the documents from MongoDB
// It only removes the Lucene index (saves storage, disables $search for that index)

// Re-syncing (if index becomes stale):
// Atlas UI → Search Indexes → Sync (forces full rebuild)
// Usually happens automatically — only trigger manually if status shows STALE for long
```

---

## Answer 8: How Does Atlas Vector Search Differ from Regular Atlas Search?

**Point**: Atlas Search (Lucene) is for keyword/text matching. Atlas Vector Search is for semantic similarity using AI-generated embeddings. They use different index types, different query operators, and serve different use cases.

**Example**:

```javascript
// Atlas Search ($search):
// Find documents containing the WORD "performance"
db.articles.aggregate([
  { $search: { text: { query: "mongodb performance", path: "title" } } }
])
// Returns: "MongoDB Performance Optimization", "Improving MongoDB Performance", etc.
// Keyword-based: "mongodb" AND "performance" as words

// Atlas Vector Search ($vectorSearch):
// Find documents SEMANTICALLY SIMILAR to "how to make queries faster"
const queryEmbedding = await openai.embeddings.create({ model: "text-embedding-ada-002",
  input: "how to make queries faster" }).then(r => r.data[0].embedding)

db.articles.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",    // type: vectorSearch (different index type!)
      path: "embedding",        // array of floats, not a text field
      queryVector: queryEmbedding,
      numCandidates: 100,
      limit: 5
    }
  }
])
// Returns: articles about "query optimization", "index selection", "explain() output"
// Even if none contain the exact words "make queries faster"

// Index type comparison:
// Atlas Search index:
{
  "type": "search",   // Lucene full-text index
  "mappings": { "fields": { "title": { "type": "string" } } }
}
// Atlas Vector Search index:
{
  "type": "vectorSearch",    // ANN (approximate nearest neighbor) index
  "fields": [{
    "type": "vector",
    "path": "embedding",
    "numDimensions": 1536,   // must match your AI model's output dimensions
    "similarity": "cosine"   // distance metric
  }]
}

// Combining both (hybrid search):
db.articles.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryEmbedding,
      numCandidates: 100,
      limit: 50    // bring back more candidates for re-ranking
    }
  },
  {
    $search: {      // WRONG: can't combine $search and $vectorSearch this way
      // ...
    }
  }
])
// For hybrid search: Atlas Search has a native compound operator combining text + vector
// (Atlas Search vector operator is separate from $vectorSearch)
```

---

## Answer 9: How Do You Migrate from $text to Atlas Search?

**Point**: Migration is a gradual process: build the Atlas Search index alongside the existing `$text` index, test with a feature flag, then switch traffic over progressively.

**Example**:

```javascript
// Step 1: Assess current $text usage
db.system.profile.find({ "command.filter.$text": { $exists: true } })
// Find all slow $text queries → understand what needs to be migrated

// Step 2: Create Atlas Search index (non-breaking, runs in parallel)
// The $text index continues to serve production queries during index build

// Step 3: Map $text features to Atlas Search:
// $text: { $search: "quick brown fox" }
// → compound.should with text operators (OR semantics)
db.articles.aggregate([
  { $search: {
      compound: {
        should: [
          { text: { query: "quick brown fox", path: "body" } }
        ],
        minimumShouldMatch: 1
      }
  }}
])

// $text: { $search: "\"exact phrase\"" } (phrase search in $text)
// → phrase operator
db.articles.aggregate([
  { $search: { phrase: { query: "exact phrase", path: "body" } } }
])

// $text: { $search: "-exclude this" } (negation in $text)
// → compound.mustNot
db.articles.aggregate([
  { $search: {
      compound: {
        should: [{ text: { query: "quick brown fox", path: "body" } }],
        mustNot: [{ text: { query: "exclude", path: "body" } }]
      }
  }}
])

// Step 4: Feature flag and gradual rollout
const ATLAS_SEARCH_ROLLOUT = parseFloat(process.env.ATLAS_SEARCH_ROLLOUT || "0")
const useAtlasSearch = Math.random() < ATLAS_SEARCH_ROLLOUT
// Start at 0.01 (1%), monitor, increase to 0.1, 0.5, 1.0 over weeks

// Step 5: After 100% rollout, drop $text index
db.articles.dropIndex("body_text_title_text")
// ✓ Write performance improves (one less B-tree to maintain)
// ✓ Storage recovered (inverted index removed)
```

---

## Answer 10: How Do You Monitor and Debug Atlas Search Performance?

**Point**: Atlas Search monitoring uses Atlas UI metrics, index status checks, and the explain() pipeline to diagnose performance issues.

**Example**:

```javascript
// 1. Check index health and sync status:
db.products.aggregate([{ $searchIndexes: {} }])
// { name: "default", status: "READY", queryable: true, numDocuments: 1247832 }

// 2. Explain a $search query:
db.products.explain("executionStats").aggregate([
  { $search: { text: { query: "laptop", path: "name" } } },
  { $limit: 10 }
])
// Look for:
// "SEARCH" stage executionTimeMillisEstimate
// nReturned vs total docs
// Whether returnStoredSource is being used

// 3. Atlas UI monitoring metrics for Atlas Search:
// - Lucene indexing latency: how long ago was the last sync?
// - Search operation latency P95, P99
// - Index disk size (watching for bloat)
// - Query queue depth (backpressure indicator)

// 4. Slow search queries:
// Atlas UI → Performance Advisor (shows slow $search queries separately)
// OR: use MongoDB profiler with $search stage detection
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find({ "command.pipeline.0.$search": { $exists: true } })
  .sort({ millis: -1 }).limit(10)

// 5. Common performance problems and fixes:
// Problem: $search returning millions of results (no limit)
// Fix: always add { $limit: N } after $search

// Problem: index size > available RAM
// Fix: use static mappings (not dynamic) + dedicated Atlas Search nodes

// Problem: sync lag > 30 seconds after writes
// Fix: check Atlas Search node health, check oplog window, consider write rate

// Problem: wildcard /* regex queries are slow
// Fix: replace with autocomplete operator or anchored patterns

// 6. Benchmark Atlas Search vs $text index:
const start = Date.now()
await db.articles.find({ $text: { $search: "mongodb" } }).limit(10).toArray()
console.log("$text:", Date.now() - start, "ms")

const start2 = Date.now()
await db.articles.aggregate([
  { $search: { text: { query: "mongodb", path: "title" } } },
  { $limit: 10 }
]).toArray()
console.log("Atlas Search:", Date.now() - start2, "ms")
// Atlas Search is typically faster for complex queries, comparable for simple ones
```
