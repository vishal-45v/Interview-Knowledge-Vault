# Chapter 10 — MongoDB Operations: Scenario Questions

---

## Scenario 1: Production Database Running Out of Disk Space

Your MongoDB Atlas M30 cluster is at 85% disk usage and growing 2% per day. You have 7 days before hitting 100%. Walk through your immediate actions and long-term fixes.

### Immediate Triage

```javascript
// Step 1: Identify what's consuming space
// Atlas UI: Cluster → Metrics → Disk space used
// Or via mongosh:
db.runCommand({ dbStats: 1, scale: 1073741824 })  // scale to GB
// {
//   "db": "mydb",
//   "storageSize": 45.2,        // GB used
//   "dataSize": 52.1,           // uncompressed size (always > storageSize)
//   "indexSize": 8.3,           // index storage
//   "totalSize": 53.5
// }

// Per-collection breakdown:
db.runCommand({ listCollections: 1 }).cursor.firstBatch.forEach(coll => {
  const stats = db.runCommand({ collStats: coll.name, scale: 1073741824 })
  print(`${coll.name}: data=${stats.storageSize.toFixed(2)}GB, index=${stats.totalIndexSize.toFixed(2)}GB`)
})
// Identify the top space consumers
```

### Root Cause Analysis

```javascript
// Finding 1: Logs collection has 200GB of old log data
// Fix A: Drop old log data beyond retention period
db.applicationLogs.deleteMany({
  createdAt: { $lt: ISODate("2025-01-01") }  // delete logs older than 1 year
})
// Run during low traffic. Use batched deletes to avoid large write locks:
async function batchDelete(collection, filter, batchSize = 10000) {
  let totalDeleted = 0
  while (true) {
    const result = await db.collection(collection).deleteMany(
      { ...filter, _id: { $lte: await getOldestIdInBatch(collection, filter, batchSize) } }
    )
    if (result.deletedCount === 0) break
    totalDeleted += result.deletedCount
    console.log(`Deleted ${totalDeleted} total`)
    await new Promise(r => setTimeout(r, 100))  // brief pause between batches
  }
}

// Fix B: Add TTL index to prevent future accumulation
db.applicationLogs.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 7776000 }  // 90 days
)

// Finding 2: Index bloat — too many indexes
db.orders.getIndexes()
// Found 12 indexes on a collection that queries use only 4
// Identify unused indexes:
db.orders.aggregate([{ $indexStats: {} }]).forEach(idx => {
  print(`${idx.name}: used ${idx.accesses.ops} times since ${idx.accesses.since}`)
})
// Drop indexes with 0 accesses (never used):
db.orders.dropIndex("old_compound_index_name")

// Finding 3: Dropped data not reclaiming disk (fragmentation)
db.runCommand({ compact: "orders" })
// compact rewrites the collection data files, reclaiming fragmented space
// CAUTION: compact blocks reads/writes during execution on standalone
// On Atlas: Atlas handles compaction automatically (WiredTiger does online compaction)
```

### Long-Term Solutions

```javascript
// Solution 1: Atlas Auto-Scaling
// Atlas UI: Cluster → Edit → Auto-scaling → Enable disk auto-scaling
// Disk scales up automatically when utilization exceeds 90%

// Solution 2: Atlas Online Archive for old data
// Move data older than 90 days to S3 (10x cheaper storage):
// Atlas UI: Cluster → Online Archive → Create Archive
// Archive rule: { "createdAt": { "$lt": { "$date": { "$subtract": [$$NOW, 7776000000] } } } }
// After archiving: query both live + archive via Data Federation

// Solution 3: Enable compression
// Already enabled by default (snappy), but verify:
db.runCommand({ collStats: "orders" }).wiredTiger.creationString
// Should include: block_compressor=snappy
// For higher compression (at CPU cost):
db.createCollection("orders_compressed", {
  storageEngine: {
    wiredTiger: { configString: "block_compressor=zstd" }
  }
})
// zstd: ~30% better compression than snappy, higher CPU
```

