# Chapter 03 — Cassandra: Structured Answers

Complete, interview-ready answers with CQL, Python, and architecture explanations.

---

## Q1: Explain Cassandra's write path and read path in detail.

**Answer:**

### Write Path

```
Client Request
      │
      ▼
Coordinator Node (any node — no special role)
      │
      ├──► Logs mutation to COMMITLOG (sequential write, durable)
      │    /var/lib/cassandra/commitlog/CommitLog-7-1704067200000.log
      │
      ├──► Writes to MEMTABLE (in-memory, sorted by partition key)
      │
      ├──► Updates Bloom filter (probabilistic membership check)
      │
      └──► Replicates asynchronously to RF-1 replica nodes
           (based on consistency level: wait for W acks before responding)

MEMTABLE FLUSH (triggered by: size threshold, commitlog segment threshold, manual flush):
      │
      ▼
   SSTable files written to disk:
   ├── Data.db        (actual row data, compressed)
   ├── Index.db       (partition key → offset in Data.db)
   ├── Summary.db     (sparse index of Index.db, in memory)
   ├── Filter.db      (Bloom filter — is this partition key here?)
   ├── CompressionInfo.db
   └── Statistics.db  (min/max timestamps, tombstone counts)

After flush: CommitLog segments that are fully flushed are recycled.
```

### Read Path

```
Client Query: SELECT * FROM orders WHERE user_id = 'alice' AND order_date = '2024-01-15'
      │
      ▼
Coordinator contacts R replicas (per consistency level)
      │
      ▼
On each replica — the read path:
      │
      ├─ 1. CHECK BLOOM FILTER for each SSTable
      │      Does this SSTable potentially contain the partition key?
      │      False positive → check deeper (wasted I/O)
      │      True negative  → skip this SSTable entirely (huge win)
      │
      ├─ 2. CHECK KEY CACHE
      │      In-memory cache of partition key → exact byte offset in Data.db
      │      Hit: jump directly to data (skip Index.db read)
      │      Miss: go to Summary.db
      │
      ├─ 3. CHECK PARTITION SUMMARY (Summary.db, in memory)
      │      Sparse index: every Nth entry from Index.db
      │      Narrows the range within Index.db to read
      │
      ├─ 4. READ PARTITION INDEX (Index.db, on disk)
      │      Full index: partition key → offset in Data.db
      │      Single disk read to find exact offset
      │
      └─ 5. READ DATA FILE (Data.db, on disk)
             Read the actual row data at the known offset

Coordinator merges results from R replicas (picks latest by timestamp)
Coordinator triggers read repair for any stale replicas (background)
Returns result to client
```

```python
# Python driver with tuned consistency levels
from cassandra.cluster import Cluster
from cassandra.policies import DCAwareRoundRobinPolicy, TokenAwarePolicy
from cassandra import ConsistencyLevel
from cassandra.query import SimpleStatement

cluster = Cluster(
    contact_points=["cassandra-1.example.com"],
    load_balancing_policy=TokenAwarePolicy(
        DCAwareRoundRobinPolicy(local_dc="us-east-1")
    ),
)
session = cluster.connect("mykeyspace")

# QUORUM read — strong consistency, single DC
read_stmt = SimpleStatement(
    "SELECT * FROM orders WHERE user_id = %s AND order_date >= %s",
    consistency_level=ConsistencyLevel.LOCAL_QUORUM,
)

# Execute prepared statement (driver sends to coordinator owning the token)
rows = session.execute(read_stmt, ("alice", "2024-01-01"))
for row in rows:
    print(row.order_id, row.total_amount)
```

---

## Q2: Explain Cassandra data modelling with a real-world example.

**Answer:**

**Core principle**: Model around your queries, not around entities. Cassandra does not
support joins, aggregations, or cross-partition queries efficiently. Each table is a
pre-computed query result.

**Example: E-commerce Order Management System**

