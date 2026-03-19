# Chapter 02 — Redis: Scenario Questions

Real-world engineering scenarios requiring deep Redis knowledge. Answer out loud first.

---

1. **The distributed rate limiter**
   Your API gateway must enforce a per-user rate limit of 1,000 requests per minute, applied
   across 12 API gateway instances running in parallel. A naive per-instance counter would allow
   each instance to permit 1,000 requests, giving 12,000 total — clearly wrong. Design a
   Redis-based rate limiter using atomic operations that enforces the global limit correctly,
   handles the "thundering herd" at the start of each minute window, and fails open gracefully
   if Redis is unavailable. Consider both fixed-window and sliding-window approaches.

2. **The cache stampede**
   Your Redis cache stores computed product recommendations, each with a 5-minute TTL. At
   midnight, 10,000 keys expire simultaneously (they were all populated at the same time during
   a batch job). All 200 application servers hit the database at once to recompute the missing
   entries, causing the database to spike to 100% CPU and cascade into a full outage. What
   Redis patterns and configuration changes prevent this? Discuss at least three strategies.

3. **The Sentinel false failover**
   Your Redis setup is a master + 2 replicas with 3 Sentinel processes. The master is in
   AWS us-east-1 and the Sentinels are distributed: 1 in us-east-1, 1 in eu-west-1, 1 in
   ap-southeast-1. A transient 10-second network blip between regions causes the 2 remote
   Sentinels to lose contact with the master. They vote and promote a replica — but the master
   in us-east-1 is still running and accepting writes. Now you have two masters (split-brain).
   How did this happen? How do you configure Sentinel to avoid it? What is `min-replicas-to-write`
   and how does it help?

4. **The hot key crisis**
   Your Redis Cluster is handling 2 million requests/second. Monitoring shows that shard 7
   is at 100% CPU while all other shards are at 15%. Investigation reveals that a single key
   `"trending:homepage"` is being read 800,000 times per second. This key is in slot 4921,
   which lives on shard 7. How do you solve this without changing the application's key naming?
   What Redis-native and application-layer solutions exist, and what are the trade-offs of each?

5. **The AOF corruption recovery**
   Your Redis instance crashed mid-write during a power failure. On restart, Redis refuses to
   load the AOF file, logging "Bad file format reading the append only file." The business
   needs the data recovered immediately. Walk through the recovery procedure step by step.
   What tool does Redis provide, what does it do to the corrupted file, and what data is
   potentially lost at the end of the recovery?

6. **The Lua script timeout**
   Your team uses a Lua script to implement a complex atomic workflow: read 5 keys, compute
   derived values, write 3 keys, delete 2 keys — all atomically. In production, one invocation
   of the script processes a key set that is unexpectedly large (10,000 items instead of 10).
   The script runs for 8 seconds. What happens to all other Redis clients during those 8
   seconds? What is `lua-time-limit`, what does Redis do when it expires, and how would
   you redesign the workflow to avoid this problem?

7. **The eviction policy selection**
   You run Redis as a session store with 50GB of RAM available. Your dataset is 80GB of session
   objects: 10% of sessions are accessed daily (hot), 30% are accessed weekly, and 60% have
   not been accessed in 30+ days. All sessions have an explicit TTL of 90 days. You are
   currently using `noeviction` and Redis OOMs every week. Walk through the choice of eviction
   policy, what `maxmemory-policy` you would set, whether you would set `maxmemory`, and how
   you would validate that the most important sessions are not being evicted.

8. **The Redis Cluster resharding data loss**
   Your team is resharding Redis Cluster to add two new shards. During the resharding operation,
   a client writes to a key whose slot is currently being migrated. The key ends up on the
   source shard. The migration tool moves it to the destination shard. But then a second
   client reads the key and gets a MOVED redirect — but the key was not there yet. Explain
   the MOVED/ASK protocol during slot migration, what the correct client behaviour should be,
   and why a client that only handles MOVED but not ASK can return wrong results during
   a live resharding.

9. **The Streams consumer crash recovery**
   You have a Redis Stream with 3 consumer groups, each with 5 consumers. One consumer (C3 in
   group "orders") crashes while processing a batch of 100 messages. Those 100 messages are
   now in C3's PEL (Pending Entry List) and are not being processed. The consumer does not
   restart for 2 hours. Design the recovery procedure: how do you detect the stale pending
   messages, how do you reclaim them for processing by another consumer, and how do you
   prevent duplicate processing if C3 restarts and replays its in-memory buffer?

10. **The multi-region Redis architecture**
    You are building a global multiplayer game backend. Players in Asia, Europe, and North
    America must all see consistent leaderboard data with sub-10ms read latency. Writes to
    the leaderboard happen at 50,000 updates/second globally. Design a Redis topology that
    meets these requirements. What consistency model will you accept for the leaderboard?
    How does Redis Cluster's lack of multi-region support affect your design, and what
    commercial or open-source solutions address this?