---

## Scenario 2: Application Experiencing Slow Queries After Data Growth

A query that used to take 10ms now takes 5 seconds after the `orders` collection grew from 1M to 50M documents. Diagnose and fix.

### Diagnosis

```javascript
// Step 1: Get the explain plan
db.orders.find({
  customerId: "cust_abc",
  status: "pending",
  createdAt: { $gte: ISODate("2026-01-01") }
}).explain("executionStats")

// Expected output (before optimization):
{
  "executionStats": {
    "nReturned": 12,
    "executionTimeMillis": 4892,     // 4.9 seconds!
    "totalKeysExamined": 250000,     // 250k index keys scanned
    "totalDocsExamined": 250000,     // 250k documents loaded
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "customerId_1"   // using single-field index
      }
    }
  }
}
// Problem: customerId index returns 250k docs for "cust_abc"
// MongoDB then filters 250k docs for status + createdAt → slow
// Need a COMPOUND index that covers all three filter fields

// Step 2: Check index statistics
db.orders.aggregate([{ $indexStats: {} }])
// Find if existing compound indexes exist but aren't being used
```

### Fix

```javascript
// Create the correct compound index (ESR rule):
// Equality → Sort → Range
db.orders.createIndex({
  customerId: 1,    // E: equality filter
  status: 1,        // E: equality filter
  createdAt: -1     // R: range filter (descending for most-recent-first queries)
})
// Name it explicitly for easy management:
db.orders.createIndex(
  { customerId: 1, status: 1, createdAt: -1 },
  { name: "orders_customer_status_date" }
)

// After index creation, explain should show:
{
  "executionStats": {
    "nReturned": 12,
    "executionTimeMillis": 2,     // 2ms!
    "totalKeysExamined": 12,      // only 12 keys (exact match)
    "totalDocsExamined": 12,
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "orders_customer_status_date",
        "indexBounds": {
          "customerId": ["[\"cust_abc\", \"cust_abc\"]"],
          "status": ["[\"pending\", \"pending\"]"],
          "createdAt": ["[MaxKey, ISODate(\"2026-01-01\"))"]
        }
      }
    }
  }
}

// Make it a covered query (no FETCH stage) by adding to projection:
db.orders.createIndex({
  customerId: 1,
  status: 1,
  createdAt: -1,
  amount: 1,     // add amount to support covered query
  _id: 1
})
// Query with projection:
db.orders.find(
  { customerId: "cust_abc", status: "pending", createdAt: { $gte: ISODate("2026-01-01") } },
  { customerId: 1, status: 1, createdAt: 1, amount: 1, _id: 1 }
)
// Now: "stage": "IXSCAN" with NO "FETCH" → covered query, zero document fetches
```

---

## Scenario 3: Designing a Real-Time Audit Log System

Your application needs a tamper-evident audit log for all data modifications: who changed what, when, and what the old and new values were. Design this using MongoDB change streams.

### Architecture Design

```javascript
// Architecture: Change Streams → Audit Log Collection
// The audit log is append-only (no updates/deletes)

// Audit log document structure:
const auditLogDoc = {
  _id: new ObjectId(),
  timestamp: new Date(),
  operationType: "update",           // insert | update | replace | delete
  collection: "users",
  documentId: ObjectId("..."),       // _id of changed document
  userId: "user_admin_123",         // WHO made the change (from app context)
  sessionId: "session_abc",
  ipAddress: "203.0.113.42",
  beforeState: {                     // what it looked like before
    email: "old@example.com",
    status: "active"
  },
  afterState: {                      // what it looks like after
    email: "new@example.com",
    status: "active"
  },
  changedFields: ["email"],          // which fields changed
  applicationVersion: "v2.3.1"
}
```

### Implementation

