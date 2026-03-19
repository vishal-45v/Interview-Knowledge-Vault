# Chapter 07 — Atlas Features: Scenario Questions

---

## Scenario 1: Setting Up Automated Alerting for Production

**Scenario**: Your Atlas production cluster handles an e-commerce platform. You need alerts for:
- High connections (> 80% of max)
- Slow queries (P99 latency > 500ms)
- Disk space > 75%
- Replication lag > 30 seconds

Show how to configure Atlas alerts and what to do when they fire.

```javascript
// Atlas Alert configuration (via Atlas API or UI):
// POST https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/alertConfigs

// 1. High connections alert
{
  "eventTypeName": "CONNECTIONS_PERCENT_OVER_THRESHOLD",
  "enabled": true,
  "threshold": {
    "operator": "GREATER_THAN",
    "threshold": 80,
    "units": "PERCENT"
  },
  "notifications": [
    { "typeName": "SLACK",       "slackApiToken": "...", "channelName": "#ops-alerts" },
    { "typeName": "EMAIL",       "emailAddress": "dba-team@example.com", "intervalMin": 60 },
    { "typeName": "PAGERDUTY",   "serviceKey": "...", "intervalMin": 5 }
  ]
}

// 2. Slow query alert (Atlas Integration with Datadog / CloudWatch)
// Atlas doesn't have a native "P99 latency" alert, but:
// - Enable Atlas Monitoring → push metrics to Datadog/Prometheus
// - Create alert on: opcounters.query > X per second + executionTimeMillis P99 > 500

// 3. Disk space alert
{
  "eventTypeName": "DISK_AUTO_SCALE_INITIATED",
  "enabled": true
}
// AND:
{
  "eventTypeName": "DISK_PARTITION_SPACE_USED_PERCENT_OVER_THRESHOLD",
  "enabled": true,
  "threshold": { "operator": "GREATER_THAN", "threshold": 75, "units": "PERCENT" }
}

// 4. Replication lag alert
{
  "eventTypeName": "REPLICATION_OPLOG_WINDOW_RUNNING_OUT",
  "enabled": true,
  "threshold": { "operator": "LESS_THAN", "threshold": 1, "units": "HOURS" }
}

// When a "high connections" alert fires:
// Step 1: Check Atlas → Metrics → Connections (see trend)
db.serverStatus().connections
// { current: 847, available: 120, totalCreated: 1240 }

// Step 2: Find which users/apps have the most connections
db.currentOp({ $ownOps: false }).then(r =>
  r.inprog.reduce((acc, op) => {
    acc[op.client] = (acc[op.client] || 0) + 1
    return acc
  }, {})
)

// Step 3: Check if connection pool is misconfigured
// Node.js MongoClient default maxPoolSize: 100
// If running 10 app servers × 100 connections = 1000 connections
// Fix: reduce maxPoolSize to 25 per server
const client = new MongoClient(uri, { maxPoolSize: 25 })

// Step 4: Consider Atlas Auto-scaling or upgrade cluster tier
```

---

## Scenario 2: Migrating to Atlas from Self-Managed MongoDB

**Scenario**: You have a production MongoDB 5.0 replica set on AWS EC2 with 500GB of data. You need to migrate to Atlas M50 with zero downtime (< 5 second disruption window).

