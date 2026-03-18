# Chapter 07 — Atlas Features: Structured Answers

---

## Answer 1: What Is MongoDB Atlas and When Would You Choose It Over Self-Managed MongoDB?

**Point**: Atlas is MongoDB's fully managed DBaaS that eliminates all infrastructure operations. The choice comes down to operational maturity, team size, and total cost of ownership.

**Reason**: Running MongoDB yourself requires dedicated DBAs or engineers to handle upgrades, backups, security patches, scaling operations, and monitoring. Atlas automates all of these, letting teams focus on application development.

**Example**:

```javascript
// Atlas handles automatically:
// - Automated backups with point-in-time restore
// - Zero-downtime version upgrades (rolling upgrades)
// - Auto-scaling (storage, compute)
// - Performance Advisor (index recommendations)
// - Security hardening (TLS mandatory, encryption at rest)
// - Multi-region replication and Global Clusters
// - Monitoring and alerting

// When to choose Atlas:
// ✓ Startup without dedicated DBA: Atlas eliminates ops overhead
// ✓ Compliance requirements (HIPAA, SOC 2): Atlas has certifications
// ✓ Variable workloads: auto-scaling handles traffic spikes
// ✓ Multi-region requirements: Global Clusters simplify geo-distribution
// ✓ Atlas-specific features needed: Search, Vector Search, Charts, Device Sync

// When to consider self-managed:
// ✓ Strict data residency in on-premises hardware (government, banking regulation)
// ✓ Air-gapped environments (no internet connectivity allowed)
// ✓ Extreme cost optimization at massive scale (M600 = $8.30/hr = $6,000/month)
// ✓ Highly customized MongoDB configuration not supported by Atlas

// Cost comparison (rough):
// Self-managed M5 equivalent: EC2 i3.xlarge (~$0.25/hr) × 3 nodes = $540/month
//   + DBA time (10 hrs/month × $150/hr) = $1,500/month DBA cost
//   + Backup infrastructure = $200/month
//   Total: ~$2,240/month
// Atlas M50 (~M5 equivalent): ~$400/month (includes all ops)
// Atlas is cheaper for small-medium teams when DBA time is included
```

---

## Answer 2: Describe Atlas Triggers and Give a Production Use Case

**Point**: Atlas Triggers are serverless functions that execute in response to database changes (database triggers), on a schedule (scheduled triggers), or on authentication events (authentication triggers). They eliminate the need for a polling loop or a separate event processing service.

**Example**:

```javascript
// Production use case: audit log generation
// Every write to the orders collection creates an immutable audit record

// Trigger configuration:
{
  "name": "orderAuditLog",
  "type": "DATABASE",
  "config": {
    "database": "ecommerce",
    "collection": "orders",
    "operation_types": ["INSERT", "UPDATE", "DELETE"],
    "full_document": true,
    "full_document_before_change": true
  }
}

exports = async function(changeEvent) {
  const db = context.services.get("mongodb-atlas").db("audit")

  await db.collection("orderAuditLog").insertOne({
    orderId: changeEvent.documentKey._id,
    operation: changeEvent.operationType,
    before: changeEvent.fullDocumentBeforeChange,
    after: changeEvent.fullDocument,
    changedFields: changeEvent.updateDescription?.updatedFields,
    timestamp: new Date(),
    userId: context.user?.id
  })
}

// Why Atlas Triggers vs. application-level change handling:
// 1. Separation of concerns — audit logic doesn't live in every service
// 2. Works even for changes made directly in Atlas UI or mongosh
// 3. No single point of failure in application code
// 4. Scales independently from the application
// 5. Change stream resume tokens handled automatically by Atlas

// Limitations to know:
// - 300-second max execution time
// - Not exactly-once (must build idempotency)
// - Not available on M0/M2/M5 or Serverless (for database triggers)
```

---

## Answer 3: Explain Atlas Search vs Native Text Indexes

**Point**: Atlas Search (Apache Lucene) provides enterprise-grade full-text search with relevance scoring, fuzzy matching, autocomplete, and facets. Native $text indexes are simpler but lack these capabilities.

**Example**:

```javascript
// Native text index — when it's sufficient:
db.articles.createIndex({ title: "text", body: "text" })
db.articles.find({ $text: { $search: "mongodb performance" } })
// OK for: exact keyword matching, simple search features
// Not OK for: typo tolerance, autocomplete, facets, relevance scoring

// Atlas Search — when to upgrade:
db.articles.aggregate([
  {
    $search: {
      index: "default",
      compound: {
        must: [{
          text: {
            query: "mongod performen",  // "mongod" = abbreviation, "performen" = typo
            path: ["title", "body"],
            fuzzy: { maxEdits: 1 }      // handles 1-char typos
          }
        }],
        should: [{
          text: {
            query: "mongod performen",
            path: "title",
            score: { boost: { value: 3 } }  // title matches score 3x higher
          }
        }]
      }
    }
  },
  {
    $project: {
      title: 1,
      score: { $meta: "searchScore" },
      highlights: { $meta: "searchHighlights" }
    }
  }
])

// Decision matrix:
// Feature              | $text | Atlas Search
// Fuzzy/typo tolerance | NO    | YES ($fuzzy)
// Autocomplete         | NO    | YES ($autocomplete)
// Facets               | NO    | YES ($searchMeta)
// Relevance scoring    | Basic | Advanced (BM25 algorithm)
// Phrase matching      | YES   | YES
// Multiple collections | NO    | YES (cross-collection index)
// Geo + text combined  | NO    | YES
// Available on M0      | YES   | NO
// Extra cost           | NO    | YES (dedicated Lucene nodes)
```

---

## Answer 4: How Does Atlas Online Archive Work and When Should You Use It?

**Point**: Atlas Online Archive automatically moves "cold" documents meeting a date/field criteria from your Atlas cluster to cloud object storage (S3), while keeping them queryable via Atlas Data Federation at ~10x lower storage cost.

**Example**:

```javascript
// Setup: archive orders older than 90 days
{
  "criteria": {
    "type": "date",
    "dateField": "createdAt",
    "expireAfterDays": 90
  },
  "partitionFields": [
    { "fieldName": "createdAt", "order": 0 },
    { "fieldName": "customerId", "order": 1 }
  ]
}

// Application uses Data Federation endpoint for historical queries:
const fedClient = new MongoClient("mongodb+srv://user:pass@data.mongodb.net/")

// This query transparently spans hot (cluster) + cold (S3) data:
const yearlyRevenue = await fedClient.db("ecommerce")
  .collection("orders")
  .aggregate([
    { $match: { createdAt: { $gte: startOf2022, $lt: startOf2024 } } },  // spans archive
    { $group: {
        _id: { $year: "$createdAt" },
        revenue: { $sum: "$amount" }
    }}
  ]).toArray()

// When to use Online Archive:
// ✓ Time-series or append-mostly data (orders, logs, events)
// ✓ Clear "hot vs cold" access boundary (> 90 days = rarely accessed)
// ✓ Regulatory requirement to retain data (7 years) at low cost
// ✓ Working set is growing and pushing Atlas cluster above comfortable memory ratio

// When NOT to use Online Archive:
// ✗ Data is randomly accessed regardless of age
// ✗ All queries need sub-100ms response (archived data: 5-30 seconds)
// ✗ Data needs updates after archiving (archive is read-only)

// Cost math example:
// 2TB hot + 18TB cold:
// Without archive: 20TB × $0.25 = $5,000/month
// With archive: 2TB × $0.25 + 18TB × $0.023 = $500 + $414 = $914/month
// Savings: $4,086/month (82% reduction!)
```

---

## Answer 5: Explain Atlas Vector Search and the RAG Pattern

**Point**: Atlas Vector Search indexes high-dimensional vector embeddings (generated by AI models) and finds documents by semantic similarity rather than keyword matching. The RAG (Retrieval Augmented Generation) pattern uses it to ground LLM responses in your specific data.

**Example**:

```javascript
// Ingestion pipeline:
// 1. Document text → AI embedding model → 1536-dimensional vector
// 2. Store vector alongside document in MongoDB

const embedding = await openai.embeddings.create({
  model: "text-embedding-ada-002",
  input: documentText
}).then(r => r.data[0].embedding)

await db.docs.insertOne({ title, body: documentText, embedding })

// Create vector search index (Atlas UI):
// { "type": "vectorSearch", "fields": [{ "type": "vector", "path": "embedding",
//   "numDimensions": 1536, "similarity": "cosine" }] }

// Query: semantic search (not keyword)
const queryEmbedding = await getEmbedding("How do I configure MongoDB indexes?")

const results = await db.docs.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",
      queryVector: queryEmbedding,
      numCandidates: 100,
      limit: 5
    }
  },
  { $project: { title: 1, body: 1, score: { $meta: "vectorSearchScore" }, embedding: 0 } }
]).toArray()

// RAG flow:
// User: "My MongoDB queries are slow. What should I do?"
// 1. Embed question → queryVector
// 2. $vectorSearch finds top 5 most semantically similar docs
//    → Finds: "ESR Rule for Compound Indexes", "Covered Query Optimization", ...
// 3. Pass documents + question to GPT-4
// 4. GPT-4 generates: "Based on the documentation:
//    1. Check explain() for COLLSCAN → add indexes...
//    2. Use ESR rule for compound indexes..."
// 5. Response is accurate to YOUR documentation (not hallucinated)

// Key advantages over keyword search for RAG:
// "how to make queries faster" semantically matches "query performance optimization"
// even though they share no common keywords
```

