# Chapter 09 — Replication & Sharding: Structured Answers

---

## Answer 1: Explain MongoDB Replica Set Architecture

**PREP Framework:**

**Point:** A replica set is a group of mongod processes that maintain identical data sets. It provides automatic failover and data redundancy.

**Reason:** Single mongod has no fault tolerance — one hardware failure means downtime. Replica sets keep multiple copies of data on different nodes, enabling automatic failover in 10-30 seconds.

**Example:**

```javascript
// 3-node replica set initialization:
rs.initiate({
  _id: "myRS",
  members: [
    { _id: 0, host: "node1:27017", priority: 2 },  // preferred primary
    { _id: 1, host: "node2:27017", priority: 1 },
    { _id: 2, host: "node3:27017", priority: 1 }
  ]
})

// All writes go to primary node1
// node2 and node3 tail the oplog and apply changes asynchronously
// Oplog entry structure:
{
  "ts": Timestamp(1710000000, 1),   // timestamp + sequence
  "t": NumberLong(5),               // term number (Raft)
  "h": NumberLong(-1234567890),     // unique hash
  "v": 2,                           // version
  "op": "i",                        // operation: i=insert, u=update, d=delete, c=command
  "ns": "mydb.orders",              // namespace
  "o": { _id: ObjectId("..."), customerId: "cust_123", amount: 49.99 }
}

// Monitor replica set health:
rs.status()    // overall status, member states
rs.conf()      // configuration
rs.printReplicationInfo()       // oplog window on primary
rs.printSecondaryReplicationInfo()  // lag on secondaries
```

**Proof:** A 3-node PSS configuration tolerates one node failure. With `w: "majority"` write concern, even if the primary fails after acknowledging writes, those writes have been committed on at least one secondary and will survive the election.

---

## Answer 2: How Does Replica Set Election Work?

**Point:** Elections use a Raft-based consensus algorithm. A secondary calls an election when it hasn't heard from the primary within `electionTimeoutMS` (default 10 seconds). The candidate with the highest priority and most up-to-date oplog wins the majority of votes.

**Reason:** Automatic failover requires a deterministic leader selection algorithm that prevents split-brain scenarios. Raft's majority-vote requirement ensures only one primary at a time.

**Example:**

```javascript
// Election trigger conditions:
// 1. Primary crashes or becomes unreachable
// 2. rs.stepDown() called manually
// 3. Priority change causes a higher-priority node to force re-election

// Election timeline:
// T=0:   Primary becomes unreachable
// T=10s: Secondaries timeout (electionTimeoutMS: 10000)
// T=10s: One secondary calls election, votes for itself
// T=11s: Other secondary receives vote request
//        Checks: is candidate's optime >= my optime? YES
//        Grants vote (a node can only grant one vote per term)
// T=12s: Candidate has 2 votes (majority of 3) → becomes primary
// T=30s: Application reconnects to new primary (driver server monitoring)

// Configure election timeout:
rs.reconfig({
  _id: "myRS",
  settings: {
    electionTimeoutMS: 10000,    // default, good for most deployments
    heartbeatTimeoutSecs: 10,    // how long to wait for heartbeat response
    heartbeatIntervalMillis: 2000  // heartbeat frequency
  },
  members: [ /* ... */ ]
})

// Force a stepdown manually (for maintenance):
rs.stepDown(60)  // don't re-elect this node for 60 seconds

// Check current term (Raft term increments with each election):
rs.status().term  // e.g., 5 means 5 elections have occurred
```

**Proof:** The majority requirement (need 2 of 3 votes) means elections only succeed when more than half the cluster agrees — preventing two primaries from existing simultaneously. A 5-node cluster tolerates 2 simultaneous failures while still electing a primary.

---

## Answer 3: How Do You Choose a Shard Key?

**Point:** The shard key determines how MongoDB distributes data and routes queries. A good shard key has high cardinality, avoids monotonically increasing values (for write distribution), and matches your dominant query pattern.

**Reason:** The shard key is immutable after sharding (pre-MongoDB 5.0) and extremely costly to change. A bad shard key causes hot shards (write bottlenecks), scatter-gather queries (read bottlenecks), or jumbo chunks (balancing failures).

**Example:**

