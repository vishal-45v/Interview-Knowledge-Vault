# Chapter 10 — MongoDB Operations: Structured Answers

---

## Answer 1: Explain the WiredTiger Storage Engine Architecture

**Point:** WiredTiger is MongoDB's default storage engine, providing document-level concurrency, compression, MVCC, and a write-ahead journal. It's the engine that makes MongoDB suitable for production workloads.

**Reason:** MongoDB's performance characteristics (concurrent reads and writes, compression ratios, crash recovery) are primarily WiredTiger's responsibility. Understanding it helps you tune correctly.

**Example:**

```javascript
// WiredTiger data path:

// 1. WRITE PATH:
// Application Write → WiredTiger Cache → Journal (WAL) → Checkpoint (disk)
// Journal flush: every 100ms (configurable via journalCommitInterval)
// Checkpoint: every 60 seconds (all dirty pages flushed to disk)
// Recovery: if crash between checkpoints, journal is replayed

// 2. READ PATH:
// Query → WiredTiger Cache hit? → Return (fast)
//      → Cache miss? → Read from disk → Load into cache → Return (slow)
// Cache eviction: LRU when cache > 80% full

// 3. CONCURRENCY:
// Document-level intent locks (not collection-level)
// MVCC: readers see a snapshot from query start, not blocked by writers
// Writers update a new version; old version visible to concurrent readers

// 4. COMPRESSION (default: snappy):
// Typical 40-60% reduction in storage
// Data block level (not field level)
// CPU cost: ~5% overhead vs no compression

// Check engine internals:
db.serverStatus().wiredTiger.cache
// "maximum bytes configured": 8589934592      ← 8GB cache
// "bytes currently in cache": 3000000000      ← 3GB in use
// "tracked dirty bytes in the cache": 50000000 ← 50MB dirty (needs flushing)
// "pages read into cache": 12345             ← disk reads (cache misses)
// "pages written from cache": 5678           ← checkpoints + evictions

// Cache ratio health check:
const cache = db.serverStatus().wiredTiger.cache
const dirtyPct = (cache["tracked dirty bytes in the cache"] / cache["maximum bytes configured"]) * 100
const usedPct = (cache["bytes currently in cache"] / cache["maximum bytes configured"]) * 100
// GOOD: dirtyPct < 5%, usedPct < 80%
// WARNING: dirtyPct > 20% → writes outpacing checkpoints
// WARNING: usedPct > 95% → cache pressure, consider RAM upgrade
```

**Proof:** WiredTiger's document-level locking was the key improvement over MMAPv1 (MongoDB's previous engine) which used collection-level locks. A collection-level lock meant a single write to one document blocked all reads on the same collection — unacceptable for high concurrency.

---

## Answer 2: How Do You Design and Optimize Aggregation Pipelines?

**Point:** Aggregation pipelines are MongoDB's primary analytics and transformation tool. Performance depends critically on stage order — reduce data volume as early as possible.

**Reason:** Each stage processes the output of the previous stage. If you sort 10M documents and then filter to 100, you wasted sorting 9.99M documents. If you filter first, you sort only 100.

**Example:**

