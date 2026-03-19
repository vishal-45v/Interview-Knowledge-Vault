# Chapter 07 — Atlas Features: Theory Questions

---

## Q1: What is MongoDB Atlas and how does it differ from self-managed MongoDB?

MongoDB Atlas is MongoDB's fully managed cloud database service (DBaaS). It runs MongoDB on AWS, GCP, and Azure and handles all infrastructure operations automatically.

```javascript
// Atlas handles automatically:
// - Provisioning and hardware management
// - MongoDB version upgrades (major and minor)
// - Automated backups (continuous cloud backups + snapshots)
// - Monitoring and alerting (Performance Advisor, Atlas Charts)
// - Security configuration (VPC peering, private endpoints, encryption)
// - Scaling (vertical and horizontal)
// - Replica set management (elections, member health)
// - Sharding configuration

// Self-managed MongoDB responsibilities:
// - Install and configure MongoDB on your own VMs/bare metal
// - Manage replica sets, elections, oplog sizing
// - Configure backups and test restores
// - Monitor performance and set up alerts
// - Apply security patches and version upgrades
// - Manage TLS certificates
// - Scale up/down manually (add nodes, resize disks)

// Atlas connection string format:
"mongodb+srv://username:password@cluster0.abc123.mongodb.net/dbname?retryWrites=true&w=majority"
// +srv: uses DNS SRV record to discover all replica set members automatically
// retryWrites=true: default in Atlas (recommended)
// w=majority: default write concern for Atlas clusters

// Atlas tiers:
// M0 (Free):        512MB, shared infrastructure, limited features
// M2/M5 (Shared):   2-5GB, shared vCPU, limited features
// M10+ (Dedicated): dedicated resources, all Atlas features available
// M30+ (Production): recommended minimum for production workloads
// M200+: high-memory for large working sets
```

---

## Q2: What is Atlas App Services? What are Atlas Triggers?

**Atlas App Services** (formerly Realm) is a serverless backend platform integrated with Atlas. It provides:
- **Atlas Triggers**: serverless functions that react to database changes or schedules
- **Atlas Functions**: Node.js serverless functions for custom backend logic
- **Atlas GraphQL API**: auto-generated GraphQL API from your schema
- **Atlas Device Sync**: real-time sync between Atlas and mobile/edge devices (Realm SDK)
- **Atlas Data API**: REST API for accessing Atlas data without a MongoDB driver

```javascript
// Atlas Database Trigger — fires on collection changes
// Configured in Atlas UI or via Atlas Admin API

// Trigger configuration (JSON):
{
  "name": "onOrderPlaced",
  "type": "DATABASE",
  "config": {
    "service_id": "atlas-cluster-service",
    "database": "ecommerce",
    "collection": "orders",
    "operation_types": ["INSERT"],
    "full_document": true
  }
}

// Trigger function (Node.js, runs in Atlas App Services):
exports = async function(changeEvent) {
  const order = changeEvent.fullDocument

  // Send confirmation email via SendGrid
  const context = this.context
  const emailService = context.services.get("sendgrid")

  await emailService.send({
    to: order.customerEmail,
    subject: `Order ${order.orderNumber} Confirmed`,
    html: `<p>Your order for ${order.items.length} items has been placed.</p>`
  })

  // Update analytics counter (Computed Pattern)
  const atlas = context.services.get("mongodb-atlas")
  const db = atlas.db("ecommerce")

  await db.collection("analytics").updateOne(
    { date: new Date().toISOString().split("T")[0] },
    { $inc: { ordersPlaced: 1, revenue: order.total } },
    { upsert: true }
  )
}

// Scheduled trigger — runs on a cron schedule
{
  "name": "dailyCleanup",
  "type": "SCHEDULED",
  "config": {
    "schedule": "0 2 * * *"   // 2 AM daily (cron syntax)
  }
}
exports = async function() {
  const atlas = context.services.get("mongodb-atlas")
  const db = atlas.db("sessions")
  const cutoff = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000)
  const result = await db.collection("sessions").deleteMany({ createdAt: { $lt: cutoff } })
  console.log(`Deleted ${result.deletedCount} expired sessions`)
}
```

---

## Q3: What is Atlas Data Federation? How does it work?

