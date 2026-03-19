# Chapter 10: Database Selection — Follow-Up Traps

## Trap 1: "NoSQL is for scale; relational is for consistency — pick based on scale"

**What most people say:**
"If you need to scale horizontally, use NoSQL. If you need ACID, use SQL."

**Correct answer:**
This framing is a decade out of date. The real dimensions are:
1. **Access patterns**: Are you doing point lookups, range scans, graph traversal, or full-text search? These determine query model, not scale.
2. **Data model**: Do you have a known schema (relational works great), highly variable attributes (document works well), or highly connected data (graph wins)?
3. **Consistency requirements**: These exist on a spectrum — not just "ACID vs eventual."
4. **Scale**: Only after 1-3 are clear.

Counterexamples to the "NoSQL = scale" myth:
- CockroachDB and Google Spanner are distributed SQL databases that scale horizontally and maintain full ACID semantics.
- MongoDB at extreme scale requires more operational expertise than PostgreSQL with Citus (distributed Postgres).
- A single well-tuned PostgreSQL instance handles 100K queries/second — far more than most startups ever need.
- DynamoDB gives you massive scale but LIMITS your query patterns severely (no joins, no arbitrary WHERE clauses).

```sql
-- PostgreSQL with Citus: horizontal sharding without leaving SQL
-- This scales to terabytes while maintaining full SQL + ACID
SELECT
    user_id,
    SUM(order_total) AS lifetime_value
FROM orders
WHERE created_at > NOW() - INTERVAL '90 days'
GROUP BY user_id
ORDER BY lifetime_value DESC LIMIT 100;
-- With Citus: this query is distributed across N worker nodes
-- Workers compute partial aggregations; coordinator combines results
```

## Trap 2: "DynamoDB is always eventually consistent"

**What most people say:**
"DynamoDB is a NoSQL database, so reads are eventually consistent."

**Correct answer:**
DynamoDB offers TWO read consistency models, and the choice has significant implications:

- **Eventually Consistent Reads (default)**: Uses any replica, may return stale data.
  Consumes 0.5 read capacity units per KB.
- **Strongly Consistent Reads**: Goes to the leader replica, always returns the latest data.
  Consumes 1 read capacity unit per KB (2x cost).

```python
import boto3

dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('UserProfiles')

# Eventually consistent read (default) — cheaper but may be stale
response = table.get_item(
    Key={'user_id': 'user-123'},
    # ConsistentRead defaults to False
)

# Strongly consistent read — always up-to-date, 2x RCU cost
response = table.get_item(
    Key={'user_id': 'user-123'},
    ConsistentRead=True   # Read from leader replica
)

# DynamoDB Transactions — fully ACID across up to 100 items
response = dynamodb.meta.client.transact_write(
    TransactItems=[
        {
            'Update': {
                'TableName': 'accounts',
                'Key': {'account_id': 'ACC-001'},
                'UpdateExpression': 'SET balance = balance - :amount',
                'ConditionExpression': 'balance >= :amount',
                'ExpressionAttributeValues': {':amount': Decimal('100')}
            }
        },
        {
            'Update': {
                'TableName': 'accounts',
                'Key': {'account_id': 'ACC-002'},
                'UpdateExpression': 'SET balance = balance + :amount',
                'ExpressionAttributeValues': {':amount': Decimal('100')}
            }
        }
    ]
)
# Transact_write is FULLY ACID — either both updates succeed or neither does
```

DynamoDB transactions exist and are ACID. The trap is assuming "DynamoDB = no transactions."

## Trap 3: "MongoDB is schema-less so you don't need to plan your schema"

**What most people say:**
"MongoDB's flexibility means you can store anything without planning. Great for agile
development because you can change schema anytime."

**Correct answer:**
MongoDB is "schema-enforced-by-application" not "schema-free." The schema is enforced
by your application code and indexes rather than by the database engine. This flexibility
is BOTH an advantage and a trap:

Advantage: Add new optional fields to new documents without migration. Works well for
sparse, variable-attribute data.

Trap: Application bugs can insert documents with wrong field names or wrong types. Without
a schema validator, you silently corrupt data. Also, without careful design, MongoDB
documents grow unboundedly (the "unbounded array" anti-pattern).

