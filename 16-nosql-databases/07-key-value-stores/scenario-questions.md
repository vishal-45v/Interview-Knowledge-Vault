# Chapter 07: Key-Value Stores — Scenario Questions

1. Your team runs a Memcached cluster (10 nodes) for session caching at 500K req/sec.
   A node fails and the client-side consistent hashing routes ~10% of traffic to adjacent
   nodes. Those nodes, now receiving extra load, start evicting LRU entries, which
   increases cache miss rate, which increases database load, which slows responses, which
   causes more requests to pile up. The system cascades into a brownout. Describe this
   failure mode by name, explain why consistent hashing alone doesn't prevent it, and
   describe your recovery and prevention strategy.

2. Your e-commerce application uses Redis for product catalog caching. A flash sale event
   starts at noon — 200,000 users simultaneously request the same set of 50 product pages.
   All 50 cache keys have TTL of exactly 1 hour and all expired at 11:59:58 AM. You observe
   a massive spike in database queries (cache stampede) and your PostgreSQL primary falls
   over. How do you fix this for the immediate incident, and how do you redesign the caching
   layer to prevent it permanently?

3. Your Kubernetes cluster uses etcd as its control plane store. During a routine upgrade,
   one of your three etcd nodes fails to restart due to a data corruption issue. Your cluster
   has approximately 50,000 keys and 2GB of etcd data. Walk through your recovery procedure,
   including how you determine if you still have quorum, how you restore from backup, and
   what Kubernetes components might be in an inconsistent state after the etcd restore.

4. You are implementing distributed leader election for a payment processing service using
   ZooKeeper. The election must tolerate network partitions, guarantee exactly one active
   leader at a time, and handle "zombie leaders" — processes that believe they are leader
   after a network partition heals. Describe your complete implementation using ephemeral
   sequential znodes and explain each edge case.

5. Your Redis Cluster (6 shards, 3 replicas each) is used for a rate limiter processing
   2M requests/sec. You notice that all traffic for customer ID ranges starting with "A"
   through "D" maps to the same hash slot range, causing one shard to receive 40% of all
   traffic while others sit at 10%. How do you diagnose this hotspot, and what are your
   short-term and long-term remediation options?

6. Your team uses a write-through cache for a financial ledger — every debit/credit is
   written to both Redis and PostgreSQL atomically. During a routine Redis memory expansion,
   you discover that Redis memory is 3x the size of the PostgreSQL database. Investigation
   reveals the write-through policy is caching every intermediate ledger calculation, not
   just the final balances. How do you redesign the caching strategy to be space-efficient
   without sacrificing the read-through benefit for balance queries?

7. You are building a feature flag system using etcd for 500 microservices. Each service
   watches 50 feature flag keys. When a flag changes, all 500 services must receive the
   update within 1 second. You also need to guarantee that no service reads a partially
   updated flag set (atomicity across multiple flags). How do you model this in etcd, and
   how do you use etcd transactions and watches to guarantee consistency?

8. Your system uses Redis as a message queue (LPUSH/BRPOP pattern) for background jobs.
   At peak load, 5,000 jobs/second are pushed to a single queue key. Workers are falling
   behind — the queue depth is growing to 10 million items. Adding more workers only
   marginally helps because they all contend on the same key. How do you redesign the
   queue architecture to scale horizontally while maintaining job ordering guarantees for
   jobs from the same user?

9. You are migrating from Memcached to Redis for a social media platform's session store.
   The platform stores 50 million active sessions, each 2KB in size (100GB total). The
   migration must be zero-downtime. Sessions expire after 30 days of inactivity. Walk
   through your migration strategy, including dual-write, read fallback, validation, and
   the final cutover procedure.

10. Your distributed system uses ZooKeeper for service discovery. Microservices register
    ephemeral znodes on startup and deregister on shutdown. You discover that during a
    rolling deployment, services are briefly registered in both the old and new version,
    causing some requests to route to old service instances after the new version requires
    a different API contract. How do you use ZooKeeper's versioning and conditional
    updates to implement a blue-green service discovery pattern?