Atlas Data Federation allows you to query data across multiple sources — Atlas clusters, Atlas Data Lake (S3), HTTP endpoints, and Atlas Online Archive — using standard MongoDB aggregation syntax from a single connection.

```javascript
// Use case: query both hot data (Atlas cluster) and cold data (S3 archive) together

// Data Federation federated endpoint:
"mongodb+srv://user:pass@data.mongodb.net/?authSource=%24external&authMechanism=MONGODB-AWS"

// Storage configuration (defined in Atlas UI):
// Sources:
//   - Atlas cluster: myCluster.ecommerce.orders (hot data, last 90 days)
//   - S3 bucket: s3://my-archive-bucket/orders/  (cold data, 90+ days old)
//     → each file: orders-2023-01.json.gz, orders-2023-02.json.gz, etc.

// Query seamlessly across both:
db.orders.aggregate([
  {
    $match: {
      createdAt: {
        $gte: ISODate("2023-01-01T00:00:00Z"),   // includes cold data
        $lt: ISODate("2024-01-01T00:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: { year: { $year: "$createdAt" }, month: { $month: "$createdAt" } },
      total: { $sum: "$amount" },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])
// Federation query planner routes recent data to Atlas, old data to S3

// Atlas Online Archive (automatic tiering):
// Automatically moves documents matching a date/time criteria to S3
// Accessible via Data Federation without any application changes
db.adminCommand({
  "archiving": {
    "collectionName": "orders",
    "dbName": "ecommerce",
    "criteria": {
      "type": "date",
      "dateField": "createdAt",
      "expireAfterDays": 90    // archive docs older than 90 days to S3
    }
  }
})
```

---

## Q4: What is Atlas Search? How does it differ from MongoDB's text indexes?

**Atlas Search** is a full-text search service built on Apache Lucene, fully integrated with Atlas. It provides advanced search capabilities not available with MongoDB's native text indexes.

```javascript
// Native text index — simple full-text search
db.articles.createIndex({ body: "text", title: "text" })
db.articles.find({ $text: { $search: "mongodb performance" } })
// Limitations:
// - One text index per collection
// - No relevance scoring control
// - No fuzzy matching, autocomplete, facets, highlighting
// - No geo + text combined queries
// - Cannot index nested fields in arrays separately

// Atlas Search — Lucene-powered, far more capable
// 1. Create search index (via Atlas UI or API):
{
  "name": "default",
  "mappings": {
    "dynamic": false,
    "fields": {
      "title":   { "type": "string", "analyzer": "lucene.english" },
      "body":    { "type": "string", "analyzer": "lucene.english" },
      "tags":    { "type": "string", "analyzer": "lucene.keyword" },
      "price":   { "type": "number" },
      "category":{ "type": "stringFacet" },
      "location":{ "type": "geo" }
    }
  }
}

// 2. Query with $search aggregation stage:
db.articles.aggregate([
  {
    $search: {
      index: "default",
      compound: {
        must: [
          { text: { query: "mongodb performance", path: ["title", "body"], fuzzy: { maxEdits: 1 } } }
        ],
        filter: [
          { equals: { path: "status", value: "published" } }
        ]
      },
      highlight: { path: ["title", "body"] },
      returnStoredSource: true
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" },
      highlights: { $meta: "searchHighlights" }
    }
  },
  { $sort: { score: { $meta: "searchScore" } } },
  { $limit: 10 }
])

// Atlas Search features NOT in text indexes:
// - Fuzzy search (typo tolerance)
// - Autocomplete (partial word completion)
// - Faceted search with $searchMeta
// - Phrase matching
// - Boosting (weight title higher than body)
// - Wildcard and regex patterns
// - Geo search combined with text
// - Highlighting (show matching terms in context)
// - Custom analyzers (tokenizers, stemming, synonyms)
// - Near (number proximity) and range filters within $search
```

---

## Q5: What is Atlas Device Sync (formerly Realm Sync)?

Atlas Device Sync enables real-time, bidirectional data synchronization between MongoDB Atlas and mobile/edge devices using the Realm SDK.