```javascript
// Audit service: standalone process tailing change streams
class AuditService {
  constructor(mongoClient) {
    this.client = mongoClient
    this.auditDb = mongoClient.db("audit")
    this.resumeToken = null
  }

  async start(collectionsToAudit) {
    // Load last resume token from database (survive restarts)
    const state = await this.auditDb
      .collection("_audit_state")
      .findOne({ _id: "resumeToken" })
    this.resumeToken = state?.token || null

    // Watch the entire database
    const pipeline = [
      {
        $match: {
          "ns.coll": { $in: collectionsToAudit },
          operationType: { $in: ["insert", "update", "replace", "delete"] }
        }
      }
    ]

    const changeStream = this.client.db("mydb").watch(pipeline, {
      resumeAfter: this.resumeToken,
      fullDocument: "updateLookup",        // get full doc after update
      fullDocumentBeforeChange: "required" // MongoDB 6.0+: get pre-change state
    })

    changeStream.on("change", this.handleChange.bind(this))
    changeStream.on("error", this.handleError.bind(this))
  }

  async handleChange(change) {
    const auditEntry = {
      timestamp: new Date(),
      operationType: change.operationType,
      collection: change.ns.coll,
      documentId: change.documentKey._id,
      afterState: change.fullDocument || null,
      beforeState: change.fullDocumentBeforeChange || null,  // MongoDB 6.0+
      changedFields: change.updateDescription?.updatedFields
        ? Object.keys(change.updateDescription.updatedFields)
        : null
    }

    // Use a session for atomic: write audit + update resume token
    const session = this.client.startSession()
    await session.withTransaction(async () => {
      await this.auditDb.collection("auditLog").insertOne(auditEntry, { session })
      await this.auditDb.collection("_audit_state").updateOne(
        { _id: "resumeToken" },
        { $set: { token: change._id } },  // change._id IS the resume token
        { upsert: true, session }
      )
    })
    this.resumeToken = change._id
  }

  async handleError(error) {
    // Log error, attempt to resume
    console.error("Change stream error:", error)
    // Restart will use stored resumeToken to pick up where we left off
  }
}

// Create TTL index on audit log (keep 2 years):
await db.auditLog.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 63072000 }  // 2 years
)

// Create indexes for audit queries:
await db.auditLog.createIndex({ documentId: 1, timestamp: -1 })
await db.auditLog.createIndex({ collection: 1, timestamp: -1 })
await db.auditLog.createIndex({ userId: 1, timestamp: -1 })
```

---

## Scenario 4: Handling the "Too Many Connections" Production Alert

At 2 PM on Monday (peak traffic), you receive an alert: "Atlas cluster approaching connection limit." The M30 cluster allows 3000 connections. You're at 2850. Diagnose and mitigate.

### Immediate Diagnosis

```javascript
// Step 1: Check current connections
db.adminCommand({ serverStatus: 1 }).connections
// {
//   "current": 2847,
//   "available": 153,
//   "totalCreated": 4521
// }

// Step 2: Find who's holding connections
db.adminCommand({ currentOp: 1, "$all": true }).inprog
  .filter(op => op.active)
  .map(op => ({
    connectionId: op.connectionId,
    client: op.client,
    desc: op.desc,
    secs: op.secs_running,
    ns: op.ns
  }))
// Look for: idle connections, long-running operations, unusual clients

// Step 3: Check if connections are from too many app instances
// Atlas UI: Metrics → Connections shows breakdown by source IP
```

### Root Cause: maxPoolSize Too High × Too Many App Instances

