# Chapter 07: Key-Value Stores — Follow-Up Traps

## Trap 1: "Redis is single-threaded so it can't be faster than Memcached"

**What most people say:**
"Memcached is multi-threaded, so it uses all CPU cores and should be faster."

**Correct answer:**
Redis is single-threaded for COMMAND PROCESSING, which is actually an advantage for
throughput predictability. Redis achieves high throughput through:
1. Epoll-based I/O multiplexing — a single thread handles thousands of connections
2. In-memory operations — commands are nanosecond-fast, not I/O-bound
3. No locking overhead — single thread means no mutex contention

Memcached's multi-threading helps on multi-core machines for network I/O, but adds
lock contention on shared data structures. Benchmarks show Redis often matches or
exceeds Memcached for single-value operations, while Memcached edges ahead only for
very large objects (>1MB) where network I/O truly dominates.

Redis 6+ added I/O threads for network read/write, but command execution remains
single-threaded:

```bash
# Redis 6 config — enable I/O threads (2-4 is typical; diminishing returns beyond 4)
io-threads 4
io-threads-do-reads yes  # Thread reads too, not just writes

# Verify threading behavior
redis-cli INFO server | grep io_threads_active
```

The real Memcached advantage: it uses less memory per key (no data structure overhead)
and has a simpler codebase. Choose Memcached when you need pure string caching with
minimal operational complexity. Choose Redis when you need persistence, Lua scripts,
pub/sub, streams, or complex data structures.

## Trap 2: "etcd is just a distributed key-value store like Redis"

**What most people say:**
"etcd stores key-value pairs and supports TTL — it's basically Redis with Raft."

**Correct answer:**
etcd is a COORDINATION system, not a caching system. The differences are fundamental:

1. **Consistency model**: etcd provides linearizable reads by default (reads go through
   the Raft leader, guaranteeing you see the latest committed value). Redis with replication
   provides eventual consistency.

2. **Watch semantics**: etcd watches are reliable — they deliver ALL changes in order and
   never miss an event (guaranteed by Raft log). Redis keyspace notifications can be missed.

3. **Transaction model**: etcd supports compare-and-set transactions with multi-key atomicity.
   Redis has MULTI/EXEC but without compare semantics in the same way.

4. **Scale**: etcd is designed for small datasets (< 8GB recommended, < 1000 keys per watch).
   Redis scales to hundreds of GBs. Using etcd as a general cache will corrupt your cluster.

```python
import etcd3

client = etcd3.client()

# etcd CAS transaction — compare-and-swap with multi-key atomicity
# This is fundamentally different from Redis MULTI/EXEC
etcd_transaction = client.transaction(
    compare=[
        client.transactions.value('/service/leader') == b'node-1',  # Precondition
    ],
    success=[
        client.transactions.put('/service/leader', 'node-2'),       # If true
        client.transactions.put('/service/leader_ts', str(time.time())),
    ],
    failure=[
        client.transactions.get('/service/leader'),                  # If false, return current
    ]
)
success, responses = etcd_transaction
# This is linearizable — other nodes will see this atomically
```

## Trap 3: "ZooKeeper watches fire exactly once — just re-register after receiving one"

**What most people say:**
"You register a watch, get notified once, then re-register. Simple."

**Correct answer:**
The one-time nature of watches creates a TOCTOU (time-of-check-time-of-use) window.
Between receiving a watch event and re-registering the watch, changes can occur that
you miss. The correct pattern is:

```python
from kazoo.client import KazooClient
from kazoo.recipe.watchers import DataWatcher

zk = KazooClient(hosts='127.0.0.1:2181')
zk.start()

# WRONG: watch fires once, re-registration has a gap
def bad_watch(event):
    data, stat = zk.get('/config/feature_flag', watch=bad_watch)
    print(f"Value changed: {data}")

# CORRECT: read-then-watch in a loop with version check
def safe_watch(event):
    # After receiving event, get current data AND re-register watch atomically
    data, stat = zk.get('/config/feature_flag', watch=safe_watch)
    current_version = stat.version
    # Process data with awareness that it's already the NEW value
    print(f"Updated to version {current_version}: {data}")

# Start the watch
data, stat = zk.get('/config/feature_flag', watch=safe_watch)

# Kazoo's DataWatcher handles this properly (persistent watch abstraction)
@zk.DataWatch('/config/feature_flag')
def watch_config(data, stat, event):
    if data is not None:
        print(f"Config: {data.decode()}, version: {stat.version}")
    # Kazoo automatically re-registers — no gap
```