```javascript
// Anti-pattern: expensive operations before filtering
// BAD pipeline:
db.orders.aggregate([
  { $lookup: { from: "products", localField: "productId", foreignField: "_id", as: "product" } },
  { $unwind: "$product" },
  { $match: { "product.category": "electronics" } },  // filter AFTER expensive $lookup
  { $sort: { createdAt: -1 } },
  { $limit: 20 }
])

// Optimized pipeline (same result, dramatically faster):
db.orders.aggregate([
  { $match: {                              // 1. Filter orders early (uses index)
      status: "completed",
      createdAt: { $gte: ISODate("2026-01-01") }
  }},
  { $sort: { createdAt: -1 } },           // 2. Sort before limit (index sort if possible)
  { $limit: 100 },                         // 3. Limit EARLY (pass only 100 to $lookup)
  { $lookup: {                             // 4. Expensive join only on 100 docs
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product",
      pipeline: [
        { $match: { category: "electronics" } },  // filter inside $lookup
        { $project: { name: 1, category: 1 } }    // project inside $lookup
      ]
  }},
  { $unwind: "$product" },
  { $limit: 20 }                           // 5. Final limit for response
])

// Allow disk use for large sorts (last resort):
db.events.aggregate(pipeline, {
  allowDiskUse: true,
  cursor: { batchSize: 1000 }  // reduce memory per batch
})

// $facet for multiple aggregations on same data (avoids multiple queries):
db.products.aggregate([
  { $match: { category: "electronics" } },
  { $facet: {
      "priceBuckets": [
        { $bucket: {
            groupBy: "$price",
            boundaries: [0, 50, 100, 500, 1000],
            default: "1000+",
            output: { count: { $sum: 1 } }
        }}
      ],
      "topBrands": [
        { $group: { _id: "$brand", count: { $sum: 1 } } },
        { $sort: { count: -1 } },
        { $limit: 10 }
      ],
      "avgPrice": [
        { $group: { _id: null, avg: { $avg: "$price" } } }
      ]
  }}
])
// One pass through data, three results → much faster than three separate queries
```

**Proof:** MongoDB's aggregation optimizer automatically moves some `$match` stages earlier in the pipeline (optimization pass), but it can't always reorder safely. Explicit stage ordering is the developer's responsibility and has a larger impact than any automatic optimization.

---

## Answer 3: How Do You Implement a Robust Change Stream Consumer?

**Point:** A change stream consumer must handle: resume tokens (for fault tolerance), idempotent processing (for at-least-once guarantees), and error recovery (for transient failures).

**Reason:** Change streams are at-least-once. Without resume tokens, a consumer crash means missing all events since the crash. Without idempotency, duplicate processing causes data corruption.

**Example:**

```javascript
class RobustChangeStreamConsumer {
  constructor(mongoClient, redisClient) {
    this.mongo = mongoClient
    this.redis = redisClient
    this.resumeTokenKey = "cs:orders:resumeToken"
    this.processedKey = "cs:orders:processed"
  }

  async start() {
    // Load resume token (survive restarts)
    const savedToken = await this.redis.get(this.resumeTokenKey)
    const resumeToken = savedToken ? JSON.parse(savedToken) : null

    const pipeline = [
      { $match: {
          operationType: { $in: ["insert", "update"] },
          "fullDocument.status": "shipped"
      }}
    ]

    const changeStream = this.mongo.db("mydb")
      .collection("orders")
      .watch(pipeline, {
        resumeAfter: resumeToken,
        fullDocument: "updateLookup"
      })

    // Process events
    for await (const change of changeStream) {
      await this.processWithRetry(change)
    }
  }

  async processWithRetry(change, maxAttempts = 3) {
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        await this.processEvent(change)
        // Save resume token after successful processing
        await this.redis.set(
          this.resumeTokenKey,
          JSON.stringify(change._id)
        )
        return
      } catch (err) {
        if (attempt === maxAttempts) {
          // Dead letter queue for permanently failed events
          await this.mongo.db("mydb").collection("_dlq")
            .insertOne({ change, error: err.message, attempts: maxAttempts })
          // Still advance resume token (skip this event)
          await this.redis.set(this.resumeTokenKey, JSON.stringify(change._id))
          return
        }
        await new Promise(r => setTimeout(r, 1000 * attempt))  // exponential backoff
      }
    }
  }

  async processEvent(change) {
    const orderId = change.documentKey._id.toString()

    // IDEMPOTENCY CHECK: skip if already processed
    const alreadyProcessed = await this.redis.sismember(this.processedKey, orderId)
    if (alreadyProcessed) return

    // Process the event (e.g., send shipping notification)
    await sendShippingNotification(change.fullDocument)

    // Mark as processed (idempotency guard)
    await this.redis.sadd(this.processedKey, orderId)
    await this.redis.expire(this.processedKey, 604800)  // keep for 7 days
  }
}
```

