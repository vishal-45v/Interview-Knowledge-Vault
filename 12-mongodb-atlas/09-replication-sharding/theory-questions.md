# Chapter 09 — Replication & Sharding: Theory Questions

---

## Q1: How does a MongoDB replica set work?

A **replica set** is a group of MongoDB instances (mongod processes) that maintain the same dataset through replication. One member is the **primary** (handles all writes), and the others are **secondaries** (replicate from the primary via the oplog).

```javascript
// Replica set architecture (3-node standard):
// Primary: accepts all reads (by default) and all writes
// Secondary 1: replicates from primary asynchronously
// Secondary 2: replicates from primary asynchronously

// Initiate a replica set (in mongosh on the primary node):
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017", priority: 2 },   // highest priority → prefers primary
    { _id: 1, host: "mongo2:27017", priority: 1 },
    { _id: 2, host: "mongo3:27017", priority: 1 }
  ]
})

// Check replica set status:
rs.status()
// Returns: stateStr ("PRIMARY" / "SECONDARY"), health (1/0), optime, lag

// How replication works:
// 1. Client writes to primary → primary stores in WiredTiger + writes to oplog
// 2. Secondaries tail the primary's oplog (using a tailable cursor)
// 3. Each secondary applies oplog entries in order
// 4. Secondaries maintain their own copy of the oplog

// Oplog entry structure:
use local
db.oplog.rs.findOne()
// {
//   ts:   Timestamp(1709289600, 1),   ← logical timestamp
//   t:    1,                           ← term number (Raft term)
//   v:    2,                           ← oplog version
//   op:   "i",                         ← i=insert, u=update, d=delete, c=command
//   ns:   "mydb.orders",              ← namespace
//   ui:   UUID("..."),                ← collection UUID
//   wall: ISODate("2024-03-01T10:00:00Z"),
//   o:    { _id: ObjectId("..."), status: "pending" }  ← the operation
// }

// Reading from a secondary (default: reads go to primary):
const client = new MongoClient(uri, {
  readPreference: "secondary"   // redirect reads to secondaries
})
// Caution: secondaries may be slightly behind primary (replication lag)
```

---

## Q2: How does replica set election work?

When the primary becomes unavailable, the remaining members hold an election to choose a new primary. MongoDB uses the Raft consensus protocol.

```javascript
// Election trigger:
// 1. Secondaries send heartbeats to all other members every 2 seconds
// 2. If a secondary doesn't receive a heartbeat from the primary for 10 seconds (electionTimeoutMS)
//    → it considers the primary unreachable
// 3. Secondary increments term number, transitions to "CANDIDATE" state
// 4. Candidate requests votes from other members
// 5. Members vote YES if:
//    - They haven't voted in this term
//    - Candidate's oplog is as up-to-date as theirs
// 6. Candidate needs majority of votes (in a 3-node set: 2 votes)
// 7. On winning majority: transitions to PRIMARY, begins accepting writes

// Why majority matters:
// 3-node set: primary needs 2 votes. If network split:
//   Node 1 can see Node 2 → Node 1 + Node 2 = 2 votes → can elect primary ✓
//   Node 3 is isolated → cannot get majority → stays secondary ✓
// This prevents split-brain (two primaries simultaneously)

// Election priority:
// Priority 0: never becomes primary (good for arbiter-like or delayed members)
// Priority 1 (default): can become primary
// Priority 2+: preferred primary (wins ties in elections)

// Force an election (controlled switchover):
rs.stepDown(60)   // current primary steps down for 60 seconds, triggers election
// Use this for planned maintenance (upgrade the primary)

// Election timeline:
// T=0:    Primary crashes or becomes unreachable
// T=10s:  Secondary detects heartbeat timeout (electionTimeoutMS: 10000)
// T=10s:  Secondary becomes candidate, requests votes
// T=11s:  Votes received, new primary elected
// T=11s:  Clients reconnect to new primary (driver auto-detects)
// Total typical downtime: 10-15 seconds

// Adjusting election timeout (trade-off: faster detection vs false elections):
rs.reconfig({ members: [...], settings: { electionTimeoutMillis: 5000 } })
// 5 seconds: faster elections but more sensitive to transient network blips
```