```javascript
// Use case: mobile app that works offline and syncs when connected

// Atlas App Services configuration:
// 1. Enable Sync on a collection
// 2. Define a schema for the synced objects
// 3. Configure permissions (who can read/write what)

// Mobile client (React Native / iOS / Android):
import Realm from "realm"

const app = new Realm.App({ id: "my-atlas-app-id" })
const credentials = Realm.Credentials.emailPassword("user@example.com", "password")
await app.logIn(credentials)

// Open a synced realm
const realm = await Realm.open({
  schema: [TaskSchema],
  sync: {
    user: app.currentUser,
    flexible: true,
    initialSubscriptions: {
      update: (subs, realm) => {
        // Subscribe to tasks belonging to this user
        subs.add(realm.objects("Task").filtered("ownerId == $0", app.currentUser.id))
      }
    }
  }
})

// Read/write — automatically syncs with Atlas
const tasks = realm.objects("Task")    // live query, auto-updates on changes

realm.write(() => {
  realm.create("Task", {
    _id: new Realm.BSON.ObjectId(),
    title: "Review PR #42",
    completed: false,
    ownerId: app.currentUser.id
  })
})
// This write:
// 1. Persists locally immediately (works offline)
// 2. Syncs to Atlas when connection is available
// 3. Other devices subscribed to this data receive the update in real-time

// Conflict resolution:
// Atlas Device Sync uses a deterministic conflict resolution algorithm
// Last-write-wins based on logical timestamp
// For complex conflicts: define custom conflict resolver

// Atlas side: synced data appears in your Atlas collection as normal MongoDB documents
```

---

## Q6: How does Atlas Backup work? What are the key backup concepts?

Atlas provides two backup mechanisms:

```javascript
// 1. CLOUD BACKUP (recommended for M10+)
// - Continuous backups: every 6 hours (configurable)
// - On-demand snapshots: manually triggered
// - Stored in cloud provider (AWS S3, GCP GCS, Azure Blob Storage)
// - Encrypted at rest (AES-256)
// - Retention: configurable (4 days to 12 months)
// - Point-in-time restore: restore to any point within the retention window

// Atlas backup policy configuration (via API or UI):
{
  "policies": [
    {
      "frequencyType": "hourly",
      "frequencyInterval": 6,        // every 6 hours
      "retentionValue": 7,
      "retentionUnit": "days"
    },
    {
      "frequencyType": "daily",
      "frequencyInterval": 1,        // once daily
      "retentionValue": 30,
      "retentionUnit": "days"
    },
    {
      "frequencyType": "weekly",
      "frequencyInterval": 1,        // once weekly (Monday)
      "retentionValue": 3,
      "retentionUnit": "months"
    },
    {
      "frequencyType": "monthly",
      "frequencyInterval": 1,        // once monthly (1st of month)
      "retentionValue": 12,
      "retentionUnit": "months"
    }
  ]
}

// 2. LEGACY BACKUP (pre-Atlas Cloud Backup — being deprecated)
// - Uses Atlas's own backup infrastructure (not cloud-provider native)
// - Less performant, higher cost

// Key capabilities:
// Point-in-time restore (PITR):
// - Restore your cluster to any second within the retention window
// - Uses continuous oplog backup in addition to snapshots
// - Atlas replays oplog from nearest snapshot to exact target time
// - Example: "restore to 2024-03-01T14:23:45Z" — exact second

// Snapshot restore options:
// 1. Restore to existing cluster (overwrites — destructive!)
// 2. Create a new Atlas cluster from snapshot (non-destructive)
// 3. Download snapshot as .tar.gz (mongodump format) for local restore

// Backup for sharded clusters:
// Atlas takes coordinated snapshots across all shards simultaneously
// Ensures snapshot consistency across shards (required for cross-shard transaction safety)

// Backup monitoring:
db.adminCommand({ listAtlasBackup: 1 })   // not a real command — done via Atlas API
// GET https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/clusters/{clusterName}/backup/snapshots
```

---

## Q7: What is the Atlas Performance Advisor?

The **Performance Advisor** is an Atlas UI tool that automatically analyzes your query performance and recommends new indexes.