```bash
# APPROACH: Live migration using Atlas Live Migration Service
# (available in Atlas UI: cluster → "..." → Migrate Data to this Cluster)

# STEP 1: Prepare source cluster
# Verify MongoDB version compatibility (Atlas supports source >= 4.2)
mongosh "mongodb://admin:pass@ec2-host:27017/admin" --eval "db.version()"
# Must be 4.2, 4.4, 5.0, 6.0, or 7.0

# STEP 2: Atlas Live Migration creates a change stream on source
# Atlas Live Migration requires:
# - Source must be a replica set (not standalone)
# - Source must have oplog with at least 1 hour window
# - Network access: Atlas IP allowlist must include source IP
# - MongoDB user with readAnyDatabase + changeStream privileges

# Create migration user on source:
db.createUser({
  user: "atlasMigration",
  pwd: "securePassword",
  roles: [
    { role: "readAnyDatabase", db: "admin" },
    { role: "clusterMonitor", db: "admin" }
  ]
})

# STEP 3: Configure Atlas Live Migration in UI
# - Provide source hostname:port
# - Provide migration user credentials
# - Atlas begins initial sync (full copy of all data)
# - While initial sync runs: Atlas also tails the oplog (captures ongoing changes)

# STEP 4: Monitor migration progress
# Atlas UI shows:
# - Documents copied: 12.3M / 15M (82%)
# - Data transferred: 410GB / 500GB
# - Lag: 2.3s (oplog lag — how far behind live writes)
# - Estimated completion: 35 minutes

# STEP 5: Cutover
# When initial sync is complete and lag is < 10 seconds:
# Option A: Atlas-guided cutover (recommended)
# 1. Stop writes to source cluster (application-level "maintenance mode")
# 2. Wait for Atlas lag to reach 0
# 3. Atlas signals "ready to cutover"
# 4. Update application connection strings to Atlas
# 5. Re-enable writes (now pointing to Atlas)
# Total downtime: typically 1-5 seconds

# STEP 6: Verify migration
mongosh "mongodb+srv://user:pass@atlas-cluster.mongodb.net/mydb" --eval "
  db.orders.countDocuments(),
  db.users.countDocuments(),
  db.serverStatus().version
"
# Verify counts match source, verify version
```

---

## Scenario 3: Atlas Triggers for Real-Time Notifications

**Scenario**: Build a real-time notification system using Atlas Triggers. When an order's status changes from "processing" to "shipped", send a notification to the user and update a notifications collection.

```javascript
// Atlas Trigger configuration:
{
  "name": "onOrderShipped",
  "type": "DATABASE",
  "config": {
    "service_id": "atlas-cluster-service",
    "database": "ecommerce",
    "collection": "orders",
    "operation_types": ["UPDATE"],
    "full_document": true,
    "full_document_before_change": true,
    "match": {
      "updateDescription.updatedFields.status": "shipped"
    }
  }
}

// Trigger function:
exports = async function(changeEvent) {
  const context = this.context

  const beforeStatus = changeEvent.fullDocumentBeforeChange?.status
  const afterStatus = changeEvent.fullDocument.status

  // Only process "processing" → "shipped" transitions
  if (beforeStatus !== "processing" || afterStatus !== "shipped") {
    return
  }

  const order = changeEvent.fullDocument
  const db = context.services.get("mongodb-atlas").db("ecommerce")

  // Step 1: Create notification document
  await db.collection("notifications").insertOne({
    recipientId: order.customerId,
    type: "order_shipped",
    title: "Your order has shipped!",
    body: `Order ${order.orderNumber} is on its way. Tracking: ${order.trackingNumber}`,
    orderId: order._id,
    isRead: false,
    createdAt: new Date(),
    expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)   // 30 days
  })

  // Step 2: Get customer details for push notification
  const customer = await db.collection("users").findOne({ _id: order.customerId })

  // Step 3: Send push notification via FCM (if customer has push token)
  if (customer?.pushToken) {
    const https = require("https")
    const payload = JSON.stringify({
      to: customer.pushToken,
      notification: {
        title: "Your order has shipped!",
        body: `Order ${order.orderNumber} is on its way`
      },
      data: { orderId: order._id.toString(), type: "order_shipped" }
    })

    await new Promise((resolve, reject) => {
      const req = https.request({
        hostname: "fcm.googleapis.com",
        path: "/fcm/send",
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `key=${context.values.get("FCM_SERVER_KEY")}`
        }
      }, res => resolve(res))
      req.on("error", reject)
      req.write(payload)
      req.end()
    })
  }

  // Step 4: Send email notification
  if (customer?.email) {
    // Using Atlas App Services HTTP service (SMTP or SendGrid)
    const emailHttp = context.services.get("emailHttp")
    await emailHttp.post({
      url: "https://api.sendgrid.com/v3/mail/send",
      headers: {
        "Authorization": [`Bearer ${context.values.get("SENDGRID_API_KEY")}`],
        "Content-Type": ["application/json"]
      },
      body: JSON.stringify({
        personalizations: [{ to: [{ email: customer.email }] }],
        from: { email: "noreply@example.com" },
        subject: "Your order has shipped!",
        content: [{ type: "text/plain", value: `Order ${order.orderNumber} is on its way!` }]
      })
    })
  }

  console.log(`Notification sent for order ${order._id}`)
}
```