```javascript
// If each of 30 Node.js instances has maxPoolSize: 100:
// 30 × 100 = 3000 connections → at limit!

// Solution 1: Reduce maxPoolSize per instance
// Each instance needs AT MOST: (cluster connections) / (num instances)
// 3000 / 30 = 100 per instance (already at limit)
// Reduce to 50 if traffic doesn't require 100 concurrent queries per instance:
const client = new MongoClient(uri, {
  maxPoolSize: 50,   // was 100
  minPoolSize: 5
})

// Solution 2: Use connection pooling middleware (PgBouncer equivalent)
// MongoDB: Use mongocryptd or a connection proxy
// Or: Upgrade to larger cluster tier (M50 = 6000 connections)

// Solution 3: Kill idle long-lived connections
// Drivers should automatically close idle connections after maxIdleTimeMS
// If they don't (misconfigured drivers):
db.adminCommand({ currentOp: 1, "$all": true, active: false }).inprog
  .filter(op => op.secs_running > 3600)  // idle for 1 hour
  .forEach(op => {
    db.adminCommand({ killOp: 1, op: op.opid })
  })

// Solution 4: Horizontal pod autoscaler + connection-aware scaling
// In Kubernetes: don't scale pods beyond (cluster_conn_limit / maxPoolSize)
// max_replicas = floor(cluster_connection_limit / maxPoolSize / safety_factor)
// max_replicas = floor(3000 / 50 / 1.2) = 50 pods
```

---

## Scenario 5: Database Migration — Adding New Required Fields to 50M Documents

Your application is adding a new required field `region` to all 50M order documents. The field will be populated based on the `shippingAddress.country` field. Design a zero-downtime migration.

### Migration Strategy: Lazy + Backfill

```javascript
// Phase 1: Update application to handle both old and new documents
// (before any data migration)
// Application code that works with AND without the new field:
function getOrderRegion(order) {
  if (order.region) return order.region  // new format
  // fall back to computing from old format
  const country = order.shippingAddress?.country
  return mapCountryToRegion(country)  // compute on the fly
}

// Phase 2: Write new documents with the field
// New order inserts include the field:
await db.orders.insertOne({
  customerId: "c1",
  shippingAddress: { country: "US", zip: "10001" },
  region: "americas",  // NEW FIELD
  amount: 99.99,
  createdAt: new Date()
})

// Phase 3: Background backfill (never block production)
// DO NOT: db.orders.updateMany({}, [...]) — locks the collection!
// DO: batched updates with throttling

async function backfillRegion() {
  const countryToRegion = {
    "US": "americas", "CA": "americas", "MX": "americas",
    "GB": "europe", "DE": "europe", "FR": "europe",
    "JP": "apac", "AU": "apac", "SG": "apac"
  }

  let lastId = null
  let totalUpdated = 0
  const BATCH_SIZE = 1000

  while (true) {
    // Process in batches, using _id as cursor position
    const filter = {
      region: { $exists: false }  // only docs missing the field
    }
    if (lastId) filter._id = { $gt: lastId }

    const batch = await db.orders.find(filter)
      .sort({ _id: 1 })
      .limit(BATCH_SIZE)
      .project({ _id: 1, "shippingAddress.country": 1 })
      .toArray()

    if (batch.length === 0) {
      console.log(`Backfill complete. Total updated: ${totalUpdated}`)
      break
    }

    // Bulk write for efficiency
    const bulkOps = batch.map(doc => ({
      updateOne: {
        filter: { _id: doc._id },
        update: {
          $set: {
            region: countryToRegion[doc.shippingAddress?.country] || "unknown"
          }
        }
      }
    }))

    await db.orders.bulkWrite(bulkOps, {
      ordered: false,
      writeConcern: { w: 1 }  // relaxed for backfill speed
    })

    totalUpdated += batch.length
    lastId = batch[batch.length - 1]._id

    console.log(`Updated ${totalUpdated} documents`)

    // Throttle: sleep 50ms between batches to limit impact
    await new Promise(r => setTimeout(r, 50))
  }
}

// Phase 4: Add schema validation after backfill is complete
db.runCommand({
  collMod: "orders",
  validator: {
    $jsonSchema: {
      required: ["region"],
      properties: {
        region: {
          bsonType: "string",
          enum: ["americas", "europe", "apac", "unknown"]
        }
      }
    }
  },
  validationLevel: "strict"
})
// NOW: all new writes must include region
// Existing documents without region are STILL valid (validationLevel: "strict"
// validates writes, not reads of existing docs)
// To enforce retroactively: use validationLevel: "moderate" carefully
```