```cql
-- REQUIREMENT ANALYSIS:
-- Q1: Get all orders for a user (paginated)
-- Q2: Get all orders in a status ("pending", "shipped") for a user
-- Q3: Get a specific order by ID
-- Q4: Get all orders placed today (admin dashboard)

-- ─── Q1 & Q2: Orders by User ────────────────────────────────────────────────
CREATE TABLE orders_by_user (
    user_id        TEXT,
    order_date     DATE,           -- clustering column (range queries)
    order_id       UUID,           -- clustering column (uniqueness within day)
    status         TEXT,
    total_amount   DECIMAL,
    item_count     INT,
    shipping_addr  TEXT,
    created_at     TIMESTAMP,
    PRIMARY KEY ((user_id), order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC, order_id ASC)
  AND default_time_to_live = 7776000;  -- 90-day TTL

-- Query Q1: all orders for user (last 30 days)
SELECT * FROM orders_by_user
WHERE user_id = 'alice'
  AND order_date >= '2024-01-01'
  AND order_date <= '2024-01-31';

-- Query Q2: filter by status (ALLOW FILTERING only if status has low cardinality
-- and partition is small — generally still an anti-pattern)
-- BETTER: maintain separate table for status-based lookups:

CREATE TABLE orders_by_user_status (
    user_id    TEXT,
    status     TEXT,
    order_date DATE,
    order_id   UUID,
    total_amount DECIMAL,
    PRIMARY KEY ((user_id, status), order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC, order_id ASC);

-- Query Q2: pending orders for a user
SELECT * FROM orders_by_user_status
WHERE user_id = 'alice' AND status = 'pending'
  AND order_date >= '2024-01-01';

-- ─── Q3: Order by ID ─────────────────────────────────────────────────────────
CREATE TABLE orders_by_id (
    order_id       UUID,
    user_id        TEXT,
    status         TEXT,
    total_amount   DECIMAL,
    line_items     LIST<FROZEN<line_item>>,
    created_at     TIMESTAMP,
    PRIMARY KEY (order_id)
);

-- ─── Q4: Orders by Date (Admin — bucketed to avoid large partition) ──────────
CREATE TABLE orders_by_date (
    date_bucket    TEXT,    -- 'YYYY-MM-DD' — limits partition size to 1 day
    created_at     TIMESTAMP,
    order_id       UUID,
    user_id        TEXT,
    total_amount   DECIMAL,
    PRIMARY KEY ((date_bucket), created_at, order_id)
) WITH CLUSTERING ORDER BY (created_at DESC, order_id ASC);

-- User-Defined Type for line items
CREATE TYPE line_item (
    product_id   TEXT,
    product_name TEXT,
    quantity     INT,
    unit_price   DECIMAL
);

-- ─── Dual Write Pattern (application layer) ──────────────────────────────────
```

```python
import uuid
from cassandra.cluster import Cluster
from cassandra import ConsistencyLevel
from cassandra.query import BatchStatement, BatchType
from datetime import date, datetime

session = Cluster().connect("ecommerce")

INSERT_BY_USER = session.prepare("""
    INSERT INTO orders_by_user
    (user_id, order_date, order_id, status, total_amount, item_count, created_at)
    VALUES (?, ?, ?, ?, ?, ?, ?)
""")

INSERT_BY_ID = session.prepare("""
    INSERT INTO orders_by_id
    (order_id, user_id, status, total_amount, created_at)
    VALUES (?, ?, ?, ?, ?)
""")

INSERT_BY_STATUS = session.prepare("""
    INSERT INTO orders_by_user_status
    (user_id, status, order_date, order_id, total_amount)
    VALUES (?, ?, ?, ?, ?)
""")

def create_order(user_id: str, total_amount: float, item_count: int) -> str:
    order_id   = uuid.uuid4()
    order_date = date.today()
    created_at = datetime.utcnow()
    status     = "pending"

    # Single-partition batch per table (efficient)
    # Cross-table writes are NOT atomic — use saga pattern for failures
    session.execute(INSERT_BY_USER,
        (user_id, order_date, order_id, status, total_amount, item_count, created_at),
        consistency_level=ConsistencyLevel.LOCAL_QUORUM,
    )
    session.execute(INSERT_BY_ID,
        (order_id, user_id, status, total_amount, created_at),
        consistency_level=ConsistencyLevel.LOCAL_QUORUM,
    )
    session.execute(INSERT_BY_STATUS,
        (user_id, status, order_date, order_id, total_amount),
        consistency_level=ConsistencyLevel.LOCAL_QUORUM,
    )
    return str(order_id)
```

