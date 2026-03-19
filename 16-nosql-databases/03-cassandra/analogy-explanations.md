# Chapter 03 — Cassandra: Analogy Explanations

Analogies that make Cassandra's distributed architecture intuitive.

---

## Analogy 1: The Ring Topology — The Circular Conveyor Belt

**The Story:**
Imagine a circular conveyor belt in a large factory, with 12 workstations placed around it
at equal intervals. Each workstation handles the items that stop in front of it. When a new
item arrives (a data write), a label is attached (a hash of the partition key). The item
travels around the belt until it reaches the workstation whose zone covers that label number.

There is no "factory manager" workstation — all 12 are equal peers. If one workstation
goes offline for maintenance, the one clockwise from it temporarily handles its items (replica
coverage). When it returns, items are transferred back. Any workstation can accept an order
from a customer and forward the right items from the right zone.

**Technical Connection:**
Cassandra's ring topology assigns each node a range of token values. Any node can be a
coordinator (accept the request), and the token of the partition key determines which node
owns the data. With RF=3, each item is placed at the three clockwise nodes from its token
position. This means any 3-node combination that covers a token range can serve that data,
and the system functions even if some nodes are offline.

```bash
# View ring token assignment
nodetool ring

# Output:
# Datacenter: us-east-1
# ======================
# Address         Rack  Status State   Load       Owns     Token
# 10.0.0.1        1a    Up     Normal  245.67 GiB 33.33%   -9223372036854775808
# 10.0.0.2        1b    Up     Normal  244.91 GiB 33.33%   -3074457345618258603
# 10.0.0.3        1c    Up     Normal  246.12 GiB 33.33%    3074457345618258602

# Check which nodes own a specific partition key
nodetool getendpoints mykeyspace mytable 'alice'
# 10.0.0.1
# 10.0.0.2
# 10.0.0.3
```

---

## Analogy 2: Cassandra Write Path — The Post Office Assembly Line

**The Story:**
When you walk into a post office with a letter (write request), three things happen:
1. The clerk immediately stamps a receipt in the logbook (commitlog) — proof it was received
2. The clerk places the letter in a tray on the desk (memtable) sorted by destination
3. The clerk calls three branch post offices to deliver copies (replication)

When the desk tray is full, all letters are bundled into sealed mailbags (SSTables) and
sent to the warehouse (disk). The logbook entries for those letters are marked "archived."
If the post office burns down before the tray is archived, the logbook survives (it's in a
fireproof cabinet — sequential disk writes) and letters can be reconstructed from it.

**Technical Connection:**
The "logbook" is the commitlog — a sequential write to disk that ensures durability even
if the memtable (in RAM) is lost. The "tray" is the memtable — an in-memory sorted data
structure for fast writes. SSTable flush converts the memtable to immutable on-disk files.
The commitlog segments are recycled only after the corresponding memtable has been flushed.

```bash
# Commitlog location and configuration
grep "commitlog_directory" /etc/cassandra/cassandra.yaml
# commitlog_directory: /var/lib/cassandra/commitlog

grep "commitlog_sync" /etc/cassandra/cassandra.yaml
# commitlog_sync: periodic          # or batch
# commitlog_sync_period_in_ms: 10000 # fsync every 10 seconds (periodic mode)

# Manually flush memtable to SSTable
nodetool flush                      # flush all keyspaces
nodetool flush mykeyspace mytable   # flush specific table

# Check memtable size
nodetool cfstats mykeyspace.mytable | grep -i memtable
# Memtable cell count: 45231
# Memtable data size: 12.34 MiB
# Memtable off-heap memory used: 0 bytes
```

---

## Analogy 3: The Bloom Filter — The Bouncer's Guest List

**The Story:**
A nightclub has thousands of rooms (SSTables), and a VIP guest (partition key) may have
been placed in any of them. Checking every room would take all night. Instead, each room
has a bouncer with a special list. The bouncer can tell you with certainty: "This guest is
DEFINITELY NOT in my room." But if the list says "maybe in my room," you still have to
go in and check — the bouncer is sometimes wrong about who *is* there, but never wrong
about who *isn't*.

The Bloom filter is this bouncer. It lets you skip most rooms (SSTables) in O(1) time by
saying "definitely not here." On false positives, you do a disk read and find nothing —
a small wasted I/O. On true positives or true negatives, it perfectly guides you to the
right SSTable.