---

## Scenario 6: Setting Up MongoDB for a Time-Series Workload

You're building an IoT platform that will ingest 1 million sensor readings per minute. Each reading: `{ sensorId, timestamp, temperature, humidity, pressure }`. Design the optimal MongoDB setup.

### Time-Series Collection (MongoDB 5.0+)

```javascript
// MongoDB 5.0+ has native time-series collection type
// Provides: automatic bucketing, columnar storage, optimized range queries

db.createCollection("sensorReadings", {
  timeseries: {
    timeField: "timestamp",   // the datetime field (required)
    metaField: "sensorId",    // the grouping field (optional but important)
    granularity: "seconds"    // seconds | minutes | hours
    // granularity controls bucket size:
    // "seconds": 1-hour buckets (best for data arriving frequently)
    // "minutes": 24-hour buckets
    // "hours": 30-day buckets
  },
  expireAfterSeconds: 7776000  // delete data older than 90 days
})

// Insert readings (looks like normal insert):
await db.sensorReadings.insertMany([
  { sensorId: "sensor_001", timestamp: new Date(), temperature: 22.5, humidity: 65.2, pressure: 1013.25 },
  { sensorId: "sensor_002", timestamp: new Date(), temperature: 21.8, humidity: 70.1, pressure: 1012.90 },
  // ... bulk insert up to 100k at once
])

// Time-series query performance:
// Range query (highly optimized):
db.sensorReadings.find({
  sensorId: "sensor_001",
  timestamp: {
    $gte: ISODate("2026-03-17T00:00:00Z"),
    $lte: ISODate("2026-03-17T23:59:59Z")
  }
}).sort({ timestamp: 1 })
// Uses built-in time-series index automatically

// Aggregation for downsampling (common time-series pattern):
db.sensorReadings.aggregate([
  { $match: {
      sensorId: "sensor_001",
      timestamp: { $gte: ISODate("2026-03-17T00:00:00Z") }
  }},
  { $group: {
      _id: {
        sensorId: "$sensorId",
        hour: { $dateTrunc: { date: "$timestamp", unit: "hour" } }
      },
      avgTemp: { $avg: "$temperature" },
      maxTemp: { $max: "$temperature" },
      minTemp: { $min: "$temperature" },
      avgHumidity: { $avg: "$humidity" },
      readingCount: { $count: {} }
  }},
  { $sort: { "_id.hour": 1 } }
])
```

### Sharding Time-Series Collections

```javascript
// For 1M readings/minute, single-node won't scale
// Shard time-series collection:
sh.shardCollection("iot.sensorReadings", {
  sensorId: 1,       // metaField = natural shard key prefix
  timestamp: 1       // time field for range queries
})

// This distributes by sensor group AND enables time-range queries within sensor

// Write throughput:
// M50 cluster (3 shards): ~300k writes/second
// 1M readings/minute = ~16,666/second → easily handled

// Batch writes for efficiency:
const BATCH_SIZE = 5000
for (let i = 0; i < readings.length; i += BATCH_SIZE) {
  await db.sensorReadings.insertMany(
    readings.slice(i, i + BATCH_SIZE),
    { ordered: false }  // parallel, continue on error
  )
}
```

---

## Scenario 7: Diagnosing and Fixing Index Bloat

Your DBA reports that the `users` collection has 15 indexes consuming 8GB, but the collection data is only 2GB. Diagnose and safely reduce index storage.

### Diagnosis