---

## Answer 6: What Are Atlas M0 vs M10 vs Serverless Tradeoffs?

**Point**: Atlas tier selection is about cost vs. capability vs. predictability. Each tier has a different pricing model and feature set.

**Example**:

```javascript
// DECISION GUIDE:

// M0 (Free): development and learning ONLY
// Shared vCPU, 512MB, 100 connections, NO Atlas Search, NO VPC peering
// Cost: $0/month
// NOT for: any production workload

// M2/M5 (Shared, $9-$25/month): small non-critical apps
// Still shared infrastructure, 2-5GB, limited connections
// Cost: $9-$25/month

// Serverless: pay-per-use, variable workloads
// No pre-provisioned server: scales to zero when idle
// Cost: ~$0 when idle, ~$0.10 per million reads
// MISSING: Atlas Search, Change Streams for external consumers
// BEST FOR: dev environments, prototypes, spiky unpredictable traffic

// M10 ($0.08/hr = $58/month): dedicated, smallest production tier
// 2GB RAM, dedicated vCPU, all Atlas features available
// Atlas Search, VPC peering, Performance Advisor
// BEST FOR: small production apps, Atlas Search development

// M30 ($0.54/hr = $389/month): production minimum recommendation
// 8GB RAM, decent IOPS, all Atlas features
// BEST FOR: medium production workloads

// M50+ ($1.04+/hr): production high-traffic
// 16GB+ RAM, higher IOPS, better network
// BEST FOR: large workloads, complex aggregations, large working sets

// Serverless vs M10 for startup:
// Serverless: $0/month in dev, ~$10-50/month in early prod → great for unpredictable growth
// M10: $58/month regardless of usage → predictable cost, all features
// Switch from Serverless to dedicated when: Atlas Search needed, stable workload, > 1M reads/day
```

---

## Answer 7: How Does Atlas Global Clusters Achieve Low-Latency Global Writes?

**Point**: Atlas Global Clusters uses zone-based sharding to physically co-locate data with the users who write and read it. Each geographic zone is a full replica set, ensuring data sovereignty and low latency.

**Example**:

```javascript
// Without Global Clusters (single-region cluster):
// EU user writes profile → goes to US-East-1 primary → 120ms round-trip latency
// EU user reads profile → reads from EU secondary → 5ms ✓
// But writes are always 120ms to US ✗

// With Global Clusters:
// EU user writes profile → goes to EU-West-1 shard (local primary) → 5ms ✓
// EU user reads profile → from EU-West-1 local → 5ms ✓
// US user reads EU user's profile → cross-zone $lookup → still works, slightly slower

// Implementation:
// Shard key: { region: 1, _id: 1 }
// Zone mapping: "EU" region → EU-West-1 shard

db.adminCommand({
  updateZoneKeyRange: {
    ns: "myapp.users",
    min: { region: "EU", _id: MinKey },
    max: { region: "EU", _id: MaxKey },
    zone: "EU"
  }
})

// Application sets region on insert:
await db.users.insertOne({
  ...userProfile,
  region: determineRegion(user.ipAddress),   // "US" | "EU" | "AP"
  _id: new ObjectId()
})

// Result: EU write → EU shard (5ms) ✓
//         GDPR: EU data physically stays in EU ✓
//         EU reads EU data → EU shard (5ms) ✓
//         US reads EU data → cross-zone hop (50-100ms) — acceptable for rare cross-region reads
```

---

## Answer 8: Explain Atlas Backup and Point-in-Time Restore

**Point**: Atlas Cloud Backup provides scheduled snapshots plus continuous oplog shipping, enabling restore to any second within the retention window.

**Example**:

```javascript
// How PITR works:
// 1. Atlas takes a full snapshot at 6 AM
// 2. Atlas continuously ships oplog to cloud storage (~every minute)
// 3. A bug in your app corrupts data at 2:47:33 PM

// Restore to 2:47:30 PM (3 seconds before corruption):
// 1. Atlas identifies nearest snapshot before 2:47:30 PM = 6:00 AM snapshot
// 2. Restores cluster from 6:00 AM snapshot
// 3. Replays oplog from 6:00 AM to exactly 2:47:30 PM
// 4. Cluster is restored to pre-corruption state
// Total time: 30-120 minutes depending on cluster size

// Backup policy for production (recommendation):
{
  "policies": [
    { "frequencyType": "hourly",   "frequencyInterval": 6, "retentionValue": 7,  "retentionUnit": "days" },
    { "frequencyType": "daily",    "frequencyInterval": 1, "retentionValue": 30, "retentionUnit": "days" },
    { "frequencyType": "weekly",   "frequencyInterval": 1, "retentionValue": 3,  "retentionUnit": "months" },
    { "frequencyType": "monthly",  "frequencyInterval": 1, "retentionValue": 12, "retentionUnit": "months" }
  ]
}

// Restore options:
// 1. Restore to existing cluster (overwrites data — irreversible until you have another backup!)
// 2. Create new cluster from snapshot (safe — doesn't touch production)
// 3. Download as .tar.gz (mongodump format) → restore locally with mongorestore

// Best practice: always restore to a NEW cluster first, verify data, then:
// Option A: redirect traffic to new cluster (atomic connection string swap)
// Option B: selective mongorestore of specific collections to production

// RTO/RPO targets with Atlas backup:
// RPO (Recovery Point Objective): up to 1 minute (oplog shipping frequency)
// RTO (Recovery Time Objective): 30-120 minutes for large clusters
// For lower RTO: use Atlas Multi-Region clusters (secondaries always available)
```