```javascript
// ANTI-PATTERN: Unbounded array in a document (common MongoDB mistake)
// A blog post with comments stored inside the post document:
{
  "_id": "post-123",
  "title": "My Blog Post",
  "content": "...",
  "comments": [       // ← This array can grow to 10K entries
    {"user": "Alice", "text": "Great post!", "timestamp": "..."},
    {"user": "Bob", "text": "Interesting.", "timestamp": "..."},
    // ... 9,998 more comments
  ]
}
// Problem: MongoDB document max size = 16MB
// Problem: To add one comment, the entire 1MB document is read, modified, rewritten
// Problem: Index on comments.user requires indexing every element in every post's array

// CORRECT PATTERN: Separate collection for high-growth arrays
// Post document:
{ "_id": "post-123", "title": "...", "comment_count": 1234 }

// Comments collection:
{ "_id": ObjectId(), "post_id": "post-123", "user": "Alice", "text": "...", "ts": "..." }
{ "_id": ObjectId(), "post_id": "post-123", "user": "Bob",   "text": "...", "ts": "..." }

// Index for efficient lookup:
db.comments.createIndex({ post_id: 1, ts: -1 })  // comments for a post, newest first

// Enforce schema with validator
db.createCollection("users", {
   validator: {
      $jsonSchema: {
         bsonType: "object",
         required: ["email", "created_at"],
         properties: {
            email: { bsonType: "string", pattern: "^.+@.+$" },
            age:   { bsonType: "int", minimum: 0, maximum: 150 }
         }
      }
   }
})
```

## Trap 4: "CQRS always means eventual consistency for reads"

**What most people say:**
"With CQRS, the read model is updated asynchronously, so reads are always stale."

**Correct answer:**
CQRS does NOT inherently require eventual consistency. The read-write separation is
an architectural pattern, and the synchronization model is a separate concern:

1. **Synchronous CQRS**: Write to both command store and read model in the same transaction
   (or synchronous propagation). Read model is always consistent. Performance cost: all
   writes must update both models before returning.

2. **Asynchronous CQRS**: Write to command store; read model is updated by a background
   process via events. Reads may be stale. Performance benefit: writes return fast.

3. **Mixed**: Strong consistency for recent data, eventual consistency for historical views.

```python
# SYNCHRONOUS CQRS: strong consistency, higher write latency
def place_order(user_id: str, items: list) -> str:
    with db.transaction():
        # Write model: normalized, ACID
        order_id = orders_table.insert({"user_id": user_id, "items": items, "status": "placed"})

        # Read model: denormalized for fast reads, updated synchronously
        order_history_cache.set(
            f"user:{user_id}:recent_orders",
            order_history_cache.get(f"user:{user_id}:recent_orders", []) + [order_id],
            ttl=3600
        )
    return order_id  # Both models consistent before return

# ASYNCHRONOUS CQRS: eventual consistency, lower write latency
def place_order_async(user_id: str, items: list) -> str:
    # Write model only
    order_id = orders_table.insert({"user_id": user_id, "items": items})

    # Emit event; read model updated asynchronously
    event_bus.publish("OrderPlaced", {"order_id": order_id, "user_id": user_id})
    return order_id  # Read model NOT yet updated

# READ-YOUR-WRITES workaround for async CQRS:
# After placing order, check write model first for "just placed" orders,
# fall back to read model for historical orders
```

## Trap 5: "Event sourcing solves all consistency problems"

**What most people say:**
"With event sourcing, every state change is recorded as an immutable event. You can
replay history and always get the correct state. It's the ultimate consistency model."

**Correct answer:**
Event sourcing introduces new consistency challenges:

1. **Snapshot staleness**: Without snapshots, rebuilding state requires replaying ALL events
   from the beginning. At 100M events, this takes hours.

2. **Event version compatibility**: If you change the structure of an event, old event
   replayers break. Versioning events is non-trivial.

3. **Query challenges**: You cannot run "SELECT * WHERE status = 'active'" against an
   event store — you need a projection (separate read model), which brings back all the
   consistency challenges of CQRS.

4. **Concurrent writes**: Two commands processed concurrently can create conflicting events.
   Optimistic concurrency with expected version is required.

```python
# Event store with optimistic concurrency
class OrderEventStore:
    def append_event(self, stream_id: str, event: dict, expected_version: int):
        with db.transaction():
            current_version = self.get_stream_version(stream_id)
            if current_version != expected_version:
                raise ConcurrencyConflictError(
                    f"Expected version {expected_version}, got {current_version}"
                )
            self.db.insert('events', {
                'stream_id': stream_id,
                'version': current_version + 1,
                'event_type': event['type'],
                'data': json.dumps(event['data']),
                'timestamp': datetime.utcnow()
            })

# Snapshot strategy (critical for performance)
def rebuild_order_aggregate(order_id: str) -> Order:
    # Check for snapshot first
    snapshot = snapshot_store.get_latest(order_id)
    start_version = 0
    state = {}

    if snapshot:
        state = snapshot['state']
        start_version = snapshot['version']

    # Replay only events AFTER the snapshot
    events = event_store.get_events_after(order_id, start_version)
    for event in events:
        state = apply_event(state, event)

    return Order(state)
```

