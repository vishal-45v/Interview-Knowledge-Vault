# Chapter 02 — Redis: Follow-up Traps

The sharpest follow-up questions on Redis, with the wrong answer and the correct deep answer.

---

## Trap 1: "You said Redis is single-threaded — so it can only use one CPU core?"

**What most people say:**
"Yes, Redis is single-threaded so you should run multiple instances per machine to use all cores."

**Correct answer:**
Redis is *single-threaded for command execution* but has been multi-threaded for I/O since
Redis 6.0 (`io-threads` configuration). The network I/O (reading from sockets, parsing
protocol, writing responses) can be parallelised across multiple threads, while the actual
command processing remains single-threaded for simplicity and atomicity.

This means:
1. You don't *need* multiple instances to saturate network I/O on a multi-core machine
2. CPU-bound workloads (complex Lua scripts, large LRANGE, SORT commands) are still
   bottlenecked on the single execution thread
3. Redis 6.0+ can push 2-4x more throughput on 10GbE networks with `io-threads 4`

Running multiple instances per host is still valid for: independent key-space isolation,
independent persistence configs, or independent eviction policies — not merely for core
utilisation.

---

## Trap 2: "appendfsync everysec means you can lose at most 1 second of data — right?"

**What most people say:**
"Yes, it fsync's every second, so worst case you lose 1 second."

**Correct answer:**
You can lose **up to 2 seconds** (or more in practice) because:

1. Redis buffers writes in the OS page cache
2. The background thread calls `fsync()` every ~1 second
3. If Redis crashes *just before* an fsync, the unfsynced buffer can contain up to ~1 second
   of writes
4. But if the *OS crashes* (kernel panic, power failure), the OS page cache is also lost,
   meaning any writes between the last completed fsync and the crash are gone
5. Under I/O pressure, `fsync()` can take >1 second, delaying the next scheduled fsync

The `appendfsync always` option calls fsync after *every* write command — this guarantees
at most one command lost, but can reduce throughput to the disk's fsync rate (~3-5k/s on HDD).

```bash
# Check current AOF config
redis-cli CONFIG GET appendfsync
# Output: appendfsync everysec

# Check AOF status
redis-cli DEBUG SLEEP 0  # Wake any blocking
redis-cli INFO persistence
# aof_enabled:1
# aof_rewrite_in_progress:0
# aof_last_bgrewrite_status:ok
# aof_current_size:1048576
# aof_base_size:524288
```

---

## Trap 3: "Redis Sentinel with quorum=2 means 2 Sentinels must agree to failover — so 2 Sentinels is enough?"

**What most people say:**
"Yes, quorum=2 and 2 Sentinel instances should work."

**Correct answer:**
You need **at least 3 Sentinel instances** for safe operation. Here's why:

The quorum in Sentinel is for *detecting* failure (agreeing the master is down). But for
*performing* the failover (promoting a replica), Sentinel needs a **majority of all Sentinel
processes** to authorise the acting Sentinel.

With 2 Sentinels: if one Sentinel goes down, the remaining 1 cannot form a majority of 2 to
authorise itself for failover. The system is completely unable to failover with quorum=2 and
only 2 Sentinels if one Sentinel fails.

With 3 Sentinels + quorum=2: any 2 Sentinels can detect failure (quorum met) AND any 2
Sentinels can authorise failover (majority of 3 = 2). Even if 1 Sentinel dies, the remaining
2 can still detect failure AND authorise failover.

```bash
# sentinel.conf
sentinel monitor mymaster 127.0.0.1 6379 2   # quorum=2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1

# Check Sentinel state
redis-cli -p 26379 SENTINEL masters
redis-cli -p 26379 SENTINEL sentinels mymaster
```

---

## Trap 4: "You can use KEYS * to find all keys matching a pattern in production — it's just one command."

**What most people say:**
"KEYS is fine for small datasets, but I'd use SCAN for large ones."

**Correct answer:**
`KEYS *` on any production Redis instance is almost always wrong, even on "small" datasets.
`KEYS` is O(N) where N is total number of keys in the entire keyspace and it *blocks* the
entire Redis event loop for its duration. On a Redis instance with 10 million keys, `KEYS *`
can block for hundreds of milliseconds, during which NO other client can execute any command.

`SCAN` is the correct alternative: it is cursor-based, returns a small batch per call, does
not block, and is safe for production use. However, SCAN does NOT guarantee atomicity across
the full scan — keys may be returned multiple times (cursor-based hashtable iteration), and
keys added/deleted during a scan may or may not be returned.

```bash
# NEVER in production:
KEYS user:*

# CORRECT: cursor-based SCAN
# Returns at most ~COUNT keys per call (not exact)
SCAN 0 MATCH user:* COUNT 100

# Python equivalent:
import redis
r = redis.Redis()
cursor = 0
while True:
    cursor, keys = r.scan(cursor, match="user:*", count=100)
    for key in keys:
        print(key)
    if cursor == 0:
        break  # Full scan complete
```

