# Chapter 09 — Replication & Sharding: Scenario Questions

---

## Scenario 1: Primary Election During Maintenance Window

Your 3-node replica set (PSS configuration) is in production. You need to perform maintenance on the primary node (patching the OS). You want to minimize downtime and ensure the application reconnects automatically.

**Walk through the correct procedure and explain what happens at the driver level.**

### Solution

```javascript
// Step 1: Check current replica set status
rs.status()
// Identify which node is primary, note replication lag on secondaries

// Step 2: Ensure secondaries are fully caught up before stepping down
rs.printReplicationInfo()
// Check "configured oplog size" and "log length"

// Step 3: Step down the primary gracefully
// This forces an election and prevents the node from being re-elected for 60 seconds
rs.stepDown(60)  // 60-second stepdown period (default: 60s)
// MongoDB 4.2+: stepDown waits up to stepDownSecs for a secondary to catch up

// Step 4: Verify the new primary
rs.status()
// Look for "stateStr": "PRIMARY" on a different node

// Step 5: Now safe to perform maintenance on the old primary (now secondary)
// Maintenance on secondary:
db.adminCommand({ shutdown: 1, timeoutSecs: 30 })
// or via mongosh:
// mongosh --eval "db.adminCommand({ shutdown: 1 })"

// Step 6: After maintenance, start mongod — it rejoins as secondary automatically
// Monitor catchup:
rs.printSecondaryReplicationInfo()
```

**Driver behavior during election:**

```javascript
// The MongoDB driver handles elections transparently
// Connection string with retryWrites=true (default in MongoDB 4.2+ drivers):
const client = new MongoClient(
  "mongodb://node1:27017,node2:27017,node3:27017/?replicaSet=myRS&retryWrites=true",
  {
    serverSelectionTimeoutMS: 30000,  // wait up to 30s for new primary
    heartbeatFrequencyMS: 10000,       // check server health every 10s
  }
)

// During election (typically 10-30 seconds):
// 1. Driver detects primary is gone (heartbeat timeout)
// 2. Driver marks all nodes as unknown
// 3. Driver rediscovers topology via remaining nodes
// 4. Driver routes writes to new primary
// 5. retryWrites=true: one failed write is automatically retried on new primary

// What the application sees:
// - With retryWrites=true: single transient error (if any), then normal operation
// - Without retryWrites: application gets a NotWritablePrimary error and must handle retry
```

**Key considerations:**
- Election takes ~10-30 seconds in a healthy cluster
- Applications with `retryWrites=true` experience at most one write failure
- `serverSelectionTimeoutMS: 30000` gives time for election to complete

---

## Scenario 2: Replication Lag Spike in Production

Your monitoring shows replication lag on secondary nodes suddenly spiked to 45 seconds during a batch import job. Primary is getting write-heavy. Describe your diagnosis and remediation approach.

### Diagnosis

```javascript
// Step 1: Check current replication lag
rs.printSecondaryReplicationInfo()
// Output shows something like:
// source: secondary1:27017
//   syncedTo: Wed Mar 01 2026 09:00:00 GMT+0000
//   0 secs (0 hrs) behind the primary

// Step 2: Check oplog window
rs.printReplicationInfo()
// Output:
// configured oplog size:   10240MB
// log length start to end: 3600 secs (1 hrs)
// oplog first event time:  Wed Mar 01 2026 08:00:00 GMT+0000
// oplog last event time:   Wed Mar 01 2026 09:00:00 GMT+0000
// now:                     Wed Mar 01 2026 09:00:45 GMT+0000
// CONCERN: If lag > oplog window, secondary needs full resync!

// Step 3: Check what's happening on the primary
db.adminCommand({ currentOp: 1, active: true })
// Look for long-running operations

// Step 4: Check secondary's replication status
db.adminCommand({ serverStatus: 1 }).repl
// Check replication metrics

// Step 5: On the secondary, check replication executor
db.adminCommand({ replSetGetStatus: 1 })
// Check "optimesDate" and "lastHeartbeatMessage" for each member
```

### Root Cause Analysis