```javascript
// How it works:
// 1. Atlas continuously samples the MongoDB profiler (system.profile collection)
// 2. Identifies slow operations (default threshold: 100ms)
// 3. Groups similar queries (by query shape — structure, not values)
// 4. Recommends compound indexes based on the ESR rule
// 5. Shows impact: estimated improvement + which queries will benefit

// What Performance Advisor analyzes:
// - COLLSCAN operations (most critical — missing index)
// - Queries with in-memory sorts (SORT stage after FETCH)
// - Low index selectivity (many docs examined for few returned)
// - Unnecessary scanned keys

// Sample Performance Advisor output:
// Recommended Index: { status: 1, createdAt: -1, customerId: 1 }
// Impact: 47 queries in the last 24 hours would benefit
// Avg improvement: 94% reduction in docs examined
// Before: avgDocsExamined: 5,000,000 — After: avgDocsExamined: 50
// Affected queries:
//   - db.orders.find({ status: "pending" }).sort({ createdAt: -1 })
//   - db.orders.find({ status: "active", customerId: id })

// Creating the recommended index from Performance Advisor:
// Click "Create Index" in the UI → builds online without blocking reads/writes

// Configuration:
// Slow operation threshold: configurable (default 100ms)
// Profiling level: Atlas sets 1 (log slow ops) by default
// Namespace filter: can focus on specific collections

// Code equivalent (what Performance Advisor analyzes):
db.setProfilingLevel(1, { slowms: 100 })   // enable profiling for ops > 100ms
db.system.profile.find({
  op: "query",
  millis: { $gt: 100 },
  "planSummary": /COLLSCAN/   // find collection scans
}).sort({ millis: -1 }).limit(20)
```

---

## Q8: What is Atlas Charts?

Atlas Charts is a native data visualization tool built into Atlas. It connects directly to your Atlas cluster data and lets you build dashboards without exporting data.

```javascript
// Atlas Charts creates:
// - Bar charts, line charts, pie charts, scatter plots
// - Geo maps (for GeoJSON fields)
// - Heatmaps
// - Number widgets (single metric display)
// - Data tables
// - Word clouds

// Key features:
// 1. No data export needed — reads directly from Atlas collections
// 2. Aggregation pipeline editor — write custom pipelines for complex visualizations
// 3. Scheduled auto-refresh (real-time dashboards)
// 4. Embedding (embed charts in external apps with auth)
// 5. Row-level security (filter data per viewer)
// 6. Dashboard sharing (public or authenticated viewers)

// Example: pipeline powering a "Daily Revenue" line chart
[
  { $match: { status: "completed", createdAt: { $gte: { $dateSubtract: { startDate: "$$NOW", unit: "month", amount: 3 } } } } },
  { $group: {
      _id: { $dateTrunc: { date: "$createdAt", unit: "day" } },
      revenue: { $sum: "$total" },
      orderCount: { $sum: 1 }
  }},
  { $sort: { _id: 1 } }
]

// Embedding charts in your app:
// Atlas Charts generates an embedding URL with optional token-based auth
// <div id="chart-div" style="width:700px; height:400px"></div>
// <script src="https://charts.mongodb.com/charts-embedding.js"></script>
// <script>
//   const sdk = ChartsEmbedSDK({ baseUrl: "https://charts.mongodb.com/charts-..." })
//   const chart = sdk.createChart({ chartId: "...", height: "400px" })
//   chart.render(document.getElementById("chart-div"))
// </script>
```

---

## Q9: What is Atlas Database Auditing?

Atlas Database Auditing records all authenticated operations against your Atlas cluster, enabling compliance, security monitoring, and forensic analysis.

```javascript
// What gets audited:
// - Authentication attempts (success and failure)
// - CRUD operations (configurable — often too verbose for production)
// - Admin operations (createUser, createIndex, dropCollection, etc.)
// - Schema validation failures
// - Authorization failures

// Atlas audit log format (JSON):
{
  "atype": "find",           // operation type
  "ts": { "$date": "2024-03-01T10:00:00.000Z" },
  "local": { "ip": "10.0.1.5", "port": 27017 },
  "remote": { "ip": "10.0.1.100", "port": 54321 },
  "users": [{ "user": "appuser", "db": "admin" }],
  "roles": [{ "role": "readWrite", "db": "myapp" }],
  "param": {
    "ns": "myapp.orders",
    "command": { "find": "orders", "filter": { "status": "pending" } }
  },
  "result": 0                // 0 = success
}

// Atlas audit filter (customize what gets logged):
// Log only authentication events + admin operations:
{
  "atype": { "$in": ["authenticate", "logout", "createUser", "dropUser",
                      "createCollection", "dropCollection", "createIndex",
                      "dropIndex", "createDatabase", "dropDatabase"] }
}

// Compliance frameworks that require auditing:
// SOC 2, HIPAA, PCI-DSS, GDPR (access to personal data must be logged)

// Atlas integrations for audit logs:
// - Amazon S3 (store audit logs for long-term retention)
// - Splunk, Datadog (SIEM integration)
// - Atlas UI: view recent audit events (24-hour window in UI)

// M10+ required for auditing
// Performance impact: negligible for admin event auditing, higher for full CRUD auditing
```

