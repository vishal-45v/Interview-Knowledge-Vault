# Chapter 09 — Replication & Sharding: Follow-Up Traps

---

## Trap 1: "Arbiters Are Free — Just Use One Instead of a Full Secondary"

**Wrong answer:** "Arbiters are great — they're free nodes that count toward quorum without storing data. I'd use an arbiter in a 2-node replica set to get a 3-node quorum cheaply."

**Why it's wrong:**
- PSA (Primary-Secondary-Arbiter) configurations are actively discouraged by MongoDB in production
- The secondary in PSA has no redundancy: if the secondary goes down, the arbiter (which stores no data) cannot help with data recovery
- Worse: if the secondary goes down, the primary is now in a cluster where the arbiter is the only other voter — the primary knows it has majority (2/3) but there's only ONE copy of the data
- MongoDB 5.0 changed election behavior specifically to prevent primaries from staying primary in this configuration (the primary steps down when it detects PSA with a non-voting secondary)

**Correct answer:** "Arbiters are a cost-saving shortcut that introduces real risk. In PSA config, losing the secondary means you have exactly one copy of your data on the primary with no failover. MongoDB recommends PSS (3 full data-bearing nodes) or use an arbiter only if you have 4+ data nodes and need an odd vote count. On Atlas, M10+ clusters use PSS by default — no arbiters."

---

## Trap 2: "Adding More Secondaries Scales Write Throughput"

**Wrong answer:** "We need more write throughput — let me add 2 more secondaries to the replica set."

**Why it's wrong:**
- ALL writes go to the primary only
- Adding secondaries does NOT increase write throughput at all
- It only adds read capacity (for queries using secondary read preference) and increases durability/redundancy
- More secondaries also means more replication overhead (primary must send oplog to all secondaries)

**Correct answer:** "Secondaries only scale reads. To scale writes, you need sharding — distributing writes across multiple primaries. Adding secondaries improves read scalability, geographic distribution, and data redundancy, but has zero impact on write throughput."

---

## Trap 3: "The Primary Always Has the Most Up-to-Date Data"

**Wrong answer:** "Always read from the primary — it's the most current."

**Why it's wrong:**
- `readConcern: "local"` on the primary returns committed data as of that instant — but this can include data that hasn't yet been replicated to majority and could be rolled back on primary failure
- `readConcern: "majority"` on a secondary returns data that will never be rolled back (durable)
- "Most up-to-date" and "most durable" are different things
- For financial transactions, a majority-acknowledged secondary read is "safer" than a local primary read

**Correct answer:** "The primary has the latest writes, but `readConcern: 'local'` on primary can return data that might be rolled back if the primary fails before replication. For data that must never be rolled back, use `readConcern: 'majority'` — which can also be satisfied by secondaries."

---

## Trap 4: "Hashed Sharding Is Always Better Than Ranged Sharding"

**Wrong answer:** "Use hashed sharding — it gives perfect distribution and avoids hotspots."

**Why it's wrong:**
- Hashed sharding destroys range query locality — `{ createdAt: { $gte: ... $lte: ... } }` becomes a scatter-gather across all shards
- Every range query hits every shard (scatter-gather) — terrible for analytics and time-series
- Hashed sharding is ideal ONLY when: (1) the query pattern is point lookups by the hashed field, (2) there's no need for range queries, (3) even write distribution is the priority

**Correct answer:** "Hashed sharding solves write hotspots at the cost of range query performance. For time-series data, use compound sharding like `{ tenantId: 1, timestamp: 1 }` — tenantId prefix distributes across shards, timestamp suffix enables range queries within a tenant. Only use hashed sharding when you have confirmed that range queries on the shard key field are not needed."

---

## Trap 5: "Scatter-Gather Queries Are Always Bad"

**Wrong answer:** "We should redesign everything to avoid any scatter-gather queries."

**Why it's wrong:**
- For low-frequency admin/analytics queries, scatter-gather is acceptable
- The cost is proportional to the number of shards — 5 shards means 5 parallel queries
- Scatter-gather with parallel execution can still be fast if the query is simple and the result set is small
- Eliminating all scatter-gather might require a shard key that creates hotspots for the common case

**Correct answer:** "Scatter-gather is acceptable for infrequent queries. The design goal is: high-frequency OLTP queries must be targeted (single or few shards); low-frequency analytics can scatter-gather. Optimize for the common case first. A query that runs once per day scatter-gathering 10 shards is fine; a query that runs 10,000 times per second must be targeted."

---

## Trap 6: "You Can Change the Shard Key After Sharding"

**Wrong answer (pre-MongoDB 5.0):** "If we pick the wrong shard key, we can just change it later."

**Correct nuance:** In MongoDB 5.0+, `reshardCollection` allows changing the shard key online (no downtime), but it is not free:
- Resharding copies ALL data to new shard layout — this takes time proportional to collection size
- During resharding: writes go to both old and new layout (temporary write amplification)
- Brief write block at commit phase (seconds)
- For a 5TB collection, resharding may take 4-8 hours

**Correct answer:** "Resharding is available in MongoDB 5.0+ and is live (no downtime), but it's a major operation. For a 5TB collection, expect several hours of slightly degraded write performance. It's not a quick fix — still worth choosing the right shard key upfront. But if you got it wrong, resharding is the path forward."