---

## Trap 5: "Redis Cluster automatically handles multi-key operations across different slots."

**What most people say:**
"Yes, Redis Cluster is transparent — you use it like a single Redis instance."

**Correct answer:**
Redis Cluster does NOT support multi-key operations (MGET, MSET, SUNION, ZUNIONSTORE, EVAL
with multiple keys, LMOVE between lists, etc.) if the keys are in different hash slots. You
will receive a `CROSSSLOT` error.

```bash
# This fails in Redis Cluster if key1 and key2 are in different slots:
MSET key1 val1 key2 val2
# Error: CROSSSLOT Keys in request don't hash to the same slot

# Solution 1: Hash Tags — force keys to the same slot
MSET {user:42}:profile "..." {user:42}:settings "..."
# Both keys contain {user:42}, so they hash to the same slot

# Solution 2: Client-side scatter/gather
# Application groups keys by slot and issues separate commands per slot

# Solution 3: Lua script (only if all keys in same slot)
EVAL "return redis.call('MGET', KEYS[1], KEYS[2])" 2 {user:42}:a {user:42}:b
```

Hash tags are the most common solution but introduce a new risk: if all your hash tags are
the same (e.g., all keys use `{global}:*`), they all land on one slot → one shard → you've
recreated a single-node bottleneck in a cluster.

---

## Trap 6: "HyperLogLog gives an exact count of unique elements."

**What most people say:**
"HyperLogLog is for counting unique visitors — it's accurate."

**Correct answer:**
HyperLogLog gives a *probabilistic estimate* with a standard error of **±0.81%**. For a
dataset with 1 million unique items, the result may be anywhere from 991,900 to 1,008,100.
It uses ~12KB of memory regardless of cardinality (versus potentially GBs for an exact set).