```javascript
// Common causes of replication lag:

// 1. Bulk writes flooding the oplog
// The batch import creates massive oplog volume
// Secondary cannot apply oplog entries fast enough

// 2. Check secondary's CPU/disk I/O
// If secondary is disk-bound, it processes oplog slowly

// 3. Replication thread bottleneck
// Single-threaded replication (pre-4.0) or thread pool saturation

// Step 6: Check batch job and throttle it
// Instead of inserting 100k docs at once:
async function throttledBatchInsert(documents, batchSize = 1000) {
  for (let i = 0; i < documents.length; i += batchSize) {
    const batch = documents.slice(i, i + batchSize)
    await db.collection('data').insertMany(batch, { ordered: false })

    // Check replication lag before next batch
    const status = await db.admin().command({ replSetGetStatus: 1 })
    const maxLag = Math.max(...status.members
      .filter(m => m.state === 2)  // secondaries
      .map(m => Math.abs(
        new Date(status.members.find(p => p.state === 1).optimeDate) -
        new Date(m.optimeDate)
      ) / 1000)
    )

    if (maxLag > 10) {
      console.log(`Replication lag: ${maxLag}s — throttling...`)
      await new Promise(resolve => setTimeout(resolve, 2000))  // pause 2s
    }
  }
}
```

### Remediation

```javascript
// Option 1: Pause the batch job during peak hours

// Option 2: Run batch job with w:1 (don't wait for secondary acknowledgment)
// but RISK: if primary fails, unacknowledged writes may be rolled back
await db.collection('data').insertMany(docs, {
  writeConcern: { w: 1 }  // only primary acknowledges
})

// Option 3: Increase oplog size to prevent "oplog window exceeded" situation
// (only works if lag hasn't exceeded oplog window yet)
// In MongoDB 4.0+:
db.adminCommand({
  replSetResizeOplog: 1,
  size: 20480  // 20GB in MB
})

// Option 4: Check secondary hardware
// If secondary disk is slower than primary, consider upgrading or
// using priority: 0 for analytics secondaries to prevent them from
// being elected primary
rs.reconfig({
  _id: "myRS",
  members: [
    { _id: 0, host: "primary:27017", priority: 2 },
    { _id: 1, host: "secondary1:27017", priority: 1 },
    { _id: 2, host: "secondary2:27017", priority: 0 }  // never elect as primary
  ]
})
```

---

## Scenario 3: Shard Key Design for Multi-Tenant SaaS

You are building a SaaS application with 10,000 tenants. Each tenant has their own database-equivalent isolation but you're using a single MongoDB cluster with a `tenantId` field. You need to shard the main `events` collection (expected 5TB, 50M writes/day). Design the shard key.

### Analysis

```javascript
// Collection structure:
// {
//   _id: ObjectId,
//   tenantId: "tenant_abc123",      ← high cardinality (10,000 values)
//   userId: "user_xyz",
//   eventType: "page_view",
//   timestamp: ISODate("2026-03-01T..."),
//   payload: { ... }
// }

// Query patterns:
// 1. Most common: { tenantId: X, timestamp: { $gte: Y, $lte: Z } }
// 2. Analytics: { tenantId: X, eventType: "purchase" }
// 3. Admin: { tenantId: X } — get all events for a tenant
// 4. RARE: { eventType: "error" } — cross-tenant analytics (scatter-gather OK)

// Option A: Single field { tenantId: 1 }
// Problem: If one tenant has 80% of data → hot shard!
// Only 10,000 unique values → limited parallelism

// Option B: { timestamp: 1 } hashed
// Problem: Monotonic → write hotspot on current chunk
// Range queries by tenantId become scatter-gather

// Option C: Compound { tenantId: 1, timestamp: 1 } ← BEST CHOICE
// Reasons:
// - Targeted queries on tenantId filter (single shard or small subset)
// - Time range queries within tenant are efficient
// - Chunks split naturally as timestamp increases within each tenant
// - No monotonic hotspot problem (tenantId prefix varies per write)

// BUT: large tenants may still create hot chunks
// Solution: Add a bucket component for large tenants

// Option D: { tenantId: 1, _id: "hashed" }
// Good distribution but loses time-range efficiency

// Final recommendation: Compound { tenantId: 1, timestamp: 1 }
sh.shardCollection("saas.events", { tenantId: 1, timestamp: 1 })
```

### Implementation