---

## Q3: What is the oplog? How do you monitor and size it?

The **oplog** (operations log) is a special capped collection (`local.oplog.rs`) that records every write operation that modifies data. Secondaries use it to replicate from the primary.

```javascript
// Oplog characteristics:
// - Capped collection (fixed size, circular overwrite)
// - Stored in the "local" database (not replicated itself)
// - Contains only idempotent operations (can be applied multiple times safely)
// - Pre-update image conversion: MongoDB converts ops to idempotent forms

// Check oplog info:
db.adminCommand({ replSetGetStatus: 1 }).members.forEach(m =>
  console.log(m.name, m.stateStr, m.optimeDate, m.lastHeartbeat)
)

rs.printReplicationInfo()
// Output:
// configured oplog size:   3999.81MB
// log length start to end: 98398 secs (27.33hrs)
// oplog first event time:  Thu Feb 29 2024 10:00:00
// oplog last event time:   Fri Mar 01 2024 13:23:18
// now:                     Fri Mar 01 2024 13:23:21
// oplog window = 27.33 hours

// Sizing the oplog:
// Rule of thumb: oplog should hold at least 24 hours (ideally 72 hours) of operations
// Why: if a secondary falls behind by more than the oplog window → it must do a full resync

// Default oplog size:
// 5% of disk space up to 50GB

// Change oplog size (Atlas: configured via cluster settings):
db.adminCommand({
  replSetResizeOplog: 1,
  size: 10240   // 10GB in MB
})

// Monitor replication lag:
rs.printSlaveReplicationInfo()
// Secondary 1:        behind the primary by 2 secs
// Secondary 2:        behind the primary by 0 secs

// High replication lag causes:
// 1. Secondary falling behind → writes faster than secondary can apply
//    Fix: increase secondary resources (CPU, disk IOPS)
// 2. Network bandwidth saturation
//    Fix: reduce oplog traffic, use compression, add bandwidth
// 3. Long-running operations on primary
//    Fix: use caution with bulk operations, chunk large jobs

// Oplog window calculation:
// Your application writes 1GB of oplog per hour
// You want a 72-hour window
// Oplog size needed: 1GB × 72 = 72GB
```

---

## Q4: What read preferences are available? When do you use each?

Read preferences control which replica set member the driver routes read operations to.

```javascript
// Five read preferences:

// 1. primary (default)
const client = new MongoClient(uri, { readPreference: "primary" })
// All reads go to the primary
// Use: when you need the absolute latest data (no staleness)
// Risk: all read traffic hits one node

// 2. primaryPreferred
new MongoClient(uri, { readPreference: "primaryPreferred" })
// Reads go to primary; falls back to secondary if primary unavailable
// Use: mostly need fresh data, but availability > freshness during failures
// Good for: reporting queries during primary maintenance windows

// 3. secondary
new MongoClient(uri, { readPreference: "secondary" })
// All reads go to secondaries (not the primary)
// Use: analytics, reporting, batch jobs — offload read traffic from primary
// Caution: replication lag means stale reads possible

// 4. secondaryPreferred
new MongoClient(uri, { readPreference: "secondaryPreferred" })
// Reads go to secondaries; falls back to primary only if no secondaries available
// Use: highest read throughput distribution, tolerate slight staleness

// 5. nearest
new MongoClient(uri, { readPreference: "nearest" })
// Reads go to the member with lowest network latency (could be primary or secondary)
// Ignores primary/secondary status
// Use: multi-region setups (read from geographically closest node)
// Atlas Global Clusters: this is the common choice

// Tag sets for routing to specific members:
rs.reconfig({
  members: [
    { _id: 0, host: "primary:27017", tags: { region: "us-east", use: "operational" } },
    { _id: 1, host: "secondary1:27017", tags: { region: "us-east", use: "analytics" } },
    { _id: 2, host: "secondary2:27017", tags: { region: "eu-west", use: "analytics" } }
  ]
})

// Route analytics queries to tagged analytics members:
const client = new MongoClient(uri, {
  readPreference: new ReadPreference("secondary",
    [{ use: "analytics" }])  // tag set filter
})

// On Atlas: separate Analytics Nodes (M10+)
// Analytics nodes: hidden from primary election, dedicated for read workloads
// Do not affect primary election
// Perfect for: BI Connector, Spark, heavy aggregation pipelines
```