Key behaviours:
- `PFADD` adds elements (returns 1 if the estimated count changed, 0 otherwise)
- `PFCOUNT` returns the estimate
- `PFMERGE` combines multiple HyperLogLogs
- Individual elements cannot be retrieved (it's a count, not a set)
- Two HyperLogLogs of the same underlying set will give consistent (but approximated) results

```bash
PFADD dau:2024-01-01 user1 user2 user3
PFADD dau:2024-01-01 user1  # duplicate, may not change estimate
PFCOUNT dau:2024-01-01      # Returns ~3 (exact for small sets, approximate for large)

PFMERGE dau:weekly dau:2024-01-01 dau:2024-01-02 dau:2024-01-03
PFCOUNT dau:weekly  # Approximate unique users across 3 days
```

Use cases where ±0.81% error is acceptable: daily active users, unique page views,
unique search queries. Use cases where it is NOT acceptable: financial transaction deduplication,
unique coupon redemption, security audit logs.

---

## Trap 7: "Write-through caching means the cache is always consistent with the database."

**What most people say:**
"Yes — on every write, you update both the cache and the DB, so they're always in sync."

**Correct answer:**
Write-through keeps the cache consistent *after a write*, but there are two failure modes:

1. **Write to cache succeeds, write to DB fails**: The cache has new data, the DB has old data.
   On next restart, the cache is populated from the DB with the stale value. This requires
   transactional write (write cache + DB atomically) or a compensating rollback.

2. **Write to DB succeeds, cache invalidation fails**: Cache has stale data, DB has new data.
   This is the "dual write problem" — without distributed transactions, atomicity across
   cache + DB is impossible.

The most operationally reliable pattern is **cache-aside with short TTLs**: write to DB, then
delete (not update) the cache entry. The next read will miss and repopulate from the DB.
Deleting is safer than setting because it avoids the "set stale value" race condition where
a slow read populates the cache with an old value after a write has already updated DB.

```python
import redis
import psycopg2

r = redis.Redis()

def update_user(user_id: str, data: dict):
    # 1. Write to database (source of truth)
    with psycopg2.connect(...) as conn:
        cursor = conn.cursor()
        cursor.execute("UPDATE users SET ... WHERE id = %s", (user_id,))
        conn.commit()

    # 2. DELETE cache entry (not SET) — avoids stale-set race condition
    r.delete(f"user:{user_id}")
    # Next read will miss and repopulate from DB with fresh data

def get_user(user_id: str) -> dict:
    cached = r.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)
    # Cache miss: read from DB
    user = db_get_user(user_id)
    r.setex(f"user:{user_id}", 300, json.dumps(user))  # 5 min TTL
    return user
```

---

## Trap 8: "You can store large blobs (10MB+ values) in Redis without any issues."

**What most people say:**
"Redis can store any binary value up to 512MB."

**Correct answer:**
Technically yes, Redis supports values up to 512MB. But large values cause severe operational
problems:

1. **Network I/O**: A 10MB GET blocks the Redis I/O thread for the duration of the TCP transfer.
   During this time, other clients' responses are queued.
2. **Replication lag**: Large values are shipped as full blobs during replication. A single
   10MB write can cause replication lag on slow inter-DC links.
3. **RDB snapshot**: If BGSAVE is triggered while a large value is being written, the
   copy-on-write fork will duplicate the entire page containing that value.
4. **Eviction**: LRU/LFU eviction decisions are made at the key level. Evicting a 10MB
   key is expensive in terms of freed memory vs. evicting 10,000 × 1KB keys.
5. **Cluster slot migration**: Moving a slot that contains a 10MB key is slow and can
   cause client-visible latency during resharding.

Best practice: values over 100KB should be stored in object storage (S3, GCS) and Redis
should store only the reference (URL or object ID). Redis is a cache, not a blob store.

---

## Trap 9: "Redis transactions (MULTI/EXEC) are ACID transactions."

**What most people say:**
"Yes — MULTI/EXEC wraps commands in a transaction."

**Correct answer:**
Redis transactions are **NOT ACID** in the relational sense:

- **Atomic**: Yes — all commands in MULTI/EXEC execute atomically (no other commands interleave)
- **Consistent**: Partially — command syntax is validated, but logic errors don't abort the block
- **Isolated**: Yes — the block executes as a unit, but WATCH is needed for optimistic locking
- **Durable**: Only as durable as persistence config (AOF/RDB settings)

The critical difference from ACID: **Redis does not support rollback on command errors**.
If one command in MULTI/EXEC fails at runtime (e.g., INCR on a string), the other commands
still execute. Only queuing errors (syntax errors) abort the transaction.

```bash
MULTI
SET counter 0
INCR counter       # valid
INCR non-integer   # will fail at runtime (if non-integer is a string key)
GET counter
EXEC
# Result: 1) OK, 2) (integer) 1, 3) (error) WRONGTYPE, 4) "1"
# Counter was incremented — no rollback!
```

For true optimistic locking (check-and-set), use WATCH:
```bash
WATCH account:42:balance
current = GET account:42:balance
MULTI
SET account:42:balance (current - 100)  # executes only if balance unchanged since WATCH
EXEC
# Returns nil if another client modified balance:42 since WATCH → retry
```

---

## Trap 10: "Redis Cluster provides automatic load balancing — you don't need to think about key distribution."

**What most people say:**
"Yes, the hash slot algorithm distributes keys evenly."

**Correct answer:**
Redis Cluster assigns hash slots evenly across shards (16,384 / N slots per shard), but this
does not guarantee *key count* or *memory* balance because:

1. Keys have different sizes (a 1KB value and a 1MB value both occupy one slot's worth of keys)
2. Access frequency is not uniform — one slot with a hot key dominates CPU
3. Hash tags bypass the slot distribution algorithm — all `{user:42}:*` keys go to one slot,
   which goes to one shard

Monitoring the distribution requires:
```bash
redis-cli --cluster info 127.0.0.1:7000
# Shows: key count per node, slot count per node, memory per node

# Identify hot keys
redis-cli --hotkeys -p 7000  # requires maxmemory-policy != noeviction
redis-cli MONITOR  # stream all commands — use in dev only, massive overhead
```

To rebalance after distribution skew:
```bash
redis-cli --cluster rebalance 127.0.0.1:7000 --cluster-use-empty-masters
# Moves slots to balance key count, not just slot count
```

---

## Trap 11: "EXPIRE and TTL operate at the key level — you can't set TTL on individual hash fields."

**What most people say:**
"That's a limitation — you need separate string keys if you need per-field TTL."

**Correct answer:**
This was true until **Redis 7.4**, which introduced **hash field expiry** via the commands
`HEXPIRE`, `HEXPIREAT`, `HPERSIST`, and `HTTL`. Before 7.4, setting per-field TTLs required
storing each field as a separate string key (with the corresponding memory overhead and loss
of atomic hash operations).

With Redis 7.4+:
```bash
HSET user:42 name "Alice" email "alice@example.com" session_token "abc123"
HEXPIRE user:42 3600 FIELDS 1 session_token  # Only session_token expires in 1 hour
HTTL user:42 FIELDS 1 session_token          # Returns remaining TTL: 3598
HTTL user:42 FIELDS 1 name                  # Returns -1: no expiry
```

Before 7.4 workaround:
```python
# Use sorted set as a TTL index
import time
r = redis.Redis()
expire_at = time.time() + 3600
r.hset("user:42", "session_token", "abc123")
r.zadd("ttl_index:user:42", {"session_token": expire_at})

# Background job scans ttl_index and deletes expired fields
```
