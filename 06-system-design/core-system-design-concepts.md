# Core System Design Concepts

---

## CAP Theorem

Any distributed data store can guarantee only 2 of 3:
- **Consistency:** All nodes see the same data at the same time
- **Availability:** System responds to every request (no errors)
- **Partition Tolerance:** System continues despite network splits

During a network partition (P), you must choose:
- CP: Consistent + Partition-Tolerant (may return errors, never stale data)
  → HBase, ZooKeeper, etcd
- AP: Available + Partition-Tolerant (always responds, may return stale data)
  → Cassandra, CouchDB, DynamoDB (eventual consistency)

In practice: Network partitions happen (must accept P), so the real choice is C vs A during partitions.

---

## Consistency Models

**Strong Consistency:** Read always returns the most recent write. Slowest.
**Eventual Consistency:** Will eventually return the most recent write. Fastest.
**Read-your-writes:** You always see your own writes. (User's own profile updates)
**Monotonic reads:** Once you read a value, subsequent reads won't return older values.
**Causal consistency:** Related operations seen in correct order.

---

## Database Scaling Patterns

```
Vertical Scaling (Scale Up):
  Bigger server — more RAM, CPU, faster disk
  Simple, but expensive and has limits

Read Replicas:
  Primary handles writes
  Replicas handle reads (async replication)
  Read:Write ratio 10:1 → 10 replicas handle reads
  Lag: replicas may be slightly behind primary

Sharding (Horizontal Scaling):
  Partition data across multiple DB servers
  Shard key: user_id % num_shards, region, hash
  Pros: Infinite horizontal scaling
  Cons: Complex queries, cross-shard joins expensive, resharding painful

CQRS (Command Query Responsibility Segregation):
  Separate write model (commands) from read model (queries)
  Write: normalized SQL DB for consistency
  Read: denormalized, optimized for specific query patterns
```

---

## Load Balancing Algorithms

```
Round Robin: Requests distributed in rotation
  [S1][S2][S3][S1][S2][S3]...
  Simple, doesn't account for server load

Least Connections: Route to server with fewest active connections
  Good when requests have varying processing time

IP Hash: Same client IP always routes to same server
  Useful for session affinity (sticky sessions)

Weighted Round Robin: Servers get requests proportional to their weight
  S1(weight=3): gets 3× traffic of S2(weight=1)

Random: Random server selection
  Surprisingly effective at scale (law of large numbers)
```

---

## Message Queue Patterns

```
Point-to-Point (Queue):
  Producer → [Queue] → Consumer
  Message consumed by exactly ONE consumer
  Use: Task distribution, work queues

Publish-Subscribe (Topic):
  Publisher → [Topic] → [Subscriber 1]
                     → [Subscriber 2]
                     → [Subscriber 3]
  Message consumed by ALL subscribers
  Use: Events, notifications, cache invalidation

Common Queue Systems:
  RabbitMQ: Low latency, complex routing, AMQP
  Kafka: High throughput, log-based, replay, ordering
  SQS: Managed, at-least-once delivery, AWS
  ActiveMQ: Java-native, JMS, legacy-friendly
```

---

## Consistent Hashing

```
Problem: Adding/removing servers in a cluster changes where keys are mapped.
Traditional hashing: key % N_servers → all keys remapped when N changes

Consistent Hashing:
  Servers and keys placed on a "ring" (0 to 2^32)
  Key is assigned to nearest server clockwise
  
  Adding server: Only keys between previous and new server moved
  Removing server: Only keys on removed server moved to next server
  
  Virtual nodes: Each physical server has multiple positions on ring
  → Better load distribution

Used by: Cassandra, Amazon DynamoDB, Memcached clusters
```

---

## CDN (Content Delivery Network)

```
Without CDN:
  User in Tokyo → request to US server → 200ms latency

With CDN:
  User in Tokyo → CDN Edge in Tokyo → 10ms latency (if cached)
              → Cache miss → origin US server → cache content → serve

CDN use cases:
  - Static files: JS, CSS, images, fonts
  - Large files: videos, downloads
  - Dynamic content: some CDNs support edge computing

Cache control headers:
  Cache-Control: max-age=31536000 (1 year for versioned assets)
  Cache-Control: no-cache (check with server on every request)
  ETag: "version-hash" (conditional caching)
```