---

## Q5: What is an arbiter? When should (and shouldn't) you use one?

An **arbiter** is a mongod instance that participates in elections but holds NO data. It votes in elections to break ties.

```javascript
// Arbiter use case: 3-node replica set without the cost of a 3rd data node
// 2 data nodes (primary + secondary) + 1 arbiter = voting majority = 2

// Adding an arbiter (only for non-Atlas self-managed deployments):
rs.addArb("arbiter:27017")
// mongod configuration for arbiter:
// mongod --replSet rs0 --dbpath /data/arb --port 27017

// WHEN TO USE AN ARBITER:
// ✓ Cost-sensitive: 3 full data nodes too expensive, but need election quorum
// ✓ Self-managed deployments in 2-node primary+secondary pairs

// WHEN NOT TO USE AN ARBITER (and why MongoDB discourages them):
// ✗ Atlas recommends AGAINST arbiters entirely
// ✗ Arbiter provides NO data redundancy (it has no data)
// ✗ If both data nodes are down: arbiter can't help recover data
// ✗ Atlas minimum: 3 data-bearing nodes for M10+ clusters
// ✗ 2 data nodes + arbiter: if primary crashes → only 1 secondary for reads
//    (secondaryPreferred still works, but capacity is cut in half)

// MongoDB's recommendation (4.0+):
// Use 3 or more data-bearing members for production
// Arbiter is acceptable for development or very budget-constrained deployments

// Priority-0 member (alternative to arbiter for data redundancy without extra primary risk):
rs.reconfig({
  members: [
    { _id: 0, host: "primary:27017",   priority: 1 },
    { _id: 1, host: "secondary:27017", priority: 1 },
    { _id: 2, host: "hidden:27017",    priority: 0, hidden: true, votes: 1 }
    // hidden member: participates in elections, holds data, NOT visible to clients
  ]
})
```

---

## Q6: What is MongoDB sharding? Explain the components.

**Sharding** is MongoDB's horizontal scaling mechanism. Data is distributed across multiple **shards** (replica sets), each holding a subset of the data.

```javascript
// 3 components of a sharded cluster:

// 1. SHARDS: each shard is a replica set
// Holds a subset of the total data (chunks)
// shard0: { status: "pending", customerId: 0-5000 } range (example)
// shard1: { status: "pending", customerId: 5001-10000 }

// 2. CONFIG SERVERS: a replica set storing cluster metadata
// Stores: chunk ranges, which shard each chunk lives on, cluster topology
// 3 config server nodes (default Atlas setup)
// Never hold application data

// 3. MONGOS: query router
// The entry point for all client connections in a sharded cluster
// Routes queries to the correct shard(s)
// Merges results from multiple shards
// Atlas: runs mongos automatically (transparent to clients)

// Architecture diagram (commands):
// Client → mongos → shard routing decision
//                 ↗ shard0 (rs0)
//                 → shard1 (rs1)
//                 ↘ shard2 (rs2)
//          ↑ config servers (metadata)

// Enable sharding on a database and collection:
sh.enableSharding("ecommerce")
sh.shardCollection("ecommerce.orders", { customerId: 1 })  // hashed or ranged

// Sharding considerations:
// - Shard key choice is PERMANENT (cannot change after sharding)
// - Bad shard key → hotspot shard (all writes to one shard) → no scalability
// - Good shard key → even distribution of writes and reads

// Atlas sharding:
// Add a shard via Atlas UI: Clusters → Shards → Add Shard
// Atlas auto-manages chunk balancing between shards
// No manual sh.moveChunk() needed (Atlas balancer does it automatically)
```

---

## Q7: How do you choose the right shard key?

Choosing the shard key is the most critical sharding decision — it cannot be changed after sharding.