```javascript
// Step 1: List all indexes with sizes
db.users.stats({ indexDetails: true })
// or:
db.users.aggregate([{ $indexStats: {} }])

// Step 2: Check usage statistics
db.users.aggregate([
  { $indexStats: {} }
]).forEach(idx => {
  const sinceDate = idx.accesses.since
  const ops = idx.accesses.ops
  const daysSince = (new Date() - sinceDate) / (1000 * 60 * 60 * 24)
  print(`${idx.name}: ${ops} ops, tracked since ${sinceDate.toDateString()} (${Math.round(daysSince)} days)`)
})

// Expected output:
// _id_: 1547823 ops, tracked since Mon Jan 01 2026 (75 days)
// email_1: 982341 ops, tracked since Mon Jan 01 2026 (75 days)
// createdAt_1_status_1: 0 ops, tracked since Mon Jan 01 2026 (75 days) ← UNUSED
// phone_1: 2 ops, tracked since Mon Jan 01 2026 (75 days) ← RARELY USED
// lastLogin_1_status_1_email_1: 0 ops, ← UNUSED
// ... etc
```

### Safe Index Removal

```javascript
// NEVER drop an index without careful analysis:
// 1. Zero ops might mean the index is used by background processes
// 2. Atlas Performance Advisor may have recently recommended this index
// 3. The index might be used by a rarely-run monthly report

// Safe approach: use index hiding (MongoDB 4.4+)
// Hidden indexes are NOT used by query planner but still maintained
// This lets you test "what happens if I drop this index?" safely

// Hide index (keeps it maintained but invisible to queries):
db.users.hideIndex("createdAt_1_status_1")
db.users.hideIndex("lastLogin_1_status_1_email_1")

// Run for 1 week — if no performance degradation:
db.users.dropIndex("createdAt_1_status_1")
db.users.dropIndex("lastLogin_1_status_1_email_1")

// If performance gets worse after hiding: unhide immediately
db.users.unhideIndex("createdAt_1_status_1")  // query planner uses it again instantly

// Check index sizes:
const collStats = db.users.stats()
Object.entries(collStats.indexSizes).forEach(([name, size]) => {
  print(`${name}: ${(size / 1073741824).toFixed(3)} GB`)
})

// After cleanup (assuming 8 unused indexes removed):
// Before: 15 indexes, 8GB
// After: 7 indexes, 3.5GB
// Savings: 4.5GB → also improves write performance (fewer indexes to maintain)

// Create partial indexes to replace full indexes for filtered queries:
// Instead of: { status: 1 } (indexes all documents)
// Use: { status: 1 } with partialFilterExpression: { status: "active" }
// Only 10% of users are "active" → 90% smaller index!
db.users.dropIndex("status_1")
db.users.createIndex(
  { status: 1, createdAt: -1 },
  { partialFilterExpression: { status: "active" } }
)
```

---

## Scenario 8: Setting Up Atlas Monitoring and Alerting for Production

You're the DBA setting up a new production MongoDB Atlas cluster for a fintech application. Configure comprehensive monitoring and alerting.

### Alert Configuration

```javascript
// Atlas alerts are configured via Atlas UI or API
// Critical alerts to configure:

// 1. Replication Lag
// Condition: replication lag > 60 seconds → WARNING
// Condition: replication lag > 300 seconds → CRITICAL
// Notification: PagerDuty (for CRITICAL), email (for WARNING)

// 2. Query Targeting Ratio
// Condition: query targeting ratio < 0.01 (less than 1% of queries are targeted)
// This means most queries are scatter-gather
// Notification: Slack #mongodb-ops channel

// 3. Connections Utilization
// Condition: connections > 80% of limit → WARNING
// Condition: connections > 95% of limit → CRITICAL

// 4. Disk Utilization
// Condition: disk > 80% → WARNING (time to plan expansion)
// Condition: disk > 90% → CRITICAL (urgent)

// 5. CPU Utilization
// Condition: system CPU > 80% (5-minute average) → WARNING
// Condition: system CPU > 95% (1-minute average) → CRITICAL

// 6. Cache Dirty Bytes
// Condition: WiredTiger cache dirty % > 20% → WARNING
// High dirty bytes means writes are outpacing cache flushes

// Programmatic alert setup via Atlas API:
const alertConfig = {
  "eventTypeName": "REPLICATION_OPLOG_WINDOW_RUNNING_OUT",
  "enabled": true,
  "threshold": {
    "operator": "LESS_THAN",
    "threshold": 1,
    "units": "HOURS"
  },
  "notifications": [
    {
      "typeName": "OPS_GENIE",
      "opsGenieApiKey": process.env.OPS_GENIE_KEY,
      "delayMin": 0,
      "intervalMin": 5
    }
  ]
}
// POST to Atlas Admin API: /api/atlas/v1.0/groups/{groupId}/alertConfigs
```

