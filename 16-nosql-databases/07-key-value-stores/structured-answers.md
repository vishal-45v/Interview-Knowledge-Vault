# Chapter 07: Key-Value Stores — Structured Answers

## Q1: Compare etcd and ZooKeeper as distributed coordination systems. When do you choose each?

**Answer:**

Both etcd and ZooKeeper solve the same core problem: providing strongly consistent,
fault-tolerant coordination primitives for distributed systems. Their architectural
differences matter significantly at scale.

**Consensus protocol:**
- **ZooKeeper**: ZAB (ZooKeeper Atomic Broadcast) — leader-based protocol where the leader
  generates monotonically increasing "zxid" (transaction IDs). More similar to Viewstamped
  Replication than Raft.
- **etcd**: Raft — well-understood, formally proven consensus algorithm. Leader election
  uses randomized timeouts; log entries are committed once a quorum acknowledges.

**Read consistency:**
- **ZooKeeper**: By default, reads are served locally from ANY replica and may be stale.
  For linearizable reads, you must call `sync()` before reading, which adds a round trip.
- **etcd**: By default, reads are linearizable (go through the leader). For faster but
  potentially stale reads, use `serializable` read option.

```python
# etcd: linearizable read (default) vs serializable read
import etcd3

client = etcd3.client()

# Linearizable: guaranteed to see latest committed value (goes to leader)
value, metadata = client.get('/config/timeout', metadata=True)

# Serializable: may read from follower (stale but faster, avoids leader bottleneck)
value, metadata = client.get('/config/timeout',
                              metadata=True,
                              serializable=True)
```

**Data model differences:**
```
ZooKeeper znodes:
/service/          (persistent directory)
├── /service/config  (persistent — survives restart)
├── /service/lock    (persistent parent)
│   └── /service/lock/node-00001  (ephemeral+sequential — disappears on disconnect)
└── /service/members (ephemeral — disappears on disconnect)

etcd keys (flat namespace with prefix convention):
/service/config     → "{'timeout': 30}"
/service/lock/      → lease-attached key
/service/members/node1 → "192.168.1.10:8080"
```

**MVCC and history:**
etcd stores ALL versions of every key (until compaction). This enables:
- Watch from a specific revision (replay missed events)
- Point-in-time reads (read what value was at revision 1000)

ZooKeeper stores only the CURRENT value of each znode.

**Performance comparison:**

| Metric              | etcd        | ZooKeeper   |
|---------------------|-------------|-------------|
| Write throughput    | ~10K ops/s  | ~15K ops/s  |
| Read throughput     | ~50K ops/s  | ~80K ops/s* |
| Watch latency       | ~1ms        | ~5ms        |
| Max data size       | 8GB (rec.)  | No hard limit|
| Max key size        | 1.5MB       | 1MB         |

*ZooKeeper local reads can be very fast but may be stale.