```javascript
// BAD shard keys and why:

// 1. { _id: 1 } with ObjectId — monotonically increasing
// All new inserts go to the same "right-edge" chunk → hot shard!
// Even though ObjectId has high cardinality

// 2. { status: 1 } — low cardinality (only a few distinct values)
// Maximum one chunk per unique status value
// Cannot split chunks beyond unique value count

// 3. { email: 1 } — high cardinality but range queries won't benefit
// "Find all users in domain example.com" → scatter-gather

// GOOD shard keys:

// For IoT sensor data (time-series):
sh.shardCollection("iot.readings", { sensorId: 1, timestamp: 1 })
// sensorId distributes across shards
// timestamp enables range queries within a sensor

// For multi-tenant SaaS:
sh.shardCollection("saas.events", { tenantId: 1, _id: 1 })
// tenantId routes all tenant data to same shard (co-location)
// _id provides uniqueness for chunk splitting

// For social media posts (even write distribution + recency queries):
sh.shardCollection("social.posts", { authorId: "hashed" })
// Hashed: even distribution, no hotspot
// Tradeoff: range queries on authorId are scatter-gather
// Acceptable if most queries are by specific authorId (point lookups)

// Shard key decision matrix:
// High write throughput?        → hashed or compound with low-entropy prefix
// Range queries by field X?     → ranged, X as first component
// Multi-tenant isolation?       → tenantId as prefix
// Geographic routing?           → region as prefix
// Transactions across docs?     → common field as prefix (co-location)
```

**Proof:** The New York Times uses `{ articleId: 1 }` for their articles collection — high cardinality, matches primary query pattern (fetch by articleId), and articles don't need range queries across all articles.

---

## Answer 4: Explain Chunk Balancing in Sharded Clusters

**Point:** The balancer is a background process on config servers that monitors chunk counts per shard and migrates chunks from over-populated shards to under-populated ones. It kicks in when the difference exceeds a threshold.

**Reason:** As data grows and new shards are added, chunks become unevenly distributed. Without balancing, some shards become "hot" (overloaded) while others sit idle, defeating the purpose of sharding.

**Example:**

```javascript
// Check balancer state:
sh.getBalancerState()  // true/false
sh.isBalancerRunning() // whether it's currently migrating a chunk

// Balancer threshold (when it triggers):
// 3 shards: triggers at imbalance of 8+ chunks
// 10+ shards: triggers at imbalance of 4+ chunks
// Config: db.getSiblingDB("config").settings.findOne({ _id: "balancer" })

// Configure balancer window (run during low-traffic hours):
use config
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "01:00", stop: "05:00" } } },
  { upsert: true }
)

// Manual chunk operations:
// Split a specific chunk at a point:
sh.splitAt("mydb.orders", { customerId: "cust_50000" })

// Move a chunk manually:
db.adminCommand({
  moveChunk: "mydb.orders",
  find: { customerId: "cust_50000" },
  to: "shard03"
})

// Diagnose jumbo chunks:
use config
db.chunks.find({ ns: "mydb.orders", jumbo: true }).count()
// Jumbo chunks can't be moved by balancer
// Fix: split them manually first, then balancer can migrate
sh.splitFind("mydb.orders", { customerId: "whale_customer_001" })

// View current chunk distribution:
db.chunks.aggregate([
  { $match: { ns: "mydb.orders" } },
  { $group: { _id: "$shard", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

**Proof:** Without the balancer, adding a new shard would leave it empty while existing shards remain overloaded. The balancer moves approximately 1 chunk per round (configurable) to gradually rebalance without overwhelming network and disk I/O.

---

## Answer 5: Describe the Impact of Read Preferences on Replica Set Performance

**Point:** Read preferences route queries to different replica set members. The choice balances freshness of data, read throughput, and operational isolation.

**Reason:** Routing all reads to the primary creates a bottleneck. Secondary reads distribute load and allow operational separation (e.g., analytics on a dedicated node), but may return slightly stale data.

**Example:**

```javascript
// Read preference options and when to use each:

// 1. primary (default)
// Use: financial reads, user-visible writes-then-reads (profile updates)
db.accounts.findOne({ userId: "u1" }, { readPreference: "primary" })
// Guarantees latest committed data (with appropriate read concern)

// 2. primaryPreferred
// Use: mostly fresh reads, but tolerate stale data if primary is down
// Good for: product catalog (slight staleness acceptable)
db.products.find({ category: "electronics" }, {
  readPreference: "primaryPreferred"
})

// 3. secondary
// Use: analytics, reporting, batch jobs that can tolerate staleness
db.orders.aggregate(
  [/* heavy aggregation */],
  { readPreference: "secondary" }
)

