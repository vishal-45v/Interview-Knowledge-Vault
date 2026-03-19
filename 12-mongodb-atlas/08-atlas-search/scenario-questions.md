# Chapter 08 — Atlas Search: Scenario Questions

---

## Scenario 1: E-Commerce Product Search with Facets

**Scenario**: Build a product search page for an e-commerce site. Users can type a search query, filter by category, price range, and rating. Show facet counts (how many products per category/price range). Handle typos. Sort by relevance.

```javascript
// Atlas Search index configuration:
{
  "name": "productSearch",
  "mappings": {
    "dynamic": false,
    "fields": {
      "name":        [
        { "type": "string",       "analyzer": "lucene.english" },
        { "type": "autocomplete", "tokenization": "edgeGram", "minGrams": 2, "maxGrams": 12 }
      ],
      "description": { "type": "string", "analyzer": "lucene.english" },
      "brand":       { "type": "string", "analyzer": "lucene.standard" },
      "category":    { "type": "stringFacet" },
      "price":       { "type": "numberFacet" },
      "rating":      { "type": "number" },
      "isActive":    { "type": "boolean" },
      "inStock":     { "type": "boolean" }
    }
  },
  "storedSource": {
    "include": ["name", "price", "category", "rating", "imageUrl", "brand"]
  }
}

// Main search function:
async function searchProducts({
  query, category = null, minPrice = null, maxPrice = null,
  minRating = null, page = 1, pageSize = 20
}) {
  const filterClauses = [
    { equals: { path: "isActive", value: true } }
  ]

  if (category) filterClauses.push({ equals: { path: "category", value: category } })
  if (minPrice !== null || maxPrice !== null) {
    filterClauses.push({ range: { path: "price",
      ...(minPrice !== null ? { gte: minPrice } : {}),
      ...(maxPrice !== null ? { lte: maxPrice } : {})
    }})
  }
  if (minRating !== null) {
    filterClauses.push({ range: { path: "rating", gte: minRating } })
  }

  const searchQuery = query ? {
    compound: {
      should: [
        { text: { query, path: "name",        score: { boost: { value: 3 } } } },
        { text: { query, path: "brand",       score: { boost: { value: 2 } } } },
        { text: { query, path: "description", score: { boost: { value: 1 } } } }
      ],
      filter: filterClauses
    }
  } : {
    compound: { filter: filterClauses }
  }

  const [results, meta] = await Promise.all([
    db.products.aggregate([
      { $search: { index: "productSearch", ...searchQuery, returnStoredSource: true } },
      { $skip: (page - 1) * pageSize },
      { $limit: pageSize },
      { $project: { name: 1, price: 1, category: 1, rating: 1, imageUrl: 1, brand: 1,
          score: { $meta: "searchScore" } } }
    ]).toArray(),

    db.products.aggregate([
      {
        $searchMeta: {
          index: "productSearch",
          facet: {
            operator: searchQuery,
            facets: {
              categoryFacet: { type: "string", path: "category", numBuckets: 15 },
              priceFacet: {
                type: "number", path: "price",
                boundaries: [0, 25, 50, 100, 200, 500, 1000],
                default: "1000+"
              }
            }
          }
        }
      }
    ]).toArray()
  ])

  return {
    results,
    facets: meta[0]?.facet || {},
    total: meta[0]?.count?.lowerBound || 0,
    page, pageSize,
    totalPages: Math.ceil((meta[0]?.count?.lowerBound || 0) / pageSize)
  }
}
```

---

## Scenario 2: Search-as-You-Type Autocomplete

**Scenario**: Implement a fast search box that shows suggestions as the user types. "mongo" should suggest "MongoDB", "Mongoose", "MongoDB Atlas", etc. Suggestions should appear in < 50ms.

