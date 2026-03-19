# Chapter 10: Database Selection — Structured Answers

## Q1: Walk through a principled database selection framework for a new system.

**Answer:**

The most common mistake is selecting a database based on popularity, team familiarity,
or what other companies use. The correct process starts with the data and its access patterns.

**Step 1: Model your access patterns (what queries does the application execute?)**

```python
# Document every access pattern BEFORE choosing a database
access_patterns = [
    # Pattern, estimated QPS, latency SLA, consistency requirement
    ("get_user_by_id",                    "10K/s", "10ms p99",   "strong consistency"),
    ("get_user_activity_feed",            "5K/s",  "50ms p99",   "eventual ok"),
    ("write_activity_event",              "20K/s", "20ms p99",   "at-least-once"),
    ("search_users_by_name_and_location", "500/s", "200ms p99",  "eventual ok"),
    ("fraud_detection_3hop_graph",        "100/s", "150ms p99",  "strong consistency"),
    ("monthly_revenue_report",            "1/hour","30s ok",     "eventual ok"),
]
# Each pattern narrows the database options significantly
```

**Step 2: Identify consistency requirements**

```
Spectrum of consistency models:
Linearizable ←──────────────────────────────────────── Eventual
  │                                                         │
  ▼                                                         ▼
PostgreSQL    MongoDB      DynamoDB    Cassandra(tunable)  DynamoDB
CockroachDB   (default)    (strong     (eventual by        (eventual)
Redis         with write   consistent  default)
Cluster       concern)     option)
Spanner

Questions to ask:
- Can users see stale data? (eventual consistency ok)
- Can TWO users simultaneously receive the "last item" in inventory? (strong required)
- Is "read your own writes" required? (at minimum causal consistency)
- Are cross-entity atomic operations required? (ACID transactions)
```

**Step 3: Assess data model fit**

```
Data characteristics → Data model → Database type

Tabular + relationships + ACID → Relational → PostgreSQL, MySQL, CockroachDB
Variable attributes + nested → Document → MongoDB, DynamoDB, Firestore
Highly connected + traversal → Graph → Neo4j, Amazon Neptune
Ordered time + high write → Time-series → TimescaleDB, InfluxDB, ClickHouse
Key lookups + low latency → Key-value → Redis, DynamoDB (simple key access)
Wide columns + high cardinality → Wide-column → Cassandra, HBase, Bigtable
Full-text + faceted → Search → Elasticsearch, OpenSearch, Meilisearch
```

**Step 4: Scale and growth projection**

```python
def estimate_scale_requirements(
    daily_active_users: int,
    writes_per_user_per_day: int,
    reads_per_user_per_day: int,
    bytes_per_record: int,
    retention_years: int
) -> dict:
    writes_per_second = (daily_active_users * writes_per_user_per_day) / 86400
    reads_per_second  = (daily_active_users * reads_per_user_per_day)  / 86400

    # Storage after retention period
    total_bytes = (daily_active_users * writes_per_user_per_day *
                   bytes_per_record * retention_years * 365)

    return {
        "writes_per_second": writes_per_second,
        "reads_per_second":  reads_per_second,
        "total_storage_tb":  total_bytes / 1e12,
        "single_postgres_feasible": writes_per_second < 10000 and total_bytes < 10e12,
    }

# For most products: single PostgreSQL handles the load for 2-3 years
# Premature optimization toward "infinite scale" databases creates unnecessary complexity
```

**Step 5: Operational considerations**

```
MANAGED (DBaaS):                    SELF-HOSTED:
+ No ops burden                     + Full control
+ Auto-scaling in some cases        + Lower per-GB cost
+ Integrated backups/failover       + No vendor lock-in
- Higher cost per GB                - Requires DBA expertise
- Vendor lock-in risk               - Backup/DR is your problem
- Less flexibility for tuning       - Security patching is your problem

Decision: managed if team < 50 engineers, self-hosted if > dedicated DBA team
```

---

## Q2: Compare the CAP/PACELC positioning of the major databases with nuance.

**Answer:**

CAP theorem is often oversimplified. The modern PACELC framework is more useful:
- When there IS a partition: C (consistency) vs A (availability)
- ELSE (no partition): L (latency) vs C (consistency)