**Technical Connection:**
Each SSTable has a Bloom filter (Filter.db). A read for partition key P checks the Bloom
filter of each SSTable. A "definitely not here" answer skips the SSTable entirely. A "maybe
here" answer triggers a disk read of the SSTable's index and data files. The false positive
rate is configurable — lower FPR requires more memory for the filter.

```bash
# Bloom filter stats
nodetool cfstats mykeyspace.mytable | grep -i bloom
# Bloom filter false positives: 234
# Bloom filter false ratio:     0.01234    (target: < 0.1 = 10%)
# Bloom filter space used:      45.67 MiB

# Adjust false positive rate (lower = more memory, fewer disk reads)
# In schema:
# WITH bloom_filter_fp_chance = 0.01  -- 1% false positive rate (default)
#      bloom_filter_fp_chance = 0.1   -- 10% FP rate (less memory, more disk reads)

# Rebuild bloom filters if corrupted
nodetool upgradesstables -a mykeyspace mytable
```

---

## Analogy 4: Tombstones — The Library "Removed" Stickers

**The Story:**
A library has millions of books spread across 3 branches (replicas). When a book is "banned"
(deleted), librarians place a "REMOVED" sticker on each branch's copy. The sticker doesn't
immediately remove the book — it marks it as deleted and tells staff to discard the book
if they see it.

Now imagine a branch was closed for renovation when the "REMOVED" sticker was issued (a node
was down during the delete). When it reopens (node rejoins), it needs to receive the sticker
before the other branches can throw their copies away (gc_grace_seconds). If they threw their
stickered copies away too early, the branch that reopened would still have the "live" copy —
and the book would magically reappear (data resurrection).

**Technical Connection:**
Tombstones are delete markers written as cells in SSTables. They are propagated to replicas
via normal replication. `gc_grace_seconds` (default: 10 days) is the minimum time a tombstone
must survive before compaction can purge it — ensuring all replicas have received it before
it disappears. Tombstone accumulation in high-delete tables causes TombstoneOverwhelmingException
when read latency becomes unacceptable.

```cql
-- Check tombstone configuration
SELECT gc_grace_seconds FROM system_schema.tables
WHERE keyspace_name = 'mykeyspace' AND table_name = 'mytable';

-- Tombstone-safe delete pattern using TTL (better than DELETE for time-series)
-- Instead of DELETE, let data expire naturally:
INSERT INTO events (user_id, ts, event_type)
VALUES ('alice', toTimestamp(now()), 'login')
USING TTL 86400;  -- 1-day TTL: Cassandra creates a TTL tombstone, not a delete tombstone
-- TTL tombstones have a known expiry and are cleaned up more predictably

-- Emergency: reduce tombstone warning threshold (not a fix, just monitoring)
-- cassandra.yaml:
-- tombstone_warn_threshold: 1000    (warn at 1k tombstones per read)
-- tombstone_failure_threshold: 100000 (fail at 100k)
```

---

## Analogy 5: Query-First Design — The Restaurant Menu vs the Ingredient List

**The Story:**
When you design a database for an RDBMS, you think like a chef organising ingredients:
"I have tomatoes, pasta, and cheese — I'll store them in separate containers and combine
them in any dish I need." The database JOIN is the recipe that assembles dishes on demand.

Cassandra forces you to think like a restaurant manager writing the menu first: "What dishes
will customers order? Pasta with marinara? Caesar salad? Tiramisu?" You create a separate
preparation station (table) for each dish, pre-assembled with exactly the ingredients needed.
If a customer wants Carbonara, you can't serve it from the Marinara station — you need a
dedicated Carbonara station.

The cost: you maintain redundant copies of ingredients (denormalised data) across stations.
The benefit: every dish is assembled instantly because all the ingredients are already together.

**Technical Connection:**
Cassandra does not support JOINs or multi-partition aggregations efficiently. Every query
pattern needs a dedicated table with a partition key that groups all the data needed for that
query in one place. "One table per query pattern" is the fundamental Cassandra design rule.