---

## Scenario 4: Using Atlas Charts for Business Dashboard

**Scenario**: The marketing team needs a self-service dashboard showing:
- Daily revenue over the last 30 days (line chart)
- Top 10 products by revenue this month (bar chart)
- Revenue by category breakdown (pie chart)
- Real-time active orders count (number widget)

Show the aggregation pipelines that power each chart.

```javascript
// Chart 1: Daily revenue over last 30 days
// Collection: orders, Field: createdAt (date), amount (number)
[
  {
    $match: {
      status: "completed",
      createdAt: {
        $gte: { $dateSubtract: { startDate: "$$NOW", unit: "day", amount: 30 } }
      }
    }
  },
  {
    $group: {
      _id: { $dateTrunc: { date: "$createdAt", unit: "day" } },
      revenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { "_id": 1 } }
]
// Atlas Charts: X-axis = _id (date), Y-axis = revenue (line), secondary Y = orderCount

// Chart 2: Top 10 products by revenue this month
[
  {
    $match: {
      status: "completed",
      createdAt: { $gte: { $dateSubtract: { startDate: "$$NOW", unit: "month", amount: 1 } } }
    }
  },
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.productName",
      revenue: { $sum: { $multiply: ["$items.price", "$items.quantity"] } },
      unitsSold: { $sum: "$items.quantity" }
    }
  },
  { $sort: { revenue: -1 } },
  { $limit: 10 }
]
// Atlas Charts: horizontal bar chart, X = revenue, Y = _id (product name)

// Chart 3: Revenue by category (pie chart)
[
  { $match: { status: "completed" } },
  { $unwind: "$items" },
  {
    $lookup: {
      from: "products",
      localField: "items.productId",
      foreignField: "_id",
      as: "productInfo"
    }
  },
  { $unwind: "$productInfo" },
  {
    $group: {
      _id: "$productInfo.category",
      revenue: { $sum: { $multiply: ["$items.price", "$items.quantity"] } }
    }
  },
  { $sort: { revenue: -1 } }
]
// Atlas Charts: pie chart, category = _id, value = revenue

// Chart 4: Real-time active orders count
[
  { $match: { status: { $in: ["placed", "processing", "shipped"] } } },
  { $count: "activeOrders" }
]
// Atlas Charts: number widget, value = activeOrders
// Set auto-refresh to 30 seconds for near-real-time

// Atlas Charts configuration tips:
// - Set chart refresh rate to match data freshness needs (1 min for near-real-time)
// - Use "Calculate" series for derived metrics (e.g., average order value)
// - Add filters to dashboards: date range picker, region dropdown
// - Configure row-level security for manager vs. executive dashboards
// - Embed dashboards in your internal admin portal:
//   const sdk = ChartsEmbedSDK({ baseUrl: "https://charts.mongodb.com/charts-abc" })
//   const chart = sdk.createChart({ chartId: "chart-id-here" })
//   await chart.render(document.getElementById("chartContainer"))
```

---

## Scenario 5: Atlas Search for Product Catalog

**Scenario**: Build a product search for an e-commerce site. Users can search by text ("wireless headphones"), filter by price range, filter by category, and sort by relevance score. The search must handle typos (fuzzy matching) and return facet counts.

