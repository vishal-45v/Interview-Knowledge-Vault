# Chapter 02 — Redis: Structured Answers

Complete, interview-ready answers with working code examples.

---

## Q1: Explain Redis data structures with their internal encodings and use cases.

**Answer:**

Redis chooses compact encodings for small collections and upgrades to full encodings as they
grow. This is transparent to clients but critical for memory planning.

```bash
# 1. STRING — binary-safe, max 512MB
SET user:42:name "Alice"              # encoding: embstr (<=44 bytes) or raw
SET counter 100                        # encoding: int (stored as actual integer)
INCR counter                           # atomic: 101
SETEX session:abc 3600 "user_data"    # with TTL
OBJECT ENCODING user:42:name          # "embstr"

# 2. LIST — ordered, allows duplicates
LPUSH queue:emails "msg1" "msg2"      # push to left
RPOP  queue:emails                    # pop from right (FIFO queue)
LRANGE mylist 0 -1                    # all elements
# Encoding: listpack (<=128 elements, <=64 bytes each) → quicklist otherwise

# 3. HASH — field-value pairs
HSET user:42 name "Alice" age 30 city "NYC"
HGET user:42 name                     # "Alice"
HGETALL user:42                       # all fields
HINCRBY user:42 age 1                 # atomic increment of a field
# Encoding: listpack (<=128 fields, <=64 bytes each) → hashtable otherwise

# 4. SET — unordered unique elements
SADD tags:post:1 "python" "redis" "nosql"
SMEMBERS tags:post:1
SINTER tags:post:1 tags:post:2        # intersection
SUNIONSTORE dest tags:post:1 tags:post:2  # union to dest key
# Encoding: listpack (<=128 members, integer-only small sets) → hashtable otherwise

# 5. SORTED SET (ZSET) — unique members with float scores
ZADD leaderboard 9500 "alice" 8200 "bob" 9900 "carol"
ZRANGE leaderboard 0 -1 WITHSCORES REV  # top 3 by score descending
ZRANGEBYSCORE leaderboard 8000 10000    # range query
ZRANK leaderboard "alice"               # 0-based rank
ZINCRBY leaderboard 100 "alice"         # increment score atomically
# Encoding: listpack (<=128 members, <=64 bytes) → skiplist+hashtable otherwise

# 6. BITMAP — bit array operations
SETBIT daily_active:2024-01-01 user_id 1  # user_id=42 was active
GETBIT daily_active:2024-01-01 42
BITCOUNT daily_active:2024-01-01          # count active users
BITOP AND result day1 day2                 # users active on both days

# 7. HYPERLOGLOG — probabilistic cardinality estimation
PFADD uv:2024-01-01 "user1" "user2" "user3"
PFCOUNT uv:2024-01-01   # ~3 (exact for small, ±0.81% for large)
PFMERGE uv:week uv:day1 uv:day2 uv:day3

# 8. STREAM — append-only log with consumer groups
XADD events * action "login" user_id "42"     # auto-generate ID
XADD events 1704067200000-0 action "purchase" # explicit ID
XREAD COUNT 10 STREAMS events 0               # read from beginning
XLEN events                                    # stream length
```

**Memory planning rule of thumb:**
Always set `hash-max-listpack-entries`, `hash-max-listpack-value`, and equivalent for other
types to keep collections in compact encoding as long as your use case allows.

---

## Q2: Explain Redis persistence: RDB vs AOF vs Hybrid.

**Answer:**

```bash
# ─── RDB (Redis Database Snapshot) ─────────────────────────────────────────
# redis.conf settings:
save 900 1      # save after 900s if ≥1 key changed
save 300 10     # save after 300s if ≥10 keys changed
save 60 10000   # save after 60s if ≥10000 keys changed
dbfilename dump.rdb
dir /var/lib/redis

# Manual trigger (non-blocking, fork + write):
BGSAVE
# Blocking trigger (do not use in production):
SAVE

# ─── AOF (Append Only File) ─────────────────────────────────────────────────
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec   # options: always | everysec | no

# AOF rewrite (compaction — eliminates redundant commands):
BGREWRITEAOF
# Automatic rewrite triggers:
auto-aof-rewrite-percentage 100    # rewrite when AOF is 100% larger than base
auto-aof-rewrite-min-size 64mb     # only if AOF is at least 64MB

# ─── RDB + AOF Hybrid (Redis 4.0+, default in Redis 7+) ────────────────────
aof-use-rdb-preamble yes
# On AOF rewrite: writes RDB snapshot as preamble, then AOF delta on top
# Faster restart (parse RDB, not thousands of SET commands) + incremental AOF

# ─── No Persistence ──────────────────────────────────────────────────────────
save ""          # disable all RDB saves
appendonly no    # disable AOF
# Use case: pure cache where data loss on restart is acceptable
```