---

## Q3: Explain Cassandra compaction strategies with workload matching.

**Answer:**

```cql
-- ─── STCS: Size-Tiered Compaction Strategy ───────────────────────────────────
-- Default strategy. Groups SSTables of similar size, compacts when 4+ in a tier.
-- Best for: write-heavy workloads, mostly INSERTs, rarely reads same data twice.
-- Problems: "space amplification" during compaction (needs 2x data space temporarily);
--           on read-heavy workloads, many overlapping SSTables → slow reads.
ALTER TABLE user_events
    WITH compaction = {
        'class': 'SizeTieredCompactionStrategy',
        'min_threshold': 4,       -- compact when 4+ SSTables of similar size
        'max_threshold': 32,      -- never compact more than 32 at once
        'bucket_size': 1.5,       -- SSTables within 1.5x of each other are "similar"
        'min_sstable_size': 52428800  -- ignore SSTables < 50MB
    };

-- ─── LCS: Leveled Compaction Strategy ────────────────────────────────────────
-- Keeps SSTables in levels. L0 is unleveled; L1+ are leveled (no overlapping key ranges).
-- Best for: read-heavy workloads, random reads, high update rate.
-- Benefits: read touches at most 1 SSTable per level → predictable read latency.
-- Problems: write amplification (each write is re-compacted multiple times);
--           higher I/O than STCS for write-heavy workloads.
ALTER TABLE user_profiles
    WITH compaction = {
        'class': 'LeveledCompactionStrategy',
        'sstable_size_in_mb': 160  -- target L1+ SSTable size
    };

-- ─── TWCS: Time Window Compaction Strategy ───────────────────────────────────
-- Groups SSTables by time window, compacts only within the current window.
-- Old (closed) windows are never recompacted — ideal for time-series data.
-- Best for: time-series metrics, logs, events with TTL; insert-only (no updates).
-- Requirements: data should be written in roughly time-order; use TTL for expiry.
-- Benefits: compaction stays bounded and predictable; old data expires naturally.
ALTER TABLE sensor_readings
    WITH compaction = {
        'class': 'TimeWindowCompactionStrategy',
        'compaction_window_unit': 'HOURS',
        'compaction_window_size': 1    -- 1-hour windows
    }
    AND default_time_to_live = 604800; -- 7-day TTL

-- Verify compaction progress
-- nodetool compactionstats
-- nodetool cfstats keyspace.table | grep "SSTable count"
```

**Decision matrix:**

```
Workload          STCS    LCS     TWCS
─────────────────────────────────────────────────────────────────
Write-heavy        ✓       ✗       ✓ (time-series only)
Read-heavy         ✗       ✓       ✗
Time-series        ~       ✗       ✓
High updates       ~       ✓       ✗ (updates kill TWCS)
Limited disk       ✗       ✓       ✓
Predictable reads  ✗       ✓       ~
```

---

## Q4: Implement a Cassandra-backed time-series data store.

**Answer:**