Event sourcing is genuinely powerful for audit logs, financial systems, and collaborative
applications, but it is NOT a universal consistency solution — it's an additional
complexity that pays off in specific scenarios.

## Trap 6: "Polyglot persistence means using the 'best tool' for every job"

**What most people say:**
"Use Redis for caching, Elasticsearch for search, Cassandra for time-series, Neo4j for
graphs, and PostgreSQL for transactions. Each is the best tool for its use case."

**Correct answer:**
Polyglot persistence has a HIDDEN COST that is often severely underestimated: cross-database
consistency and operational complexity.

Adding a new database technology means:
1. New operational expertise required (one team cannot be expert in 5 databases)
2. No atomic transactions across databases (each write is two separate operations)
3. Data synchronization bugs (what happens when Elasticsearch is updated but PostgreSQL is not?)
4. Monitoring/alerting for N different systems
5. Backup/restore procedures for N different systems
6. Security patching and upgrades for N different systems

```python
# The cross-database consistency trap:
def create_product(product: dict) -> str:
    # Step 1: Insert into PostgreSQL
    product_id = postgres.insert('products', product)

    # Step 2: Index in Elasticsearch
    # --- FAILURE WINDOW: PostgreSQL has data, ES doesn't ---
    # If the process crashes here, search is inconsistent
    elasticsearch.index('products', product_id, product)

    # Step 3: Cache in Redis
    # --- FAILURE WINDOW: both above succeed but cache is wrong ---
    redis.set(f"product:{product_id}", json.dumps(product), ex=3600)

    return product_id  # 3 separate writes, no atomicity guarantee

# The correct approach: eventual consistency via events
def create_product_with_events(product: dict) -> str:
    # Single atomic write to PostgreSQL + outbox table
    with postgres.transaction():
        product_id = postgres.insert('products', product)
        # Outbox ensures the event is reliably published
        postgres.insert('outbox', {
            'event_type': 'ProductCreated',
            'payload': json.dumps({**product, 'id': product_id}),
            'status': 'pending'
        })

    # Background workers read outbox and update Elasticsearch + Redis
    # If they fail, they retry from the outbox (at-least-once delivery)
    return product_id
```

The decision to add a new database technology should require a "database adoption review"
with a high bar: can PostgreSQL + extensions handle this with acceptable performance?
Only add a new database when the performance or query model advantage is clear and the
team has operational capacity to support it.

## Trap 7: "The Strangler Fig pattern is safe — you can roll back at any time"

**What most people say:**
"The Strangler Fig gradually moves traffic to the new system. If anything goes wrong,
route traffic back to the old system."

**Correct answer:**
Rolling back is safe ONLY if you have maintained bidirectional data synchronization.
During the migration, writes may go to either system (or both). Rollback after a
significant period means you must reconcile writes that happened in the new system
but not the old one.

```
MIGRATION TIMELINE:
Day 1-30:   All traffic → Old DB
Day 31-60:  Dual-write to Old DB + New DB; reads from Old DB
            [Old DB has ALL data; New DB has data from day 31]
Day 61:     Switch reads to New DB
Day 62:     Bug found: New DB missing data from day 45 (bug in dual-write)
ROLLBACK:   Switch reads back to Old DB
            Old DB has everything (safe)
            New DB has partial data → must re-sync from Old DB

BUT: What if you also moved WRITES to New DB on day 61?
Day 61-62:  Writes go to New DB
ROLLBACK:   Old DB is missing writes from day 61-62!
            Must dual-write BACK to Old DB before completing rollback
            → Very complex reconciliation required

CORRECT APPROACH:
1. Always keep old DB as source of truth during migration
2. Only switch writes to new DB AFTER reads are confirmed working (not before)
3. Keep dual-write running in BOTH directions for a "backup period" (2-4 weeks)
   Old DB → New DB (for any legacy paths still writing to Old DB)
   New DB → Old DB (for the new paths writing to New DB)
4. Verify data equivalence with automated comparison jobs before final cutover
5. Final cutover: stop dual-write, decommission old DB
```

```python
# Automated data comparison during migration
def compare_databases(entity_ids: list) -> dict:
    mismatches = []
    for entity_id in entity_ids:
        old_data = old_db.get(entity_id)
        new_data = new_db.get(entity_id)

        if normalize(old_data) != normalize(new_data):
            mismatches.append({
                'id': entity_id,
                'old': old_data,
                'new': new_data,
                'diff': diff(old_data, new_data)
            })

    return {
        'total_checked': len(entity_ids),
        'mismatches': len(mismatches),
        'mismatch_rate': len(mismatches) / len(entity_ids),
        'examples': mismatches[:10]
    }
    # Target: < 0.001% mismatch rate before switching reads
```