**Recovery time comparison:**

```python
# Approximate startup times for 10GB dataset:
startup_times = {
    "RDB only":          "10-30 seconds (parse binary snapshot)",
    "AOF only":          "3-10 minutes (replay all commands)",
    "RDB+AOF hybrid":    "15-45 seconds (parse RDB preamble + short AOF delta)",
    "No persistence":    "< 1 second (empty keyspace)",
}

# Durability comparison:
durability = {
    "RDB every 60s":     "Can lose up to 60 seconds of writes",
    "AOF everysec":      "Can lose up to ~2 seconds of writes",
    "AOF always":        "Can lose at most 1 command (but ~3-5k writes/sec max)",
    "No persistence":    "Lose everything on restart",
}
```

**Interview key point**: AOF `always` mode performance depends entirely on disk fsync speed.
On an NVMe SSD, this can be 100k+ commands/sec. On a network-attached volume (EBS gp2),
it can drop to 3,000 commands/sec — a 30x performance difference.

---

## Q3: Design a Redis-based distributed rate limiter with sliding window.

**Answer:**

```python
import redis
import time
from typing import Tuple

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

# ─── Fixed Window Counter (simple, but allows 2x burst at window boundary) ──
def rate_limit_fixed_window(
    user_id: str,
    limit: int = 1000,
    window_seconds: int = 60
) -> Tuple[bool, int]:
    """Returns (allowed, remaining_requests)."""
    window_key = int(time.time()) // window_seconds
    key = f"rl:fixed:{user_id}:{window_key}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, window_seconds * 2)  # keep for 2 windows
    count, _ = pipe.execute()

    remaining = max(0, limit - count)
    return count <= limit, remaining


# ─── Sliding Window Log (exact, but memory-intensive) ───────────────────────
def rate_limit_sliding_log(
    user_id: str,
    limit: int = 1000,
    window_seconds: int = 60
) -> Tuple[bool, int]:
    """Stores every request timestamp in a sorted set."""
    now = time.time()
    window_start = now - window_seconds
    key = f"rl:log:{user_id}"

    with r.pipeline() as pipe:
        pipe.zremrangebyscore(key, 0, window_start)   # remove old entries
        pipe.zadd(key, {str(now): now})               # add this request
        pipe.zcard(key)                               # count in window
        pipe.expire(key, window_seconds + 1)
        _, _, count, _ = pipe.execute()

    return count <= limit, max(0, limit - count)


# ─── Sliding Window Counter (approximate, memory-efficient) ─────────────────
def rate_limit_sliding_counter(
    user_id: str,
    limit: int = 1000,
    window_seconds: int = 60
) -> Tuple[bool, int]:
    """
    Approximation using two fixed windows:
    current_count + previous_count * (elapsed_fraction)
    """
    now = time.time()
    current_window = int(now) // window_seconds
    prev_window = current_window - 1
    elapsed_in_current = (now % window_seconds) / window_seconds

    curr_key = f"rl:sw:{user_id}:{current_window}"
    prev_key = f"rl:sw:{user_id}:{prev_window}"

    with r.pipeline() as pipe:
        pipe.incr(curr_key)
        pipe.expire(curr_key, window_seconds * 2)
        pipe.get(prev_key)
        curr_count, _, prev_count = pipe.execute()

    prev_count = int(prev_count or 0)
    # Weight previous window by remaining fraction
    weighted = int(prev_count * (1 - elapsed_in_current)) + curr_count
    return weighted <= limit, max(0, limit - weighted)


# ─── Token Bucket via Lua (most production-accurate) ────────────────────────
TOKEN_BUCKET_SCRIPT = """
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])  -- tokens per second
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

-- Refill tokens based on elapsed time
local elapsed = math.max(0, now - last_refill)
tokens = math.min(capacity, tokens + elapsed * refill_rate)

local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) * 2)

return {allowed, math.floor(tokens)}
"""

token_bucket_sha = r.script_load(TOKEN_BUCKET_SCRIPT)

def rate_limit_token_bucket(
    user_id: str,
    capacity: int = 1000,
    refill_rate: float = 16.67,  # tokens/sec = 1000/60
) -> Tuple[bool, int]:
    key = f"rl:bucket:{user_id}"
    now = time.time()
    allowed, remaining = r.evalsha(
        token_bucket_sha, 1, key, capacity, refill_rate, now, 1
    )
    return bool(allowed), remaining
```

---

## Q4: Explain Redis Sentinel vs Redis Cluster — when to use each.

**Answer:**