ZooKeeper 3.6+ added persistent watches (`addWatch` API) that fire on every change
without requiring re-registration, eliminating the TOCTOU window.

## Trap 4: "Cache-aside and read-through are basically the same pattern"

**What most people say:**
"Both patterns check cache first, then load from DB on miss — they're the same."

**Correct answer:**
The key difference is WHERE THE CACHE POPULATION LOGIC LIVES:

- **Cache-aside**: Application code handles cache misses. If cache is down, app reads
  directly from DB. Cache is an optimization, not required for operation.
- **Read-through**: Cache handles its own misses by fetching from DB. Application only
  talks to cache. If cache is down, application is down.

```python
# Cache-aside: application manages the cache
def get_user_cache_aside(user_id: str) -> dict:
    # Application checks cache
    cached = redis_client.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    # Application queries DB on miss (cache is transparent)
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    if user:
        # Application populates cache
        redis_client.setex(f"user:{user_id}", 3600, json.dumps(user))
    return user

# Read-through: cache handles its own misses (e.g., using Redis + custom loader)
# Application only calls cache.get() — never calls DB directly
class ReadThroughCache:
    def get(self, key: str):
        value = self.redis.get(key)
        if value is None:
            # Cache fetches from DB and populates itself
            value = self._load_from_db(key)
            self.redis.set(key, value, ex=3600)
        return value
```

The failure mode difference:
- Cache-aside failure: cache down → all reads hit DB → DB overload (graceful degradation)
- Read-through failure: cache down → ALL requests fail (cache is in critical path)

Read-through is simpler for application code but creates a harder operational dependency.

## Trap 5: "Adding more Redis Cluster nodes always improves performance"

**What most people say:**
"More shards = more distributed load = better performance."

**Correct answer:**
Adding shards introduces CROSS-SLOT OPERATION OVERHEAD. Redis Cluster prohibits
multi-key operations across different hash slots. Operations like MGET, SUNION, and
Lua scripts that touch keys in different slots fail with a CROSSSLOT error:

```bash
# This FAILS in Redis Cluster if keys hash to different slots
redis-cli MGET user:1001 user:1002 user:1003
# Error: CROSSSLOT Keys in request don't hash to the same slot

# SOLUTION 1: Hash tags force related keys to the same slot
# Keys with {user_id} in the name hash only on the part in braces
redis-cli MGET {user:1001}:profile {user:1001}:settings {user:1001}:cache
# All hash to slot of "user:1001" — guaranteed same shard

# SOLUTION 2: Pipeline individual GETs (more round trips but works)
pipeline = redis_client.pipeline()
for user_id in [1001, 1002, 1003]:
    pipeline.get(f"user:{user_id}")
results = pipeline.execute()

# SOLUTION 3: For atomic multi-key ops, use a single-node Redis with Sentinel
# and accept the single-machine write throughput ceiling
```

The performance cliff: if your hottest keys all hash to the same slot (even with many
shards), you get no distribution benefit. Always analyze your key distribution before
sharding.

## Trap 6: "Write-through caching guarantees strong consistency"

**What most people say:**
"Write-through writes to both cache and DB atomically, so they're always in sync."

**Correct answer:**
Write-through is NOT atomic — it's two separate operations. The cache and DB can
diverge if either write fails:

```python
import redis
import psycopg2

def update_user_write_through(user_id: str, data: dict):
    # Step 1: Write to DB
    with db.cursor() as cur:
        cur.execute("UPDATE users SET name=%s WHERE id=%s",
                    (data['name'], user_id))
        db.commit()
    # --- FAILURE WINDOW: DB updated, cache not yet ---
    # If Redis is down or the process crashes here, cache is stale

    # Step 2: Write to cache
    redis_client.setex(f"user:{user_id}", 3600, json.dumps(data))
    # If this fails, cache has stale data until TTL expires

# SAFER PATTERN: Write DB first, then INVALIDATE (not update) cache
def update_user_safe(user_id: str, data: dict):
    with db.cursor() as cur:
        cur.execute("UPDATE users SET name=%s WHERE id=%s",
                    (data['name'], user_id))
        db.commit()
    # DELETE forces next read to repopulate from DB (fresh data)
    # Worst case: cache miss → DB read (acceptable)
    # Never: stale cache entry (unacceptable)
    redis_client.delete(f"user:{user_id}")
```

