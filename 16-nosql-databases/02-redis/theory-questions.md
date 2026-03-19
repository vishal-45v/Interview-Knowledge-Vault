# Chapter 02 — Redis: Theory Questions

Questions covering Redis internals, data structures, persistence, clustering, and operations.
No answers provided — use as verbal practice prompts.

---

1. List all eight core Redis data structure types. For each, describe the internal implementation
   Redis uses (e.g., ziplist, listpack, skiplist, hashtable) and the threshold at which Redis
   upgrades from the compact to the full representation.

2. What is the time complexity of ZADD, ZRANGE, ZRANGEBYSCORE, and ZRANK in a Redis Sorted Set?
   Why is a skip list used instead of a balanced BST (like AVL or red-black tree)?

3. Explain Redis persistence modes: RDB snapshots, AOF (Append Only File), RDB+AOF hybrid, and
   no persistence. What are the durability guarantees and recovery time trade-offs of each?

4. What are the three `appendfsync` options for AOF? Which option provides the best durability
   guarantee, which provides the best performance, and what is the actual durability guarantee
   of `appendfsync everysec` (what can you lose)?

5. What is AOF rewrite? Why is it needed, what triggers it, and how does Redis perform it without
   blocking the main thread?

6. Explain Redis replication. What is the role of the replication backlog (`repl-backlog-size`)?
   What is the difference between PSYNC (partial sync) and full resync, and when does each occur?

7. What is Redis Sentinel? List its four responsibilities. What is the quorum configuration in
   Sentinel, and how many Sentinel processes do you need to safely handle a single Redis master
   failure?

8. Explain Redis Cluster hash slot assignment. How many hash slots exist, how is a key mapped to
   a slot, and what is a hash tag and why is it used?

9. What is the difference between a MOVED redirect and an ASK redirect in Redis Cluster?
   In what scenario does an ASK redirect occur?

10. What is Redis Cluster's gossip protocol? What information is exchanged, how often, and what
    is its role in failure detection?

11. Explain the difference between Redis Pub/Sub and Redis Streams. When would you choose one
    over the other? What does Redis Streams provide that Pub/Sub does not?

12. What is a Redis Stream consumer group? Explain the concepts of XREADGROUP, XACK, and
    the PEL (Pending Entry List). What happens to messages in the PEL if a consumer crashes?

13. Explain `EVAL` and Lua scripting in Redis. Why are Lua scripts atomic, and what is the
    implication of a long-running Lua script for other Redis clients?

14. List all Redis eviction policies. Under what workload characteristics would you choose
    `allkeys-lfu` over `allkeys-lru`? What is the `maxmemory-samples` configuration and
    how does it affect eviction accuracy?

15. What is the cache-aside pattern? What is write-through? What is write-behind (write-back)?
    For each, describe the cache invalidation scenario and the risk of stale data.

16. What is a Redis HyperLogLog? What problem does it solve, what is its error rate, and what
    is its memory usage regardless of the cardinality being estimated?

17. What is the Redis OBJECT ENCODING command? Give examples of at least four keys with
    different encodings and explain what determines the encoding chosen.

18. Explain Redis keyspace notifications. What configuration is required to enable them, what
    are the event categories, and what is the performance cost of enabling them?

19. What is RediSearch (now Redis Stack's Search module)? How does it index data, and how does
    it differ from running a KEYS pattern scan for search?

20. What new features did Redis 7.x introduce? Specifically, what is Redis Functions
    (replacing Lua EVAL in some use cases), and what are Redis 7.0's multi-part AOF files?