```cql
-- For a social media platform:

-- Q1: Get user's timeline (posts from people they follow)
CREATE TABLE user_timeline (
    user_id     TEXT,
    post_ts     TIMESTAMP,
    post_id     UUID,
    author_id   TEXT,
    content     TEXT,
    like_count  COUNTER,  -- note: counter columns require separate table
    PRIMARY KEY ((user_id), post_ts, post_id)
) WITH CLUSTERING ORDER BY (post_ts DESC, post_id ASC);

-- Q2: Get posts by a specific user (their profile page)
CREATE TABLE posts_by_user (
    author_id   TEXT,
    post_ts     TIMESTAMP,
    post_id     UUID,
    content     TEXT,
    PRIMARY KEY ((author_id), post_ts, post_id)
) WITH CLUSTERING ORDER BY (post_ts DESC, post_id ASC);

-- Q3: Get a specific post by ID
CREATE TABLE posts_by_id (
    post_id     UUID,
    author_id   TEXT,
    content     TEXT,
    created_at  TIMESTAMP,
    PRIMARY KEY (post_id)
);

-- When Alice posts, write to ALL THREE tables atomically (dual-write):
-- Cassandra denormalisation = data in multiple tables simultaneously
```

---

## Analogy 6: Compaction Strategies — Sorting Your Email Inbox

**The Story:**
You receive 1,000 emails per day (writes). You need to search them quickly.

**STCS (Size-Tiered)**: You keep all emails in piles on your desk, sorted by date received
in each pile. When you have too many small piles, you merge them into one big pile. Good for
archiving but you have to search through many piles for any recent email.

**LCS (Leveled)**: You maintain a perfectly alphabetically sorted filing cabinet. Every new
email goes into the cabinet immediately in the right spot. The cabinet is always sorted,
so finding any email takes exactly the same time. But filing is 10x more work than just
throwing it in a pile — you move lots of emails around to maintain order.

**TWCS (Time-Windowed)**: You have a separate folder for each hour of the day. Emails from
9am–10am go in the "9am folder." Once 10am starts, the "9am folder" is sealed and never
touched again (no updates to old data!). Old folders (old time windows) are eventually
thrown away (TTL). Perfect for sensor data, logs, and events.

**Technical Connection:**

```bash
# Check which compaction strategy a table uses
SELECT compaction FROM system_schema.tables
WHERE keyspace_name = 'mykeyspace' AND table_name = 'mytable';

# Monitor compaction activity
nodetool compactionstats
# pending tasks: 5
# id            compaction type  keyspace  table       completed    total    unit   progress
# abc123        Compaction       iot       readings    1048576      52428800 bytes  2.00%

# Throttle compaction to reduce I/O impact
nodetool setcompactionthroughput 32  # 32MB/s (default: 16MB/s, 0 = unlimited)

# Trigger compaction manually (blocks, use carefully)
nodetool compact iot sensor_readings

# For TWCS: verify window configuration matches write pattern
nodetool cfstats iot.sensor_readings | grep "SSTable count"
# Ideally: 1-3 SSTables per time window (compacted promptly)
# If SSTable count grows unbounded: writes are out of time-order (TWCS anti-pattern)
```

---

## Analogy 7: Virtual Nodes — The Fairground Work Roster

**The Story:**
A fairground has 8 rides (nodes) and 160 attraction zones (token ranges). Instead of giving
each ride operator 1 large zone to manage (original Cassandra single token), each operator
is given 20 small scattered zones across the fairground (virtual nodes).

When a new operator joins, they take over 20 scattered zones from different existing operators
— no single operator loses all of their territory at once. The workload moves from all 8
existing operators to the new one gradually and in parallel. If one operator is sick, their
20 zones are redistributed across all other operators — everyone takes a small additional load.

**Technical Connection:**
With `num_tokens: 256` (default in Cassandra 3.x+), each node claims 256 non-contiguous token
ranges. Adding a node → all existing nodes stream a portion of their data simultaneously.
This prevents the "bootstrap bottleneck" where a single node streams all data. It also
naturally rebalances the cluster when nodes of different capacity are added.

```bash
# Set vnodes in cassandra.yaml before joining the ring
grep "num_tokens" /etc/cassandra/cassandra.yaml
# num_tokens: 256

# View token distribution (with vnodes, each node shows 256 tokens)
nodetool ring | head -50

# Check data balance after adding a node
nodetool status
# Datacenter: us-east-1
# Status=Up/Down, State=Normal/Leaving/Joining/Moving
# -- Address          Load       Tokens  Owns(effective)
# UN 10.0.0.1         245.67 GiB  256    24.8%   # should be ~25% with 4 nodes
# UN 10.0.0.2         244.91 GiB  256    25.1%
# UN 10.0.0.3         246.12 GiB  256    25.0%
# UN 10.0.0.4         243.88 GiB  256    25.1%   # new node, balanced
```