// 4. secondaryPreferred
// Use: read-heavy workloads where slight staleness is fine (social feed)
db.posts.find({ userId: { $in: followingList } }, {
  readPreference: "secondaryPreferred"
})

// 5. nearest
// Use: latency-sensitive, geo-distributed deployments
// Routes to lowest-latency node regardless of primary/secondary status
const client = new MongoClient(uri, {
  readPreference: "nearest",
  readPreferenceTags: [{ region: "us-east" }]  // prefer local region
})

// Tag-based routing (requires rs.conf() tag configuration):
rs.reconfig({
  _id: "myRS",
  members: [
    { _id: 0, host: "primary:27017", tags: { region: "us-east", type: "oltp" } },
    { _id: 1, host: "secondary1:27017", tags: { region: "us-east", type: "oltp" } },
    { _id: 2, host: "secondary2:27017", tags: { region: "us-east", type: "analytics" }, priority: 0 }
  ]
})
// Analytics queries route to secondary2:
db.events.aggregate(pipeline, {
  readPreference: ReadPreference.SECONDARY,
  readPreferenceTags: [{ type: "analytics" }]
})
```

**Proof:** MongoDB Atlas provides Analytics Nodes — dedicated secondaries with `priority: 0` and separate compute. Atlas routes `$$SEARCH` queries and BI Connector reads to these nodes, completely isolating analytics I/O from OLTP.

---

## Answer 6: How Do Transactions Work in a Sharded Cluster?

**Point:** Sharded cluster transactions use 2-Phase Commit (2PC) coordinated by the mongos. This adds latency but ensures atomicity across multiple shards.

**Reason:** When a transaction spans documents on different shards, a crash between "commit on shard1" and "commit on shard2" would leave data inconsistent. 2PC ensures either all shards commit or all abort.

**Example:**

```javascript
// Single-shard transaction (fast — no 2PC):
// customerId routes to shard02 for both collections
const session = client.startSession()
await session.withTransaction(async () => {
  await db.orders.insertOne(
    { _id: orderId, customerId: "cust_123", amount: 99.99 },
    { session }
  )
  await db.inventory.updateOne(
    { productId: "p1", customerId: "cust_123" },  // same shard key!
    { $inc: { reserved: 1 } },
    { session }
  )
  // Both ops on shard02 → no 2PC needed
})

// Cross-shard transaction (slow — uses 2PC):
await session.withTransaction(async () => {
  // order lands on shard02 (customerId: "cust_123")
  await db.orders.insertOne({ customerId: "cust_123", ... }, { session })
  // inventory lands on shard01 (productId: "p1" is shard key)
  await db.inventory.updateOne({ productId: "p1" }, { $inc: { stock: -1 } }, { session })
  // Different shard keys → different shards → 2PC required
})

// 2PC process:
// Phase 1 (Prepare):
//   mongos → shard01: "prepare" (shard01 writes prepare record, holds locks)
//   mongos → shard02: "prepare" (shard02 writes prepare record, holds locks)
//   Both respond: "ready"
// Phase 2 (Commit):
//   mongos → shard01: "commit" (shard01 applies changes, releases locks)
//   mongos → shard02: "commit" (shard02 applies changes, releases locks)
// If any shard says "abort" in Phase 1: entire transaction aborts

// Optimization: co-locate data to minimize cross-shard transactions
// Shard orders on { customerId: 1, _id: 1 }
// Shard orderItems on { customerId: 1, orderId: 1 }
// → Both collections for same customer → same shard → no 2PC
```

**Proof:** Benchmarks show cross-shard transactions are 3-10x slower than single-shard transactions due to 2PC overhead and distributed lock contention. Co-location via shared shard key prefix is the primary optimization strategy.

---

## Answer 7: How Do You Handle a Sharded Cluster Hot Shard?

**Point:** A hot shard occurs when one shard receives disproportionate read/write load. Causes: monotonic shard key (write hotspot), low-cardinality shard key, or uneven data distribution.

**Reason:** Sharding is only effective if load is distributed. A hot shard wastes your other shards and creates a performance bottleneck that scaling other shards cannot fix.

**Example:**

```javascript
// Diagnosis:
// Check per-shard operation counts (mongostat output):
// mongostat --discover
// shard01: insert: 4500/s  ← HOT
// shard02: insert: 100/s
// shard03: insert: 80/s