```
DATABASE     CAP    PACELC   EXPLANATION
═══════════════════════════════════════════════════════════════════════════════
PostgreSQL   CA*    ELC      *Single node: no network partition possible
                             Under partition: lose availability (stop accepting writes)
                             Normal operation: low latency is sacrificed for consistency
                             (reads go to same node as writes)
                             Read replicas = eventual consistency (EL)

CockroachDB  CP     ELC      Distributed SQL, Raft consensus
                             Partition: majority partition remains available
                             Minority partition: unavailable (CP)
                             Normal: reads from leader (ELC) OR
                             stale reads from any node (EL trade-off available)

Cassandra    AP     EL       Default: eventual consistency, write to any replica
                             Tunable: QUORUM reads = CP-like behavior
                             EACH_QUORUM = strong consistency per datacenter
                             Normal: writes/reads go to nearest replica (low L)
                             Trade-off: lower latency but higher inconsistency risk

DynamoDB     CP*    ELC      *Eventually consistent reads = AP behavior
                             Strongly consistent reads = CP
                             Transactions = CP
                             Normal: EC reads are EL; strong reads pay latency for C

MongoDB      CP*    ELC      *With default write concern (w:1) = AP-like
                             With w:majority + j:true = CP
                             Normal: reads from primary = ELC
                             Stale reads from secondaries available = EL trade-off

Redis        CP*    EL       *Redis Sentinel: CP in single datacenter
                             Redis Cluster: AP (continues with degraded capacity)
                             Normal: sub-millisecond = extremely EL
                             Persistence vs performance trade-off (AOF = C, vs RDB = L)

Elasticsearch AP    EL       Primary shards = writes; replicas for reads
                             Partition: continues with degraded shard availability
                             Normal: near-realtime (1s delay) for search visibility
                             Trade-off: availability and low latency for strong C
```

**Practical implications:**

```python
# PostgreSQL: CP read with read replica = AP read

# CP read (strong consistency, higher latency):
def get_account_balance_consistent(account_id: str) -> Decimal:
    # Always read from primary — synchronous replication means up-to-date
    return primary_db.execute(
        "SELECT balance FROM accounts WHERE id = %s", account_id
    ).scalar()

# AP read (eventual consistency, lower latency — may be stale):
def get_account_balance_eventual(account_id: str) -> Decimal:
    # Read from replica — may lag behind primary by milliseconds to seconds
    return read_replica.execute(
        "SELECT balance FROM accounts WHERE id = %s", account_id
    ).scalar()

# For a financial account balance: always use CP
# For a user's display name: AP is fine (stale display name for 100ms is harmless)
```

---

## Q3: Design the polyglot persistence architecture for a large-scale e-commerce platform.

**Answer:**

**System requirements mapped to databases:**

```
USE CASE            CHOSEN DB    JUSTIFICATION
══════════════════════════════════════════════════════════════════════
Product catalog     PostgreSQL   Rich attributes, complex queries,
                                 full-text search via pg_trgm or FTS
                                 Flexible JSON columns for custom attrs

Shopping cart       Redis        Session-scoped, high read/write (100K/s),
                                 hash data structure maps perfectly to cart,
                                 TTL for abandoned cart expiry

Order management    PostgreSQL   ACID transactions required,
                                 join product+user+payment in one TX,
                                 audit log via event table

Inventory           PostgreSQL   Strong consistency for stock levels,
(stock levels)                   prevent overselling via row-level locking,
                                 triggers for stock-level events

Product search      Elasticsearch Full-text search, faceted navigation,
                                 typo tolerance, geo-search, complex scoring

Recommendation      Redis         Sorted sets for "top-N" recs,
engine (hot         + offline ML  pre-computed score vectors per user,
recommendations)                  real-time updates via event processing

User activity       Cassandra     High write throughput (activity events),
feed                              time-series access pattern (latest N events),
                                  per-user partition key = fast reads

Session auth        Redis         Sub-millisecond token validation,
tokens                            automatic TTL on session expiry,
                                  atomic get-and-refresh operations
```

**Cross-database consistency via the outbox pattern:**