---

## Q10: What is Atlas Global Clusters?

Atlas Global Clusters is a multi-region sharded cluster that places data close to users geographically, reducing latency for globally distributed applications.

```javascript
// Architecture:
// - Shards distributed across multiple geographic regions (e.g., US-East, EU-West, AP-Southeast)
// - Zone sharding: documents are routed to the shard zone closest to their associated location
// - Each zone is a replica set in the specified region
// - Reads and writes can be served from the zone nearest the user

// Zone-based sharding example (for a global SaaS):
// Collection: users, sharded on { location: 1, _id: 1 }
// Zone mapping:
//   "US" zone  → US-East-1 shard  → handles US users
//   "EU" zone  → EU-West-1 shard  → handles EU users
//   "APAC" zone → AP-Southeast-1 shard → handles Asia-Pacific users

db.adminCommand({
  addShardToZone: "shard0001",
  zone: "US"
})

db.adminCommand({
  updateZoneKeyRange: {
    ns: "myapp.users",
    min: { location: "US", _id: MinKey },
    max: { location: "US", _id: MaxKey },
    zone: "US"
  }
})

// Application routing: write to the zone nearest the user
// Client library automatically routes based on configured read/write preference

// Benefits:
// - Low latency reads/writes for users in their region (~5-20ms vs 200ms cross-ocean)
// - Data residency compliance (EU user data stays in EU → GDPR compliance)
// - Disaster recovery (one region down → other regions still serve their users)

// Global Cluster limitations:
// - Higher cost (dedicated resources per zone)
// - Schema design constraint: shard key must include the location field
// - Cross-region transactions have higher latency (cross-shard = cross-region)
// - Not all operations can be zone-aware (global aggregations still touch all zones)

// M30+ required for Global Clusters
```

---

## Q11: What is the Atlas Data API?

The Atlas Data API provides a REST-like HTTP API for accessing and modifying Atlas data without requiring a MongoDB driver or network connection from driver-compatible languages.

```bash
# Base URL: https://data.mongodb-api.com/app/{app-id}/endpoint/data/v1

# Find one document
curl -X POST \
  https://data.mongodb-api.com/app/myapp-abc123/endpoint/data/v1/action/findOne \
  -H "Content-Type: application/json" \
  -H "api-key: YOUR-API-KEY" \
  -d '{
    "dataSource": "Cluster0",
    "database": "ecommerce",
    "collection": "orders",
    "filter": { "_id": { "$oid": "6398b56f..." } }
  }'

# Insert one document
curl -X POST \
  https://data.mongodb-api.com/app/myapp-abc123/endpoint/data/v1/action/insertOne \
  -H "Content-Type: application/json" \
  -H "api-key: YOUR-API-KEY" \
  -d '{
    "dataSource": "Cluster0",
    "database": "ecommerce",
    "collection": "products",
    "document": { "name": "Blue Widget", "price": 29.99 }
  }'

# Run an aggregation pipeline
curl -X POST \
  https://data.mongodb-api.com/app/myapp-abc123/endpoint/data/v1/action/aggregate \
  -H "Content-Type: application/json" \
  -H "api-key: YOUR-API-KEY" \
  -d '{
    "dataSource": "Cluster0",
    "database": "ecommerce",
    "collection": "orders",
    "pipeline": [
      { "$match": { "status": "completed" } },
      { "$group": { "_id": "$customerId", "total": { "$sum": "$amount" } } }
    ]
  }'
```