// Cause 1: Monotonic shard key (_id or timestamp)
// All new inserts hash to the same "latest" chunk

// Fix A: Switch to compound hashed shard key
// (requires resharding in MongoDB 5.0+)
db.adminCommand({
  reshardCollection: "mydb.events",
  key: { userId: 1, _id: "hashed" }
  // userId distributes by user, _id hashed prevents monotonic hotspot within user
})

// Fix B: Pre-split chunks for hashed sharding
// At sharding time, use numInitialChunks to pre-distribute:
sh.shardCollection(
  "mydb.events",
  { userId: "hashed" },
  false,  // unique
  { numInitialChunks: 100 }  // create 100 initial chunks across all shards
)
// Prevents all initial writes going to one chunk

// Cause 2: "Whale" customer (one customerId has 80% of data)
// Fix: Zone sharding — dedicate more shards to large customers
sh.addShardTag("shard01", "enterprise")
sh.addShardTag("shard02", "enterprise")
sh.addShardTag("shard03", "standard")

sh.addTagRange(
  "mydb.events",
  { customerId: "enterprise_bigcorp", _id: MinKey },
  { customerId: "enterprise_bigcorp", _id: MaxKey },
  "enterprise"
)

// Cause 3: Balancer not running
sh.startBalancer()
use config
db.settings.updateOne(
  { _id: "balancer" },
  { $unset: { activeWindow: "" } }  // remove time restriction
)
```

**Proof:** The monotonic hotspot is so common that MongoDB 4.2 changed ObjectId default behavior to reduce sequential patterns in cluster environments. MongoDB 5.0 introduced `numInitialChunks` for hashed sharding to pre-distribute immediately at collection creation.

---

## Answer 8: Explain Replication Lag and Its Impact on Applications

**Point:** Replication lag is the time difference between when a write is committed on the primary and when it's applied on a secondary. It impacts applications that use secondary reads, read concerns, and failover scenarios.

**Reason:** Lag means secondaries serve stale data. Excessive lag risks exceeding the oplog window — if a secondary falls too far behind, the oplog entries it needs may have been overwritten, requiring a full resync (hours of downtime).

**Example:**

```javascript
// Monitor replication lag:
rs.printSecondaryReplicationInfo()
// source: secondary1:27017
//   syncedTo: Wed Mar 17 2026 08:00:00 GMT+0000
//   10 secs (0 hrs) behind the primary  ← acceptable

// Programmatic lag check:
const status = db.adminCommand({ replSetGetStatus: 1 })
const primary = status.members.find(m => m.state === 1)
const secondaries = status.members.filter(m => m.state === 2)
secondaries.forEach(sec => {
  const lagMs = new Date(primary.optimeDate) - new Date(sec.optimeDate)
  console.log(`${sec.name}: ${lagMs/1000}s lag`)
})

// Impact on applications:
// 1. Secondary reads may return stale data
db.users.findOne({ userId: "u1" }, {
  readPreference: "secondary",
  readConcern: { level: "majority" }  // returns only majority-committed data
  // but still lag behind primary's latest commits
})

// 2. readConcern: "linearizable" forces primary + additional wait
//    → not practical for high-throughput reads

// 3. Causal consistency after-clusterTime:
//    Forces secondary to wait until it's caught up to a specific timestamp
//    before serving the read — effective but adds latency

// Causes and fixes:
// A. Network bandwidth saturation → upgrade network, compress replication traffic
// B. Secondary disk I/O bottleneck → upgrade secondary storage
// C. Heavy batch writes on primary → throttle batch jobs, add write delays
// D. Oplog too small → rs.printReplicationInfo() shows small window
//    Fix: db.adminCommand({ replSetResizeOplog: 1, size: 51200 })  // 50GB

// Alert threshold:
// Alert if lag > 60 seconds (typical oplog window: hours)
// Critical if lag > (oplog window - 10 minutes)
```

**Proof:** Netflix, which runs MongoDB at scale, implements application-level lag monitoring and automatically falls back to primary reads when secondary lag exceeds their staleness tolerance threshold for user-facing data.

---

## Answer 9: How Do You Plan a Shard Key for an E-Commerce Order System?

**Point:** Model the access patterns first, then choose the shard key. The e-commerce order system has mixed patterns — high-frequency lookups by customerId, time-range analytics, and cross-region operations.

**Reason:** E-commerce orders require: (1) fast customer order history (point lookups by customerId), (2) order line items co-located with orders (transaction safety), (3) time-series analytics (range queries on date), (4) geographic routing for global deployment.

**Example:**

```javascript
// Access patterns:
// 1. Customer views order history: { customerId: "c1" } — 80% of queries
// 2. Admin views orders by date: { createdAt: { $gte: ... } } — 5% of queries
// 3. Order detail + items: transaction touching orders + orderItems — 10%
// 4. Analytics by region/date: { region: "US", createdAt: {...} } — 5%