```javascript
// Enable sharding on the database
sh.enableSharding("saas")

// Create the shard key index first (required before sharding)
db.events.createIndex({ tenantId: 1, timestamp: 1 })

// Shard the collection
sh.shardCollection("saas.events", { tenantId: 1, timestamp: 1 })

// For large tenants, implement zone sharding to dedicate shards
// Step 1: Tag shards
sh.addShardTag("shard01", "large-tenant-zone")
sh.addShardTag("shard02", "large-tenant-zone")
sh.addShardTag("shard03", "small-tenant-zone")
sh.addShardTag("shard04", "small-tenant-zone")

// Step 2: Assign zones for large tenants
// Large tenants (top 10 by volume) get dedicated shards
const largeTenants = ["enterprise_001", "enterprise_002", "enterprise_003"]
largeTenants.forEach(tenantId => {
  sh.addTagRange(
    "saas.events",
    { tenantId: tenantId, timestamp: MinKey },  // start of range
    { tenantId: tenantId, timestamp: MaxKey },  // end of range
    "large-tenant-zone"
  )
})

// Step 3: All other tenants route to small-tenant-zone shards
// (default: no zone = balanced across all shards, or assign a default zone)

// Application query (automatically targeted):
db.events.find(
  { tenantId: "enterprise_001", timestamp: { $gte: ISODate("2026-03-01") } }
).explain()
// Should show IXSCAN + shards: ["shard01"] or ["shard02"]
// NOT a scatter-gather across all shards
```

---

## Scenario 4: Recovering from a Split-Brain Scenario

Your 3-node replica set experiences a network partition. Node A (was primary) is partitioned away from Nodes B and C. Node A continues accepting writes for 30 seconds before being shut down by ops. Node B is elected new primary. When network is restored, how do you reconcile?

### Understanding What Happened

```javascript
// TIMELINE:
// T=0:   A=PRIMARY, B=SECONDARY, C=SECONDARY
// T=10s: Network partition: A isolated from B+C
// T=10s: A still thinks it's primary (has not seen enough heartbeat failures yet)
//        B+C timeout on A's heartbeat
// T=20s: B+C elect B as new primary (2/3 votes = majority)
//        A is still acting as stale primary (no majority acknowledgment)
// T=30s: Ops shuts down A
// T=60s: Network partition healed, A restarted

// KEY: A could only accept writes if a CLIENT sent them with w:1
// With w:majority, A's writes would have failed (couldn't reach majority)
// So 30 seconds of w:1 writes on A are now ORPHANED
```

### Rollback Process

```javascript
// When A reconnects to the replica set, MongoDB automatically handles rollback:

// Step 1: A discovers B is the new primary (higher optime on B)
// Step 2: A identifies divergence point (where oplogs diverge)
// Step 3: MongoDB rolls back A's writes that were never replicated to majority
//         Rolled-back documents are saved to the rollback directory:
//         /data/db/rollback/

// Check rollback directory on Node A:
// ls /data/db/rollback/
// saas.events.2026-03-17T09-15-00.0.bson
// This BSON file contains the rolled-back documents

// Step 4: Review what was rolled back
mongorestore --dryRun /data/db/rollback/saas.events.2026-03-17T09-15-00.0.bson

// Step 5: Decide whether to re-apply rolled-back writes
// Option A: If these were non-critical (page view events), discard
// Option B: If critical (payment records), manually re-insert
const rollbackDocs = require('/data/db/rollback/parsed-rollback.json')
// Re-insert with idempotency check:
for (const doc of rollbackDocs) {
  await db.events.updateOne(
    { _id: doc._id },
    { $setOnInsert: doc },
    { upsert: true }
  )
  // $setOnInsert + upsert: only inserts if _id doesn't exist
  // If B already has this document, no-op
}
```

### Prevention

```javascript
// Prevent split-brain write acceptance with proper write concern:
// w:majority means A's writes FAIL during partition (no majority)
// This is the correct production configuration

const client = new MongoClient(connectionString, {
  writeConcern: {
    w: "majority",
    j: true,          // journaled
    wtimeout: 5000    // fail fast if majority not available
  }
})

// For the replica set config, set a high priority on the intended primary:
rs.reconfig({
  _id: "myRS",
  members: [
    { _id: 0, host: "nodeA:27017", priority: 10 },  // preferred primary
    { _id: 1, host: "nodeB:27017", priority: 1 },
    { _id: 2, host: "nodeC:27017", priority: 1 }
  ]
})
```