```javascript
// The search index already has autocomplete mapping (from Scenario 1)

// Debounced autocomplete endpoint:
// Client-side: fire after 150ms of no typing
// Server-side: cache common queries in Redis with TTL

async function autocomplete(partialQuery, userId = null) {
  if (partialQuery.length < 2) return []

  // Check Redis cache first
  const cacheKey = `autocomplete:${partialQuery.toLowerCase()}`
  const cached = await redis.get(cacheKey)
  if (cached) return JSON.parse(cached)

  const suggestions = await db.products.aggregate([
    {
      $search: {
        index: "productSearch",
        autocomplete: {
          query: partialQuery,
          path: "name",
          tokenOrder: "sequential",
          fuzzy: { maxEdits: 1, prefixLength: 1 }  // first char exact, then fuzzy
        }
      }
    },
    { $limit: 8 },
    { $project: { name: 1, category: 1, _id: 0 } },
    // Deduplicate by name prefix
    { $group: { _id: "$name", category: { $first: "$category" } } },
    { $limit: 5 }
  ]).toArray()

  // Cache for 60 seconds (popular queries change slowly)
  await redis.setex(cacheKey, 60, JSON.stringify(suggestions))

  return suggestions
}

// Also track search queries for analytics (popular searches feature)
async function trackSearch(query, userId, resultsCount) {
  await db.searchAnalytics.insertOne({
    query: query.toLowerCase(),
    userId: userId ? ObjectId(userId) : null,
    resultsCount,
    timestamp: new Date()
  })
}

// Popular searches (Computed Pattern):
db.searchAnalytics.aggregate([
  { $match: { timestamp: { $gte: new Date(Date.now() - 24 * 60 * 60 * 1000) } } },
  { $group: { _id: "$query", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

---

## Scenario 3: Multi-Language Product Search

**Scenario**: Global marketplace with products in English, Spanish, French, and German. Users search in their language. "laptop" in English, "portátil" in Spanish, "ordinateur portable" in French should all find the right products.

```javascript
// Atlas Search index with multiple language analyzers:
{
  "name": "multiLingualSearch",
  "mappings": {
    "dynamic": false,
    "fields": {
      "name.en": { "type": "string", "analyzer": "lucene.english" },
      "name.es": { "type": "string", "analyzer": "lucene.spanish" },
      "name.fr": { "type": "string", "analyzer": "lucene.french" },
      "name.de": { "type": "string", "analyzer": "lucene.german" },
      "category": { "type": "stringFacet" },
      "price": { "type": "number" }
    }
  }
}

// Document structure:
{
  _id: ObjectId("..."),
  sku: "LAPTOP-001",
  name: {
    en: "Laptop Computer 15 inch",
    es: "Ordenador Portátil 15 pulgadas",
    fr: "Ordinateur Portable 15 pouces",
    de: "Laptop Computer 15 Zoll"
  },
  price: NumberDecimal("599.99"),
  category: "computers"
}

// Search in user's language:
async function multiLingualSearch(query, language, filters = {}) {
  const langPath = `name.${language}`

  return db.products.aggregate([
    {
      $search: {
        index: "multiLingualSearch",
        compound: {
          must: [
            {
              text: {
                query: query,
                path: langPath,
                fuzzy: { maxEdits: 1 }
              }
            }
          ],
          filter: Object.entries(filters).map(([key, val]) => ({
            equals: { path: key, value: val }
          }))
        }
      }
    },
    { $limit: 20 },
    {
      $project: {
        name: `$name.${language}`,
        price: 1,
        category: 1,
        score: { $meta: "searchScore" }
      }
    }
  ]).toArray()
}

// Synonym support per language:
db.spanishSynonyms.insertMany([
  { mappingType: "equivalent", synonyms: ["portátil", "laptop", "computadora portátil"] },
  { mappingType: "equivalent", synonyms: ["móvil", "celular", "teléfono inteligente"] }
])
// Reference in index: { "name": "spanishSynonyms", "analyzer": "lucene.spanish", "source": { "collection": "spanishSynonyms" } }
```

---

## Scenario 4: Full-Text Search with Permissions/Access Control

**Scenario**: A document management system where users can only search documents they have access to. Documents have a `permissions` array of user IDs. How do you implement this without leaking restricted documents?

```javascript
// Document schema:
{
  _id: ObjectId("..."),
  title: "Q4 Financial Report",
  body: "Confidential financial data...",
  permissions: {
    userIds: [ObjectId("alice_id"), ObjectId("bob_id")],
    roles: ["finance_team", "executive"]
  },
  departmentId: "finance",
  isPublic: false
}

