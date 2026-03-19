# Chapter 07: Key-Value Stores — Theory Questions

1. Explain Memcached's slab allocator. How does it allocate memory, what is a "slab class,"
   and why does this design lead to internal fragmentation? How does this compare to
   Redis's memory allocation strategy using jemalloc?

2. What is consistent hashing and why do client-side key-value stores like Memcached use it?
   How does adding or removing a node affect key distribution, and what is the virtual node
   technique for improving balance?

3. Redis is single-threaded for command processing (pre-Redis 6). How does it achieve high
   throughput despite single-threaded I/O? What changed in Redis 6 with I/O threads, and
   what is still single-threaded even in Redis 6?

4. Describe the etcd Raft consensus protocol. What is a "term," what is a "leader election,"
   and how does etcd guarantee that at most one leader exists at any time?

5. What is a lease/TTL in etcd? How is it different from a regular key TTL? Describe the
   process of implementing a distributed lock using etcd leases.

6. Explain etcd's MVCC (Multi-Version Concurrency Control). What is a "revision," how
   does etcd store multiple versions of the same key, and when does compaction occur?

7. What is the etcd Watch API? How does it work internally, and what guarantees does it
   provide about event ordering? How is it used in Kubernetes for controller reconciliation?

8. Describe Apache ZooKeeper's ZAB (ZooKeeper Atomic Broadcast) protocol. How does it
   differ from Raft, and what are its consistency guarantees?

9. What are the three types of znodes in ZooKeeper (persistent, ephemeral, sequential)?
   How does each type behave, and what use case is each designed for?

10. Explain ZooKeeper's watch mechanism. What events trigger watches, are watches
    one-time or persistent, and how do you implement a reliable "watch forever" pattern?

11. Compare the cache-aside pattern to the read-through pattern. When does each fail, and
    what does failure look like from the application's perspective?

12. What is a cache stampede (also called thundering herd)? Describe three distinct
    approaches to preventing it, and explain the probabilistic early expiration technique.

13. What is write-behind (write-back) caching? How does it differ from write-through?
    What are the data loss and consistency risks, and how do you mitigate them?

14. Explain event-driven cache invalidation. How does a CDC (Change Data Capture) pipeline
    from a relational database integrate with Redis to maintain cache consistency?

15. What is the difference between Redis Sentinel and Redis Cluster? What does each solve,
    and can you use them together?

16. Compare etcd and ZooKeeper as distributed coordination systems. Both provide
    linearizable reads, leader election, and distributed locks — where do they differ in
    architecture, performance characteristics, and operational complexity?

17. What is the "cache tag" or "cache versioning" invalidation strategy? How does it allow
    invalidating a group of related cache entries without maintaining explicit key lists?

18. Explain the refresh-ahead cache pattern. What problem does it solve that TTL-based
    expiration cannot, and what are its drawbacks?