```javascript
// When to use Data API vs MongoDB driver:
// Data API: serverless functions, edge computing (Cloudflare Workers, Vercel Edge),
//           environments without MongoDB driver support, quick prototypes
// MongoDB Driver: production applications, complex operations,
//                 transaction support, aggregation pipelines, better performance

// Limitations of Data API:
// - Slower than native driver (HTTP overhead vs TCP + binary protocol)
// - No transaction support
// - Limited response size (16MB document limit still applies)
// - Rate limiting (varies by tier)
// - Authentication via API key (simpler but less flexible than driver auth)
```

---

## Q12: What is Atlas Vector Search?

Atlas Vector Search (MongoDB 6.0.11+, GA in Atlas) enables semantic similarity search using vector embeddings stored directly in MongoDB documents alongside your operational data.

```javascript
// Create a vector search index (Atlas UI or API):
{
  "name": "vector_index",
  "type": "vectorSearch",
  "fields": [
    {
      "type": "vector",
      "path": "embedding",
      "numDimensions": 1536,          // OpenAI text-embedding-ada-002 dimensions
      "similarity": "cosine"          // cosine | dotProduct | euclidean
    },
    {
      "type": "filter",
      "path": "category"              // for pre-filtering before vector search
    }
  ]
}

// Store documents with embeddings generated by AI model (e.g., OpenAI):
const { OpenAI } = require("openai")
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY })

const text = "MongoDB Atlas is a fully managed cloud database service."
const response = await openai.embeddings.create({
  model: "text-embedding-ada-002",
  input: text
})
const embedding = response.data[0].embedding   // [0.0023, -0.0158, ..., 0.0041] (1536 floats)

await db.articles.insertOne({
  title: "MongoDB Atlas Overview",
  body: text,
  category: "database",
  embedding: embedding                           // stored as an array of numbers
})

// Semantic similarity search:
const queryText = "cloud database hosting"
const queryEmbedding = await getEmbedding(queryText)

db.articles.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryEmbedding,
      numCandidates: 100,            // scan top 100 candidates (higher = more accurate, slower)
      limit: 5,                       // return top 5 most similar
      filter: { category: "database" }  // optional pre-filter
    }
  },
  {
    $project: {
      title: 1,
      body: 1,
      score: { $meta: "vectorSearchScore" }
    }
  }
])

// RAG (Retrieval Augmented Generation) pattern:
// 1. User asks a question
// 2. Embed the question → query vector
// 3. $vectorSearch finds most relevant documents
// 4. Pass relevant documents + question to LLM (GPT-4, Claude, etc.)
// 5. LLM generates answer grounded in your data
```

---

## Q13: What is Atlas Online Archive?

Atlas Online Archive automatically tiers "cold" data (old documents) from your Atlas cluster to a managed cloud object store (S3-compatible), making it queryable via Atlas Data Federation at a much lower cost.

```javascript
// Configure Online Archive (via Atlas UI or API):
{
  "collName": "orders",
  "dbName": "ecommerce",
  "criteria": {
    "type": "date",
    "dateField": "createdAt",
    "dateFormat": "ISODATE",
    "expireAfterDays": 90      // archive docs where createdAt > 90 days ago
  },
  "schedule": {
    "type": "daily",
    "startTime": "02:00"       // run archiving at 2 AM
  }
}

// Or use custom query criteria (field-based archiving):
{
  "criteria": {
    "type": "custom",
    "query": { "status": "archived", "closedAt": { "$lt": "2023-01-01T00:00:00Z" } }
  }
}

// What happens:
// 1. Atlas archiving job runs on schedule
// 2. Scans for documents matching criteria
// 3. Copies matching documents to cloud object store (compressed Parquet format)
// 4. Deletes originals from Atlas cluster
// 5. Archived data becomes queryable via Data Federation endpoint

// Cost comparison:
// Atlas M30 storage: ~$0.25/GB/month
// Atlas Online Archive storage: ~$0.023/GB/month (S3 pricing)
// → 10x cheaper for cold data!

// Querying archived + live data together:
// Applications connect to Data Federation endpoint (not the cluster directly)
// Data Federation automatically routes queries to the right source
// Archived data queries (no index on S3): slower than Atlas queries
// Best for: compliance data, historical reports, audit logs

// Limitations:
// - Read-only (cannot write to archived documents)
// - No indexes on archived data (full scan per query on S3)
// - Query performance slower than Atlas (cold storage access)
// - 90-day minimum archive age (configurable)
```