```
Redis Sentinel:
- Single master + N replicas
- Sentinel monitors master health
- Automatic failover: promotes a replica to master
- No data sharding — all data on one master
- Use when: dataset fits in single node, need HA but not horizontal scale
- Typical: cache layers, session stores under 50GB

Redis Cluster:
- N master shards, each with M replicas
- 16,384 hash slots distributed across masters
- Automatic failover per shard
- Horizontal scale: add shards to grow capacity
- Use when: dataset > single node, write throughput > single node
- Typical: large caches, real-time data stores
```

```python
# Redis Sentinel client configuration (Python redis-py)
from redis.sentinel import Sentinel

sentinel = Sentinel(
    [
        ("sentinel-1.example.com", 26379),
        ("sentinel-2.example.com", 26379),
        ("sentinel-3.example.com", 26379),
    ],
    socket_timeout=0.5,
)

# Sentinel client automatically routes to master for writes
master = sentinel.master_for("mymaster", socket_timeout=0.5, decode_responses=True)
# Routes to replica for reads (potentially stale)
replica = sentinel.slave_for("mymaster", socket_timeout=0.5, decode_responses=True)

master.set("key", "value")
value = replica.get("key")  # may be slightly stale
```

```python
# Redis Cluster client configuration
from redis.cluster import RedisCluster, ClusterNode

startup_nodes = [
    ClusterNode("redis-cluster-1.example.com", 7000),
    ClusterNode("redis-cluster-2.example.com", 7001),
]

rc = RedisCluster(startup_nodes=startup_nodes, decode_responses=True)

# Cluster client handles MOVED/ASK redirects automatically
rc.set("user:42", "alice")   # routes to correct shard
rc.get("user:42")

# Hash tags force keys to same slot
rc.mset({"user:{42}:name": "Alice", "user:{42}:age": "30"})  # same slot
```

**Key operational differences:**

| Aspect | Sentinel | Cluster |
|--------|----------|---------|
| Max write throughput | Single master limit | N masters × single master limit |
| Max dataset size | Single master RAM | N masters × single master RAM |
| Failover granularity | Whole master fails over | Per-shard failover |
| Multi-key atomicity | Full support | Only within same slot / hash tag |
| Client complexity | Simple sentinel-aware client | Cluster-aware client required |
| Resharding | N/A | Online resharding (with ASK redirects) |

---

## Q5: Implement Redis Streams with consumer groups for reliable event processing.

**Answer:**

```python
import redis
import json
import time
from typing import Optional

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

STREAM_KEY = "orders:events"
GROUP_NAME = "order-processor"
CONSUMER_NAME = "worker-1"
PENDING_TIMEOUT_MS = 30_000  # 30 seconds

def setup_consumer_group():
    """Create the consumer group (idempotent)."""
    try:
        r.xgroup_create(STREAM_KEY, GROUP_NAME, id="0", mkstream=True)
        print(f"Consumer group '{GROUP_NAME}' created")
    except redis.ResponseError as e:
        if "BUSYGROUP" in str(e):
            print(f"Consumer group '{GROUP_NAME}' already exists")
        else:
            raise

def produce_order_event(order_id: str, event_type: str, payload: dict) -> str:
    """Add event to stream, returns auto-generated message ID."""
    message_id = r.xadd(STREAM_KEY, {
        "order_id": order_id,
        "event_type": event_type,
        "payload": json.dumps(payload),
        "timestamp": str(time.time()),
    })
    return message_id

def process_messages(batch_size: int = 10):
    """
    Read and process messages, acknowledging only after successful processing.
    Uses XREADGROUP with '>' to read only new (undelivered) messages.
    """
    messages = r.xreadgroup(
        GROUP_NAME,
        CONSUMER_NAME,
        {STREAM_KEY: ">"},   # '>' = only new messages for this group
        count=batch_size,
        block=5000,          # block for 5 seconds if no messages
    )

    if not messages:
        return 0

    processed = 0
    for stream_name, entries in messages:
        for message_id, data in entries:
            try:
                handle_order_event(message_id, data)
                # ACK removes message from PEL (Pending Entry List)
                r.xack(STREAM_KEY, GROUP_NAME, message_id)
                processed += 1
            except Exception as e:
                print(f"Failed to process {message_id}: {e}")
                # Do NOT ack — message stays in PEL for recovery

    return processed

def handle_order_event(message_id: str, data: dict):
    """Business logic — process one order event."""
    event_type = data.get("event_type")
    payload = json.loads(data.get("payload", "{}"))
    print(f"Processing [{message_id}] event_type={event_type} payload={payload}")
    # ... actual processing logic

def recover_pending_messages(older_than_ms: int = PENDING_TIMEOUT_MS):
    """
    Reclaim messages stuck in PEL (dead consumer's unacked messages).
    Uses XAUTOCLAIM (Redis 7.0+) or XCLAIM (older).
    """
    # XAUTOCLAIM: atomically reclaim messages idle > older_than_ms
    result = r.xautoclaim(
        STREAM_KEY,
        GROUP_NAME,
        CONSUMER_NAME,
        older_than_ms,
        "0-0",           # start from beginning of PEL
        count=100,
    )
    next_cursor, reclaimed, deleted = result

    if reclaimed:
        print(f"Reclaimed {len(reclaimed)} pending messages")
        for message_id, data in reclaimed:
            try:
                handle_order_event(message_id, data)
                r.xack(STREAM_KEY, GROUP_NAME, message_id)
            except Exception as e:
                print(f"Recovery failed for {message_id}: {e}")

def inspect_stream_health():
    """Operational health check for the stream."""
    info = r.xinfo_groups(STREAM_KEY)
    for group in info:
        print(f"Group: {group['name']}")
        print(f"  Pending:       {group['pending']}")
        print(f"  Last delivered:{group['last-delivered-id']}")

    # List consumers and their pending counts
    consumers = r.xinfo_consumers(STREAM_KEY, GROUP_NAME)
    for c in consumers:
        print(f"Consumer: {c['name']}, pending={c['pending']}, idle={c['idle']}ms")

    # Stream length
    print(f"Stream length: {r.xlen(STREAM_KEY)}")

# ─── Main processing loop ────────────────────────────────────────────────────
if __name__ == "__main__":
    setup_consumer_group()

    # Produce some events
    for i in range(5):
        mid = produce_order_event(f"order-{i}", "OrderPlaced", {"items": i})
        print(f"Produced: {mid}")

    # Process events
    while True:
        count = process_messages(batch_size=10)
        if count > 0:
            print(f"Processed {count} messages")
        # Periodically recover stuck messages
        recover_pending_messages()
```