```cql
-- Schema: IoT sensor readings, distributed by sensor + time bucket
-- Bucket prevents unbounded partition growth (1 hour = ~3600 rows at 1/sec)

CREATE KEYSPACE iot
    WITH replication = {
        'class': 'NetworkTopologyStrategy',
        'us-east-1': 3,
        'eu-west-1': 2
    };

USE iot;

CREATE TABLE sensor_readings (
    sensor_id    TEXT,
    bucket       TEXT,        -- 'YYYY-MM-DD-HH' (1-hour bucket)
    ts           TIMESTAMP,
    value        DOUBLE,
    unit         TEXT,
    quality      SMALLINT,    -- 0=bad, 1=uncertain, 2=good
    PRIMARY KEY ((sensor_id, bucket), ts)
) WITH CLUSTERING ORDER BY (ts DESC)
  AND compaction = {
      'class': 'TimeWindowCompactionStrategy',
      'compaction_window_unit': 'HOURS',
      'compaction_window_size': 1
  }
  AND default_time_to_live = 2592000  -- 30-day TTL
  AND compression = {
      'class': 'LZ4Compressor',
      'chunk_length_in_kb': 64
  };

-- Index for querying latest value per sensor across buckets
CREATE TABLE sensor_latest (
    sensor_id    TEXT,
    ts           TIMESTAMP STATIC,  -- static column: one per partition
    value        DOUBLE STATIC,
    PRIMARY KEY (sensor_id)
);
```

```python
import arrow  # arrow library for time manipulation
from cassandra.cluster import Cluster
from cassandra.policies import TokenAwarePolicy, DCAwareRoundRobinPolicy
from cassandra import ConsistencyLevel
from cassandra.query import SimpleStatement
from datetime import datetime
import uuid

cluster = Cluster(
    contact_points=["cassandra-1.iot.internal"],
    load_balancing_policy=TokenAwarePolicy(
        DCAwareRoundRobinPolicy(local_dc="us-east-1")
    ),
)
session = cluster.connect("iot")

# Prepare statements once at startup
INSERT_READING = session.prepare("""
    INSERT INTO sensor_readings (sensor_id, bucket, ts, value, unit, quality)
    VALUES (?, ?, ?, ?, ?, ?)
    USING TTL ?
""")

SELECT_READINGS = session.prepare("""
    SELECT ts, value, unit, quality FROM sensor_readings
    WHERE sensor_id = ? AND bucket = ? AND ts >= ? AND ts <= ?
    ORDER BY ts ASC
""")

def get_bucket(ts: datetime) -> str:
    """Returns 'YYYY-MM-DD-HH' bucket for a timestamp."""
    return ts.strftime("%Y-%m-%d-%H")

def get_buckets_in_range(start: datetime, end: datetime) -> list:
    """Returns all hourly buckets covering the time range."""
    buckets = []
    current = arrow.get(start).floor("hour")
    end_arrow = arrow.get(end)
    while current <= end_arrow:
        buckets.append(current.format("YYYY-MM-DD-HH"))
        current = current.shift(hours=1)
    return buckets

def write_reading(sensor_id: str, ts: datetime, value: float,
                  unit: str = "celsius", quality: int = 2, ttl: int = 2592000):
    bucket = get_bucket(ts)
    session.execute(
        INSERT_READING,
        (sensor_id, bucket, ts, value, unit, quality, ttl),
        consistency_level=ConsistencyLevel.LOCAL_ONE,  # write-heavy: fast writes
    )

def read_range(sensor_id: str, start: datetime, end: datetime) -> list:
    """Read sensor data across multiple time buckets."""
    all_rows = []
    for bucket in get_buckets_in_range(start, end):
        rows = session.execute(
            SELECT_READINGS,
            (sensor_id, bucket, start, end),
            consistency_level=ConsistencyLevel.LOCAL_QUORUM,
        )
        all_rows.extend(rows)

    # Sort across buckets (each bucket is already sorted)
    all_rows.sort(key=lambda r: r.ts)
    return all_rows

# Example: ingest 1,000 readings
from datetime import timedelta
now = datetime.utcnow()
for i in range(1000):
    ts = now - timedelta(seconds=i)
    write_reading("gateway-001", ts, 22.5 + (i % 10) * 0.1)

# Query last hour
one_hour_ago = now - timedelta(hours=1)
readings = read_range("gateway-001", one_hour_ago, now)
print(f"Retrieved {len(readings)} readings")
```