---

## Scenario 5: Migrating a Collection to a New Shard Key (Resharding)

Your `orders` collection was sharded on `{ customerId: 1 }` two years ago. Now you have 50 "whale" customers causing hot shards. The query pattern has shifted to primarily range queries on `{ region: 1, createdAt: 1 }`. Plan the live resharding in MongoDB 5.0+.

### Pre-Resharding Assessment

```javascript
// Step 1: Analyze current distribution
db.adminCommand({ listShards: 1 })
sh.status()

// Check chunk distribution:
use config
db.chunks.aggregate([
  { $match: { ns: "mydb.orders" } },
  { $group: { _id: "$shard", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
// Output might show:
// { _id: "shard01", count: 4521 }  ← hot shard (whale customers)
// { _id: "shard02", count: 312 }
// { _id: "shard03", count: 198 }

// Step 2: Verify new shard key is suitable
// { region: 1, createdAt: 1 }
// region has 8 distinct values (US, EU, APAC, etc.) — low cardinality alone
// But compound with createdAt (timestamps) → good distribution
// createdAt is monotonic BUT region prefix distributes across shards

// Step 3: Check index exists (or will be created automatically)
db.orders.getIndexes()
// MongoDB 5.0 resharding creates the index automatically
```

### Resharding Process

```javascript
// MongoDB 5.0+ live resharding (online, no downtime!)
db.adminCommand({
  reshardCollection: "mydb.orders",
  key: { region: 1, createdAt: 1 }
})

// Monitor resharding progress:
db.adminCommand({ currentOp: 1, type: "op", ns: "mydb.orders" })
// Output includes:
// {
//   "type": "op",
//   "desc": "ReshardingRecipientService ...",
//   "totalApproxBytesToCopy": 524288000,
//   "bytesCopied": 104857600,
//   "remainingOperationTimeEstimated": { "seconds": 1800 }
// }

// Resharding phases:
// Phase 1: Clone — copy documents to new shard layout (parallel to live writes)
// Phase 2: Catch-up — apply oplog changes that occurred during cloning
// Phase 3: Commit — brief write block (seconds), then switch to new layout
// Phase 4: Cleanup — old chunks deleted from original shards

// During resharding:
// - Reads: fully available
// - Writes: available (slightly slower due to dual-write to old+new layout)
// - No application code changes needed
// - Connection string unchanged
```

### Post-Resharding Verification

```javascript
// Verify new distribution
sh.status()
// Should show even distribution across shards

// Verify queries are now targeted
db.orders.find(
  { region: "US", createdAt: { $gte: ISODate("2026-01-01") } }
).explain("executionStats")
// "shards" should show 1-2 shards, not all shards

// Verify index was created
db.orders.getIndexes()
// { key: { region: 1, createdAt: 1 }, name: "region_1_createdAt_1" }

// Optional: pre-split chunks for the new key to avoid initial hotspot
// (MongoDB 5.0 does this automatically for hashed keys, manual for ranged)
```

---

## Scenario 6: Read Scaling with Analytics Workloads

Your 3-node replica set runs OLTP on the primary. An analytics team runs complex aggregations that take 5-30 minutes and consume 100% CPU. This is impacting OLTP latency. Design a solution.

### Solution: Dedicated Analytics Secondary

```javascript
// Step 1: Add a 4th node as a dedicated analytics secondary
rs.add({
  host: "analytics-node:27017",
  priority: 0,      // never elected primary
  hidden: true,     // invisible to normal read preference routing
  votes: 0,         // doesn't participate in elections
  tags: { "nodeType": "analytics" }
})

// Step 2: Verify the configuration
rs.conf()
// analytics-node shows: priority: 0, hidden: true, votes: 0

// Wait for sync to complete:
rs.status()
// analytics-node shows "stateStr": "SECONDARY", "health": 1

// Step 3: Analytics team connects with specific read preference
// Connection string pointing to analytics node:
const analyticsClient = new MongoClient(connectionString, {
  readPreference: ReadPreference.SECONDARY,
  readPreferenceTags: [{ nodeType: "analytics" }]
})

// Or using mongosh:
db.getMongo().setReadPref("secondary", [{ nodeType: "analytics" }])

// Step 4: Run heavy aggregation on analytics node
const result = await analyticsClient.db("mydb")
  .collection("orders")
  .aggregate([
    { $match: { createdAt: { $gte: ISODate("2026-01-01") } } },
    { $group: {
        _id: { region: "$region", month: { $month: "$createdAt" } },
        totalRevenue: { $sum: "$amount" },
        orderCount: { $count: {} }
    }},
    { $sort: { totalRevenue: -1 } }
  ], {
    allowDiskUse: true,     // allow spilling to disk for large sorts
    maxTimeMS: 1800000      // 30 minute timeout
  })
  .toArray()
```