// Atlas Search index:
{
  "mappings": {
    "fields": {
      "title": { "type": "string", "analyzer": "lucene.english" },
      "body":  { "type": "string", "analyzer": "lucene.english" },
      "permissions.userIds": { "type": "objectId" },
      "permissions.roles":   { "type": "string", "analyzer": "lucene.keyword" },
      "isPublic": { "type": "boolean" },
      "departmentId": { "type": "string", "analyzer": "lucene.keyword" }
    }
  }
}

// Secure search function:
async function searchDocuments(query, currentUser) {
  const userRoles = currentUser.roles   // e.g., ["employee", "finance_team"]

  return db.documents.aggregate([
    {
      $search: {
        compound: {
          must: [
            { text: { query, path: ["title", "body"] } }
          ],
          filter: [
            {
              // Access control: public OR user in permissions OR role in permissions
              compound: {
                should: [
                  { equals: { path: "isPublic", value: true } },
                  { equals: { path: "permissions.userIds", value: currentUser._id } },
                  // Can't do $in directly in Atlas Search — use should for multiple roles
                  ...userRoles.map(role => ({
                    equals: { path: "permissions.roles", value: role }
                  }))
                ],
                minimumShouldMatch: 1
              }
            }
          ]
        }
      }
    },
    { $limit: 20 },
    {
      $project: {
        title: 1,
        excerpt: { $substr: ["$body", 0, 200] },
        departmentId: 1,
        score: { $meta: "searchScore" }
      }
    }
  ]).toArray()
}

// Important: always validate access control in Atlas Search query, NOT just in application
// A malicious client bypassing the application could call MongoDB directly
// Defense in depth: also use MongoDB role-based access control at DB level
```

---

## Scenario 5: Relevance-Tuned Job Search

**Scenario**: A job board search. Job seekers search by title and skills. Requirements:
- Title match is most important (10x boost)
- Recent jobs score higher than old jobs (decay by age)
- Jobs with salary listed score slightly higher
- User can filter by location, job type (full-time/contract), salary range

```javascript
// Atlas Search index:
{
  "name": "jobSearch",
  "mappings": {
    "fields": {
      "title":       { "type": "string", "analyzer": "lucene.english" },
      "skills":      { "type": "string", "analyzer": "lucene.standard" },
      "description": { "type": "string", "analyzer": "lucene.english" },
      "location":    { "type": "string", "analyzer": "lucene.keyword" },
      "jobType":     { "type": "string", "analyzer": "lucene.keyword" },
      "postedAt":    { "type": "date" },
      "salaryMin":   { "type": "number" },
      "salaryMax":   { "type": "number" },
      "hasSalary":   { "type": "boolean" }
    }
  }
}

async function searchJobs({ query, location, jobType, minSalary, maxSalary, page = 1 }) {
  const filter = []
  if (location) filter.push({ equals: { path: "location", value: location } })
  if (jobType)  filter.push({ equals: { path: "jobType", value: jobType } })
  if (minSalary !== null) filter.push({ range: { path: "salaryMin", gte: minSalary } })
  if (maxSalary !== null) filter.push({ range: { path: "salaryMax", lte: maxSalary } })

  return db.jobs.aggregate([
    {
      $search: {
        index: "jobSearch",
        compound: {
          must: query ? [
            {
              compound: {
                should: [
                  // Title match: 10x boost
                  { text: { query, path: "title",       score: { boost: { value: 10 } } } },
                  // Skills match: 3x boost
                  { text: { query, path: "skills",      score: { boost: { value: 3 } } } },
                  // Description match: 1x (baseline)
                  { text: { query, path: "description" } }
                ],
                minimumShouldMatch: 1
              }
            }
          ] : [],
          should: [
            // Recency boost: jobs posted in last 7 days score higher
            {
              near: {
                path: "postedAt",
                origin: new Date(),
                pivot: 7 * 24 * 60 * 60 * 1000   // 7 days in ms
              }
            },
            // Salary listed boost
            { equals: { path: "hasSalary", value: true } }
          ],
          filter: filter
        },
        count: { type: "lowerBound", threshold: 1000 }
      }
    },
    { $skip: (page - 1) * 20 },
    { $limit: 20 },
    {
      $project: {
        title: 1, company: 1, location: 1, jobType: 1,
        salaryMin: 1, salaryMax: 1, postedAt: 1,
        score: { $meta: "searchScore" }
      }
    }
  ]).toArray()
}
```

---

## Scenario 6: Atlas Search for Slack-Like Message Search

**Scenario**: A team messaging app (like Slack) needs message search. Users search their workspace's messages. Requirements:
- Fast (< 100ms for results)
- Search by keywords in message body
- Filter by channel, date range, author
- Highlight matching terms in the message
- Support phrase search (exact phrase in quotes)

```javascript
// Atlas Search index for messages:
{
  "name": "messageSearch",
  "mappings": {
    "dynamic": false,
    "fields": {
      "body":      { "type": "string", "analyzer": "lucene.standard" },
      "channelId": { "type": "objectId" },
      "authorId":  { "type": "objectId" },
      "sentAt":    { "type": "date" },
      "workspaceId": { "type": "objectId" }
    }
  },
  "storedSource": {
    "include": ["body", "authorId", "channelId", "sentAt"]
  }
}