```python
# Product created: must appear in PostgreSQL AND Elasticsearch
# Naive approach (inconsistent):
def create_product_naive(product: dict):
    product_id = postgres.insert('products', product)
    es_client.index('products', id=product_id, body=product)
    # If ES index fails: PostgreSQL has product, search doesn't show it

# Correct approach: outbox pattern
def create_product(product: dict) -> str:
    with postgres.transaction():
        # 1. Write to primary store
        product_id = postgres.insert('products', product)

        # 2. Write event to outbox IN THE SAME TRANSACTION
        postgres.insert('outbox_events', {
            'id': uuid.uuid4(),
            'event_type': 'product.created',
            'aggregate_id': product_id,
            'payload': json.dumps(product),
            'created_at': datetime.utcnow(),
            'status': 'pending'
        })
        # Both writes are atomic: either both succeed or both fail
    return product_id

# Background worker: reads outbox and propagates to other databases
# Uses at-least-once delivery (idempotent consumers handle duplicates)
def outbox_processor():
    while True:
        pending = postgres.query(
            "SELECT * FROM outbox_events WHERE status = 'pending' LIMIT 100"
        )
        for event in pending:
            try:
                if event.event_type == 'product.created':
                    product = json.loads(event.payload)
                    es_client.index('products', id=event.aggregate_id, body=product)
                    # Update Elasticsearch only AFTER primary DB committed
                postgres.execute(
                    "UPDATE outbox_events SET status = 'processed' WHERE id = %s",
                    event.id
                )
            except Exception as e:
                postgres.execute(
                    "UPDATE outbox_events SET status = 'failed', error = %s WHERE id = %s",
                    (str(e), event.id)
                )
```

**Inventory consistency (preventing overselling):**

```sql
-- Use SELECT FOR UPDATE to prevent race condition on last item
BEGIN;
SELECT id, stock_level
FROM inventory
WHERE product_id = 'PROD-001'
  AND stock_level > 0
FOR UPDATE;                    -- Row-level lock: only one transaction can hold this

-- Check if stock available and decrement atomically
UPDATE inventory
SET stock_level = stock_level - :quantity,
    updated_at = NOW()
WHERE product_id = 'PROD-001'
  AND stock_level >= :quantity  -- Double-check (prevents negative stock even with lock)
RETURNING stock_level;

-- If UPDATE returned 0 rows: out of stock → ROLLBACK
-- If UPDATE returned 1 row: success → COMMIT
COMMIT;
```

---

## Q4: Design a CQRS + Event Sourcing architecture for a financial ledger.

**Answer:**

**System requirements:**
- Immutable transaction records (regulatory compliance)
- Real-time balance queries (p99 < 50ms)
- Reconstruct any account's balance at any historical point
- 10K transactions/second
- 7-year retention

**Architecture:**

