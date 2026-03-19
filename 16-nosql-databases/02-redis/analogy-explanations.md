# Chapter 02 — Redis: Analogy Explanations

Analogies that make Redis internals stick. Each includes a technical connection and code.

---

## Analogy 1: Redis Single-Threaded Event Loop — The Master Chef's Kitchen

**The Story:**
A Michelin-star restaurant has one master chef and a kitchen with a single stove. Orders come
in from 500 tables. The chef processes each order completely before starting the next. There
is no multi-tasking, no leaving a dish half-cooked. If you need the chef to poach an egg, the
chef poaches the egg start-to-finish, then moves to the next ticket.

This sounds slow, but it means every dish is perfect and there are no collisions, no "oops
I added salt while you were adding pepper" races. The kitchen is simple, correct, and
blazingly fast because there are no locks, no context switches, and no coordination overhead.

The restaurant does have multiple waiters (I/O threads since Redis 6.0) who carry orders to
and from tables in parallel, but the chef (command execution) is always just one.

**Technical Connection:**
Redis's event loop processes one command at a time. This eliminates the need for locks on
data structures, making operations like `INCR`, `ZADD`, and `HSET` inherently atomic.
The single-threaded model achieves 1M+ ops/sec because operations are in-memory and take
nanoseconds. Redis 6.0's I/O threading parallelises the network read/write but not execution.

```bash
# Everything is sequential — INCR is always atomic
INCR page_views       # No race condition possible, even with 10,000 concurrent clients
INCR page_views
# Result is always perfectly incremented, never partially updated

# A Lua script is the "whole recipe" — runs atomically from start to finish
EVAL "
  local val = redis.call('GET', KEYS[1])
  if not val then val = 0 end
  redis.call('SET', KEYS[1], tonumber(val) + ARGV[1])
  return tonumber(val) + tonumber(ARGV[1])
" 1 counter 5
```

---

## Analogy 2: Redis RDB vs AOF — The Photographer vs The Journalist

**The Story:**
A photographer (RDB) takes a snapshot of a party every hour. If you want to know what the
party looked like, you look at the last photo. But if the party ended between photos, you only
have the last snapshot — anything that happened in the last 59 minutes is lost.

A journalist (AOF) writes down every single thing that happens in their notebook, continuously.
If you need to reconstruct the party, you read the journalist's notes from the beginning.
It's slower to reconstruct, but nothing is lost (unless the journalist dropped their pen
right before writing the last note).

The hybrid approach (RDB+AOF) is a journalist who takes a photo every hour AND writes notes
for what happens between photos. To reconstruct: start from the last photo, then read the
notes since then. Fastest reconstruction, best of both worlds.

**Technical Connection:**
RDB is a point-in-time binary snapshot. Recovery is fast (parse binary) but you can lose
up to `save` interval worth of data. AOF is a sequential log of every write command. Recovery
requires replaying all commands (slow for large datasets). The hybrid mode writes an RDB
preamble in the AOF file on each rewrite, then appends incremental commands since the snapshot.

```bash
# RDB: periodic snapshots
CONFIG SET save "900 1 300 10 60 10000"
BGSAVE  # trigger manual snapshot (non-blocking fork)
DEBUG RELOAD  # reload from RDB (test recovery in dev)

# AOF: every-second durability
CONFIG SET appendonly yes
CONFIG SET appendfsync everysec
BGREWRITEAOF  # compact the AOF by rewriting minimal set of commands

# Hybrid: best restart performance
CONFIG SET aof-use-rdb-preamble yes
# AOF file structure after rewrite:
# [RDB binary snapshot ← fast to parse]
# [SET key1 val1 ← commands since snapshot]
# [ZADD leaderboard 100 alice ← incremental AOF]
```

---

## Analogy 3: Redis Sorted Set Skip List — The Highway Exit System

**The Story:**
Imagine a highway with 1 million exits, each numbered. You need to find exit 742,831 quickly.
The regular road (linked list) requires driving past exits 1 through 742,830 — slow.
A skip list adds express lanes: one lane with every 10th exit (100,000 stops), another with
every 100th (10,000 stops), another with every 1,000th. You take the highest express lane
possible, then drop down to lower lanes as you narrow in.

This gives O(log N) search while maintaining sorted order — you can still enumerate exits in
order (unlike a hash table). A balanced BST (like AVL) gives the same O(log N) complexity but
is much harder to update concurrently and requires complex rotation logic. A skip list is
lock-friendly and simpler to implement with good cache locality.

**Technical Connection:**
Redis implements Sorted Sets using a skip list for range operations (ZRANGE, ZRANGEBYSCORE,
ZRANGEBYLEX) and a hash table for O(1) score lookups by member (ZSCORE, ZRANK). The skip list
has configurable max levels (default: 32) giving O(log N) for N up to 2^32 elements.