async function searchMessages({ query, workspaceId, channelId, authorId, startDate, endDate, page = 1 }) {
  const filter = [
    { equals: { path: "workspaceId", value: ObjectId(workspaceId) } }
  ]
  if (channelId) filter.push({ equals: { path: "channelId", value: ObjectId(channelId) } })
  if (authorId)  filter.push({ equals: { path: "authorId",  value: ObjectId(authorId) } })
  if (startDate || endDate) {
    filter.push({ range: {
      path: "sentAt",
      ...(startDate ? { gte: new Date(startDate) } : {}),
      ...(endDate   ? { lte: new Date(endDate)   } : {})
    }})
  }

  // Parse query: if wrapped in quotes, use phrase search
  const isPhrase = query.startsWith('"') && query.endsWith('"')
  const searchText = query.replace(/"/g, '')

  const textClause = isPhrase
    ? { phrase: { query: searchText, path: "body" } }         // exact phrase
    : { text:   { query: searchText, path: "body",             // keyword search
          fuzzy: { maxEdits: 1, prefixLength: 3 } } }

  return db.messages.aggregate([
    {
      $search: {
        index: "messageSearch",
        compound: {
          must:   [ textClause ],
          filter: filter
        },
        highlight: {
          path: "body",
          maxCharsToExamine: 1000,
          maxNumPassages: 1
        },
        returnStoredSource: true
      }
    },
    { $sort: { sentAt: -1 } },    // most recent first
    { $skip: (page - 1) * 50 },
    { $limit: 50 },
    {
      $project: {
        body: 1, authorId: 1, channelId: 1, sentAt: 1,
        score: { $meta: "searchScore" },
        highlights: { $meta: "searchHighlights" }
      }
    }
  ]).toArray()
}

// Using highlights in the UI:
// highlights: [{ path: "body", texts: [
//   { value: "Deploy to ", type: "text" },
//   { value: "MongoDB", type: "hit" },       ← highlighted term
//   { value: " Atlas cluster", type: "text" }
// ]}]
// Render: "Deploy to **MongoDB** Atlas cluster"
```

---

## Scenario 7: Atlas Search for Healthcare Provider Directory

**Scenario**: Build a healthcare provider search. Patients can search for doctors by:
- Specialty (cardiologist, dermatologist)
- Name or practice name
- Insurance accepted
- Location (city + distance)
- Languages spoken
- Availability (accepting new patients)

```javascript
// Atlas Search index (with geo support):
{
  "name": "providerSearch",
  "mappings": {
    "dynamic": false,
    "fields": {
      "name":            { "type": "string", "analyzer": "lucene.standard" },
      "practiceName":    { "type": "string", "analyzer": "lucene.standard" },
      "specialty":       { "type": "string", "analyzer": "lucene.keyword" },
      "specialtySearch": { "type": "string", "analyzer": "lucene.english" },
      "insurances":      { "type": "string", "analyzer": "lucene.keyword" },
      "languages":       { "type": "string", "analyzer": "lucene.keyword" },
      "acceptingNewPatients": { "type": "boolean" },
      "location": {
        "type": "geo"    // GeoJSON Point
      }
    }
  },
  "synonyms": [
    {
      "name": "medicalSynonyms",
      "analyzer": "lucene.english",
      "source": { "collection": "medicalSynonyms" }
    }
  ]
}

// Medical synonyms:
db.medicalSynonyms.insertMany([
  { mappingType: "equivalent", synonyms: ["cardiologist", "heart doctor", "cardiac specialist"] },
  { mappingType: "equivalent", synonyms: ["dermatologist", "skin doctor", "skin specialist"] },
  { mappingType: "equivalent", synonyms: ["OB/GYN", "obstetrician", "gynecologist"] }
])

async function findProviders({ query, specialty, insurance, language, city, radiusMiles, acceptingNew }) {
  const filter = []
  if (specialty)    filter.push({ equals: { path: "specialty", value: specialty } })
  if (insurance)    filter.push({ equals: { path: "insurances", value: insurance } })
  if (language)     filter.push({ equals: { path: "languages", value: language } })
  if (acceptingNew) filter.push({ equals: { path: "acceptingNewPatients", value: true } })

  const mustClauses = []
  if (query) {
    mustClauses.push({
      compound: {
        should: [
          { text: { query, path: ["name", "practiceName"], score: { boost: { value: 3 } } } },
          { text: { query, path: "specialtySearch", synonyms: "medicalSynonyms" } }
        ],
        minimumShouldMatch: 1
      }
    })
  }

  // Geo filter (if city coordinates provided)
  if (city?.coordinates) {
    filter.push({
      geoWithin: {
        circle: {
          center: { type: "Point", coordinates: city.coordinates },
          radius: radiusMiles * 1609.34  // miles to meters
        },
        path: "location"
      }
    })
  }

  return db.providers.aggregate([
    {
      $search: {
        index: "providerSearch",
        compound: {
          must: mustClauses,
          filter: filter
        }
      }
    },
    { $limit: 20 },
    {
      $project: {
        name: 1, practiceName: 1, specialty: 1,
        location: 1, phone: 1, acceptingNewPatients: 1,
        score: { $meta: "searchScore" }
      }
    }
  ]).toArray()
}
```

---

## Scenario 8: Migrating from $text to Atlas Search

**Scenario**: Your application uses native `$text` indexes. Performance is degrading as your collection grows (10M documents). Users complain about irrelevant results and no autocomplete. How do you migrate to Atlas Search?

```javascript
// BEFORE: native $text index
db.articles.createIndex({ title: "text", body: "text", tags: "text" },
  { weights: { title: 3, tags: 2, body: 1 } })

db.articles.find(
  { $text: { $search: "mongodb performance optimization" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })

// MIGRATION PLAN:
// Step 1: Create Atlas Search index (parallel to existing $text index)
// → No impact on production queries yet
// → Index builds asynchronously in background

// Step 2: Test Atlas Search queries in staging/development
db.articles.aggregate([
  {
    $search: {
      index: "articleSearch",
      compound: {
        should: [
          { text: { query: "mongodb performance optimization",
              path: "title", score: { boost: { value: 3 } } } },
          { text: { query: "mongodb performance optimization",
              path: "tags", score: { boost: { value: 2 } } } },
          { text: { query: "mongodb performance optimization",
              path: "body" } }
        ]
      }
    }
  },
  { $limit: 10 },
  { $project: { title: 1, score: { $meta: "searchScore" } } }
])

// Step 3: Compare results quality (A/B test)
// Run both queries, compare:
// - Relevance of top 10 results
// - Response time (Atlas Search usually faster for complex queries)
// - Typo tolerance (Atlas Search handles "mongod" → "mongodb", $text does not)

// Step 4: Feature flag-controlled switch
async function searchArticles(query, useAtlasSearch = false) {
  if (useAtlasSearch) {
    return db.articles.aggregate([
      { $search: { compound: { should: [
            { text: { query, path: "title", score: { boost: { value: 3 } } } },
            { text: { query, path: "body" } }
      ]}}}
    ]).toArray()
  } else {
    return db.articles.find(
      { $text: { $search: query } },
      { score: { $meta: "textScore" } }
    ).sort({ score: { $meta: "textScore" } }).toArray()
  }
}

// Step 5: Gradual rollout (10% → 25% → 50% → 100%)
const useAtlasSearch = Math.random() < 0.10   // 10% of requests

// Step 6: After full migration, drop $text index
db.articles.dropIndex("title_text_body_text_tags_text")
// Removing unused index improves write performance!

// Added Atlas Search features post-migration:
// - Autocomplete (new product feature unlocked)
// - Fuzzy matching (better user experience)
// - Highlighting (show matched terms in snippets)
// - Faceted search (new filtering UI)
```