```javascript
// Step 1: Create Atlas Search index
{
  "name": "productSearch",
  "mappings": {
    "dynamic": false,
    "fields": {
      "name": [
        { "type": "string", "analyzer": "lucene.english" },
        { "type": "autocomplete", "tokenization": "edgeGram", "minGrams": 2, "maxGrams": 15 }
      ],
      "description": { "type": "string", "analyzer": "lucene.english" },
      "brand": { "type": "string", "analyzer": "lucene.keyword" },
      "category": { "type": "stringFacet" },       // for facet counts
      "price": { "type": "number" },
      "rating": { "type": "number" },
      "isActive": { "type": "boolean" }
    }
  }
}

// Step 2: Full-text search with facets and filters
async function searchProducts(query, filters) {
  const { text, category, minPrice, maxPrice, page = 1, pageSize = 20 } = filters

  const searchStage = {
    $search: {
      index: "productSearch",
      compound: {
        // Must match text (with fuzzy for typo tolerance)
        must: text ? [
          {
            text: {
              query: text,
              path: ["name", "description"],
              fuzzy: { maxEdits: 1, prefixLength: 3 },  // allow 1 typo, exact first 3 chars
              score: { boost: { path: "rating", undefined: 1 } }  // boost by rating
            }
          }
        ] : [],
        // Filter conditions (don't affect relevance score)
        filter: [
          { equals: { path: "isActive", value: true } },
          ...(category ? [{ equals: { path: "category", value: category } }] : []),
          ...(minPrice || maxPrice ? [{
            range: {
              path: "price",
              gte: minPrice || 0,
              lte: maxPrice || 999999
            }
          }] : [])
        ]
      }
    }
  }

  const [results, facets] = await Promise.all([
    // Main search results
    db.products.aggregate([
      searchStage,
      {
        $project: {
          name: 1, price: 1, category: 1, brand: 1, rating: 1,
          imageUrl: 1,
          score: { $meta: "searchScore" }
        }
      },
      { $sort: { score: { $meta: "searchScore" } } },
      { $skip: (page - 1) * pageSize },
      { $limit: pageSize }
    ]).toArray(),

    // Facet counts (no documents returned, just counts)
    db.products.aggregate([
      {
        $searchMeta: {
          index: "productSearch",
          compound: {
            must: text ? [{ text: { query: text, path: ["name", "description"] } }] : [],
            filter: [{ equals: { path: "isActive", value: true } }]
          },
          facet: {
            operator: { exists: { path: "category" } },
            facets: {
              categoryFacet: {
                type: "string",
                path: "category",
                numBuckets: 10
              },
              priceFacet: {
                type: "number",
                path: "price",
                boundaries: [0, 25, 50, 100, 200, 500, 1000],
                default: "other"
              }
            }
          }
        }
      }
    ]).toArray()
  ])

  return {
    results,
    facets: facets[0]?.facet || {},
    totalCount: facets[0]?.count?.lowerBound || 0
  }
}

// Autocomplete endpoint (for search-as-you-type):
db.products.aggregate([
  {
    $search: {
      index: "productSearch",
      autocomplete: {
        query: partialText,      // "wire" → suggests "wireless headphones", "wire stripper"
        path: "name",
        tokenOrder: "sequential",
        fuzzy: { maxEdits: 1 }
      }
    }
  },
  { $limit: 5 },
  { $project: { name: 1, _id: 0 } }
])
```

---

## Scenario 6: Atlas Online Archive + Data Federation for Compliance

**Scenario**: Financial services company must retain 7 years of transaction history. Active data (last 90 days) must be sub-100ms. Historical data (90 days to 7 years) needs to be queryable for compliance audits but can take seconds.