**Proof:** The resume token is the change stream's "bookmark." MongoDB's oplog can replay events from any point within its window (typically hours to days). The consumer only needs to save the last successfully processed token — restarts pick up exactly where they left off with no gaps.

---

## Answer 4: How Do You Approach Index Strategy for a New Application?

**Point:** Start with the minimum indexes, measure query patterns in production (or staging load tests), then add indexes for observed slow queries. Don't over-index upfront.

**Reason:** Query patterns in production often differ from development assumptions. Indexes that seem necessary may never be used, while indexes for unexpected query patterns are missing.

**Example:**

```javascript
// Phase 1: Mandatory indexes at launch
// - Unique indexes for uniqueness constraints
// - Shard key index (if sharding)
// - Indexes for authentication queries (login by email)

db.users.createIndex({ email: 1 }, { unique: true })  // mandatory: unique login
db.sessions.createIndex({ token: 1 }, { unique: true })  // mandatory: session lookup
db.sessions.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })  // TTL: auto-delete

// Phase 2: Enable profiling in staging/production
db.setProfilingLevel(1, { slowms: 50 })  // log queries > 50ms

// Phase 3: After 1 week, analyze slow queries
db.system.profile.aggregate([
  { $match: { millis: { $gt: 50 } } },
  { $group: {
      _id: {
        op: "$op",
        ns: "$ns",
        planSummary: "$planSummary"
      },
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" },
      maxMs: { $max: "$millis" }
  }},
  { $sort: { avgMs: -1 } },
  { $limit: 20 }
])

// Atlas Performance Advisor shows this automatically and suggests indexes:
// "We recommend creating the following indexes:
//  db.orders.createIndex({ customerId: 1, createdAt: -1 })"

// Phase 4: Create recommended indexes with naming convention
db.orders.createIndex(
  { customerId: 1, createdAt: -1 },
  { name: "idx_orders_customer_date_v1" }
)

// Phase 5: Verify improvement
// Before: COLLSCAN, 3000ms
// After: IXSCAN, 2ms
db.orders.find({ customerId: "c1" }).sort({ createdAt: -1 }).explain("executionStats")

// Phase 6: Regular index audit (monthly)
db.orders.aggregate([{ $indexStats: {} }]).forEach(idx => {
  if (idx.accesses.ops < 10 && idx.name !== "_id_") {
    console.log(`UNUSED: ${idx.name} — consider removing`)
  }
})
```

**Proof:** MongoDB Atlas Performance Advisor uses slow query log analysis to recommend indexes. It calculates "impact score" based on query frequency and latency improvement, showing which indexes to create for the maximum performance gain.

---

## Answer 5: Describe MongoDB's Security Architecture

**Point:** MongoDB security is layered: network isolation (firewall/VPC), transport encryption (TLS), authentication (who you are), and authorization (what you can do). Defense in depth — no single layer is sufficient.

**Reason:** A database without authentication is completely exposed. Authentication without authorization means any authenticated user can do anything. Network restrictions without TLS expose credentials to man-in-the-middle attacks.

**Example:**