---

## Trap 7: "w:majority Guarantees No Data Loss"

**Wrong answer:** "With w:majority, data is completely safe — it can never be lost."

**Why it's wrong:**
- `w:majority` ensures the write was acknowledged by a majority of nodes and is durable to disk
- BUT: if `j:false` (default for majority in some drivers), the write is in memory on the secondary nodes — a simultaneous crash of all nodes before journal flush could lose data
- Catastrophic scenarios (entire data center fire, storage corruption) lose data regardless of write concern
- Write concern is about normal failure scenarios (single node failure, network partition), not catastrophic failures

**Correct answer:** "`w:majority, j:true` provides strong durability guarantees against single-node failures and primary elections. It means the write is journaled on a majority of nodes. But no software-level write concern protects against physical media destruction — that requires backup strategy (Atlas Cloud Backup with off-region snapshots)."

---

## Trap 8: "More Config Servers = More Reliability"

**Wrong answer:** "For a production sharded cluster, I'd run 5 config servers for high availability."

**Why it's wrong:**
- MongoDB requires config servers to run as a replica set (since MongoDB 3.4)
- The config server replica set follows the same rules as any replica set
- Having 5 config servers means you need 3 to be alive for writes (majority of 5)
- Having 3 config servers means you need 2 to be alive (majority of 3)
- 5 config servers provides only marginal reliability improvement over 3 and adds cost/complexity
- MongoDB explicitly recommends exactly 3 config servers

**Correct answer:** "Run exactly 3 config servers (as a 3-node replica set). This gives you failure tolerance for 1 config server. You can still READ cluster metadata with only 1 config server alive, and new operations require a majority (2 of 3). Running 5 adds cost without meaningful reliability improvement for config servers specifically."

---

## Trap 9: "Transactions in Sharded Clusters Work the Same as in Replica Sets"

**Wrong answer:** "I'll design for single-node transactions and then just shard later — transactions work the same everywhere."

**Why it's wrong:**
- Sharded cluster transactions require 2-Phase Commit across shards (coordinated by mongos)
- 2PC adds 2-5x latency overhead compared to single-shard transactions
- Shard coordinator node is a single point of failure for in-flight transactions
- Transactions cannot exceed 1,000 operations across shards by default (chunked operations)
- Each shard in the transaction holds locks until commit → higher contention in distributed scenario

**Correct answer:** "Transactions in sharded clusters use 2PC via mongos as coordinator. This adds significant latency. The design goal should be single-shard transactions via co-location: shard all related collections on the same key prefix (e.g., `customerId`), so order + orderItems + payments for the same customer live on the same shard. A transaction that touches only one shard has replica-set-level performance."

---

## Trap 10: "Hidden Members Don't Count for Read Preference"

**Wrong answer (incomplete):** "Hidden members are completely invisible to the application."

**Why it's nuanced:**
- Hidden members are invisible to ALL read preferences (primary, secondary, nearest, secondaryPreferred) — you CANNOT route reads to them via standard read preference
- But hidden members CAN be used for: backups (mongodump runs against hidden replica), analytics (via direct connection bypassing the replica set), and they DO participate in elections (if votes: 1, which is the default)
- A hidden member with `votes: 1` can still vote in elections even though it can't serve reads

**Correct answer:** "Hidden members are invisible to read preference routing — clients never see them in server selection. But they still replicate data, participate in elections (votes: 1 by default), and can serve as backup targets via direct connection. If you want a node that's purely for backup without voting, set both `hidden: true` AND `votes: 0`."

---

## Trap 11: "The Balancer Should Always Be On"

**Wrong answer:** "The balancer should always run to keep chunks balanced."

**Why it's wrong:**
- Chunk migration is I/O and network intensive: it reads chunks from source shard and writes to destination
- During heavy OLTP load, balancer migrations can degrade query performance
- Production best practice: configure balancer to run only during low-traffic windows

**Correct answer:** "Configure a balancer active window that aligns with low-traffic periods (e.g., 2 AM - 6 AM). This way, chunk balancing happens without impacting production load. The downside: cluster stays slightly imbalanced during peak hours, but this is almost always acceptable. In Atlas, the balancer window can be configured per cluster."

```javascript
// Configure balancer window:
use config
db.settings.updateOne(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: {
        start: "02:00",  // 2 AM
        stop: "06:00"    // 6 AM (UTC)
      }
    }
  },
  { upsert: true }
)
```

---

## Trap 12: "You Can Always Add Tags to Existing Shard Keys"

**Wrong answer:** "I'll set up zone sharding later once we see traffic patterns."

**Why it's wrong:**
- Zone sharding requires that the shard key includes the zone-routing field as a prefix
- If you shard on `{ _id: 1 }` and later want zone-based routing by `region`, you CANNOT add tags that work correctly — `region` is not in the shard key
- Adding zones to an existing collection with a wrong shard key requires resharding
- Zone configuration must be planned upfront and the shard key must accommodate it

**Correct answer:** "Zone sharding design must be part of your initial shard key selection. If you plan to route by geography, the shard key MUST start with the geographic field (e.g., `{ region: 1, _id: 1 }`). The tag ranges you configure map directly to shard key value ranges — if `region` isn't in the shard key, zone tags have no effect. Plan this before sharding."