**Decision framework:**
- Choose **etcd** for: Kubernetes (it's the only supported option), new systems, when
  you want strong defaults and simpler ops, when you need watch-from-revision for event
  replay, Go/gRPC ecosystem.
- Choose **ZooKeeper** for: HBase (tightly integrated), Kafka (ZooKeeper mode),
  existing Hadoop ecosystem, when you need the hierarchical namespace model,
  Java/JVM ecosystem.

---

## Q2: Implement a distributed lock using etcd leases with fencing tokens. Explain why a naive implementation is unsafe.

**Answer:**

The fundamental problem with distributed locks: a process can believe it holds a lock
after the lease expires (due to GC pause, OS scheduling, network partition).

**The unsafe naive implementation:**

```python
# UNSAFE: race condition between lock acquisition and action
def unsafe_distributed_lock():
    granted = client.put('/locks/payment', 'node-1', lease=lease)
    # ------------ DANGER ZONE ------------
    # What if GC pauses here for 31 seconds (longer than lease TTL)?
    # Lease expires, another node acquires the lock, THEN this resumes
    # Now TWO nodes think they hold the lock simultaneously
    process_payment()  # DOUBLE PAYMENT possible
```

**Safe implementation with fencing tokens:**

```python
import etcd3
import time
import threading
from contextlib import contextmanager

class EtcdDistributedLock:
    def __init__(self, client: etcd3.Etcd3Client, lock_key: str, ttl: int = 30):
        self.client = client
        self.lock_key = lock_key
        self.ttl = ttl
        self.lease = None
        self._keepalive_thread = None

    @contextmanager
    def acquire(self):
        # Create lease
        self.lease = self.client.lease(self.ttl)

        # Fencing token = the etcd revision at which we acquired the lock
        # Any write to the protected resource must include this token
        # The resource rejects writes with stale tokens
        try:
            # Atomic compare-and-set: only PUT if key doesn't exist
            success, _ = self.client.transaction(
                compare=[
                    self.client.transactions.version(self.lock_key) == 0
                ],
                success=[
                    self.client.transactions.put(
                        self.lock_key,
                        f"holder={self._node_id()},time={time.time()}",
                        lease=self.lease
                    )
                ],
                failure=[]
            )

            if not success:
                raise LockNotAcquiredError(f"Lock {self.lock_key} already held")

            # Get the revision — this is the fencing token
            _, metadata = self.client.get(self.lock_key)
            fencing_token = metadata.mod_revision

            # Start keepalive thread
            self._start_keepalive()

            yield fencing_token  # Caller uses this in all downstream operations

        finally:
            self._stop_keepalive()
            if self.lease:
                self.lease.revoke()  # Immediately release

    def _start_keepalive(self):
        def keepalive():
            for _ in self.client.refresh_lease(self.lease.id):
                time.sleep(self.ttl // 3)

        self._keepalive_thread = threading.Thread(target=keepalive, daemon=True)
        self._keepalive_thread.start()

    def _stop_keepalive(self):
        if self._keepalive_thread:
            self._keepalive_thread.join(timeout=1)

    def _node_id(self):
        import socket
        return socket.gethostname()


# Usage with fencing token
def process_payment_safely(payment_id: str):
    lock = EtcdDistributedLock(client, f"/locks/payment/{payment_id}", ttl=30)

    with lock.acquire() as fencing_token:
        # Pass fencing token to storage layer
        # Storage layer rejects operations with lower token than previously seen
        result = payment_db.debit(
            payment_id=payment_id,
            amount=100.00,
            fencing_token=fencing_token  # Storage uses this to reject stale writes
        )
        return result
```

**Why fencing tokens are necessary:**
Even with keepalive, a GC pause or kernel schedule can suspend your process for longer
than the lease TTL. The lock expires, another process acquires it with a HIGHER revision
number (higher fencing token). Your process resumes and tries to write to the database.
The database sees your OLD fencing token < current token and REJECTS the write.
This prevents the double-write even in the face of partial failures.

---

## Q3: Explain all five caching patterns with their failure modes and when to use each.

**Answer:**

**1. Cache-Aside (Lazy Loading)**
```python
def get_product(product_id: str):
    # 1. Check cache
    cached = redis.get(f"product:{product_id}")
    if cached:
        return json.loads(cached)

    # 2. Cache miss — load from DB
    product = db.find_product(product_id)

    # 3. Populate cache (application's responsibility)
    if product:
        redis.setex(f"product:{product_id}", 3600, json.dumps(product))
    return product

# Failure mode: Cache miss surge after cold start or cache flush
# Mitigation: warm cache before going live (pre-population script)
# Use when: most keys are never read (sparse access patterns)
```

**2. Read-Through**
```python
class ReadThroughCache:
    """Cache handles its own misses — application only calls cache.get()"""
    def get(self, key: str):
        value = self.redis.get(key)
        if value is None:
            # Cache itself calls DB loader function
            value = self.db_loader(key)
            if value:
                self.redis.setex(key, 3600, json.dumps(value))
        return json.loads(value) if value else None

# Failure mode: Cache is a hard dependency — cache down = system down
# Use when: all data is potentially accessed (dense access patterns)
```

**3. Write-Through**
```python
def update_product(product_id: str, data: dict):
    # Write to DB first
    db.update_product(product_id, data)
    # Write to cache immediately (cache is always warm)
    redis.setex(f"product:{product_id}", 3600, json.dumps(data))

# Failure mode: Two writes mean two failure points. DB write ok + cache write fails
#               = stale cache until TTL expires. Use invalidation instead of update.
# Use when: read-heavy workloads that need low read latency immediately after write
```

**4. Write-Behind (Write-Back)**
```python
import queue

write_queue = queue.Queue()

def update_product_write_behind(product_id: str, data: dict):
    # Write to cache ONLY — fast, synchronous
    redis.setex(f"product:{product_id}", 3600, json.dumps(data))
    # Queue DB write for async processing
    write_queue.put((product_id, data, time.time()))
    return  # Return immediately — DB write is async

# Background worker drains the queue
def db_writer_worker():
    while True:
        product_id, data, queued_at = write_queue.get(timeout=5)
        try:
            db.update_product(product_id, data)
        except Exception as e:
            # DANGER: if DB write fails, data is in cache but not DB
            # Need dead-letter queue and alerting
            dead_letter_queue.put((product_id, data, e))

# Failure mode: Process crash before queue drains = DATA LOSS
# Mitigation: Use Redis Streams for durable queue instead of in-memory queue
# Use when: Write throughput >> DB write capacity (IoT, metrics ingestion)
```

**5. Refresh-Ahead**
```python
import threading

def get_with_refresh_ahead(key: str, ttl: int, refresh_at: float = 0.8):
    """
    Proactively refresh cache before expiry.
    refresh_at=0.8 means refresh when 80% of TTL has elapsed.
    """
    value, remaining_ttl = redis.get_with_ttl(key)

    if value and remaining_ttl > 0:
        # Check if we're in the refresh window
        elapsed_fraction = 1.0 - (remaining_ttl / ttl)
        if elapsed_fraction >= refresh_at:
            # Trigger async refresh — serve current value while refreshing
            threading.Thread(
                target=_async_refresh,
                args=(key, ttl),
                daemon=True
            ).start()
        return json.loads(value)

    # Cache miss — synchronous fetch
    return _sync_fetch(key, ttl)

# Failure mode: Refresh thread fails silently; cache item eventually expires
#               and causes synchronous miss. Need proper error tracking.
# Use when: Items are expensive to compute, access patterns are predictable
```

---

## Q4: Explain Memcached's slab allocator and why it causes memory fragmentation.

**Answer:**

Memcached pre-allocates memory in fixed-size "slabs" to avoid the overhead of
malloc/free for every cache operation. Each slab class handles objects within a
specific size range.

**Slab allocator structure:**

```
MEMCACHED MEMORY LAYOUT:
─────────────────────────────────────────────────────────────
Total Memory: 1GB (set with -m 1024)

Slab Class 1: items 0-96 bytes     → 1 slab page (1MB)
  └── 10,922 chunk slots × 96 bytes each

Slab Class 2: items 97-120 bytes   → 1 slab page (1MB)
  └── 8,738 chunk slots × 120 bytes each

Slab Class 3: items 121-152 bytes  → 1 slab page (1MB)
  └── 6,898 chunk slots × 152 bytes each

... (continues with ~1.25x growth factor between classes)

Slab Class 42: items up to 1MB     → 1 slab page (1MB)
  └── 1 chunk slot × 1,048,576 bytes
─────────────────────────────────────────────────────────────
```

**The fragmentation problem:**

```bash
# Inspect Memcached slab statistics
echo "stats slabs" | nc localhost 11211

# Sample output:
STAT 1:chunk_size 96          # Slab class 1: items up to 96 bytes
STAT 1:chunks_per_page 10922
STAT 1:total_pages 5
STAT 1:total_chunks 54610
STAT 1:used_chunks 54000      # Almost full — good utilization
STAT 1:free_chunks 610

STAT 3:chunk_size 152
STAT 3:total_chunks 6898
STAT 3:used_chunks 100        # Mostly empty — wasted memory
STAT 3:free_chunks 6798
```

**Internal fragmentation example:**
- You store a 97-byte item → goes into Slab Class 2 (120-byte slots) → wastes 23 bytes
- You store a 145-byte item → goes into Slab Class 3 (152-byte slots) → wastes 7 bytes
- Worst case: a 97-byte item wastes 23 bytes (24% waste) in its chunk

**The imbalance problem:**
If all your items are 100-byte strings, all memory goes to Slab Class 2. If you later
start storing 200-byte items, Memcached CANNOT reallocate slab pages from Class 2 to
accommodate Class 3. You hit OOM for 200-byte items while 96% of Class 2 sits unused.

```bash
# Fix: slab_reassign (Memcached 1.4.11+) allows moving slab pages between classes
# But it's disruptive — moved chunks are evicted

# Check if rebalancing is needed
echo "stats slabs" | nc localhost 11211 | grep "total_pages\|used_chunks"

# Enable auto-rebalancing
memcached -o slab_reassign,slab_automove=2
# automove=2: aggressive rebalancing, good for variable item size workloads
```

**Redis comparison:**
Redis uses jemalloc, which dynamically allocates and frees individual items. This has
higher per-operation overhead but eliminates the slab imbalance problem. Redis memory
usage is more proportional to actual data size, while Memcached memory usage depends
heavily on the distribution of item sizes matching slab classes.

---

## Q5: Design a rate limiter using Redis that handles distributed traffic, supports burst, and is accurate to within 1%.

**Answer:**

**Sliding window counter using sorted sets:**

```python
import redis
import time
import math

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def is_allowed(user_id: str, limit: int, window_seconds: int) -> tuple[bool, dict]:
    """
    Sliding window rate limiter using Redis sorted set.
    Returns (allowed: bool, metadata: dict)
    """
    now = time.time()
    window_start = now - window_seconds
    key = f"ratelimit:{user_id}"

    pipe = redis_client.pipeline()

    # Remove expired entries outside the window
    pipe.zremrangebyscore(key, '-inf', window_start)

    # Count current requests in window
    pipe.zcard(key)

    # Add current request with score = current timestamp
    # Member = timestamp:random to handle multiple requests in same millisecond
    member = f"{now}:{id(pipe)}"
    pipe.zadd(key, {member: now})

    # Set expiry on the key to auto-cleanup
    pipe.expire(key, window_seconds * 2)

    results = pipe.execute()
    current_count = results[1]  # Count BEFORE adding current request

    allowed = current_count < limit

    # If not allowed, remove the optimistically added entry
    if not allowed:
        redis_client.zrem(key, member)

    return allowed, {
        'limit': limit,
        'remaining': max(0, limit - current_count - (1 if allowed else 0)),
        'reset_at': math.ceil(window_start + window_seconds),
        'retry_after': window_seconds if not allowed else 0
    }


# Token bucket for burst support (allows short bursts above average rate)
RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local capacity = tonumber(ARGV[1])    -- Max tokens (burst capacity)
local rate = tonumber(ARGV[2])         -- Tokens per second (average rate)
local now = tonumber(ARGV[3])          -- Current timestamp
local requested = tonumber(ARGV[4])    -- Tokens needed for this request

local last_ts = tonumber(redis.call('hget', key, 'ts') or now)
local tokens = tonumber(redis.call('hget', key, 'tokens') or capacity)

-- Refill tokens based on elapsed time
local elapsed = now - last_ts
local new_tokens = math.min(capacity, tokens + elapsed * rate)

-- Check if enough tokens available
local allowed = 0
if new_tokens >= requested then
    allowed = 1
    new_tokens = new_tokens - requested
end

-- Store updated state
redis.call('hset', key, 'ts', now, 'tokens', new_tokens)
redis.call('expire', key, math.ceil(capacity / rate) + 10)

return {allowed, math.floor(new_tokens), math.ceil((requested - new_tokens) / rate)}
"""

token_bucket_script = redis_client.register_script(RATE_LIMIT_SCRIPT)

def token_bucket_check(user_id: str, capacity: int, rate: float, tokens_needed: int = 1):
    """
    Token bucket rate limiter.
    capacity = max burst size (e.g., 100 requests)
    rate = sustained rate (e.g., 10 requests/second)
    """
    result = token_bucket_script(
        keys=[f"tokenbucket:{user_id}"],
        args=[capacity, rate, time.time(), tokens_needed]
    )
    allowed, remaining_tokens, retry_after = result
    return bool(allowed), {'remaining': remaining_tokens, 'retry_after': retry_after}


# Distributed rate limiting across multiple Redis Cluster shards
# Problem: keys on different shards → no atomic multi-key ops
# Solution: use hash tags to pin all rate limit keys to same shard
def get_rate_limit_key(user_id: str, resource: str) -> str:
    # {user_id} forces all rate limit keys for this user to same hash slot
    return f"{{{user_id}}}:ratelimit:{resource}"
```

**Accuracy analysis:**
- Sliding window sorted set: exact, but O(log N) per request for ZREMRANGEBYSCORE
- Fixed window counter: O(1) but allows 2x burst at window boundaries
- Token bucket: O(1), allows controlled burst, requires Lua for atomicity
- For 1% accuracy requirement: sliding window is exact, token bucket is exact,
  fixed window can have up to ~100% inaccuracy at window boundaries

**Production considerations:**
```python
# Rate limit response headers (standard X-RateLimit headers)
def apply_rate_limit(request, user_id: str):
    allowed, metadata = is_allowed(user_id, limit=100, window_seconds=60)

    headers = {
        'X-RateLimit-Limit': str(metadata['limit']),
        'X-RateLimit-Remaining': str(metadata['remaining']),
        'X-RateLimit-Reset': str(metadata['reset_at']),
    }

    if not allowed:
        headers['Retry-After'] = str(metadata['retry_after'])
        return 429, headers, {'error': 'Too Many Requests'}

    return None, headers, None  # Proceed
```
