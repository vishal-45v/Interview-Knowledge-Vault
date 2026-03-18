# Chapter 07 — Atlas Features: Follow-Up Traps

---

## Trap 1: "Atlas M0 (Free Tier) Has All Features"

**Wrong answer**: "I'll develop and test everything on M0, then deploy the same to production."

**The limitations of M0 you must know**:

```javascript
// M0 (Free tier) MISSING features:
// ✗ No Atlas Search
// ✗ No Atlas Triggers / App Services (limited — basic functions only)
// ✗ No VPC Peering or Private Endpoints
// ✗ No Performance Advisor
// ✗ No Atlas Charts (limited)
// ✗ No Database Auditing
// ✗ No Online Archive
// ✗ No Advanced Monitoring (limited metric retention)
// ✗ No Auto-scaling
// ✗ No dedicated IOPS (shared infrastructure = noisy neighbor risk)
// ✗ Shared cluster = your queries compete with other M0 users
// ✗ Connection limit: 100 connections (vs thousands for M10+)
// ✗ Max storage: 512MB
// ✗ No backup (point-in-time restore not available)

// M10 minimum for:
// - Atlas Search
// - Performance Advisor
// - Database Auditing
// - VPC Peering
// - Cloud Backup

// Practical impact:
// Developer uses Atlas Search on local Atlas Search (M10+)
// Pushes to M0 free cluster for "testing"
// Atlas Search queries fail silently or with error:
// MongoServerError: Cannot run $search command as Atlas Search is not enabled

// CORRECT approach:
// - Use M0 for learning and basic CRUD testing
// - For full-featured testing: Atlas free trial with M10+ for 7 days
// - OR: use Atlas Serverless (smaller cost but more features than M0)
// - OR: run MongoDB locally with Atlas Search replaced by text index
```

---

## Trap 2: "Atlas Automatically Backs Up My Data"

**Wrong answer**: "Atlas handles backups automatically — I don't need to think about it."

**What you actually need to configure**:

```javascript
// Atlas Cloud Backup is NOT automatically enabled on new clusters!
// You must:
// 1. Enable Cloud Backup when creating the cluster (or after)
// 2. Configure the backup policy (frequency + retention)
// 3. Set up backup window (when to take snapshots — avoid peak hours)

// Default state for a NEW Atlas cluster:
// Cloud Backup: DISABLED
// You must explicitly enable it

// Enabling backup via Atlas API:
// PATCH https://cloud.mongodb.com/api/atlas/v1.0/groups/{groupId}/clusters/{clusterName}
{
  "providerBackupEnabled": true,
  "backupPolicy": {
    "restoreWindowDays": 7
  }
}

// What "Cloud Backup enabled" gives you:
// - Snapshots on your configured schedule
// - Point-in-time restore within retention window
// - Ability to download snapshots
// - Cross-region backup (for DR)

// What it does NOT do automatically:
// - Test restores (you must periodically restore and verify!)
// - Application-level backups (e.g., exporting specific collections with mongodump)
// - Cross-cluster validation
// - Alert on backup failures (configure backup failure alerts in Atlas UI)

// Critical: test your backup
const restored = await atlasAPI.createRestoreJob({
  deliveryType: "create",
  targetClusterName: "restore-test-cluster",
  snapshotId: latestSnapshotId
})
// Verify restored cluster has correct data:
db.orders.countDocuments()   // should match source
```

---

## Trap 3: "Atlas Triggers Are Reliable and Execute Exactly Once"

**Wrong answer**: "Atlas Triggers guarantee exactly-once execution — no need for idempotency."

**The reality**:

```javascript
// Atlas Triggers MIGHT execute more than once in rare scenarios:
// - If the trigger function times out and the retry policy retries it
// - After an Atlas infrastructure event
// - In rare cases of exactly-once delivery failure in change stream processing

// Atlas Trigger retry policy:
// Default: retry failed executions automatically
// If your trigger is NOT idempotent, a retry can cause:
// - Duplicate emails sent to users
// - Duplicate notifications created
// - Duplicate charges processed

// CORRECT: always write idempotent trigger functions
exports = async function(changeEvent) {
  const orderId = changeEvent.fullDocument._id

  // Idempotency: check if we already processed this event
  const db = context.services.get("mongodb-atlas").db("ecommerce")

  const alreadyNotified = await db.collection("processedNotifications").findOne({
    changeEventId: changeEvent._id.toString()   // change stream event has unique _id
  })

  if (alreadyNotified) {
    console.log(`Already processed event ${changeEvent._id} — skipping`)
    return
  }

  // Process the event
  await db.collection("notifications").insertOne({
    orderId,
    type: "order_confirmed",
    createdAt: new Date()
  })

  // Mark as processed
  await db.collection("processedNotifications").insertOne({
    changeEventId: changeEvent._id.toString(),
    processedAt: new Date()
  })
}

// Additional considerations:
// - Atlas Triggers timeout: 300 seconds max execution time
// - If trigger does HTTP calls: implement timeout + error handling
// - Trigger queue depth: if trigger falls behind, Atlas may skip events
// - Use trigger logs to monitor failures: Atlas UI → App Services → Logs
```

---

## Trap 4: "Atlas Private Endpoint Is the Same as VPC Peering"

**Wrong answer**: "I have VPC peering set up, so my Atlas data is fully private."

**The difference**:

```javascript
// VPC Peering:
// - Your VPC and Atlas VPC are directly connected (peering route)
// - Traffic stays within cloud provider network
// - Atlas cluster is accessible from your VPC via private IPs
// - BUT: Atlas still has a public IP (can be accessed externally if IP allowlist allows)
// - Shared peering infrastructure — other Atlas customers use the same Atlas VPC

// Private Endpoint (AWS PrivateLink / GCP Private Service Connect / Azure Private Link):
// - Creates a private endpoint ENI INSIDE your VPC
// - Atlas appears as if it has an IP address IN YOUR VPC
// - No direct internet exposure at all
// - Only your VPC can access this endpoint
// - Much stronger isolation than VPC peering

// Choosing between them:
// VPC Peering: cheaper, works for most use cases, Atlas still has public endpoint
// Private Endpoint: stricter security requirements (finance, healthcare), zero public exposure

// Practical difference:
// With VPC Peering: nmap shows Atlas cluster reachable on public internet (but restricted by allowlist)
// With Private Endpoint: Atlas cluster has no public internet endpoint at all

// Private Endpoint setup (AWS):
// 1. In Atlas UI: create a Private Endpoint
//    → Atlas gives you a "Service Name": com.amazonaws.vpce.us-east-1.vpce-svc-0123456789abcdef0
// 2. In AWS Console: create a VPC Endpoint using that service name
//    → AWS gives you an endpoint ID: vpce-01234567890abcdef
// 3. Back in Atlas: add the VPC Endpoint ID to the private endpoint config
// 4. Atlas gives you a special connection string:
//    "mongodb+srv://user:pass@cluster0-pl-0.abc123.mongodb.net/"
//    (different DNS resolves to private endpoint)
// 5. Remove public IP allowlist entries — cluster no longer accessible publicly
```

---

## Trap 5: "Atlas Performance Advisor Recommends Indexes I Should Always Create"

**Wrong answer**: "I follow all Performance Advisor recommendations and create every suggested index."

**The nuanced answer**:

```javascript
// Performance Advisor recommendations are suggestions, not mandates.
// Always evaluate before creating:

// EXAMPLE: Performance Advisor suggests:
// Index: { lastLoginAt: 1, userId: 1 }
// Impact: 15 slow queries last week

// BEFORE creating, ask:
// 1. How often is lastLoginAt updated?
//    If users update it on every login: 1M users × 1 login/day = 1M index updates/day
//    → 1M extra B-tree operations daily just for this index
//    → May SLOW DOWN your system more than it helps

// 2. What's the index's selectivity?
db.users.aggregate([
  { $group: { _id: "$lastLoginAt", count: { $sum: 1 } } },
  { $group: { _id: null, uniqueValues: { $sum: 1 }, total: { $sum: "$count" } } }
])
// If most lastLoginAt values cluster in the last 7 days, selectivity is poor
// → Index won't help much because too many documents match

// 3. How many indexes does the collection already have?
db.users.getIndexes().length   // if > 10, evaluate carefully

// 4. Will the query benefit from a covered query?
// If yes: include projection fields in the index → totalDocsExamined: 0

// When to IGNORE Performance Advisor recommendations:
// - The "slow queries" only ran during an unusual event (data migration, job)
// - The query is already fast enough (100ms is the threshold, but 80ms is fine too)
// - The field has very low cardinality (boolean — only 2 values)
// - Adding the index would hurt write throughput more than it helps read performance
// - You're already over-indexed (10+ indexes on the collection)

// Atlas also has $indexStats to see which indexes are actually used:
db.users.aggregate([{ $indexStats: {} }])
// { name: "lastLoginAt_1_userId_1", accesses: { ops: 0, since: ISODate("2024-01-01") } }
// ops: 0 means this index has NEVER been used → candidate for deletion!
```

---

## Trap 6: "Atlas Search Is Always Faster Than MongoDB Text Indexes"

**Wrong answer**: "Atlas Search is always better than $text indexes — I should replace all text indexes with Atlas Search."

**When native text indexes are fine**:

```javascript
// Atlas Search advantages:
// - Fuzzy matching, autocomplete, facets, highlighting, relevance scoring
// - Better tokenization (stemming, stop words, synonyms)
// - Phrase matching, proximity queries
// - Compound queries (text + geo + numeric range)

// Atlas Search disadvantages:
// - Asynchronous index sync (small delay between write and searchability)
// - Additional infrastructure cost (Lucene nodes)
// - More configuration required (search index schema)
// - Not available on M0/M2/M5 tiers
// - Not available on Atlas Serverless

// Native $text index is fine when:
// - Simple keyword matching is sufficient
// - Exactly one field or a small set of fields
// - No fuzzy matching or autocomplete needed
// - You're on a shared tier (M0/M2/M5)
// - Cost matters and the extra features aren't needed

// Example where $text is sufficient:
// Internal admin search: "find all users with username containing 'admin'"
db.users.createIndex({ username: "text" })
db.users.find({ $text: { $search: "admin" } })
// Simple, fast, sufficient for exact keyword matching

// Example where Atlas Search is needed:
// Public product search: "wireless headphons" (typo) → "wireless headphones"
// Atlas Search fuzzy: { fuzzy: { maxEdits: 1 } }
// $text: won't match "headphons" ≠ "headphones"

// Atlas Search index sync delay:
// When you insert/update a document:
// → Atlas cluster receives it immediately
// → Atlas Search index is updated asynchronously (~seconds delay)
// → A search right after an insert might not find the new document
// → For Atlas vector search: same asynchronous behavior
// If you need immediate searchability: use findOne by _id, not $search
```

---

## Trap 7: "Online Archive Makes Old Data Completely Unavailable"

**Wrong answer**: "Once data is archived to Online Archive, it's gone from MongoDB — you need to query S3 directly."

**The reality**:

```javascript
// Online Archive moves data FROM your Atlas cluster TO S3 (object store)
// The DATA IS STILL QUERYABLE via Atlas Data Federation

// Two query paths after archiving:
// 1. Atlas cluster (hot data): fast (indexed), last 90 days
// 2. Data Federation (includes both hot + archived cold data): unified query interface

// Connection string for Data Federation:
// mongodb+srv://user:pass@data.mongodb.net/
// This endpoint transparently queries BOTH the cluster AND the S3 archive

// Same aggregation pipeline, different endpoint:
// Hot data (cluster endpoint — fast):
const hotResult = await hotClient.db("finance").collection("txns")
  .find({ customerId: id, createdAt: { $gte: ninetyDaysAgo } })
  .toArray()

// Combined hot + cold (federation endpoint — slower for old data):
const fullResult = await fedClient.db("finance").collection("txns")
  .find({ customerId: id, createdAt: { $gte: sevenYearsAgo } })
  .toArray()

// The IMPORTANT difference from regular Atlas queries:
// Archived data on S3 has NO indexes
// Full partition scan instead of B-tree index lookup
// → Expected: 10-120 seconds for large archives
// → vs 1-10ms on Atlas cluster

// What you CANNOT do with archived data:
// - Write to it (read-only)
// - Update the archive schema (frozen as-of archiving)
// - Get sub-second response times on complex queries
// - Use atlas transactions referencing archived documents

// Best practice: set partition fields in the archive config
// Partition fields create S3 prefix-based "pseudo-indexes"
// If your query filter matches the partition field, S3 scan is narrowed:
// partitionFields: [{ fieldName: "customerId" }, { fieldName: "createdAt" }]
// Query: { customerId: "alice", createdAt: { $gte: 2023-01-01 } }
// → Only scans alice's 2023 partition files (not all files)
```

---

## Trap 8: "Atlas Auto-Scaling Scales Infinitely"

**Wrong answer**: "I set up auto-scaling on Atlas, so I don't need to worry about capacity planning."

**The limits of Atlas Auto-Scaling**:

```javascript
// Atlas Auto-Scaling capabilities:
// 1. Storage auto-scaling: automatically increases disk size when 90% full
//    - Max scale: up to the cluster's tier-maximum storage
//    - Scales UP only (cannot scale storage down)
//    - No downtime during storage scaling (online operation)

// 2. Compute auto-scaling (M10+): automatically scales cluster tier up/down
//    - Min/max tier bounds must be configured
//    - Example: auto-scale between M10 ($0.09/hr) and M50 ($0.52/hr)
//    - Triggers: CPU > 75% sustained or memory pressure
//    - Scale up: ~5 minute delay (rolling restart of nodes)
//    - Scale down: ~5 minute delay (happens when utilization drops)

// What auto-scaling CANNOT do:
// 1. Scale OUT (add shards) — only scales UP (bigger machines)
//    → If throughput exceeds even M600 capacity: no auto-scale solution
//    → Must manually add shards

// 2. Scale connection count
//    → M30 max connections: 3,000
//    → If you need 10,000 concurrent connections: must scale manually to larger tier

// 3. React instantly to sudden spikes
//    → Auto-scale has a 5-minute lag (collects metrics, evaluates, applies)
//    → A sudden 10x traffic spike overwhelms the cluster before auto-scale kicks in
//    → Solution: capacity planning + Atlas connection pooling proxy (Atlas Proxy / mongocryptd)

// 4. Scaling sharded clusters
//    → Auto-scaling for individual shards applies independently
//    → Adding new shards requires manual intervention or Atlas Stream Processing triggers

// Best practice: set auto-scale BOUNDS tightly
// Too wide: scale up to very expensive tier, then scale down slowly → bill shock
// Too narrow: auto-scale doesn't help when you actually need it
// Right: P95 of your load should fit in the mid-range tier

// Monitor auto-scale events:
// Atlas UI → Metrics → "Cluster Tier" metric shows tier changes over time
// Set alert: "Cluster tier changed" → notify ops team
```

---

## Trap 9: "Atlas Data API Is Equivalent to Using the MongoDB Driver"

**Wrong answer**: "I can use the Atlas Data API instead of a MongoDB driver for my production Node.js application."

**Why it's not equivalent**:

```javascript
// Atlas Data API:
// - HTTP/1.1 based (high latency: 50-200ms per request)
// - No connection pooling (each request is a new HTTP connection)
// - No transactions across operations
// - Rate limited (varies by tier)
// - JSON only (no BSON types like ObjectId natively — special syntax required)
// - No streaming results (must paginate manually)
// - No cursor-based queries

// MongoDB Driver (Node.js, Python, Java, etc.):
// - MongoDB wire protocol over TCP (low latency: 1-10ms per operation)
// - Persistent connection pool (amortizes connection overhead)
// - Full transaction support
// - Binary BSON protocol (efficient for large documents)
// - Streaming cursor support (memory-efficient for large result sets)
// - No rate limits from Atlas
// - Access to ALL MongoDB features

// Atlas Data API special JSON format for BSON types:
{ "_id": { "$oid": "507f1f77bcf86cd799439011" } }    // ObjectId
{ "price": { "$numberDecimal": "29.99" } }             // Decimal128
{ "createdAt": { "$date": { "$numberLong": "1648000000000" } } }  // Date

// Native driver format:
{ "_id": ObjectId("507f1f77bcf86cd799439011") }
{ "price": NumberDecimal("29.99") }
{ "createdAt": new Date() }

// When Data API IS appropriate:
// - Edge computing (Cloudflare Workers, Vercel Edge Functions)
//   → These environments can't run MongoDB driver (no TCP, limited Node.js)
// - Quick prototypes or webhook handlers
// - IoT devices with limited memory (can't run MongoDB driver)
// - Environments with strict outbound port restrictions (only 443 allowed)
// - Mobile apps (though Atlas Device Sync is better for mobile)

// Never use Data API for:
// - High-throughput production applications (too slow)
// - Bulk data processing
// - Applications requiring transactions
// - Anything requiring cursor-based streaming
```

---

## Trap 10: "Atlas Vector Search Works Out of the Box With Any Text Field"

**Wrong answer**: "I can use $vectorSearch on any field in my collection."

**The requirements**:

```javascript
// Atlas Vector Search requirements:
// 1. A search index MUST be created with type "vectorSearch"
//    Without the index, $vectorSearch throws an error:
//    MongoServerError: Search index 'vector_index' does not exist.

// 2. The field being searched must be an ARRAY OF NUMBERS (vector embedding)
//    - NOT a text string
//    - NOT an array of objects
//    - Must be a flat array: [0.001, -0.023, 0.154, ...]

// 3. The embedding dimensions must match the index configuration
//    - Index says: numDimensions: 1536
//    - Document embedding must be EXACTLY 1536 floats
//    - If your model changes (e.g., switch from Ada-002 to GPT-4 embeddings),
//      you must RE-EMBED ALL DOCUMENTS and rebuild the index

// 4. Query vector must match index dimensions
db.articles.aggregate([
  {
    $vectorSearch: {
      index: "vector_index",
      path: "embedding",       // must be the ARRAY OF NUMBERS field
      queryVector: [0.001, -0.023, ...],   // MUST be same length as numDimensions
      numCandidates: 100,
      limit: 10
    }
  }
])

// 5. Atlas Search must be available on your tier (M10+ or Atlas dedicated)
//    Not on M0/M2/M5, not on Serverless

// Common mistakes:
// Mistake 1: storing embedding as a string (e.g., "[0.001, -0.023, ...]")
//   → Fix: always store as native BSON array

// Mistake 2: embedding with wrong model dimensions after index is built
//   → Fix: delete and recreate the search index with correct numDimensions

// Mistake 3: using $vectorSearch without a $project to exclude the embedding
//   → Result: returns embedding array (1536 floats) in each result = massive response
db.articles.aggregate([
  { $vectorSearch: { ... } },
  {
    $project: {
      title: 1, body: 1,
      score: { $meta: "vectorSearchScore" },
      embedding: 0    // EXCLUDE the embedding from results!
    }
  }
])

// Mistake 4: assuming Atlas Vector Search is a keyword search
//   → Vector search finds SEMANTICALLY similar content (meaning-based)
//   → "good article" might match "excellent piece" (semantic)
//   → "good article" might NOT match "good articles" if embeddings differ significantly
//   → For exact keyword matching: use Atlas Search text query, not vector search
```