### Atlas Analytics Nodes (Atlas-specific)

```javascript
// In Atlas, use dedicated Analytics Nodes (no self-managed equivalent)
// Atlas UI: Cluster → Edit → Analytics Nodes → Add Analytics Node

// Analytics Node connection string (Atlas provides separate endpoint):
// mongodb+srv://cluster0-analytics.mongodb.net/mydb

// Benefits:
// - Physically separate compute from OLTP nodes
// - Dedicated RAM and CPU for analytics
// - Data is replicated but analytics node never becomes primary
// - OLTP latency is completely unaffected

// Atlas Data Federation as alternative:
// For very heavy analytics, push data to Atlas Online Archive
// Run analytics against archived data (S3) via Data Federation
// Zero impact on live cluster
const pipeline = [
  { $search: { /* ... */ } },
  // This runs against S3 data, not the live cluster
]
```

---

## Scenario 7: Chunk Imbalance After Shard Addition

You added a new shard (`shard04`) to your 3-shard cluster. After 24 hours, `shard04` still has very few chunks while `shard01` has 40% of all chunks. Diagnose and fix.

### Diagnosis

```javascript
// Step 1: Check balancer status
sh.getBalancerState()
// false ← balancer is OFF! This is the problem.

// Or balancer is on but there's a window configured:
use config
db.settings.findOne({ _id: "balancer" })
// {
//   "_id": "balancer",
//   "stopped": false,
//   "activeWindow": {
//     "start": "02:00",
//     "stop": "04:00"   ← only runs 2-4 AM
//   }
// }
// Current time is 10 AM → balancer not running

// Step 2: Check if balancer has been running
use config
db.changelog.find({ what: "moveChunk.from" }).sort({ time: -1 }).limit(10)
// Check if any recent chunk migrations occurred

// Step 3: Check for jumbo chunks (chunks that can't be split/moved)
use config
db.chunks.find({ ns: "mydb.orders", jumbo: true })
// Jumbo chunks are a common cause of imbalance
```

### Fix Imbalance

```javascript
// Fix 1: Enable balancer (if it was off)
sh.startBalancer()
sh.getBalancerState()  // should return true

// Fix 2: Remove time window restriction (if 24/7 balancing is acceptable)
use config
db.settings.updateOne(
  { _id: "balancer" },
  { $unset: { activeWindow: "" } }
)

// Fix 3: Manually trigger balancing (MongoDB 6.0+)
db.adminCommand({ balancerStart: 1 })

// Fix 4: Handle jumbo chunks
// Jumbo chunks exist because shard key has too few unique values in that range
// Fix: If using ranged sharding, manually split the jumbo chunk:
sh.splitAt("mydb.orders", { customerId: "whale_customer_50000" })
// This creates two chunks at that exact key value

// After splitting, clear jumbo flag:
use config
db.chunks.updateOne(
  { ns: "mydb.orders", min: { customerId: "whale_customer_49999" } },
  { $unset: { jumbo: "" } }
)
// MongoDB 4.2+: jumbo flag is cleared automatically after successful split

// Fix 5: Monitor balancing progress
while (true) {
  const balancerRunning = db.adminCommand({ balancerStatus: 1 }).inBalancerRound
  const chunkCounts = db.getSiblingDB("config").chunks.aggregate([
    { $match: { ns: "mydb.orders" } },
    { $group: { _id: "$shard", count: { $sum: 1 } } }
  ]).toArray()
  print(JSON.stringify({ balancerRunning, chunkCounts }))
  sleep(30000)  // check every 30s
}
```

---

## Scenario 8: Cross-Shard Transaction Performance Problem