---

## Q6: Explain Redis eviction policies with production tuning guidance.

**Answer:**

```bash
# Set maximum memory (must be set for eviction to work)
CONFIG SET maxmemory 4gb
CONFIG SET maxmemory-policy allkeys-lfu

# Eviction policies:
# noeviction       — Return error when memory full. Use for persistent stores where
#                    you cannot afford to lose data.
# allkeys-lru      — Evict least recently used key (from all keys). Use for general
#                    caches where any key might be evicted.
# volatile-lru     — Evict LRU key among keys WITH TTL set. Use when some keys are
#                    critical (no TTL) and others are cache-only (with TTL).
# allkeys-lfu      — Evict least frequently used key (from all keys). Better than LRU
#                    for skewed access patterns where "recently used" ≠ "important".
# volatile-lfu     — Evict LFU key among keys with TTL.
# allkeys-random   — Evict random key. Rarely the right choice.
# volatile-random  — Evict random key with TTL.
# volatile-ttl     — Evict key with nearest expiry first. Good for TTL-based caches
#                    where you want to preserve longer-lived items.

# LRU/LFU approximation accuracy (default=5, higher=more accurate but more CPU)
CONFIG SET maxmemory-samples 10
```

```python
import redis
import time

r = redis.Redis()

def setup_tiered_cache():
    """
    Demonstrate volatile-lru: critical keys (no TTL) vs cache keys (with TTL).
    """
    # Critical config — no TTL, will NEVER be evicted by volatile-lru
    r.set("config:feature_flags", '{"new_ui": true}')
    r.set("config:api_keys", '["key1", "key2"]')

    # User session cache — with TTL, eligible for eviction
    for user_id in range(10000):
        r.setex(f"session:{user_id}", 3600, f"session_data_{user_id}")

    # Product cache — with TTL, eligible for eviction
    for product_id in range(50000):
        r.setex(f"product:{product_id}", 600, f"product_data_{product_id}")

def monitor_evictions():
    """Check eviction stats."""
    info = r.info("stats")
    print(f"Evicted keys total:    {info['evicted_keys']}")
    print(f"Keyspace hits:         {info['keyspace_hits']}")
    print(f"Keyspace misses:       {info['keyspace_misses']}")
    hit_rate = info['keyspace_hits'] / (info['keyspace_hits'] + info['keyspace_misses'])
    print(f"Cache hit rate:        {hit_rate:.2%}")

    # LFU-specific: check object frequency counter
    r.set("hot_key", "frequently_accessed")
    for _ in range(1000):
        r.get("hot_key")
    print(f"hot_key frequency: {r.object('freq', 'hot_key')}")
```

**Production rule of thumb:**
- Session/user data cache with access skew → `allkeys-lfu`
- Full-page or API response cache with even access → `allkeys-lru`
- Mixed dataset with critical + ephemeral keys → `volatile-lru` or `volatile-lfu`
- Sentinel/persistence store where loss is unacceptable → `noeviction` + alert on memory