```bash
# Skip list enables efficient range operations
ZADD leaderboard 9500 "alice" 8200 "bob" 9900 "carol" 7800 "dave"

# O(log N) range query — skip list traversal
ZRANGEBYSCORE leaderboard 9000 10000    # Returns: alice, carol
ZRANGE leaderboard 0 -1 WITHSCORES REV  # Returns all, sorted by score desc

# O(log N) rank lookup
ZRANK leaderboard "alice"   # Returns: 2 (0-indexed, ascending)
ZREVRANK leaderboard "alice" # Returns: 1 (from top)

# O(1) score lookup via hash table
ZSCORE leaderboard "alice"  # Returns: 9500
```

---

## Analogy 4: Redis Cluster Hash Slots — The Post Office Sorting System

**The Story:**
A city's mail system assigns every postal code (zip code) to one of 16,384 sorting rooms.
Every mail carrier knows which room handles which postal code. When you hand a letter to any
carrier, they check the postal code, walk directly to the right room, and deposit it there.
If the sorting room configuration changes (a new room opens), the post office sends out
updated maps to all carriers within seconds. Letters already in transit get a note: "This
room moved — go to the new address" (MOVED) or "This room is in the middle of moving — try
the new address but come back here if not found" (ASK).

**Technical Connection:**
Redis Cluster uses CRC16(key) % 16384 to map each key to one of 16,384 hash slots. Each slot
is owned by exactly one master shard. Cluster clients cache the slot-to-node map. During
resharding, slots move from source to destination shard. MOVED redirects update the client's
map permanently; ASK redirects are temporary (during migration) and do not update the map.

```bash
# Client maps key to slot
# CRC16("user:42") % 16384 = slot 5649
# Slot 5649 is on shard 2 (127.0.0.1:7002)

redis-cli -c -p 7000 SET user:42 "alice"
# Client connects to 7000, which returns: MOVED 5649 127.0.0.1:7002
# Client reconnects to 7002 and executes SET
# Client updates its slot map: slot 5649 → 7002

# During resharding (slot 5649 moving from 7002 → 7003):
redis-cli -c -p 7002 GET user:42
# Returns: ASK 5649 127.0.0.1:7003
# Client sends ASKING to 7003, then resends GET to 7003
# Client does NOT update slot map (migration may not be complete)

# Hash tag forces key to specific slot
# {user:42}:profile and {user:42}:settings both use CRC16("user:42")
SET "{user:42}:profile" "..."
SET "{user:42}:settings" "..."
# Both land on same slot → same shard → MGET works
```

---

## Analogy 5: Redis Sentinel — The Hospital Monitoring System

**The Story:**
A hospital has one attending physician (master Redis) and several residents (replicas). Three
nurse monitoring stations (Sentinels) continuously check whether the physician is responsive —
taking vital signs every 5 seconds. If two of the three nurses (quorum) independently confirm
the physician is unresponsive, they escalate: a majority of nurses authorise the most
senior resident to take over (failover). Meanwhile, the hospital's intercom system tells all
nurses the new physician's name (client notification via Pub/Sub).

If only one nurse loses communication (network blip), no action is taken — it might just be
a miscommunication, not an actual emergency. You need independent confirmation.

**Technical Connection:**
Sentinels send `PING` to the master and replicas every `sentinel down-after-milliseconds`.
If the master doesn't respond within this window, one Sentinel marks it `SDOWN` (subjectively
down). When `quorum` Sentinels independently mark it SDOWN, it becomes `ODOWN` (objectively
down). Then a majority of Sentinels vote to authorise the failover, and one Sentinel acts as
the leader to execute promotion.

```python
from redis.sentinel import Sentinel
import time

# Sentinel configuration
sentinel = Sentinel(
    [("sentinel1", 26379), ("sentinel2", 26379), ("sentinel3", 26379)],
    socket_timeout=0.5,
    password="sentinel_password"
)

def resilient_write(key: str, value: str, retries: int = 3):
    """Write with automatic retry after Sentinel failover."""
    for attempt in range(retries):
        try:
            master = sentinel.master_for("mymaster", socket_timeout=0.5)
            master.set(key, value)
            return True
        except Exception as e:
            if attempt < retries - 1:
                time.sleep(0.5)  # Wait for Sentinel to elect new master
            else:
                raise
    return False
```

---

## Analogy 6: Redis Pub/Sub vs Streams — Radio Broadcast vs DVR Recording

**The Story:**
Redis Pub/Sub is like a live radio broadcast. If you tune in late, you missed the earlier
songs — they're gone forever. If your radio breaks for 10 minutes, you miss 10 minutes of
content. No rewind, no replay, no acknowledgement that you heard the song.

Redis Streams is like a DVR (TiVo). Every broadcast is recorded and kept on the hard drive.
You can watch from the beginning, from where you left off, or skip to a specific timestamp.
Multiple people can watch the same recording at their own pace. If you miss something, you
can go back. You can even create "watch history" for each viewer (consumer groups) that tracks
where each viewer is in the recording.