```javascript
// Step 1: Configure Online Archive
// Archive transactions older than 90 days
{
  "collName": "transactions",
  "dbName": "finance",
  "criteria": {
    "type": "date",
    "dateField": "createdAt",
    "dateFormat": "ISODATE",
    "expireAfterDays": 90
  },
  "schedule": {
    "type": "daily",
    "startTime": "01:00"   // 1 AM UTC, low traffic window
  },
  "partitionFields": [
    { "fieldName": "createdAt", "order": 0 },
    { "fieldName": "customerId", "order": 1 }
  ]
}

// Step 2: Application connects to Data Federation (not directly to cluster)
// Two connection strings maintained:
const hotClient = new MongoClient("mongodb+srv://user:pass@cluster.mongodb.net/")  // hot data
const coldClient = new MongoClient("mongodb+srv://user:pass@data.mongodb.net/")     // federation

// Step 3: Smart routing layer
async function queryTransactions(customerId, startDate, endDate) {
  const ninetyDaysAgo = new Date(Date.now() - 90 * 24 * 60 * 60 * 1000)

  const isHotOnly = startDate >= ninetyDaysAgo
  const isColdOnly = endDate < ninetyDaysAgo
  const isMixed = !isHotOnly && !isColdOnly

  if (isHotOnly) {
    // Use cluster directly (fast)
    return hotClient.db("finance").collection("transactions").find({
      customerId: ObjectId(customerId),
      createdAt: { $gte: startDate, $lte: endDate }
    }).toArray()
  } else {
    // Use Data Federation (handles both hot and cold)
    return coldClient.db("finance").collection("transactions").aggregate([
      { $match: {
          customerId: ObjectId(customerId),
          createdAt: { $gte: startDate, $lte: endDate }
      }},
      { $sort: { createdAt: -1 } }
    ]).toArray()
  }
}

// Compliance audit query (7-year lookback):
async function complianceAudit(customerId, year) {
  const start = new Date(`${year}-01-01T00:00:00Z`)
  const end = new Date(`${year}-12-31T23:59:59Z`)

  // This query goes to S3 archive via Data Federation
  // Expected latency: 5-30 seconds (cold data from object store)
  return coldClient.db("finance").collection("transactions").aggregate([
    { $match: { customerId: ObjectId(customerId), createdAt: { $gte: start, $lte: end } } },
    { $group: {
        _id: { month: { $month: "$createdAt" } },
        total: { $sum: "$amount" },
        count: { $sum: 1 }
    }},
    { $sort: { "_id.month": 1 } }
  ]).toArray()
}

// Cost optimization:
// Atlas M50 cluster: 2TB at $0.25/GB = $500/month for hot data
// Online Archive (S3): 18TB at $0.023/GB = $414/month for cold data
// vs. keeping everything in Atlas: 20TB at $0.25/GB = $5,000/month
// SAVINGS: ~90% reduction in storage costs for compliance data
```

---

## Scenario 7: Atlas Vector Search for RAG Chatbot

**Scenario**: Build a customer support chatbot that answers questions using your product documentation. The chatbot should find the most relevant documentation sections for any user question.

```javascript
// Step 1: Ingest documentation with embeddings
const { OpenAI } = require("openai")
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })

async function ingestDocumentation(docs) {
  const chunks = []

  for (const doc of docs) {
    // Split document into ~500-word chunks for better retrieval
    const paragraphs = doc.content.split("\n\n").filter(p => p.length > 50)

    for (const chunk of paragraphs) {
      // Generate embedding for each chunk
      const response = await openai.embeddings.create({
        model: "text-embedding-ada-002",
        input: chunk
      })

      chunks.push({
        documentId: doc._id,
        title: doc.title,
        section: chunk,
        embedding: response.data[0].embedding,   // 1536-dimensional vector
        category: doc.category,
        updatedAt: new Date()
      })
    }
  }

  await db.docChunks.insertMany(chunks)
  console.log(`Ingested ${chunks.length} documentation chunks`)
}

// Atlas Vector Search index:
{
  "name": "docs_vector_index",
  "type": "vectorSearch",
  "fields": [
    { "type": "vector", "path": "embedding", "numDimensions": 1536, "similarity": "cosine" },
    { "type": "filter", "path": "category" }
  ]
}

// Step 2: RAG query function
async function answerQuestion(userQuestion, category = null) {
  // 1. Embed the user's question
  const queryEmbedding = await openai.embeddings.create({
    model: "text-embedding-ada-002",
    input: userQuestion
  })

  // 2. Find most relevant documentation chunks
  const filter = category ? { category: category } : {}
  const relevantChunks = await db.docChunks.aggregate([
    {
      $vectorSearch: {
        index: "docs_vector_index",
        path: "embedding",
        queryVector: queryEmbedding.data[0].embedding,
        numCandidates: 50,
        limit: 5,
        filter: filter
      }
    },
    {
      $project: {
        title: 1,
        section: 1,
        category: 1,
        score: { $meta: "vectorSearchScore" }
      }
    }
  ]).toArray()

  // Filter by minimum relevance score
  const contextChunks = relevantChunks.filter(c => c.score > 0.75)

  if (contextChunks.length === 0) {
    return "I couldn't find relevant documentation for your question. Please contact support."
  }

  // 3. Build context for LLM
  const context = contextChunks
    .map((c, i) => `[Source ${i+1}: ${c.title}]\n${c.section}`)
    .join("\n\n")

  // 4. Generate answer with LLM
  const completion = await openai.chat.completions.create({
    model: "gpt-4o",
    messages: [
      {
        role: "system",
        content: "You are a helpful product documentation assistant. Answer questions ONLY based on the provided documentation context. If the answer is not in the context, say so."
      },
      {
        role: "user",
        content: `Documentation context:\n${context}\n\nUser question: ${userQuestion}`
      }
    ],
    temperature: 0,  // deterministic answers
    max_tokens: 500
  })

  const answer = completion.choices[0].message.content

  // 5. Log the query for analytics
  await db.chatHistory.insertOne({
    question: userQuestion,
    answer,
    sourceDocs: contextChunks.map(c => c._id),
    topScore: contextChunks[0]?.score,
    createdAt: new Date()
  })

  return {
    answer,
    sources: contextChunks.map(c => ({ title: c.title, score: c.score }))
  }
}
```