```javascript
// GOOD shard key properties:
// 1. High cardinality: many unique values (not boolean, not low-enum fields)
// 2. Even write distribution: writes spread across all shards
// 3. Targeted queries: query filter includes the shard key (avoids scatter-gather)
// 4. Not monotonically increasing: avoids write hotspot on last shard

// CARDINALITY PROBLEM (bad shard key):
sh.shardCollection("users", { country: 1 })
// Only ~200 unique countries → all US users on 1 shard → hotspot!

// MONOTONIC INCREASING PROBLEM (bad shard key):
sh.shardCollection("orders", { createdAt: 1 })
// All new orders go to the shard holding the highest time range
// → "last shard" hotspot: every INSERT goes to 1 shard

// FIX 1: Hashed shard key (solves monotonic increasing problem)
sh.shardCollection("orders", { _id: "hashed" })
// MongoDB hashes the _id value → random distribution across shards
// Downside: range queries on _id are now scatter-gather (hits all shards)

// FIX 2: Compound shard key (for targeted queries + even distribution)
sh.shardCollection("orders", { customerId: 1, createdAt: 1 })
// customerId: high cardinality (millions of customers)
// Query: db.orders.find({ customerId: "alice" }) → targeted to 1 shard
// Query: db.orders.find({ createdAt: ... }) → scatter-gather (no shard key)
// Write: distributed by customerId → no hotspot

// ZONE SHARDING (Global Clusters):
sh.shardCollection("users", { region: 1, _id: 1 })
// region provides data placement (US users on US shards, EU on EU)
// _id provides cardinality within the region

// Checking chunk distribution:
sh.status()
// Shows: chunks per shard, min/max values per chunk
// Even distribution = good. One shard with 90% chunks = hotspot

// MongoDB 5.0+: resharding!
// Can change shard key without full data copy
db.adminCommand({
  reshardCollection: "ecommerce.orders",
  key: { customerId: 1, createdAt: 1 }  // new shard key
})
// Background operation, online (reads/writes continue)
// Duration: hours to days for large collections
```

---

## Q8: What is chunk balancing in sharded clusters?

MongoDB divides the shard key range into logical **chunks** (default max 128MB). The **balancer** runs periodically to move chunks between shards so data is evenly distributed.

```javascript
// Chunks:
// Each chunk covers a range of the shard key:
// Shard0: { customerId: MinKey → 10000 }
// Shard1: { customerId: 10001 → 50000 }
// Shard2: { customerId: 50001 → MaxKey }

// As data grows, chunks split:
// { customerId: MinKey → 10000 } → too large (>128MB)
// → splits into:
//   { customerId: MinKey → 5000 }
//   { customerId: 5001 → 10000 }

// Balancer detects imbalance:
// Shard0: 100 chunks
// Shard1: 60 chunks
// Shard2: 40 chunks
// → Imbalance! Balancer moves chunks from Shard0 to Shard2

// Balancing operations:
sh.getBalancerState()   // is balancer running?
sh.isBalancerRunning()  // actively migrating right now?
sh.startBalancer()
sh.stopBalancer()       // stop during maintenance

// Balancer window (avoid balancing during business hours):
use config
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "23:00", stop: "06:00" } } },
  { upsert: true }
)
// Balancer only runs between 11 PM and 6 AM

// Monitor chunk migrations:
use config
db.changelog.find({ what: "moveChunk.commit" }).sort({ time: -1 }).limit(5)

// Jumbo chunks (cannot be split or moved):
// A chunk is "jumbo" if ALL its documents have the same shard key value
// Example: { country: 1 } shard key, and 1 million users from "US"
// → All go to ONE chunk → that chunk can never split (all have { country: "US" })
// → That shard is stuck with all US users → hotspot
// Fix: change shard key to have higher cardinality
```

---

## Q9: What is scatter-gather query? How do you minimize them?

A **scatter-gather** query is one where mongos must send the query to ALL shards because the shard key is not included in the filter. Every shard executes the query independently and sends results to mongos for merging.