```python
# COMMAND SIDE: Write model (event store)
class FinancialEventStore:
    """Append-only event store for financial transactions."""

    def record_transaction(self, command: TransferCommand) -> str:
        """
        Record a financial transaction atomically.
        Returns event_id if successful, raises ConflictError if
        concurrent modification detected.
        """
        with self.db.transaction():
            # Get current version for optimistic concurrency control
            current_version = self._get_account_version(command.from_account_id)

            if current_version != command.expected_version:
                raise ConcurrencyConflictError(
                    f"Account {command.from_account_id} was modified concurrently"
                )

            # Append events to event store (immutable, append-only)
            event_id = self.db.insert('financial_events', {
                'event_id': str(uuid.uuid4()),
                'event_type': 'MoneyTransferred',
                'account_id': command.from_account_id,
                'version': current_version + 1,
                'timestamp': datetime.utcnow(),
                'payload': json.dumps({
                    'from_account': command.from_account_id,
                    'to_account': command.to_account_id,
                    'amount': str(command.amount),  # Use string for exact decimal
                    'currency': command.currency,
                    'reference': command.reference,
                    'initiated_by': command.user_id,
                })
            })

            # Write to outbox for query side update
            self.db.insert('event_outbox', {
                'event_id': event_id,
                'status': 'pending'
            })

        return event_id


# QUERY SIDE: Read model (materialized account balances)
class AccountBalanceProjection:
    """
    Denormalized read model optimized for balance queries.
    Updated asynchronously from event store via outbox.
    """

    def rebuild_from_events(self, account_id: str, as_of: datetime = None) -> Decimal:
        """
        Reconstruct balance at any point in time from event history.
        Uses snapshots to avoid full replay for recent state.
        """
        # Find most recent snapshot before 'as_of'
        snapshot_query = """
            SELECT balance, event_version, snapshot_time
            FROM account_snapshots
            WHERE account_id = %s
              AND snapshot_time <= COALESCE(%s, NOW())
            ORDER BY event_version DESC LIMIT 1
        """
        snapshot = self.db.query(snapshot_query, (account_id, as_of))

        start_version = 0
        balance = Decimal('0')

        if snapshot:
            balance = snapshot['balance']
            start_version = snapshot['event_version']

        # Replay events after snapshot
        events_query = """
            SELECT event_type, payload, version
            FROM financial_events
            WHERE account_id = %s
              AND version > %s
              AND (%(as_of)s IS NULL OR timestamp <= %(as_of)s)
            ORDER BY version ASC
        """
        events = self.db.query(events_query, {
            'account_id': account_id,
            'start_version': start_version,
            'as_of': as_of
        })

        for event in events:
            payload = json.loads(event['payload'])
            if event['event_type'] == 'MoneyTransferred':
                if payload['from_account'] == account_id:
                    balance -= Decimal(payload['amount'])
                else:
                    balance += Decimal(payload['amount'])
            elif event['event_type'] == 'MoneyDeposited':
                balance += Decimal(payload['amount'])

        return balance

    def get_current_balance(self, account_id: str) -> Decimal:
        """Fast balance read from materialized view — O(1)."""
        result = self.cache.get(f"balance:{account_id}")
        if result is not None:
            return Decimal(result)

        # Fall back to read model table (updated asynchronously)
        balance = self.db.query(
            "SELECT balance FROM account_balances WHERE account_id = %s",
            account_id
        )
        self.cache.set(f"balance:{account_id}", str(balance), ex=60)
        return balance
```

**Snapshot strategy (critical for large event stores):**

```sql
-- Create snapshot every 1000 events per account
-- Triggered by projection worker
CREATE OR REPLACE FUNCTION create_account_snapshot(p_account_id TEXT) RETURNS VOID AS $$
DECLARE
    v_balance DECIMAL(20, 8);
    v_version BIGINT;
BEGIN
    -- Compute current balance from all events
    SELECT
        SUM(CASE
            WHEN event_type = 'MoneyDeposited' THEN (payload->>'amount')::DECIMAL
            WHEN event_type = 'MoneyTransferred' AND payload->>'to_account' = p_account_id
                THEN (payload->>'amount')::DECIMAL
            WHEN event_type = 'MoneyTransferred' AND payload->>'from_account' = p_account_id
                THEN -(payload->>'amount')::DECIMAL
            ELSE 0
        END),
        MAX(version)
    INTO v_balance, v_version
    FROM financial_events
    WHERE account_id = p_account_id;

    -- Upsert snapshot
    INSERT INTO account_snapshots (account_id, balance, event_version, snapshot_time)
    VALUES (p_account_id, v_balance, v_version, NOW())
    ON CONFLICT (account_id) DO UPDATE
        SET balance = EXCLUDED.balance,
            event_version = EXCLUDED.event_version,
            snapshot_time = EXCLUDED.snapshot_time
    WHERE EXCLUDED.event_version > account_snapshots.event_version;
END;
$$ LANGUAGE plpgsql;
```

---

## Q5: Compare PostgreSQL vs Cassandra vs DynamoDB for a global social media feed.

**Answer:**

**Workload characteristics:**
- 500M users, average 200 follows
- 50M posts/day
- Feed read: get last 100 posts from followed users
- Write: post goes to all followers' feeds (fan-out)

**Option 1: PostgreSQL with fan-out on read**

```sql
-- Schema: posts(id, author_id, content, created_at), follows(follower_id, following_id)

-- Get feed: JOIN-based, fan-out on read
SELECT p.id, p.content, p.created_at, u.name
FROM posts p
JOIN follows f ON f.following_id = p.author_id
JOIN users u ON u.id = p.author_id
WHERE f.follower_id = :user_id
  AND p.created_at > NOW() - INTERVAL '7 days'
ORDER BY p.created_at DESC LIMIT 100;

-- Analysis:
-- For 200 follows: JOIN scans up to 200 users' recent posts → manageable at small scale
-- At 500M users: this query hits 200 different partitions, slows to seconds
-- Problem: celebrities with 50M followers → fan-out is O(followers) per post write
```