Your application uses multi-document transactions that span multiple shards. You're seeing 3-5 second transaction latencies that are unacceptable (target: <100ms). Diagnose and fix.

### Diagnosis

```javascript
// Step 1: Confirm cross-shard transaction overhead
db.adminCommand({ currentOp: 1, $all: true, op: "command" })
// Look for "transaction" type operations

// Step 2: Explain the slow transaction's queries
// (run outside transaction for explain plan):
db.orders.find({ customerId: "cust_123" }).explain("executionStats")
// Check if query is scatter-gather across shards

// Step 3: Enable slow query logging
db.adminCommand({ setParameter: 1, slowOpThresholdMs: 100 })
// Check mongos logs for slow transactions
```

### Root Cause: Co-location Problem

```javascript
// The transaction touches these collections:
// orders (sharded on customerId)
// orderItems (sharded on orderId — DIFFERENT shard key!)
// payments (sharded on paymentId — DIFFERENT shard key!)

// When customer "cust_123" places order "order_456":
// - order document: on shard02 (customerId hashes to shard02)
// - orderItems: on shard01 (orderId hashes to shard01)
// - payment: on shard03 (paymentId hashes to shard03)
// → 2PC across 3 shards = massive latency

// FIX: Co-locate related data on the same shard
// Use the SAME shard key prefix for related collections

// orders collection: shard key { customerId: 1, _id: 1 }
sh.shardCollection("mydb.orders", { customerId: 1, _id: 1 })

// orderItems collection: shard key { customerId: 1, orderId: 1 }
// customerId prefix ensures items are on same shard as their order
sh.shardCollection("mydb.orderItems", { customerId: 1, orderId: 1 })

// payments collection: shard key { customerId: 1, _id: 1 }
sh.shardCollection("mydb.payments", { customerId: 1, _id: 1 })

// Now: all documents for customerId "cust_123" are on the SAME shard
// Transaction becomes single-shard → no 2PC overhead → <5ms
```

### Alternative: Embed to Avoid Cross-Collection Transactions

```javascript
// For high-frequency transactions, consider embedding instead of referencing
// Instead of separate collections:
const orderWithItems = {
  _id: ObjectId(),
  customerId: "cust_123",
  createdAt: new Date(),
  status: "pending",
  items: [                    // embedded, not separate collection
    { productId: "p1", qty: 2, price: 29.99 },
    { productId: "p2", qty: 1, price: 49.99 }
  ],
  payment: {                  // embedded, not separate collection
    method: "card",
    last4: "4242",
    amount: 109.97,
    status: "captured"
  }
}

// Single-document write = atomic by default
// No transaction needed at all → fastest possible writes
await db.orders.insertOne(orderWithItems)
// writeConcern: majority → durable, atomic, no transaction overhead
```

---

## Scenario 9: Oplog Exhaustion — Secondary Needs Full Resync

Your secondary's replication lag has exceeded the oplog window (the primary's oplog only covers the last 2 hours, but the secondary is 3 hours behind). The secondary enters RECOVERING state. Handle this without extended downtime.

### Understanding the Problem

```javascript
// On the primary, check oplog window:
rs.printReplicationInfo()
// log length start to end: 7200 secs (2 hrs)  ← oplog covers only 2 hours
// now: 2026-03-17T12:00:00

// On the secondary:
rs.printSecondaryReplicationInfo()
// source: secondary1:27017
//   syncedTo: 2026-03-17T09:00:00  ← 3 hours behind
//   STALE: secondary is not in sync and is not able to catch up

// secondary1 shows state: RECOVERING (not SECONDARY)
// It cannot apply oplog entries because the entries it needs are gone
```

### Resolution: Initial Sync