```javascript
// Sharded on: { customerId: 1, createdAt: 1 }

// TARGETED query (fast — only hits 1 shard):
db.orders.find({ customerId: "alice" })
// mongos knows: alice's orders are on shard2 → sends to shard2 only
// Result: 1 shard queried, result returned immediately

// SCATTER-GATHER query (slow — hits all shards):
db.orders.find({ status: "pending" })
// mongos has no idea which shard has "pending" orders
// → sends query to ALL shards (shard0, shard1, shard2)
// → each shard executes independently
// → mongos merges results
// Overhead: 3x the work, plus merge latency

// MINIMIZING SCATTER-GATHER:
// 1. Design shard key to match most common queries
//    If most queries filter by customerId → shard on customerId
//    If most queries filter by orderId → shard on orderId

// 2. Include shard key in query predicates
//    Add customerId to your UI filters even if the user doesn't set it:
async function getOrdersByStatus(status, customerId = null) {
  const filter = { status }
  if (customerId) filter.customerId = customerId  // enables targeted query
  return db.orders.find(filter).toArray()
}

// 3. Use zone sharding for geographic distribution
//    EU users → EU shard, US users → US shard → most queries are targeted by region

// 4. Accept some scatter-gather for analytics
//    Reports, dashboards: scatter-gather is acceptable (not user-facing latency)
//    Use dedicated analytics nodes or Atlas Data Federation for these

// Identifying scatter-gather in Atlas:
// Atlas UI → Performance Advisor → look for "mongos" in query plans
// explain() on a sharded collection:
db.orders.find({ status: "pending" }).explain("executionStats")
// { "queryPlanner": { "winningPlan": { "stage": "SHARD_MERGE" } } }
// SHARD_MERGE = scatter-gather occurred
// vs. "SINGLE_SHARD" or "shard targeting" = targeted query
```

---

## Q10: What is the difference between hashed and ranged sharding?

```javascript
// RANGED SHARDING:
// Shard key values are divided into contiguous ranges
// Documents with adjacent shard key values are co-located on the same shard
sh.shardCollection("products", { category: 1, price: 1 })
// Shard0: { category: "books", price: 0-50 }
// Shard1: { category: "books", price: 51-200 }
// Shard2: { category: "electronics", price: 0-100 }

// Benefits:
// ✓ Range queries are efficient (only 1-2 shards): find({ category: "books", price: { $lte: 50 } })
// ✓ Sort operations can use shard order
// ✓ Zone sharding (geographic): regions map to value ranges

// Problems:
// ✗ Monotonically increasing keys (like ObjectId, timestamp) → hotspot on last shard
// ✗ Uneven distribution if data isn't uniformly spread across ranges

// HASHED SHARDING:
// MongoDB hashes the shard key field value → random distribution
sh.shardCollection("orders", { _id: "hashed" })
// hash(_id: "abc") → shard2
// hash(_id: "def") → shard0
// hash(_id: "ghi") → shard1
// Random-looking distribution → no hotspot even for ObjectId/timestamp

// Benefits:
// ✓ Even distribution for any shard key (even monotonically increasing)
// ✓ Eliminates write hotspots for ordered keys
// ✓ Good for high-write collections with ObjectId as shard key

// Problems:
// ✗ Range queries are ALWAYS scatter-gather (hash breaks the ordering)
// ✗ No zone sharding (can't map hash ranges to geographic zones)

// COMPOUND HASHED SHARDING (MongoDB 4.4+):
// Range on first field + hash on second:
sh.shardCollection("orders", { region: 1, _id: "hashed" })
// region: ranged (zone sharding still works per region)
// _id within region: hashed (even distribution within each region's shard)
// → Geo-targeted queries + even write distribution per region

// Choosing:
// Time-series / high-write append: hashed
// Range query-heavy workloads: ranged (but avoid monotonic key for first field)
// Geo-distributed: zone sharding (ranged on region field)
// Complex workloads: compound with mix
```

---

## Q11: What is the pre-splitting strategy for sharded collections?

