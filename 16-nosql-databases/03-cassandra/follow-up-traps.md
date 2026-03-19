# Chapter 03 — Cassandra: Follow-up Traps

The follow-up questions that reveal whether a candidate truly understands Cassandra's internals.

---

## Trap 1: "Cassandra is eventually consistent, so it's not suitable for financial data."

**What most people say:**
"That's correct — use PostgreSQL or a CP database for anything financial."

**Correct answer:**
Cassandra's consistency is *tunable*. By setting both read and write consistency to `ALL`
(W=N, R=N), Cassandra is strongly consistent — it reads the latest write from all replicas.
More practically, `QUORUM + QUORUM` (W+R > N) provides strong consistency while tolerating
one node failure with RF=3.

Cassandra is used in financial systems at scale: Apple, Netflix (billing), and many banks use
Cassandra for specific financial workloads. The key design question is not "is it consistent?"
but "what consistency level does this specific access pattern require, and what is the
latency/availability cost of that level?"

For fraud detection (write: QUORUM, read: QUORUM) and real-time balance reads (read: LOCAL_ONE
with application-layer conflict resolution), Cassandra is entirely appropriate. For multi-entity
ACID transactions (transfer from account A to account B atomically), you need either a
relational database or careful modelling with LWT + application-layer compensation.

---

## Trap 2: "A partition key with high cardinality is always better for distribution."

**What most people say:**
"Yes — use a UUID as partition key for maximum distribution."

**Correct answer:**
High cardinality distributes data evenly, but it also destroys range query performance.
If your query pattern is "get all orders for user X in the last 7 days," a UUID partition
key means user X's orders are scattered across all nodes — you must query all partitions
and merge the results (a scatter-gather query), which is expensive.

The correct design principle is: **choose the partition key based on your most frequent
and most latency-sensitive query**. The partition key should group the rows you most
often access together.

Example: For an orders table queried by `(user_id, order_date)`:
```cql
-- Bad: UUID partition key — great distribution, terrible range queries
CREATE TABLE orders_bad (
    order_id UUID,
    user_id  TEXT,
    ...
    PRIMARY KEY (order_id)
);

-- Good: user_id as partition key, date as clustering column
CREATE TABLE orders_good (
    user_id    TEXT,
    order_date DATE,
    order_id   UUID,
    ...
    PRIMARY KEY ((user_id), order_date, order_id)
);
-- Query: SELECT * FROM orders_good WHERE user_id = 'alice'
--        AND order_date >= '2024-01-01' — single partition, efficient
```

The trade-off: `user_id` as partition key groups orders by user (great for per-user queries)
but a "power user" with 10 million orders creates a partition that is gigabytes in size — a
large partition problem. The fix is to bucket by user + time period:
```cql
PRIMARY KEY ((user_id, bucket), order_date, order_id)
-- where bucket = YEAR(order_date) || '-' || WEEK(order_date)
```

---

## Trap 3: "You said compaction merges SSTables — so Cassandra never has duplicate data."

**What most people say:**
"Correct — compaction removes duplicates."

**Correct answer:**
Between compaction cycles, the *same key* can exist in multiple SSTables. Cassandra uses
a merge-on-read strategy during compaction: it compares all versions of a cell across
SSTables and keeps only the most recent (by timestamp). But *before* compaction runs,
reading a key requires checking multiple SSTables — this is why read performance degrades
when compaction falls behind.

Additionally, tombstones (delete markers) are themselves stored in SSTables. A tombstone
is not actually removed until compaction runs AND `gc_grace_seconds` has elapsed (default:
10 days). This prevents "zombie resurrections" — if a node was down when a delete was issued
and the tombstone was compacted away before the node rejoined, the node would re-introduce
the deleted data.

The implication: on high-write tables with frequent deletes, you can have many SSTables,
each containing overlapping key ranges with tombstones mixed in. This causes the read path
to touch many SSTables and scan many tombstones — the classic "tombstone problem."

```bash
# Check SSTable count and tombstone metrics
nodetool cfstats keyspace.table | grep -E "SSTables|Tombstones"
# SSTable count:             47       ← too many → compaction lagging
# Tombstone cells scanned:   8234     ← high → delete-heavy workload

# Force compaction (blocking, use with caution in production)
nodetool compact keyspace table

# Check if a specific SSTable has a compaction problem
nodetool compactionstats
```

---

## Trap 4: "gc_grace_seconds=0 means tombstones are cleaned up immediately — great for delete-heavy workloads."

**What most people say:**
"Yes, setting gc_grace_seconds=0 solves the tombstone problem."

**Correct answer:**
Setting `gc_grace_seconds=0` is almost always wrong and can cause data resurrection
(deleted data reappearing). Here's why:

When Cassandra writes a tombstone (delete marker), it needs to be propagated to ALL replicas.
If one replica is temporarily down when the delete occurs, it does not receive the tombstone.
`gc_grace_seconds` is the period during which the tombstone must remain in SSTables — long
enough for any down replica to come back online, receive the tombstone (via repair or
hinted handoff), and only THEN garbage-collect it.

If `gc_grace_seconds=0`:
1. A delete is written (tombstone on 2 of 3 replicas, node 3 is down)
2. Compaction runs immediately and purges the tombstone
3. Node 3 comes back online (it has the original data, no tombstone)
4. A read at QUORUM picks up node 3's stale data → deleted data reappears

Safe practice:
- `gc_grace_seconds` must be >= the maximum time a node can be down before it is permanently
  removed from the cluster (typically 10 days or your repair cycle)
- Never set `gc_grace_seconds=0` unless the table has RF=1 or you have guaranteed all replicas
  receive the tombstone before compaction

```cql
-- Check gc_grace_seconds
SELECT gc_grace_seconds FROM system_schema.tables
WHERE keyspace_name = 'mykeyspace' AND table_name = 'mytable';

-- Carefully lower for a TTL-only table with guaranteed repair
ALTER TABLE mykeyspace.myevents
    WITH gc_grace_seconds = 86400  -- 1 day (requires repair daily)
    AND default_time_to_live = 2592000;  -- 30-day TTL
```

---

## Trap 5: "Cassandra handles schema changes without downtime — ALTER TABLE is always safe."

**What most people say:**
"Yes — Cassandra schema changes are online and non-blocking."

**Correct answer:**
Cassandra schema changes are *lightweight* — they don't require locking the table or
rewriting data. However, they are NOT always safe:

1. **Dropping a column that is part of a secondary index**: the secondary index must be
   dropped first, and all reads that used that index will break.

2. **Changing a column type that is part of the primary key**: not supported — you must
   create a new table and migrate data.

3. **Adding a column to a table with many nodes**: schema changes use Gossip to propagate
   to all nodes. In large clusters, schema disagreements (schema version mismatches)
   can cause `NoHostAvailableException` for new queries referencing the new column
   until all nodes have received the schema update.

4. **Dual-read period**: if old application code reads a new column that doesn't exist
   on some nodes yet (schema propagation in progress), it will get null values or errors
   depending on the driver.

Best practice: use a schema migration tool (Liquibase Cassandra extension, Flyway for
Cassandra, or manual `IF NOT EXISTS` guards) and deploy with backward-compatible schema
changes (add columns before updating application code to use them).

```cql
-- Safe: adding a nullable column (backward compatible)
ALTER TABLE users ADD last_login_at TIMESTAMP;
-- Existing rows will return null for last_login_at — safe for old readers

-- Risky: adding a column used in a new query before all nodes have schema
-- Mitigation: check schema agreement before deploying application
-- In Python driver:
from cassandra.cluster import Cluster
cluster = Cluster()
session = cluster.connect()
# Check schema agreement — all nodes must have the same schema version
print(cluster.metadata.schema_agreement())
# Returns dict: {version_uuid: [list of nodes on that version]}
# Should have only 1 key (all nodes agree) before deploying
```

---

## Trap 6: "Cassandra Materialized Views are the best way to support multiple query patterns."

**What most people say:**
"Yes — create a materialized view for each query pattern and Cassandra will keep them in sync."

**Correct answer:**
Cassandra Materialized Views (MVs) are convenient but have serious production caveats:

1. **Performance cost**: Every write to the base table triggers an additional write to each
   MV. For a table with 3 MVs, every write becomes 4 writes internally.

2. **Inconsistency risk**: MVs use an internal "view build" process that is NOT atomic with
   the base table write. Under failure scenarios, a base table write can succeed while the
   MV write fails → stale MV data. This is an eventual consistency problem *within a single
   write operation*.

3. **Known instability**: Cassandra MVs have had long-standing production instability issues.
   The Cassandra community's current recommendation (as of Cassandra 4.x) is to prefer
   "denormalised tables" with application-layer dual-write over MVs for production workloads.

4. **Repair complexity**: Repairing a table with MVs requires repairing the MVs too, which
   increases repair time proportionally.

The recommended alternative is application-side dual-write:
```python
def save_user(user: dict):
    # Write to primary table
    session.execute(
        "INSERT INTO users (user_id, email, name) VALUES (%s, %s, %s)",
        (user['id'], user['email'], user['name'])
    )
    # Write to query-optimised secondary table
    session.execute(
        "INSERT INTO users_by_email (email, user_id, name) VALUES (%s, %s, %s)",
        (user['email'], user['id'], user['name'])
    )
    # If second write fails: use a repair queue or saga pattern to retry
```

---

## Trap 7: "More consistency = slower reads — so use LOCAL_ONE for read performance."

**What most people say:**
"Yes, LOCAL_ONE is the fastest read consistency level."

