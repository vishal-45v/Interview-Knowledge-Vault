# Chapter 07: Key-Value Stores — Analogy Explanations

## Analogy 1: Memcached Slab Allocator — Egg Cartons of Different Sizes

**The Story:**
Imagine a warehouse that stores eggs of wildly different sizes — quail eggs, chicken eggs,
ostrich eggs. The manager pre-manufactures cartons in three sizes: 12-slot small cartons,
6-slot medium cartons, and 1-slot large cartons. When an egg arrives, it goes into the
smallest carton that fits. A quail egg in a small carton slot wastes minimal space. But
if all the large carton slots are occupied by medium-sized eggs (that don't fit in small
cartons but don't fill large cartons either), the warehouse is inefficient — small eggs
are being evicted even though large carton slots sit mostly empty.

**Connection to the database:**
Memcached's slab classes are the carton sizes. Items go into the smallest slab class that
fits. Slab pages (1MB each) are assigned to slab classes at startup and cannot easily be
moved. If your workload shifts — say, all your cached items suddenly grow from 100 bytes to
500 bytes — the slab classes for 100-byte items are full and you can't reallocate those
pages to the 500-byte class without evicting everything in the small-item class.

```bash
# Diagnose slab imbalance
echo "stats slabs" | nc memcached-host 11211 | awk '
/total_pages/ { pages[$0] = $2 }
/used_chunks/ { used[$0] = $2 }
/total_chunks/ { total[$0] = $2 }
{
    if (match($1, /STAT ([0-9]+):used_chunks/, arr)) {
        class = arr[1]
        printf "Class %s: %s%% utilized\n", class,
               (used[class] / total[class] * 100)
    }
}
'
# Classes with < 20% utilization are wasting slab pages
# Classes with > 90% utilization are evicting aggressively

# Fix with slab_reassign
memcached -o slab_reassign,slab_automove=1
```

---

## Analogy 2: Consistent Hashing — The Clock Face Ring

**The Story:**
Imagine a clock with 12 positions (0-11) and 3 workers (A, B, C) sitting at positions 2,
6, and 10 on the clock face. Each job that arrives is "hashed" to a position on the clock,
and goes to the next worker clockwise. If a new job hashes to position 4, it goes to
Worker B (the next clockwise from 4). If Worker B leaves (node failure), their jobs
simply go to Worker C (next clockwise). Only B's jobs move — A's jobs are unaffected.

In contrast, a modulo approach (job_id % 3 determines server) requires redistributing
ALL jobs when any server changes — adding server 4 changes the assignments of every job.

**Connection to the database:**

```python
import hashlib
import bisect
from typing import List

class ConsistentHash:
    def __init__(self, nodes: List[str], virtual_nodes: int = 150):
        """
        virtual_nodes: number of positions per physical node on the ring.
        More virtual nodes = more uniform distribution but higher memory.
        """
        self.ring = {}        # position → node name
        self.sorted_keys = [] # sorted ring positions
        self.virtual_nodes = virtual_nodes

        for node in nodes:
            self.add_node(node)

    def add_node(self, node: str):
        for i in range(self.virtual_nodes):
            key = self._hash(f"{node}:{i}")
            self.ring[key] = node
            bisect.insort(self.sorted_keys, key)

    def remove_node(self, node: str):
        for i in range(self.virtual_nodes):
            key = self._hash(f"{node}:{i}")
            self.ring.pop(key, None)
            idx = bisect.bisect_left(self.sorted_keys, key)
            if idx < len(self.sorted_keys):
                self.sorted_keys.pop(idx)

    def get_node(self, item: str) -> str:
        if not self.ring:
            raise ValueError("No nodes in ring")
        hash_val = self._hash(item)
        idx = bisect.bisect(self.sorted_keys, hash_val) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

# With 3 nodes and 150 virtual nodes each, each node handles ~33% of keys
ring = ConsistentHash(['memcache-1', 'memcache-2', 'memcache-3'])
# memcache-2 fails — only ~33% of keys (those mapped to memcache-2) remapped
ring.remove_node('memcache-2')
# All other keys still map to their original nodes — no thundering herd
```

Virtual nodes are the key insight: without them, the three workers cluster unevenly on the
clock face and one worker gets 60% of jobs while another gets 10%. Virtual nodes give each
physical server 150 positions on the ring, spreading load evenly regardless of how the
initial hashes fall.

---

## Analogy 3: etcd Raft Consensus — A Board of Directors Vote

**The Story:**
A five-member board of directors must approve all major company decisions. To pass, any
decision needs at least 3 votes (a majority). The CEO (Leader) proposes decisions. The CEO
sends the proposal to all board members, waits for 3 confirmations, then announces the
decision is final and logs it. If the CEO resigns (node failure), board members hold an
election — whoever sends their nomination first and gets 3 votes wins. During an election,
no new decisions can be made. If two members nominate themselves simultaneously, a runoff
election happens with randomized voting delays to break the tie.

**Connection to the database:**

```
RAFT LEADER ELECTION:

All nodes start as Followers
         │
         │  Election timeout expires (150-300ms randomized)
         ▼
Node becomes Candidate
         │
         │  Sends RequestVote RPC to all peers
         ▼
    Wait for votes
         │
    ┌────┴────────┐
    │             │
  Got majority   Timeout
  (>= N/2+1)    (split vote)
    │             │
    ▼             ▼
Becomes Leader  New election
                (random delay
                 prevents repeat
                 split vote)
```

```python
# etcd client observes leader changes via maintenance API
import etcd3

client = etcd3.client()

# Check current cluster status
status = client.status()
print(f"Leader: {status.leader}")
print(f"Raft index: {status.raft_index}")
print(f"DB size: {status.db_size} bytes")

# Watch for leader changes (important for apps that need leader affinity)
for event in client.watch_prefix('/'):
    print(f"Key: {event.key}, Revision: {event.mod_revision}")
    # If leader changes during watch, stream continues from new leader
    # No events are lost — Raft log ensures delivery
```

The randomized election timeout (150-300ms in etcd default config) is crucial. If all
nodes had the same timeout, they would all start elections simultaneously and split votes
indefinitely. Randomization ensures one node usually wins the "first to nominate" race.

---

## Analogy 4: Cache Stampede — The Restaurant That Ran Out of Soup

**The Story:**
A popular restaurant puts the "Soup of the Day" special on a chalkboard. The chalkboard
is updated every hour. At exactly 1:00 PM, the old soup message is erased. Three hundred
customers walk in at 1:00:01 PM, see no soup on the board, and ALL simultaneously ask the
chef "What's the soup today?" The chef is overwhelmed making soup (fetching from DB) while
all 300 customers wait. The chef can only make soup for one person at a time (database
sequential writes). The line grows, customers abandon, the restaurant loses business.

Better approach: One customer asks the chef. The chef hands them a "number ticket" (mutex
lock). Other customers wait for the chalk menu to be updated (serve from stale cache while
rebuilding), or the chef starts preparing the next soup 10 minutes BEFORE the hour
(refresh-ahead) so there's never a gap.

**Connection to the database:**

```python
import redis
import threading
import time

redis_client = redis.Redis()

# Mutex lock approach: first request rebuilds, others wait
def get_with_mutex(key: str, ttl: int, rebuild_fn):
    value = redis_client.get(key)
    if value:
        return value.decode()

    lock_key = f"lock:{key}"
    lock_acquired = redis_client.set(lock_key, "1", nx=True, ex=10)  # 10s lock timeout

    if lock_acquired:
        # This process rebuilds the cache
        try:
            fresh = rebuild_fn()
            redis_client.setex(key, ttl, str(fresh))
            return fresh
        finally:
            redis_client.delete(lock_key)
    else:
        # Other processes poll until cache is populated
        for attempt in range(20):
            time.sleep(0.5)
            value = redis_client.get(key)
            if value:
                return value.decode()
        # Fallback: go to DB directly if cache never populated
        return rebuild_fn()

# Better: Stale-while-revalidate (serve old value while refreshing in background)
def get_stale_while_revalidate(key: str, ttl: int, stale_ttl: int, rebuild_fn):
    """
    Serve stale content immediately, refresh in background.
    ttl: time to serve fresh content
    stale_ttl: additional time to serve stale content while refreshing
    """
    value = redis_client.get(key)
    remaining = redis_client.ttl(key)

    if value and remaining > 0:
        # Still fresh
        return value.decode()

    if value and remaining < 0:
        # Expired but within stale window (we use PERSIST after expiry via a flag)
        # Return stale immediately and refresh in background
        threading.Thread(
            target=lambda: redis_client.setex(key, ttl, str(rebuild_fn())),
            daemon=True
        ).start()
        return value.decode()  # Stale but immediate

    # True miss: synchronous rebuild
    fresh = rebuild_fn()
    redis_client.setex(key, ttl, str(fresh))
    return fresh
```

---

## Analogy 5: ZooKeeper Ephemeral Nodes — Hotel Room Key Cards

**The Story:**
When you check into a hotel, you get a key card. The key card works only while you're
a registered guest. When you check out — or if the system detects you've abandoned your
room (no heartbeat) — the room is automatically marked available. You don't need to
explicitly "release" the room; the connection between you and the room is automatically
terminated when your session ends.

ZooKeeper ephemeral znodes work exactly this way. When a microservice registers itself
(creates an ephemeral znode), the "key card" is the ZooKeeper session. If the service
crashes (connection lost), ZooKeeper automatically deletes the ephemeral znode after
the session timeout — no cleanup code required.

**Connection to the database:**

```python
from kazoo.client import KazooClient
import socket
import json

zk = KazooClient(hosts='zookeeper:2181',
                 timeout=10.0)     # Session timeout: 10 seconds
zk.start()

service_name = "payment-service"
instance_id = socket.gethostname()
service_info = {
    "host": socket.gethostbyname(socket.gethostname()),
    "port": 8080,
    "version": "v2.1.3",
    "started_at": "2024-01-15T10:00:00Z"
}

# Create ephemeral node — disappears automatically if this process dies
node_path = f"/services/{service_name}/{instance_id}"
zk.ensure_path(f"/services/{service_name}")

# EPHEMERAL: automatically deleted when session ends
zk.create(
    node_path,
    json.dumps(service_info).encode(),
    ephemeral=True   # This is the key — no manual cleanup on crash
)

print(f"Registered: {node_path}")

# Service discovery: list all instances of payment-service
def get_service_instances(service: str):
    try:
        instances = zk.get_children(f"/services/{service}")
        result = []
        for instance in instances:
            data, _ = zk.get(f"/services/{service}/{instance}")
            result.append(json.loads(data))
        return result
    except Exception:
        return []

# No cleanup needed — when this process exits, ZooKeeper session expires,
# ephemeral node is deleted, and all service discovery clients see it disappear
```

Sequential ephemeral nodes extend this for leader election:
`/election/candidate-0000000001`, `/election/candidate-0000000002`, etc.
The node with the lowest sequence number is the leader. If the leader dies,
its ephemeral node is deleted, and the next lowest sequence number takes over.

---

## Analogy 6: Write-Behind Caching — The Restaurant Order Pad

**The Story:**
A waiter takes your order, writes it on a pad, and returns immediately to take other
orders. They don't wait at the kitchen window while the chef cooks your food. Every few
minutes, the waiter delivers batched orders to the kitchen. The service is fast and the
waiter is efficient. But if the restaurant catches fire and the waiter's pad burns,
the orders that were taken but not yet delivered to the kitchen are lost forever.

This is write-behind (write-back) caching. The application writes to cache (the waiter's
pad) and returns immediately. The cache flushes to the database (kitchen) asynchronously
in batches. Blazing fast writes, potential data loss on failure.

**Connection to the database:**

```python
import redis
import json
import time
import threading
from collections import defaultdict

class WriteBehindCache:
    def __init__(self, redis_client, db, flush_interval=5, batch_size=100):
        self.redis = redis_client
        self.db = db
        self.flush_interval = flush_interval
        self.batch_size = batch_size
        self.dirty_queue_key = "writebehind:dirty_queue"  # Redis Stream for durability

        # Start background flusher
        self._flusher = threading.Thread(target=self._flush_loop, daemon=True)
        self._flusher.start()

    def set(self, key: str, value: dict, ttl: int = 3600):
        """Write to cache immediately, queue DB write asynchronously."""
        # Update cache
        self.redis.setex(key, ttl, json.dumps(value))

        # Enqueue for DB flush using Redis Stream (survives cache restart)
        self.redis.xadd(self.dirty_queue_key, {
            'key': key,
            'value': json.dumps(value),
            'timestamp': str(time.time())
        }, maxlen=100000)  # Keep last 100K writes in queue

    def _flush_loop(self):
        while True:
            try:
                self._flush_batch()
            except Exception as e:
                print(f"Flush error: {e}")  # Alert here in production
            time.sleep(self.flush_interval)

    def _flush_batch(self):
        # Read batch from stream
        messages = self.redis.xread(
            {self.dirty_queue_key: '0'},  # Read from beginning
            count=self.batch_size
        )

        if not messages:
            return

        batch = {}
        last_id = None
        for stream, entries in messages:
            for entry_id, entry in entries:
                key = entry[b'key'].decode()
                value = json.loads(entry[b'value'])
                batch[key] = value  # Deduplicate — only latest write per key
                last_id = entry_id

        # Batch upsert to DB
        if batch:
            self.db.batch_upsert(batch)  # Single round trip to DB
            # Acknowledge processed messages
            self.redis.xdel(self.dirty_queue_key, *[e[0] for _, entries in messages
                                                     for e in entries])
```

The critical difference from the naive implementation: using a Redis Stream (instead of
an in-memory queue) provides durability for the pending writes. If the application
process crashes, the Redis Stream retains the unprocessed writes. On restart, the
flusher picks up from where it left off. This eliminates the "burning waiter's pad" failure mode.

---

## Analogy 7: etcd MVCC and Revision — The Library Book Ledger

**The Story:**
A library keeps a ledger of all book checkouts. Every time a book changes status
(checked out, returned, lost, replaced), a new ledger entry is added — the OLD entry
is never erased. If you want to know the status of a book at a specific date, you
look up the ledger entry for that date. The librarian can tell you exactly what changed
between any two dates.

etcd's MVCC works the same way. Every write creates a new "revision" (ledger entry).
The old revision is kept until compaction. This enables watches to replay from a
specific revision (no missed events) and point-in-time queries.

**Connection to the database:**

```python
import etcd3

client = etcd3.client()

# Write a key — creates revision 5 (example)
client.put('/config/timeout', '30')
# Write again — creates revision 6
client.put('/config/timeout', '60')
# Write again — creates revision 7
client.put('/config/timeout', '90')

# Read current value
value, metadata = client.get('/config/timeout')
print(f"Current: {value}, Revision: {metadata.mod_revision}")
# Output: Current: b'90', Revision: 7

# The etcd revision is GLOBAL — it increments for every write to any key
# mod_revision: the revision when THIS key was last modified
# create_revision: the revision when this key was first created
# version: how many times this specific key has been modified

# Watch from a specific revision — replay events you might have missed
events_iterator, cancel = client.watch('/config/timeout',
                                        start_revision=5)  # Get events from rev 5
for event in events_iterator:
    print(f"Event at revision {event.mod_revision}: {event.value}")
    # Delivers events at rev 5, 6, 7 in order — never misses one
    if event.mod_revision >= 7:
        cancel()
        break

# COMPACTION: pruning old revisions to reclaim disk space
# After compaction at revision 6, you can no longer read revision 5
client.compact(6)
# Best practice: compact to (current_revision - 1000) periodically
```

The MVCC revision is why etcd is preferred over ZooKeeper for Kubernetes: if a Kubernetes
controller crashes and restarts, it resumes watching from the last known revision, ensuring
it processes every cluster state change in order — no reconciliation loop needed for missed events.