Cache invalidation (delete) is SAFER than cache update in write-through because a
delete causes a cache miss (performance degradation, acceptable), while a failed cache
update causes stale data (correctness issue, unacceptable).

## Trap 7: "etcd lease TTL is the same as a key TTL"

**What most people say:**
"You set a TTL on the key in etcd, same as Redis EXPIRE."

**Correct answer:**
etcd leases are a SEPARATE OBJECT that keys are ATTACHED TO. Multiple keys can share
one lease. When the lease expires, ALL attached keys are deleted atomically. The leaseholder
must actively renew (keepalive) the lease, or it expires automatically.

```python
import etcd3
import threading

client = etcd3.client()

# Create a lease with 30-second TTL
lease = client.lease(30)

# Attach multiple keys to the same lease
client.put('/service/node1/host', '192.168.1.10', lease=lease)
client.put('/service/node1/port', '8080',          lease=lease)
client.put('/service/node1/version', 'v2.1',       lease=lease)

# All three keys are deleted atomically when the lease expires
# This is CRUCIAL for service discovery — partial registration is impossible

# Keep-alive in background (must run continuously while service is alive)
def keepalive_loop(lease):
    for response in client.refresh_lease(lease.id):
        # Renew every 10 seconds (< 30-second TTL)
        # If this thread dies, the lease expires and all keys are deleted
        pass

keepalive_thread = threading.Thread(target=keepalive_loop, args=(lease,))
keepalive_thread.daemon = True
keepalive_thread.start()

# When the service shuts down gracefully
lease.revoke()  # Immediately deletes all attached keys
```

The lease model provides a crucial safety property for distributed systems: a service
that crashes will automatically deregister ALL its keys when its lease expires — no
manual cleanup required. A key TTL (like Redis EXPIRE) is per-key and cannot be renewed
collectively.

## Trap 8: "Probabilistic early expiration is complex and not worth implementing"

**What most people say:**
"Just use a mutex lock to prevent stampede — one request rebuilds the cache, others wait."

**Correct answer:**
Mutex lock prevents stampede but introduces its own problem: a slow cache-rebuild makes
ALL requests for that key wait, creating a different kind of latency spike. Probabilistic
early expiration eliminates this by rebuilding in the background BEFORE expiration:

```python
import time
import math
import random

def fetch_with_probabilistic_early_expiration(
    key: str,
    ttl: int,
    beta: float = 1.0
) -> dict:
    """
    XFetch algorithm: probabilistically trigger early re-fetch
    before cache expires to avoid stampede.
    """
    cached = redis_client.hgetall(f"cache:{key}")

    if cached:
        expiry_time = float(cached['expiry'])
        delta = float(cached['delta'])  # Time it took to fetch from DB last time

        # Probability of re-fetching increases as expiry approaches
        # Formula: current_time - delta * beta * log(random()) > expiry
        should_early_fetch = (
            time.time() - delta * beta * math.log(random.random()) > expiry_time
        )

        if not should_early_fetch:
            return json.loads(cached['value'])

    # Fetch from DB (either cache miss or probabilistic early re-fetch)
    start = time.time()
    fresh_data = db.fetch(key)
    delta = time.time() - start  # How long DB fetch takes

    # Store with TTL + metadata for next probabilistic calculation
    redis_client.hset(f"cache:{key}", mapping={
        'value': json.dumps(fresh_data),
        'expiry': time.time() + ttl,
        'delta': delta
    })
    redis_client.expire(f"cache:{key}", ttl + 10)  # Slight buffer after expiry

    return fresh_data
```

XFetch is elegant: items that take a LONG TIME to rebuild (large delta) trigger
early re-fetch more aggressively, because the cost of a stampede is higher for them.
Items that rebuild quickly tolerate waiting until closer to actual expiry.