**Option 2: Cassandra with fan-out on write (pre-computed feeds)**

```python
# Schema: user_feed (user_id, post_time, post_id, author_id, content)
# Partition key: user_id → one partition per user's feed
# Clustering key: post_time DESC → newest first within partition

# Write: fan out to all followers' feeds
def publish_post(author_id: str, content: str):
    post_id = str(uuid.uuid4())
    post_time = datetime.utcnow()

    # 1. Store post in posts table (source of truth)
    cassandra.execute(
        "INSERT INTO posts (post_id, author_id, content, created_at) VALUES (%s,%s,%s,%s)",
        (post_id, author_id, content, post_time)
    )

    # 2. Fan-out: insert into EVERY follower's feed
    # This is the expensive part — O(followers) writes
    followers = get_followers(author_id)  # Could be 50M for celebrities!

    for follower_id in followers:
        cassandra.execute_async("""
            INSERT INTO user_feed (user_id, post_time, post_id, author_id, content)
            VALUES (%s, %s, %s, %s, %s)
            USING TTL 604800  -- 7-day TTL on feed items
        """, (follower_id, post_time, post_id, author_id, content))

# Read: single partition scan — O(1) no matter how many follows
def get_feed(user_id: str, limit: int = 100):
    return cassandra.execute("""
        SELECT post_time, post_id, author_id, content
        FROM user_feed
        WHERE user_id = %s
        ORDER BY post_time DESC
        LIMIT %s
    """, (user_id, limit))
```

**Option 3: Hybrid (DynamoDB + Redis + selective fan-out)**

```python
# Problem with pure fan-out-on-write: celebrity with 50M followers creates 50M writes
# Solution: fan-out to cache for normal users, pull-on-read for celebrity follows

def publish_post_hybrid(author_id: str, content: str):
    post_id = str(uuid.uuid4())
    post_time = int(datetime.utcnow().timestamp() * 1000)

    # 1. Store post in DynamoDB
    posts_table.put_item(Item={
        'post_id': post_id,
        'author_id': author_id,
        'content': content,
        'created_at': post_time
    })

    follower_count = get_follower_count(author_id)

    if follower_count < 1000:  # Regular user: fan-out to Redis feeds
        followers = get_followers(author_id)
        for follower_id in followers:
            redis.lpush(f"feed:{follower_id}", json.dumps({
                'post_id': post_id, 'author_id': author_id,
                'content': content, 'time': post_time
            }))
            redis.ltrim(f"feed:{follower_id}", 0, 999)  # Keep last 1000 items
    # For celebrities (>1000 followers): don't fan-out, pull on read instead

def get_feed_hybrid(user_id: str) -> list:
    # Get pre-computed feed from Redis
    feed_items = redis.lrange(f"feed:{user_id}", 0, 99)
    feed = [json.loads(item) for item in feed_items]

    # Merge in celebrity posts (pull-on-read for followed celebrities)
    celeb_follows = get_celebrity_follows(user_id)  # Users followed who have >1K followers
    for celeb_id in celeb_follows[:10]:  # Limit to 10 celebrities
        recent_posts = posts_table.query(
            IndexName='author_id-created_at-index',
            KeyConditionExpression=Key('author_id').eq(celeb_id),
            ScanIndexForward=False,
            Limit=10
        )['Items']
        feed.extend(recent_posts)

    # Sort merged feed by time, take top 100
    feed.sort(key=lambda x: x['time'], reverse=True)
    return feed[:100]
```

**Final recommendation by scale:**
- < 1M users: PostgreSQL with read replicas (simple, ACID, full SQL)
- 1M-10M users: Cassandra with fan-out-on-write (predictable read latency)
- > 10M users: Hybrid approach (fan-out for regular users, pull-on-read for celebrities)

The celebrity problem is the key insight — no single approach handles both 200-follower
users and 50M-follower celebrities optimally.