```javascript
// Layer 1: Network — IP allowlist + VPC Peering (Atlas)
// Only allowed IP ranges can connect at all

// Layer 2: TLS — encrypt data in transit
// All Atlas connections use TLS by default
// Self-managed: mongod.conf
// net:
//   tls:
//     mode: requireTLS
//     certificateKeyFile: /etc/ssl/server.pem

// Layer 3: Authentication
// Atlas: Create database users in Atlas UI or via API
// Self-managed:
db.getSiblingDB("admin").createUser({
  user: "appUser",
  pwd: passwordPrompt(),  // prompts for password (never hardcode!)
  roles: [{ role: "readWrite", db: "mydb" }]
})

// Layer 4: Authorization — least privilege principle
// Application user should have MINIMUM required permissions:
db.createRole({
  role: "orderService",
  privileges: [
    { resource: { db: "mydb", collection: "orders" }, actions: ["find", "insert", "update"] },
    { resource: { db: "mydb", collection: "inventory" }, actions: ["find", "update"] }
    // Cannot: drop collections, create indexes, access other collections
  ],
  roles: []
})

// Layer 5: Encryption at rest
// Atlas: AES-256 by default, BYOK (Bring Your Own Key) for compliance
// Self-managed: WiredTiger Encrypted Storage Engine (Enterprise only)
// Or: encrypt at file system level (LUKS, EBS encryption)

// Layer 6: Field-level encryption (Client-Side Field Level Encryption)
// Encrypt specific sensitive fields before writing to MongoDB
// Even DBA with db access cannot see plaintext:
const encryption = new ClientEncryption(client, { keyVaultNamespace, kmsProviders })
const encryptedCard = await encryption.encrypt(
  "4111111111111111",  // credit card number
  { algorithm: "AEAD_AES_256_CBC_HMAC_SHA_512_Deterministic", keyId: dataKeyId }
)
await db.payments.insertOne({ userId: "u1", cardNumber: encryptedCard })
// Stored as binary, unreadable without the DEK (Data Encryption Key)

// Layer 7: Audit logging
// Atlas: enables audit log for all operations
// Can filter by: IP, user, operation type, collection
// Essential for SOC 2 / PCI DSS compliance
```

**Proof:** NIST cybersecurity framework requires defense in depth. The 2019 MongoDB ransomware attacks targeted instances with no authentication enabled (`--noauth` flag). Proper authentication + network restrictions would have prevented all of those attacks.

---

## Answer 6: How Do You Handle a Production Incident — Database Not Responding?

**Point:** A systematic triage process: verify connectivity, check disk/CPU/memory, identify blocking operations, kill if necessary, and investigate root cause.

**Reason:** Under pressure, systematic thinking prevents making things worse. Random restarts without diagnosis often leave root cause unaddressed.

**Example:**

```javascript
// STEP 1: Verify the problem
// Can you connect at all?
mongosh "mongodb+srv://cluster.mongodb.net" --username admin
// Timeout → connectivity issue (network/cluster down)
// Connects but hangs → cluster responsive but operations queued

// STEP 2: Check Atlas status page
// https://status.mongodb.com — is this an Atlas-wide issue?

// STEP 3: If connected, check current operations
db.adminCommand({ currentOp: 1, "$all": true, active: true })
// Look for:
// - Long-running operations (secs_running > 60)
// - Lock state (lockStats with high timeAcquiringMicros)
// - Many queued operations waiting for locks

// STEP 4: Check for resource saturation
db.adminCommand({ serverStatus: 1 }).wiredTiger.concurrentTransactions
// {
//   "write": { "out": 128, "available": 0, "totalTickets": 128 },
//   "read": { "out": 128, "available": 0, "totalTickets": 128 }
// }
// available: 0 → all tickets in use → everything is queuing!

// STEP 5: Kill the offending operations
// Find the blocking query:
const blocking = db.adminCommand({ currentOp: 1 }).inprog
  .filter(op => op.secs_running > 30)
  .sort((a, b) => b.secs_running - a.secs_running)

blocking.forEach(op => {
  console.log(`Killing op ${op.opid}: running ${op.secs_running}s, ns: ${op.ns}`)
  db.adminCommand({ killOp: 1, op: op.opid })
})

// STEP 6: Identify root cause
// Was it a missing index (COLLSCAN on 50M documents)?
// Was it a runaway batch job?
// Was it a new code deployment with an unoptimized query?

// STEP 7: Prevention
// Set maxTimeMS on all queries:
db.orders.find({ status: "pending" }).maxTimeMS(5000)
// This prevents runaway queries from blocking the database
// A query that can't complete in 5 seconds fails fast rather than blocking indefinitely

// Atlas: Set operation timeout in cluster config
// Atlas UI → Cluster → Advanced Options → Set Max Operation Time
```

**Proof:** The most common MongoDB production incident is a missing index on a new query that was accidentally added via deployment. A COLLSCAN on 50M documents with multiple concurrent requests exhausts all read tickets, making the entire cluster unresponsive. `maxTimeMS` is the circuit breaker.