---

## Q5: Explain Cassandra Lightweight Transactions (LWT) with implementation example.

**Answer:**

```cql
-- LWT uses Paxos under the hood (4 round trips instead of 1)
-- Phase 1: Prepare (Paxos ballot)
-- Phase 2: Promise
-- Phase 3: Propose (with the conditional value)
-- Phase 4: Commit

-- Use case 1: Unique username registration
INSERT INTO users (username, user_id, email, created_at)
VALUES ('alice', uuid(), 'alice@example.com', toTimestamp(now()))
IF NOT EXISTS;
-- Returns: [applied] = true (success) or [applied] = false + current row (conflict)

-- Use case 2: Compare-and-swap (optimistic lock)
UPDATE inventory
SET quantity = 99
WHERE product_id = 'SKU-001'
IF quantity = 100;  -- only if current quantity is exactly 100

-- Use case 3: Account balance debit (prevent overdraft)
UPDATE accounts
SET balance = 450.00
WHERE account_id = 'acc-42'
IF balance = 550.00 AND balance >= 100.00;
```

```python
from cassandra.cluster import Cluster
from cassandra import ConsistencyLevel
from cassandra.query import SimpleStatement

session = Cluster().connect("myapp")

def register_username(username: str, email: str) -> bool:
    """
    Atomically register a username only if it doesn't exist.
    LWT provides serial consistency — safe even with concurrent requests.
    """
    stmt = SimpleStatement(
        """
        INSERT INTO users (username, email, created_at)
        VALUES (%s, %s, toTimestamp(now()))
        IF NOT EXISTS
        """,
        consistency_level=ConsistencyLevel.LOCAL_QUORUM,
        serial_consistency_level=ConsistencyLevel.LOCAL_SERIAL,  # Paxos quorum
    )
    result = session.execute(stmt, (username, email))
    row = result.one()
    return row.applied  # True = registered, False = username taken

def debit_account(account_id: str, amount: float, expected_balance: float) -> dict:
    """
    Debit an account with optimistic locking via LWT.
    Returns {'success': True} or {'success': False, 'current_balance': X}
    """
    new_balance = expected_balance - amount
    stmt = SimpleStatement(
        """
        UPDATE accounts
        SET balance = %s, updated_at = toTimestamp(now())
        WHERE account_id = %s
        IF balance = %s
        """,
        consistency_level=ConsistencyLevel.LOCAL_QUORUM,
        serial_consistency_level=ConsistencyLevel.LOCAL_SERIAL,
    )
    result = session.execute(stmt, (new_balance, account_id, expected_balance))
    row = result.one()
    if row.applied:
        return {"success": True}
    else:
        return {"success": False, "current_balance": row.balance}

# LWT Performance warning:
# LWT costs ~4x normal write latency (4 Paxos round trips instead of 1)
# At 500 LWT/sec per partition, Paxos contention causes queuing
# Mitigation: partition LWT writes (e.g., shard by username prefix)
# Or: use application-level idempotency tokens + regular writes + compensation
```

**Key LWT facts for interviews:**
- Consistency level for the read component of LWT: `SERIAL` or `LOCAL_SERIAL`
- Consistency level for the write component: specified normally (e.g., `LOCAL_QUORUM`)
- `SERIAL` = majority of *all* DCs; `LOCAL_SERIAL` = majority of *local* DC only
- LWT holds a Paxos lock on the partition during the 4-round-trip operation — concurrent
  LWT operations on the same partition queue behind each other, causing latency spikes
  at high concurrency