```javascript
// Problem: when you first shard a collection, ALL data starts on 1 shard
// Even with hashed sharding: initial chunks all on shard0
// MongoDB gradually splits and balances → can take hours for large collections

// Pre-splitting: create empty chunks BEFORE loading data
// Requires: stopping the balancer, creating chunks manually, then re-enabling

// Pre-split with hashed shard key:
sh.stopBalancer()

// For 3 shards, create 9 empty chunks (3 per shard):
const shardKey = { _id: "hashed" }
const numberOfChunks = 9   // at least 2x number of shards

sh.splitAt("mydb.events", { _id: NumberLong("-6917529027641081856") })
sh.splitAt("mydb.events", { _id: NumberLong("-3458764513820540928") })
sh.splitAt("mydb.events", { _id: NumberLong("0") })
sh.splitAt("mydb.events", { _id: NumberLong("3458764513820540928") })
// ... etc.

// Manually move initial chunks to balance across shards:
sh.moveChunk("mydb.events", { _id: NumberLong("0") }, "shard1")
sh.moveChunk("mydb.events", { _id: NumberLong("3458764513820540928") }, "shard2")

sh.startBalancer()

// Modern approach (MongoDB 5.0+): pre-splitting happens automatically during shardCollection
sh.shardCollection("mydb.events", { _id: "hashed" }, false, { numInitialChunks: 9 })
// numInitialChunks: creates 9 empty chunks distributed across shards immediately
// → No hotspot during initial bulk load

// Atlas: automatic pre-splitting for hashed sharding
// When you add a shard in Atlas, it pre-balances automatically
```

---

## Q12: What is the impact of adding a shard to an existing cluster?

```javascript
// Adding a shard:
// 1. New shard starts empty
// 2. Balancer moves chunks from existing shards to the new shard
// 3. This takes time (hours for large clusters): chunk migration involves:
//    - Copying data to new shard
//    - Redirecting new writes to new shard
//    - Deleting from old shard (happens after confirmation)

// Monitoring shard addition:
sh.status()
// Shows: chunks per shard — new shard starts at 0, grows as balancer migrates

// During chunk migration:
// - Reads/writes continue normally (no downtime)
// - Migrating chunk: mongos routes reads to source shard, migrates in background
// - At migration commit: mongos atomically switches routing to destination
// - Migration is transparent to the application

// Performance impact during migration:
// - Source shard: extra read I/O (reading data to send to new shard)
// - Destination shard: extra write I/O (receiving migrated data)
// - Network: migration traffic between shards (can saturate if network is slow)
// - Mitigation: set balancer window to off-peak hours
sh.setBalancerWindow("02:00", "06:00")  // only balance 2-6 AM

// Removing a shard (draining):
db.adminCommand({ removeShard: "shard2" })
// All chunks on shard2 are migrated to other shards
// shard2 must be drained (empty) before it can be removed
// Database entries pointing to shard2 must be moved to another shard

// MongoDB Atlas: add/remove shards via Atlas UI
// Atlas handles the migration automatically with the built-in balancer
// No manual chunk management needed
```

---

## Q13: What is the `mongos` router? What happens if it goes down?

```javascript
// mongos responsibilities:
// 1. Accept all client connections (driver connects to mongos, not shards directly)
// 2. Route operations to correct shard(s) based on shard key
// 3. Merge results from multiple shards (for scatter-gather or $lookup)
// 4. Cache cluster metadata from config servers
// 5. Perform $lookup across shards (limited support)

// mongos is STATELESS:
// - Holds no data
// - Gets metadata from config servers
// - Can be restarted instantly (no data loss)
// - Multiple mongos instances for high availability

// High availability setup:
// Run at least 2 mongos instances
// Client connection string lists multiple mongos:
"mongodb://mongos1:27017,mongos2:27017,mongos3:27017/mydb?readPreference=nearest"

// When a mongos goes down:
// - Driver retries with the other mongos instances (from connection string)
// - No data loss (mongos is stateless)
// - Brief interruption until driver detects and reconnects (~30-60 seconds with retryable writes)

// mongos cache:
// mongos caches chunk routing information from config servers
// If config servers are unreachable: mongos serves stale cached routes
// For a few minutes: mongos can still route correctly from cache
// If config servers are unavailable for long: mongos eventually can't route correctly

// Atlas: mongos is managed automatically
// Atlas runs multiple mongos instances per cluster
// Connection string (atlas) points to mongos via DNS SRV:
"mongodb+srv://user:pass@cluster.mongodb.net/"
// SRV resolves to multiple mongos addresses

// Performance considerations:
// mongos is on the read/write hot path → must be fast
// Run mongos close to the application (low network latency to mongos is critical)
// In Atlas: mongos runs in the same AWS region as your application cluster
```