---

## Answer 7: How Do You Implement Efficient Pagination in MongoDB?

**Point:** There are two pagination methods: skip/limit (simple but slow on large offsets) and cursor-based (consistent and fast for all page positions).

**Reason:** `skip(N)` scans and discards N documents — for page 100 with 20 items per page, MongoDB scans and discards 1,980 documents. For deep pagination on large collections, this becomes prohibitively slow.

**Example:**

```javascript
// Method 1: Skip/Limit (avoid for large offsets)
// Simple but O(offset) scan:
function getPageSimple(page, pageSize) {
  return db.orders
    .find({ status: "completed" })
    .sort({ createdAt: -1 })
    .skip((page - 1) * pageSize)  // O(n) scan!
    .limit(pageSize)
    .toArray()
}
// Page 1 is fast, page 1000 scans and discards 999 × pageSize documents

// Method 2: Cursor-based pagination (recommended)
// Use the last document's sort key as the cursor:
async function getFirstPage(pageSize) {
  const docs = await db.orders
    .find({ status: "completed" })
    .sort({ createdAt: -1, _id: -1 })  // _id as tiebreaker for stable sort
    .limit(pageSize)
    .toArray()

  const cursor = docs.length > 0
    ? { createdAt: docs[docs.length - 1].createdAt, _id: docs[docs.length - 1]._id }
    : null

  return { docs, cursor }
}

async function getNextPage(cursor, pageSize) {
  if (!cursor) return { docs: [], cursor: null }

  const docs = await db.orders
    .find({
      status: "completed",
      $or: [
        { createdAt: { $lt: cursor.createdAt } },  // strictly older
        { createdAt: cursor.createdAt, _id: { $lt: cursor._id } }  // same time, later _id
      ]
    })
    .sort({ createdAt: -1, _id: -1 })
    .limit(pageSize)
    .toArray()

  const nextCursor = docs.length > 0
    ? { createdAt: docs[docs.length - 1].createdAt, _id: docs[docs.length - 1]._id }
    : null

  return { docs, cursor: nextCursor }
}
// Page 1000 is as fast as page 1: uses an index seek to the cursor position

// Index to support cursor-based pagination:
db.orders.createIndex({ status: 1, createdAt: -1, _id: -1 })

// Limitation: cursor-based pagination doesn't support "jump to page N"
// For that use case: faceted search with $searchMeta (Atlas Search)
// or accept skip/limit with a "max offset" limit (e.g., 1000 items max)
```

**Proof:** Twitter's Timeline API, Instagram's feed API, and most social media APIs use cursor-based pagination for exactly this reason. Skip/limit is only acceptable for small, bounded data sets where deep pagination doesn't occur.

---

## Answer 8: How Do You Implement Database Health Monitoring?

**Point:** Comprehensive monitoring tracks: operation latency, resource utilization (CPU/RAM/disk/IOPS), replication health, connection pool, and slow query rate.

**Reason:** Database problems manifest as multiple signals. Replication lag alone doesn't tell you if the problem is disk I/O, CPU, or network. Correlating multiple metrics quickly identifies root cause.

**Example:**