### Custom Monitoring Queries

```javascript
// Health check script (run every 5 minutes via cron/cronjob):
async function mongoHealthCheck(client) {
  const report = {}

  // 1. Replication lag check
  const replStatus = await client.db("admin")
    .command({ replSetGetStatus: 1 })
  const primary = replStatus.members.find(m => m.state === 1)
  const secondaries = replStatus.members.filter(m => m.state === 2)
  report.maxReplicationLagSeconds = Math.max(...secondaries.map(s =>
    Math.abs(new Date(primary.optimeDate) - new Date(s.optimeDate)) / 1000
  ))

  // 2. Slow operations
  const currentOps = await client.db("admin")
    .command({ currentOp: 1, secs_running: { $gt: 10 } })
  report.slowOpsCount = currentOps.inprog.length
  report.slowOps = currentOps.inprog.map(op => ({
    opid: op.opid,
    ns: op.ns,
    secsRunning: op.secs_running,
    clientIP: op.client
  }))

  // 3. Connection pool
  const serverStatus = await client.db("admin").command({ serverStatus: 1 })
  report.connectionsUsed = serverStatus.connections.current
  report.connectionsAvailable = serverStatus.connections.available

  // 4. WiredTiger cache
  const cache = serverStatus.wiredTiger.cache
  report.cacheDirtyPct =
    (cache["tracked dirty bytes in the cache"] / cache["maximum bytes configured"]) * 100
  report.cacheUsePct =
    (cache["bytes currently in cache"] / cache["maximum bytes configured"]) * 100

  // Alert if thresholds exceeded:
  if (report.maxReplicationLagSeconds > 60) {
    await sendAlert("WARNING: Replication lag " + report.maxReplicationLagSeconds + "s")
  }
  if (report.cacheDirtyPct > 20) {
    await sendAlert("WARNING: WiredTiger dirty cache " + report.cacheDirtyPct.toFixed(1) + "%")
  }

  return report
}
```

---

## Scenario 9: Implementing Data Archival with Atlas Online Archive

Your MongoDB Atlas cluster stores 3 years of order data. Orders older than 6 months are read very rarely (< 1% of reads) but must be kept for compliance (7 years). Implement a cost-effective archival strategy.

### Setup

```javascript
// Atlas Online Archive configuration (via Atlas UI or API):
// 1. Go to: Cluster → Online Archive → Create Archive
// Archive rule configuration:
const archiveRule = {
  "collectionName": "orders",
  "dbName": "mydb",
  "criteria": {
    "type": "DATE",
    "dateField": "createdAt",
    "dateFormat": "ISODATE",
    "expireAfterDays": 180  // archive orders older than 6 months
  },
  "partitionFields": [
    { "fieldName": "createdAt", "order": 0 },   // partition by date (required for queries)
    { "fieldName": "region", "order": 1 }       // partition by region for geographic queries
  ]
}

// Atlas Online Archive runs automatically:
// - Scans collection for documents matching criteria (daily)
// - Moves matching documents to Atlas-managed S3 storage
// - Automatically deletes from live cluster after successful archive
// - Data is queryable via Atlas Data Federation (same connection string!)

// Cost comparison (example):
// Live Atlas M30 storage: ~$0.25/GB/month
// Atlas Online Archive (S3): ~$0.023/GB/month
// 500GB of archived data: $125/month vs $11.50/month → 90% savings!
```