**Technical Connection:**
Pub/Sub: fire-and-forget, no persistence, no consumer state, no replay. Messages are lost if
no subscriber is listening. Redis Streams: persistent log, consumer group state tracking,
explicit acknowledgement (XACK), back-pressure via PEL, and replay from any position.

```bash
# PUB/SUB: fire and forget
SUBSCRIBE channel1
# In another client:
PUBLISH channel1 "hello"         # If no subscriber: message is lost
PUBLISH channel1 "world"         # Subscriber gets it only if connected

# STREAMS: persistent, replayable
XADD notifications * user_id 42 message "You have a new follower"
XADD notifications * user_id 99 message "Your post was liked"

# Read from beginning (replay all messages)
XREAD COUNT 100 STREAMS notifications 0

# Consumer group: each consumer tracks its own position
XGROUP CREATE notifications email-workers $ MKSTREAM
XREADGROUP email-workers worker-1 COUNT 10 STREAMS notifications >
XACK notifications email-workers 1704067200000-0  # ACK after processing

# Stream trimming (keep last 10,000 messages)
XTRIM notifications MAXLEN ~ 10000  # ~ = approximate trimming (more efficient)
```

---

## Analogy 7: Redis Eviction Policies — The Library Book Cull

**The Story:**
A small library (maxmemory) has limited shelf space. When new books arrive and the shelves
are full, the librarian must remove some books. Different librarians follow different rules:

- **allkeys-lru**: Remove the book that hasn't been checked out the longest ("dustiest").
- **allkeys-lfu**: Remove the book checked out the fewest times ever ("least popular").
- **volatile-lru**: Only remove books with a "return by" sticker (TTL) — keep permanent reference
  books safe.
- **volatile-ttl**: Remove books whose "return by" date is soonest (expires first).
- **noeviction**: Refuse to accept new books until someone returns one ("library full, sorry").

LFU wins over LRU when old-but-popular books should stay: a textbook checked out 500 times
but not in the last week should beat a novel checked out once yesterday.

**Technical Connection:**
Redis implements LRU/LFU approximately using the `maxmemory-samples` configuration. It samples
N random keys and evicts the best candidate from the sample — not from the full keyspace.
LFU uses a logarithmic counter that decays over time (Morris counter), controlled by
`lfu-decay-time` (minutes per decay unit) and `lfu-log-factor` (counter increment probability).

```bash
# View a key's LFU counter
OBJECT FREQ mykey        # Returns approximate access frequency

# LFU tuning: how quickly does frequency decay?
CONFIG SET lfu-decay-time 1       # Decay by 1 unit per minute (more aggressive)
CONFIG SET lfu-log-factor 10      # Default: probability of increment = 1/(counter*lfu-log-factor)

# Monitor eviction in real-time
INFO stats | grep evicted_keys
# evicted_keys:0 → good
# evicted_keys growing rapidly → maxmemory too low or wrong policy

# Alert setup (Python)
import redis
r = redis.Redis()
stats = r.info("stats")
eviction_rate = stats["evicted_keys"]
if eviction_rate > 0:
    print(f"WARNING: {eviction_rate} keys evicted — check maxmemory config")
```

---

## Analogy 8: Redis Replication Backlog — The Paused Newspaper Delivery

**The Story:**
A newspaper publisher (master) sends daily papers to 3 subscribers (replicas). Subscriber A
goes on vacation for 2 weeks. The publisher keeps a stack of the last 7 days' papers in a
buffer room (replication backlog). When A returns, the publisher says: "Here are the 7 days
you missed — I have them right here." A catches up quickly.

But if A was gone for 14 days and the buffer only holds 7 days, the publisher says: "I can't
just give you the missing papers, I don't have them all. You'll need to get a full copy of
the entire archive from scratch." This is a *full resync* — expensive.

**Technical Connection:**
The replication backlog (`repl-backlog-size`, default 1MB) is a circular buffer on the master
storing recent write commands. If a replica reconnects after a short disconnect, it sends its
`repl_offset` and the master replies with only the delta (partial sync / PSYNC). If the replica
was offline longer than the backlog can cover, a full resync (FULLRESYNC + RDB transfer) is
required. Setting `repl-backlog-size` too small causes frequent full resyncs under any replica
hiccup, generating enormous network traffic.

```bash
# Check replication backlog config and status
CONFIG GET repl-backlog-size        # default: 1048576 (1MB)
CONFIG SET repl-backlog-size 104857600  # 100MB — prevent full resync on short outages
CONFIG GET repl-backlog-ttl         # how long to keep backlog after last replica disconnect

# Monitor replication lag
INFO replication
# role:master
# connected_slaves:2
# slave0:ip=10.0.0.2,port=6380,state=online,offset=12345678,lag=0
# slave1:ip=10.0.0.3,port=6380,state=online,offset=12345600,lag=1
# master_repl_offset:12345678
# repl_backlog_size:104857600
# repl_backlog_first_byte_offset:12000000  ← oldest offset in backlog

# If slave offset < repl_backlog_first_byte_offset → full resync needed
```