// CHOSEN shard key: { customerId: 1, _id: 1 }
// Reasons:
// - customerId prefix: customer queries are targeted (single shard)
// - _id suffix: prevents jumbo chunks for high-volume customers
// - All customer's orders on same shard → single-shard transactions
// - Cardinality: 1M+ customers × unique _ids → very high cardinality

sh.shardCollection("ecommerce.orders", { customerId: 1, _id: 1 })

// Co-locate orderItems:
sh.shardCollection("ecommerce.orderItems", { customerId: 1, orderId: 1 })
// customerId prefix → same shard as order

// Indexed together for range queries (not shard key, just index):
db.orders.createIndex({ createdAt: 1 })  // for analytics (scatter-gather OK)
db.orders.createIndex({ region: 1, createdAt: 1 })  // for regional analytics

// Transaction is now single-shard:
const session = client.startSession()
await session.withTransaction(async () => {
  await db.orders.insertOne({ customerId: "c1", _id: orderId, ... }, { session })
  await db.orderItems.insertMany(items.map(item => ({
    customerId: "c1",     // ← shard key prefix
    orderId: orderId,
    ...item
  })), { session })
  // Both insertions route to same shard → no 2PC
})
```

**Proof:** This is the recommended pattern from MongoDB's own e-commerce reference architecture. The key insight: the shard key must include the "parent" entity ID (customerId) for all "child" collections (orderItems, payments) to ensure co-location.

---

## Answer 10: How Do You Monitor and Maintain a Sharded Cluster?

**Point:** Sharded cluster monitoring covers: shard balance, chunk distribution, mongos health, balancer status, and per-shard replication health.

**Reason:** Sharded clusters have more failure modes than replica sets. A failed config server, stale mongos cache, or imbalanced shards can silently degrade performance without triggering obvious errors.

**Example:**

```javascript
// Daily health checks (automate these):

// 1. Overall shard status
sh.status()
// Shows: cluster status, databases with sharding enabled, chunk distribution

// 2. Balancer running
sh.getBalancerState()
db.adminCommand({ balancerStatus: 1 })
// { "mode": "full", "inBalancerRound": false, "numBalancerRounds": 42312 }

// 3. Per-shard stats
db.adminCommand({ listShards: 1 })
// Shows each shard's connection info and tags

// 4. Chunk counts per shard for each collection
use config
db.chunks.aggregate([
  { $group: { _id: { shard: "$shard", ns: "$ns" }, count: { $sum: 1 } } },
  { $sort: { "_id.ns": 1, count: -1 } }
])

// 5. Orphaned documents (left after failed chunk migration)
// MongoDB 4.4+ auto-cleans these, but check:
db.runCommand({ cleanupOrphaned: "mydb.orders", startingFromKey: MinKey })

// 6. Mongos cache staleness
// Mongos caches chunk routing table; force refresh if needed:
db.adminCommand({ flushRouterConfig: 1 })

// 7. Check for and clear jumbo chunks
use config
db.chunks.find({ jumbo: true }).forEach(chunk => {
  print(`Jumbo chunk: ${chunk.ns} on ${chunk.shard}: ${JSON.stringify(chunk.min)} → ${JSON.stringify(chunk.max)}`)
})

// Atlas monitoring dashboard:
// Atlas → Cluster → Metrics shows:
// - Operation counts per shard
// - Data size per shard
// - Chunk distribution visualization
// - Replication lag per shard

// Alert configuration (Atlas):
// - Replication lag > 60s: WARNING
// - Replication lag > 300s: CRITICAL
// - Shard imbalance > 30%: WARNING
// - Mongos connection pool exhaustion: CRITICAL
```

**Proof:** MongoDB Atlas provides built-in shard monitoring with automatic alerts for chunk imbalance, replication lag, and balancer failures — dramatically reducing the operational burden of sharded cluster management compared to self-managed deployments.