---

## Q14: How do you handle transactions in a sharded cluster?

```javascript
// Multi-shard transactions (MongoDB 4.2+):
// - Supported but with overhead
// - Uses 2-phase commit (coordinator shard + participants)
// - All shards holding modified documents participate

// Transaction that spans multiple shards:
const session = client.startSession()
await session.withTransaction(async () => {
  // Write to shard0 (orders collection, sharded on customerId)
  await db.orders.insertOne({
    customerId: "alice",   // goes to shard containing "alice" range
    items: [...]
  }, { session })

  // Write to shard1 (inventory, sharded on productId)
  await db.inventory.updateOne(
    { productId: "widget" },   // goes to different shard
    { $inc: { qty: -1 } },
    { session }
  )
  // Cross-shard transaction! mongos coordinates 2PC
})

// Performance impact:
// Single-shard transaction: ~5-15ms overhead
// Two-shard transaction: ~15-30ms overhead (extra 2PC round-trips)
// Three-shard transaction: ~25-50ms overhead

// Best practice: minimize cross-shard transactions
// Design schema to keep related data on the same shard:
// Co-located: orders + order_items both sharded on customerId
// → A customer's orders and items are always on the same shard
// → Transaction only touches 1 shard → no 2PC overhead

// Cross-shard $lookup (for reads only — not in transactions):
// Only allowed if the "from" collection is unsharded or co-located
// $lookup across different shards → not supported in transactions
// Alternative: denormalize (Extended Reference Pattern) to avoid $lookup

// Monitor transaction metrics:
db.serverStatus().transactions
// {
//   totalStarted: 1234,
//   totalCommitted: 1228,
//   totalAborted: 6,
//   currentOpen: 2
// }
```

---

## Q15: What is a hidden replica set member? What about delayed members?

```javascript
// HIDDEN MEMBER:
// - Not visible to clients (driver won't route reads to it)
// - Participates in elections (votes) but priority: 0 (never becomes primary)
// - Holds a full copy of the data
// - Use cases: dedicated reporting node, backup source, Atlas Analytics Nodes

rs.reconfig({
  members: [
    { _id: 0, host: "primary:27017",    priority: 1 },
    { _id: 1, host: "secondary:27017",  priority: 1 },
    { _id: 2, host: "hidden:27017",     priority: 0, hidden: true, votes: 1 }
  ]
})

// Client's rs.isMaster() (or hello) doesn't include hidden member
// Application reads won't be routed to hidden member
// But: hidden member still replicates all data
// Access directly: db = new MongoClient("mongodb://hidden:27017/")
// (only possible if you connect directly to the hidden member's address)

// DELAYED MEMBER:
// - Holds data that's delayed by a fixed time window (e.g., 1 hour behind primary)
// - Priority: 0, hidden: true (usually)
// - Use case: protection against accidental data deletion
//   → If DBA accidentally drops a collection, delayed member still has it for 1 hour

rs.reconfig({
  members: [
    { _id: 0, host: "primary:27017",    priority: 1 },
    { _id: 1, host: "secondary:27017",  priority: 1 },
    { _id: 2, host: "delayed:27017",    priority: 0, hidden: true,
      votes: 1, secondaryDelaySecs: 3600 }  // 1 hour delay
  ]
})

// Delayed member oplog:
// The delayed member applies oplog entries 1 hour late
// Primary drops collection at 2:00 PM
// Delayed member: still has the collection until 3:00 PM
// Recovery window: you have 1 hour to realize the mistake and restore from delayed member

// Atlas equivalent: Cloud Backup with point-in-time restore
// Most teams use PITR instead of delayed members in production
// Delayed member: useful for self-managed MongoDB, ops-team controlled

// Oplog sizing for delayed members:
// If delay = 1 hour, must have oplog window > 1 hour (otherwise delayed member falls behind forever)
// Typical: oplog = delay + 24 hours safety margin
```