---

## Answer 9: How Do You Secure an Atlas Cluster for Production?

**Point**: Production Atlas security requires defense-in-depth: network isolation (Private Endpoints), least-privilege database access, encryption at rest + in transit, and database auditing.

**Example**:

```javascript
// LAYER 1: Network isolation (choose one or combine)
// Option A: IP allowlist (simpler, still has public endpoint)
// Allow: your app servers' IP range
// Allow: VPN/office CIDR range
// NEVER allow: 0.0.0.0/0 in production

// Option B: VPC Peering (better)
// Atlas cluster accessible only from your VPC private CIDR
// Public endpoint still exists but only reachable from peered network

// Option C: Private Endpoint (best)
// Atlas cluster has NO public internet endpoint
// Only accessible via PrivateLink endpoint inside your VPC

// LAYER 2: Database users (least privilege)
// Application user: read/write on specific database ONLY
db.createUser({
  user: "app_user",
  pwd: passwordFromVault,
  roles: [{ role: "readWrite", db: "myapp" }]
  // NOT dbAdmin, NOT clusterAdmin
})

// Read-only analytics user:
db.createUser({
  user: "analytics_user",
  pwd: passwordFromVault,
  roles: [{ role: "read", db: "myapp" }]
})

// LAYER 3: Encryption
// At rest: Atlas encrypts data at rest by default (AES-256)
// In transit: TLS 1.2+ mandatory (cannot disable on Atlas)
// KMIP/BYOK (Bring Your Own Key): for regulated industries
//   → Atlas integrates with AWS KMS, GCP Cloud KMS, Azure Key Vault
//   → Your team controls the encryption key, not MongoDB

// LAYER 4: Auditing
// Enable for authentication events + admin operations:
{
  "filter": {
    "atype": { "$in": ["authenticate", "logout", "createUser", "dropUser",
                        "dropCollection", "dropDatabase", "createIndex", "dropIndex"] }
  }
}

// LAYER 5: Secrets management
// Never hardcode credentials:
// Store in AWS Secrets Manager, GCP Secret Manager, HashiCorp Vault
// Rotate credentials quarterly using Atlas API
const password = await secretsManager.getSecretValue({ SecretId: "atlas-app-user" })
const client = new MongoClient(`mongodb+srv://app_user:${password.SecretString}@...`)
```

---

## Answer 10: What Is Atlas App Services and When Would You Use It?

**Point**: Atlas App Services is a serverless backend platform that lets you build backend logic (triggers, functions, GraphQL APIs, Device Sync) directly within Atlas without managing server infrastructure.

**Example**:

```javascript
// Core App Services components:

// 1. Atlas Functions — Node.js serverless functions
// Call from: mobile apps, frontend via HTTP, other Functions, Triggers
exports = async function myFunction(payload) {
  const db = context.services.get("mongodb-atlas").db("myapp")
  const user = context.user   // currently authenticated App Services user

  // Can call external APIs, query MongoDB, call other functions
  const orders = await db.collection("orders")
    .find({ userId: user.id })
    .toArray()

  return orders
}

// 2. Atlas GraphQL — auto-generated from your schema
// Configure schema in App Services UI → get instant GraphQL endpoint
// query {
//   order(query: { status: "pending" }) {
//     _id, customerId, total, createdAt
//   }
// }

// 3. Atlas Device Sync — offline-first mobile
// Mobile app writes locally → syncs to Atlas when online → other devices see changes
// Great for: to-do apps, field service apps, chat apps

// 4. Authentication providers
// Google, Facebook, Apple, email/password, API key, custom JWT
// All tied to Atlas App Services user management

// When to use App Services vs. self-hosted backend:
// App Services: rapid development, small team, low/medium scale
// Self-hosted: complex business logic, high scale, specific runtime requirements

// When NOT to use App Services:
// - You need Node.js 20+ specific features (App Services runtime is Node.js 16)
// - Complex background job scheduling
// - Long-running operations (> 5 minutes)
// - Workloads requiring > 256MB memory per function invocation
```