```javascript
// Option A: Full initial sync (automatic when MongoDB detects STALE state)
// MongoDB will automatically start initial sync on the stale secondary
// Monitor on the secondary:
rs.status()
// { "stateStr": "STARTUP2", "infoMessage": "initial sync" }

// Monitor initial sync progress (MongoDB 4.4+):
db.adminCommand({ replSetGetStatus: 1 }).initialSyncStatus
// {
//   "failedInitialSyncAttempts": 0,
//   "maxFailedInitialSyncAttempts": 10,
//   "initialSyncStart": ISODate("..."),
//   "initialSyncElapsedMillis": 45000,
//   "remainingInitialSyncEstimatedMillis": 120000
// }

// Option B: Seed from a backup (faster for large datasets)
// 1. Take a backup from primary or another secondary
mongodump --host primary:27017 --out /backup/seed/

// 2. Stop mongod on the stale secondary
// 3. Clear the data directory
rm -rf /data/db/*

// 4. Restore the backup
mongorestore --host localhost:27017 /backup/seed/

// 5. Start mongod — it reconnects and applies only the recent oplog
// Much faster than full initial sync for terabyte-scale databases

// Option C: Rsync from another secondary (for very large datasets)
// 1. Stop mongod on source secondary
// 2. Rsync data files to stale secondary
rsync -avz --progress secondary2:/data/db/ /data/db/
// 3. Start mongod on stale secondary — applies oplog delta only

// Prevention: Increase oplog size before this happens
db.adminCommand({
  replSetResizeOplog: 1,
  size: 51200  // 50GB — large enough for 48+ hours of typical load
})
```

---

## Scenario 10: Global Deployment — Minimizing Cross-Region Write Latency

Your application serves users in North America, Europe, and Asia Pacific. All writes currently go to a primary in us-east-1, causing 200ms+ latency for EU/APAC users. Design a low-latency write architecture.

### Option A: Atlas Global Clusters (Zone Sharding)

```javascript
// Architecture:
// Zone "americas": shards in us-east-1 + us-west-2
// Zone "europe": shards in eu-west-1 + eu-central-1
// Zone "apac": shards in ap-southeast-1 + ap-northeast-1

// Shard key: { region: 1, _id: 1 }
// region = "US" | "EU" | "APAC" — determined at write time

// Zone configuration:
sh.addShardTag("shard-us-east", "americas")
sh.addShardTag("shard-us-west", "americas")
sh.addShardTag("shard-eu-west", "europe")
sh.addShardTag("shard-eu-central", "europe")
sh.addShardTag("shard-ap-sea", "apac")
sh.addShardTag("shard-ap-ne", "apac")

sh.addTagRange("mydb.users",
  { region: "US", _id: MinKey },
  { region: "US", _id: MaxKey },
  "americas"
)
sh.addTagRange("mydb.users",
  { region: "EU", _id: MinKey },
  { region: "EU", _id: MaxKey },
  "europe"
)
sh.addTagRange("mydb.users",
  { region: "APAC", _id: MinKey },
  { region: "APAC", _id: MaxKey },
  "apac"
)

// Result: EU user writes go to EU shard primary (<10ms local write)
//         US user writes go to US shard primary (<10ms local write)
//         Cross-region replication happens async (eventual consistency)
```

### Application-Level Routing

```javascript
// Determine region from user context
function getUserRegion(userIp) {
  // GeoIP lookup or user profile
  if (userIp.startsWith('52.') || userIp.startsWith('54.')) return 'US'
  if (userIp.startsWith('176.') || userIp.startsWith('178.')) return 'EU'
  return 'APAC'
}

// Write user data with region field
async function createUser(userData, userIp) {
  const region = getUserRegion(userIp)
  const user = {
    ...userData,
    region: region,  // REQUIRED for zone-based shard key
    createdAt: new Date()
  }

  // mongos routes to correct zone based on shard key
  await db.users.insertOne(user)
  // EU user → EU shard primary → <10ms write latency
}

// For reads: use nearest read preference
const client = new MongoClient(connectionString, {
  readPreference: "nearest"  // routes to geographically closest replica
})
// EU user reading → EU shard secondary → <5ms read latency
```

### Tradeoffs and Limitations

```javascript
// Limitation: Documents are permanently bound to their zone
// A user who moves from EU to US is still in the EU zone
// Solution: Planned zone migration (offline process)

// Cross-region queries (accessing data in multiple zones):
// { } — no shard key filter → scatter-gather across all zones
// Accept this for admin queries, not for hot paths

// Multi-region write concern:
// w: "majority" applies per-shard (within zone)
// Cross-region replication is async → EU can be recovered independently
// Use "majority" for durability within zone
await db.users.insertOne(user, {
  writeConcern: { w: "majority", j: true }
  // Majority of EU shard nodes acknowledge → durable in EU
  // APAC/US replication follows async
})
```