```javascript
// Health monitoring script (run via Atlas Triggers or external cron):
async function collectHealthMetrics(db) {
  const serverStatus = await db.adminCommand({ serverStatus: 1 })

  return {
    timestamp: new Date(),

    // Operations
    opsPerSec: {
      queries: serverStatus.opcounters.query / serverStatus.uptime,
      inserts: serverStatus.opcounters.insert / serverStatus.uptime,
      updates: serverStatus.opcounters.update / serverStatus.uptime,
      deletes: serverStatus.opcounters.delete / serverStatus.uptime
    },

    // WiredTiger cache
    cache: {
      usedPct: (serverStatus.wiredTiger.cache["bytes currently in cache"] /
                serverStatus.wiredTiger.cache["maximum bytes configured"]) * 100,
      dirtyPct: (serverStatus.wiredTiger.cache["tracked dirty bytes in the cache"] /
                 serverStatus.wiredTiger.cache["maximum bytes configured"]) * 100,
      readFromDisk: serverStatus.wiredTiger.cache["pages read into cache"]
    },

    // Connections
    connections: {
      current: serverStatus.connections.current,
      available: serverStatus.connections.available,
      utilizationPct: (serverStatus.connections.current /
        (serverStatus.connections.current + serverStatus.connections.available)) * 100
    },

    // Replication (primary only)
    replication: await getReplicationMetrics(db),

    // Slow queries (from profiler)
    slowQueries: await db.system.profile
      .find({ ts: { $gte: new Date(Date.now() - 60000) }, millis: { $gt: 1000 } })
      .count()
  }
}

// Send to time-series database (InfluxDB, Datadog, Atlas Charts):
// Atlas Charts can graph these metrics directly from the monitoring collection
await db.metricsHistory.insertOne(await collectHealthMetrics(db))

// Alert thresholds:
const THRESHOLDS = {
  cacheUsedPct: { warn: 85, critical: 95 },
  cacheDirtyPct: { warn: 10, critical: 20 },
  connectionUtilization: { warn: 80, critical: 95 },
  slowQueriesPerMinute: { warn: 5, critical: 20 },
  replicationLagSeconds: { warn: 60, critical: 300 }
}
```

**Proof:** MongoDB Atlas provides built-in metrics dashboards and alerts. For self-managed, the MongoDB Ops Manager or Percona Monitoring and Management (PMM) provide similar capabilities. The key is proactive monitoring — finding the problem at 30% disk usage rather than at 99%.

---

## Answer 9: How Do You Handle a Large-Scale Data Deletion?

**Point:** Deleting large amounts of data in MongoDB requires batched operations with throttling. A single `deleteMany` on millions of documents generates massive oplog entries and causes replication lag.

**Reason:** `deleteMany` generates one oplog entry per deleted document. Deleting 10M documents at once creates 10M oplog entries simultaneously, overwhelming secondary replication and consuming enormous oplog space.

**Example:**

```javascript
async function batchDelete(collectionName, filter, options = {}) {
  const {
    batchSize = 10000,
    delayMs = 100,          // pause between batches
    writeConcern = { w: 1 } // relaxed for speed during bulk delete
  } = options

  const collection = db.collection(collectionName)
  let totalDeleted = 0
  let lastId = null

  while (true) {
    // Build query with _id cursor for efficient batching
    const batchFilter = { ...filter }
    if (lastId) batchFilter._id = { $gt: lastId }

    // Get IDs of documents to delete (lighter than fetching full docs)
    const docsToDelete = await collection
      .find(batchFilter, { projection: { _id: 1 } })
      .sort({ _id: 1 })
      .limit(batchSize)
      .toArray()

    if (docsToDelete.length === 0) {
      console.log(`Batch delete complete. Total: ${totalDeleted}`)
      break
    }

    const ids = docsToDelete.map(d => d._id)
    const result = await collection.deleteMany(
      { _id: { $in: ids } },
      { writeConcern }
    )

    totalDeleted += result.deletedCount
    lastId = ids[ids.length - 1]

    console.log(`Deleted batch: ${result.deletedCount}, Total: ${totalDeleted}`)

    // Monitor replication lag between batches
    const replStatus = await db.adminCommand({ replSetGetStatus: 1 })
    const primary = replStatus.members.find(m => m.state === 1)
    const maxLag = Math.max(...replStatus.members
      .filter(m => m.state === 2)
      .map(m => (new Date(primary.optimeDate) - new Date(m.optimeDate)) / 1000)
    )

    if (maxLag > 30) {
      console.log(`Replication lag ${maxLag}s — pausing...`)
      await new Promise(r => setTimeout(r, 5000))  // pause 5s
    } else {
      await new Promise(r => setTimeout(r, delayMs))  // normal pause
    }
  }
}

// Usage:
await batchDelete(
  "events",
  { createdAt: { $lt: ISODate("2025-01-01") } },  // delete events older than 2025
  { batchSize: 5000, delayMs: 200 }
)

// After large deletion: reclaim disk space
// WiredTiger handles this automatically (background compaction)
// Atlas: no manual compaction needed
// Self-managed: if urgent, run compact during maintenance window
```