---

## Scenario 8: Multi-Region Atlas Deployment

**Scenario**: Your application serves users in the US, Europe, and Southeast Asia. Latency requirements: reads and writes < 50ms for each region. How do you architect this on Atlas?

```javascript
// OPTION 1: Multi-region replica set (simpler, less expensive)
// Primary in US-East, secondaries in EU-West and AP-Southeast

// Atlas cluster configuration:
{
  "clusterType": "REPLICASET",
  "replicationSpecs": [
    {
      "regionConfigs": [
        { "regionName": "US_EAST_1", "priority": 7,  "electableNodes": 1, "analyticsNodes": 0, "readOnlyNodes": 0 },
        { "regionName": "EU_WEST_1", "priority": 6,  "electableNodes": 1, "analyticsNodes": 0, "readOnlyNodes": 0 },
        { "regionName": "AP_SOUTHEAST_1", "priority": 5, "electableNodes": 1, "analyticsNodes": 0, "readOnlyNodes": 0 }
      ]
    }
  ]
}

// Application reads from nearest node (readPreference: nearest)
const client = new MongoClient(uri, {
  readPreference: "nearest",
  readConcern: { level: "majority" }
})
// EU user → read from EU-West secondary (~5ms)
// AP user → read from AP-Southeast secondary (~5ms)
// Writes → always go to US-East primary → high latency for EU/AP writes!

// OPTION 2: Atlas Global Clusters (true multi-region writes)
// Zone sharding: EU user data → EU shard, AP user data → AP shard

// Collection: users, sharded on { region: 1, _id: 1 }
db.adminCommand({ shardCollection: "app.users", key: { region: 1, _id: 1 } })

// Zone mappings:
db.adminCommand({ addShardToZone: "shard-us", zone: "US" })
db.adminCommand({ addShardToZone: "shard-eu", zone: "EU" })
db.adminCommand({ addShardToZone: "shard-ap", zone: "AP" })

db.adminCommand({
  updateZoneKeyRange: {
    ns: "app.users",
    min: { region: "US", _id: MinKey },
    max: { region: "US", _id: MaxKey },
    zone: "US"
  }
})
// Similar for EU and AP zones

// Application routing:
async function writeUser(user) {
  const region = detectRegion(user.ipAddress)   // "US" | "EU" | "AP"
  await db.users.insertOne({ ...user, region })
  // Mongos routes to the zone nearest the region field value
  // US users → US-East shard (5ms write latency for US users)
  // EU users → EU-West shard (5ms write latency for EU users)
}

async function readUser(userId, userRegion) {
  const user = await db.users.findOne(
    { _id: ObjectId(userId), region: userRegion },   // include shard key for targeted query
    { readPreference: new ReadPreference("nearest") }
  )
  return user
}

// GDPR benefit: EU users' data physically stays in EU region
// Configure: EU zone → EU-West-1 (Ireland or Frankfurt)
// Required by GDPR Article 44-49 for EU personal data
```