---

## Q14: What is Atlas Serverless? How does it differ from Atlas shared/dedicated clusters?

Atlas Serverless is a deployment mode that automatically scales based on workload and charges per operation rather than per hour.

```javascript
// DEPLOYMENT COMPARISON:
// ─────────────────────────────────────────────────────────────────────
// Shared (M0/M2/M5):
//   - Fixed resources (shared infrastructure)
//   - Cannot scale beyond tier limits
//   - No VPC peering, no private endpoints
//   - Best for: development, learning, tiny apps
//   - Price: $0-$25/month

// Dedicated (M10+):
//   - Dedicated resources (no noisy neighbors)
//   - Manual/Auto-scaling available
//   - Full Atlas feature set
//   - Best for: production workloads, predictable traffic
//   - Price: $0.08+/hour per cluster

// Serverless:
//   - No pre-provisioned cluster
//   - Scales to zero (no cost when not in use)
//   - Scales up automatically for bursts
//   - Pay per operation (read unit / write unit)
//   - Best for: unpredictable traffic, development, sporadic workloads
//   - Price: $0.10 per million reads, $1.00 per million writes (approximate)

// Serverless limitations (vs dedicated):
// - No $lookup across different clusters
// - No Change Streams (Atlas Triggers still work via Atlas App Services)
// - No Atlas Search (as of 2024)
// - No transactions across collections (transactions still work within single collection)
// - No Analytics Nodes
// - No dedicated network peering

// Connection string — same format as dedicated:
"mongodb+srv://user:pass@serverless-cluster.abc123.mongodb.net/?retryWrites=true&w=majority"

// Best for:
// - Startups with unpredictable growth
// - Dev/staging environments (pay only when used)
// - Apps with very spiky traffic (normal: near zero, peaks: massive)
// - Side projects and prototypes
```

---

## Q15: How do you configure Atlas network security?

Atlas provides multiple layers of network security:

```javascript
// LAYER 1: IP Access List (allowlist)
// Only connections from listed IP addresses/CIDR blocks are accepted
// Configure in Atlas UI: Security → Network Access

// Allow a specific IP:
// 203.0.113.42/32 → your office NAT IP

// Allow a CIDR range:
// 10.0.0.0/8 → entire private network

// Allow from anywhere (development only — dangerous in production):
// 0.0.0.0/0 → NOT recommended for production

// LAYER 2: VPC Peering
// Connect Atlas cluster directly to your AWS VPC / GCP VPC / Azure VNet
// Traffic never leaves cloud provider's network

// Benefits:
// - No public internet exposure
// - Lower latency (private routing)
// - Can remove the public IP allowlist entirely

// Atlas VPC Peering setup (AWS):
// 1. Create peering connection in Atlas (provide your VPC ID and region)
// 2. Accept the peering connection request in AWS Console
// 3. Update AWS route tables to route Atlas CIDR to peering connection
// 4. Update Atlas IP access list to allow your VPC CIDR

// LAYER 3: Private Endpoints (PrivateLink / Private Service Connect)
// Most secure option: traffic goes through cloud provider's private endpoint
// Atlas appears as if it's inside your VPC

// AWS PrivateLink:
// 1. Create Atlas PrivateLink endpoint → Atlas gives you a service name
// 2. Create an AWS VPC Endpoint using that service name
// 3. Atlas gives you a private connection string with your VPC Endpoint ID

// Private endpoint connection string:
"mongodb+srv://user:pass@cluster0-pl-0.abc123.mongodb.net/"
// → resolves to private endpoint IP, not public internet

// LAYER 4: Database Access Controls
// Users with specific roles and network restrictions
db.createUser({
  user: "appReadWrite",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "myapp" }
  ],
  authenticationRestrictions: [
    { serverAddress: ["10.0.1.0/24"] }   // can only auth from this CIDR
  ]
})

// LAYER 5: TLS encryption (mandatory on Atlas — cannot disable)
// All connections use TLS 1.2 minimum
// Atlas manages the TLS certificates automatically
```