**Proof:** Netflix ran a 600M document deletion job using batched deletes and reported that the main operational concern was the TTL index approach (too slow for urgent cleanup) versus the batch script approach. Batching at 10k per iteration with monitoring took 4 hours but had zero production impact.

---

## Answer 10: How Do You Migrate from One MongoDB Schema to Another Without Downtime?

**Point:** Schema migration in MongoDB uses the lazy migration pattern (Schema Versioning): deploy new code that reads both old and new formats, gradually migrate documents, then remove backward compatibility code after migration completes.

**Reason:** Unlike SQL ALTER TABLE (which blocks all writes during execution), MongoDB schema changes don't require a data migration to proceed. New code can be deployed before data is migrated, enabling zero-downtime migration.

**Example:**

```javascript
// MIGRATION SCENARIO:
// Old: { firstName: "Jane", lastName: "Doe" }
// New: { name: { first: "Jane", last: "Doe" }, schemaVersion: 2 }

// STEP 1: Deploy code that handles BOTH formats
class UserService {
  getFullName(user) {
    if (user.schemaVersion === 2) {
      // New format
      return `${user.name.first} ${user.name.last}`
    } else {
      // Old format (backward compatible)
      return `${user.firstName} ${user.lastName}`
    }
  }

  async saveUser(userId, updates) {
    // Always write in new format
    const update = {
      $set: {
        name: { first: updates.firstName, last: updates.lastName },
        schemaVersion: 2
      },
      $unset: { firstName: "", lastName: "" }  // remove old fields
    }
    return db.users.updateOne({ _id: userId }, update)
  }
}

// STEP 2: Background migration (lazy migration triggered on reads)
async function migrateOnRead(userId) {
  const user = await db.users.findOne({ _id: userId })
  if (user && user.schemaVersion !== 2) {
    // Migrate on read (lazy)
    await db.users.updateOne(
      { _id: userId },
      {
        $set: {
          name: { first: user.firstName, last: user.lastName },
          schemaVersion: 2
        },
        $unset: { firstName: "", lastName: "" }
      }
    )
    return { ...user, name: { first: user.firstName, last: user.lastName }, schemaVersion: 2 }
  }
  return user
}

// STEP 3: Background batch migration for inactive users
async function batchMigrateToV2() {
  let cursor = null
  let migrated = 0

  while (true) {
    const filter = { schemaVersion: { $ne: 2 } }
    if (cursor) filter._id = { $gt: cursor }

    const batch = await db.users.find(filter)
      .sort({ _id: 1 }).limit(1000).toArray()

    if (batch.length === 0) break

    const bulkOps = batch.map(user => ({
      updateOne: {
        filter: { _id: user._id },
        update: {
          $set: { name: { first: user.firstName, last: user.lastName }, schemaVersion: 2 },
          $unset: { firstName: "", lastName: "" }
        }
      }
    }))

    await db.users.bulkWrite(bulkOps, { ordered: false })
    migrated += batch.length
    cursor = batch[batch.length - 1]._id
    await new Promise(r => setTimeout(r, 100))  // throttle
  }

  console.log(`Migration complete: ${migrated} users migrated to v2`)
}

// STEP 4: After 100% migration, remove backward compatibility code
// STEP 5: Add schema validation for v2 format
// STEP 6: Drop old indexes (firstName, lastName individual indexes)
//         Create new indexes (name.first, name.last)
```

**Proof:** Stripe, Airbnb, and most large-scale MongoDB deployments use this exact pattern. The `schemaVersion` field is the key — it makes backward compatibility decisions deterministic and testable. Complete migration might take days or weeks for large datasets, but there's zero downtime throughout.