### Querying Archived Data

```javascript
// After archiving, you can query BOTH live and archived data via Data Federation:
// Connection string changes from:
// mongodb+srv://cluster.mongodb.net
// To Data Federation endpoint:
// mongodb+srv://federated.data.mongodb.net

// Transparent query across live + archived:
// This query automatically checks both live cluster AND S3 archive:
const archivedOrders = await db.orders.find({
  customerId: "cust_123",
  createdAt: { $gte: ISODate("2023-01-01"), $lte: ISODate("2024-01-01") }
}).toArray()
// MongoDB Data Federation routes the query to S3 (archived) automatically
// because the date range is in the archived period

// Verify where documents are:
// Live cluster: orders from last 6 months
// Archive (S3): orders from 6+ months ago
// Both queryable via same collection name through Data Federation

// IMPORTANT: Archived data has higher query latency (S3 I/O)
// Acceptable for compliance queries, NOT for production hot paths

// Partitioning is critical for performance:
// partitionField: createdAt → queries with date filter read fewer S3 files
// partitionField: region → queries with region filter skip other regions' files
// Without partition, every query scans ALL archived S3 files → very slow
```

---

## Scenario 10: Building a Zero-Downtime Index Creation Process

Your production collection has 100M documents. You need to add a compound index. Downtime is not acceptable. Design the process.

### MongoDB's Non-Blocking Index Build

```javascript
// Good news: MongoDB 4.2+ builds indexes WITHOUT blocking reads or writes
// (using a "Simultaneous Index Build" protocol)
// Bad news: index build DOES consume CPU and I/O, potentially degrading performance

// Step 1: Identify current index build capabilities
db.adminCommand({ serverStatus: 1 }).indexBuilds
// Check if any other index builds are in progress (don't run concurrent builds)

// Step 2: Create the index (non-blocking)
db.orders.createIndex(
  { customerId: 1, status: 1, createdAt: -1 },
  {
    name: "orders_customer_status_date_v2",
    background: true  // legacy flag, all builds are non-blocking in 4.2+
    // In 4.2+, background: true is redundant but doesn't hurt
  }
)
// The createIndex call BLOCKS the connection until complete
// Use a separate connection for monitoring, not the one running createIndex

// Step 3: Monitor progress in another connection
db.adminCommand({ currentOp: 1, type: "op" }).inprog
  .filter(op => op.command?.createIndexes)
  .forEach(op => {
    const progress = op.progress
    print(`Index build: ${progress?.done || 0}/${progress?.total || "?"} docs processed`)
  })

// For Atlas: Monitor via Atlas UI → Cluster → Metrics → Index Build

// Step 4: Resource throttling for index builds (MongoDB 6.0+)
// Prevent index build from consuming too much I/O:
db.adminCommand({
  setParameter: 1,
  maxIndexBuildMemoryUsageMegabytes: 200  // limit index build memory
})

// Step 5: Build on secondary first (Atlas approach for large collections)
// Atlas builds index on one secondary, verifies it, then rolls to all nodes
// Means the primary has the new index added last (minimum impact on writes)

// Step 6: Verify index is active after build:
db.orders.getIndexes()
// New index should appear in the list

// Verify it's being used:
db.orders.find({ customerId: "c1", status: "pending" }).explain("executionStats")
// Should show new index name in "indexName"

// Step 7: Drop old redundant index if it exists:
// Only after verifying new index is in use and performant
db.orders.hideIndex("old_customer_index")  // hide first to test
// Wait 24 hours, verify no performance regression
db.orders.dropIndex("old_customer_index")  // then drop safely
```