**Correct answer:**
LOCAL_ONE is indeed the lowest-latency read, but it has a correctness risk that is subtle
and often ignored: **speculative reads and read repair**.

With LOCAL_ONE, the coordinator reads from *one* replica in the local DC. If that replica has
stale data (hasn't received a recent write), the client gets stale data with no indication.
There is no version comparison, no read repair across replicas.

For workloads where stale data causes business logic errors (e.g., "read balance before debit"),
LOCAL_ONE is dangerous. The correct approach is:

1. For truly cacheable, eventually consistent data (product descriptions, static configs):
   LOCAL_ONE is fine — staleness of seconds is acceptable
2. For user-generated content you just wrote: use read-your-writes pattern (route reads to
   the same node that received the write, or use LOCAL_QUORUM for the post-write read)
3. For financial/inventory data: LOCAL_QUORUM minimum

```python
from cassandra import ConsistencyLevel
from cassandra.query import SimpleStatement

# Different consistency per query type
session = cluster.connect('mykeyspace')

# Fast read — acceptable staleness
fast_stmt = SimpleStatement(
    "SELECT * FROM product_catalog WHERE product_id = %s",
    consistency_level=ConsistencyLevel.LOCAL_ONE
)

# Consistent read — must have latest data
consistent_stmt = SimpleStatement(
    "SELECT balance FROM accounts WHERE account_id = %s",
    consistency_level=ConsistencyLevel.LOCAL_QUORUM
)
```

---

## Trap 8: "Cassandra BATCH means all statements in the batch execute atomically."

**What most people say:**
"Yes, BATCH provides atomicity like a SQL transaction."

**Correct answer:**
Cassandra BATCH has two types with very different semantics:

1. **Unlogged BATCH**: Reduces round trips by grouping statements, but provides NO atomicity
   guarantee. If the batch partially fails, some statements may succeed and others fail.

2. **Logged BATCH (default)**: Uses a batch log (Paxos-style coordination) to provide
   atomicity *for statements within the same partition key*. Cross-partition logged batches
   are supported but introduce 4x the normal write latency (batch log write → execute →
   delete batch log) and significant coordinator overhead.

Critical anti-pattern: using BATCH to insert into multiple tables as a "performance
optimisation." This is actually *slower* than individual inserts for cross-partition batches
because of the batch log coordination overhead. Batch is only a performance win when:
- Statements affect the same partition (single partition batch)
- You need atomicity across multiple rows in the same partition (rare)

```cql
-- BAD: Cross-partition batch (slower than individual inserts)
BEGIN BATCH
    INSERT INTO users (id, name) VALUES (1, 'Alice');
    INSERT INTO users_by_email (email, id) VALUES ('alice@ex.com', 1);
    INSERT INTO user_events (id, ts, event) VALUES (1, now(), 'signup');
APPLY BATCH;
-- This is a cross-partition batch: logged (slow) and the coordinator becomes a
-- bottleneck if used at high frequency

-- GOOD: Single-partition batch (atomic, efficient)
BEGIN BATCH
    INSERT INTO user_activity (user_id, ts, action) VALUES (1, '2024-01-01', 'login');
    INSERT INTO user_activity (user_id, ts, action) VALUES (1, '2024-01-01', 'view');
APPLY BATCH;
-- Same partition (user_id=1) → no batch log → fast
```

---

## Trap 9: "Cassandra can replace a message queue for event streaming."

**What most people say:**
"Yes — you can INSERT events with a timestamp as the clustering column and consume them."

**Correct answer:**
Using Cassandra as a message queue is a well-known anti-pattern ("queue anti-pattern").
The problem is tombstone accumulation: when you delete consumed messages, you create
tombstones that pile up in the partition. A partition representing "pending tasks" that
receives thousands of inserts and deletes per second will accumulate millions of tombstones.
Every read must scan these tombstones to find remaining unprocessed messages.

Signs of the queue anti-pattern:
- Partition with many insertions and deletions (insert event, delete after processing)
- High tombstone count per read
- Degrading read performance over time even as "queue depth" stays constant

Solutions:
1. Use a dedicated message queue (Kafka, RabbitMQ, SQS) for event streaming
2. If Cassandra is required: use a "append-only + offset" pattern without deletes
   (like Kafka's model), mark events as "processed" with an UPDATE rather than DELETE
3. Use time-bucketed partitions with TTL so tombstones expire naturally with the partition

```cql
-- Instead of delete-on-consume (anti-pattern):
DELETE FROM tasks WHERE task_id = ?;  -- accumulates tombstones

-- Use status UPDATE + TTL (lesser evil):
UPDATE tasks SET status = 'processed', processed_at = toTimestamp(now())
    WHERE task_id = ?;
-- Set TTL on the whole row so it expires naturally:
INSERT INTO tasks (task_id, status, payload)
    VALUES (?, 'pending', ?) USING TTL 86400;  -- 24-hour TTL
```
