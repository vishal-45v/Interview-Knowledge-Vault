# Chapter 10: Database Selection — Analogy Explanations

## Analogy 1: Database Selection Framework — Choosing a Vehicle for a Trip

**The Story:**
Nobody chooses a vehicle based on its maximum speed first. You ask: How many people?
How much luggage? What terrain? What budget? A sports car is amazing for a solo trip
on a highway but terrible for moving furniture across a muddy field. A pickup truck
handles moving day but wastes fuel on the daily commute. The wrong vehicle for the job
costs money and causes frustration even if the vehicle is technically excellent.

**Connection to databases:**
The biggest mistake in database selection is asking "which database is the fastest/most
popular/what Netflix uses?" before answering:
- What shape is my data? (structured rows, documents, graphs, time-ordered)
- What questions do I need to answer? (point lookups, range scans, joins, full-text)
- What consistency do I need? (real-time balance, or eventually correct feed)
- What scale do I realistically need in year 1 vs year 3?

```
ACCESS PATTERN → DATABASE SELECTION:

"Get user by email"         →  Indexed lookup  →  Any relational or key-value
"Friends of friends"        →  Graph traversal →  Neo4j, Amazon Neptune
"Articles containing word X" → Full-text search → Elasticsearch, PostgreSQL FTS
"Avg sensor reading by hour" → Time aggregation → TimescaleDB, InfluxDB, ClickHouse
"User's last 100 actions"   →  Time-ordered list →  Cassandra, DynamoDB, Redis Stream
"Is this user allowed to X" →  Sub-ms lookup   →  Redis, DynamoDB
"Run monthly revenue report" → Complex joins    →  PostgreSQL, BigQuery, Redshift

The vehicle is selected AFTER the route is known.
```

---

## Analogy 2: CAP Theorem — The Two Generals Problem at a Restaurant

**The Story:**
Imagine two restaurant managers who must agree on whether to order more food before a
busy weekend. They communicate by phone (the network). If the call drops (network partition),
they have three options:
1. Neither orders until they can reconnect (choose Consistency over Availability)
2. Both order based on their best guess (choose Availability over Consistency)
3. Pretend the problem doesn't exist (lose either consistency or availability, unpredictably)

There is no option 4 where both managers independently make the perfect coordinated decision
without communication. This is the CAP theorem — you cannot have both Consistency AND
Availability during a Partition.

**Connection to the database:**
Modern PACELC extends this to normal (non-partitioned) operation:

```
RESTAURANT ANALOGY → DATABASE BEHAVIOR:

POSTGRESQL (CP):
"During phone outage, CLOSE the restaurant rather than risk different daily specials"
→ Under partition: stop accepting writes (maintain consistency)
→ Normal operation: low latency OR strong consistency (you choose per query)

CASSANDRA (AP, tunable):
"During outage, EACH MANAGER makes their best decision and we'll reconcile later"
→ Under partition: both halves accept writes (availability maintained)
→ Eventual consistency: reads may return stale data
→ Write reconciliation: last-write-wins with vector clocks
→ Tunable: QUORUM read/write = CP-like behavior at cost of latency

DYNAMODB (configurable):
"Some decisions need full confirmation (financial); others are ok with best-effort (catalog)"
→ Per-request consistency model: choose CP or AP per query
→ EventuallyConsistent reads = AP, 0.5 RCU per KB
→ StronglyConsistent reads  = CP, 1.0 RCU per KB
→ Transactions              = fully ACID, 2x cost
```

```python
# The right consistency level depends on the business operation

# ALWAYS strong consistency: financial operations
balance = dynamodb_table.get_item(
    Key={'account_id': 'ACC-001'},
    ConsistentRead=True   # CP — pay 2x, ensure accuracy
)['Item']['balance']

# Eventual consistency acceptable: product catalog browsing
product = dynamodb_table.get_item(
    Key={'product_id': 'PROD-001'},
    # ConsistentRead=False (default) — AP, save 50% on read cost
)['Item']
# If product details are 100ms stale: who cares? Shopper doesn't notice.
# If account balance is 100ms stale: double-spend risk. Unacceptable.
```

---

## Analogy 3: Polyglot Persistence — The Hospital's Specialized Departments

**The Story:**
A hospital doesn't use one room for everything. There's an ER for emergencies (fast,
triage-based), an ICU for critical monitoring (continuous telemetry), an MRI suite for
imaging (specialized equipment), a pharmacy (inventory management), and administrative
offices (records and billing). Each room has specialized equipment optimized for its
purpose. The hospital's total capability is greater than any one room.

But coordinating across rooms creates complexity: if an ER doctor orders a medication,
the pharmacy must fulfill it, the ICU must be notified, and billing must record it —
all consistently. The hospital solves this with shared patient records (the database)
and standardized communication protocols (the messaging system).

**Connection to databases:**
Each hospital department is a specialized database. The patient record system is the
event bus that keeps everything synchronized.

```
POLYGLOT PERSISTENCE: E-commerce platform

PostgreSQL (Administrative Office — ACID source of truth)
├── user accounts, orders, payments, inventory
└── "The authoritative patient record"

Redis (ER — fast triage, ephemeral state)
├── shopping carts (session data, auto-expires)
├── rate limiting counters
└── "Emergency triage: fast but no long-term records"

Elasticsearch (MRI Suite — deep analysis)
├── product search with full-text + facets + geo
└── "Specialized imaging: can't store primary data but reveals what others can't"

Cassandra (ICU — continuous telemetry monitoring)
├── user activity feed, click events, behavioral data
└── "Constant monitoring: high-frequency writes, time-ordered reads"

Neo4j (Pharmacy — relationship tracking)
├── product recommendations (users who bought X also bought Y)
└── "Complex relationship lookups: which treatments interact with which drugs"

EVENT BUS (Kafka) — the hospital intercom:
├── PostgreSQL order event → update Cassandra feed
├── PostgreSQL product update → reindex in Elasticsearch
└── Cassandra click event → update Neo4j recommendation graph
```

```python
# The critical challenge: keeping departments synchronized
# Outbox pattern = hospital's standardized handoff note
def place_order(user_id: str, cart: list):
    with postgres.transaction():
        # Admit to hospital (primary record)
        order_id = postgres.insert_order(user_id, cart)

        # Write handoff notes for other departments (outbox)
        postgres.insert('outbox', {
            'event': 'order_placed',
            'payload': {'order_id': order_id, 'user_id': user_id, 'cart': cart}
        })
    # Background workers deliver the notes to other departments
    # If a worker fails, the note is re-delivered (at-least-once)
```

---

## Analogy 4: Strangler Fig Migration — The Building Renovation While Occupied

**The Story:**
A historic building is too small and structurally outdated. Demolishing it and rebuilding
from scratch would require all tenants to leave for 2 years — unacceptable. Instead,
a new structure is built AROUND the old building, floor by floor. As each new floor is
ready, tenants move in. The old building slowly shrinks as tenants leave until only the
facade remains, then is finally removed. Business continues throughout.

The Strangler Fig pattern works the same way for database migrations: build the new
system around the old one, gradually move capabilities, and strangle the old system's
scope until it can be decommissioned.

**Connection to the database:**

```
STRANGLER FIG MIGRATION: PostgreSQL monolith → microservices

PHASE 1: Identify boundaries (2 months)
PostgreSQL monolith:
  users table ───────────────────────────────► User Service (future)
  products table ────────────────────────────► Product Service (future)
  orders table ──────────────────────────────► Order Service (future)
  inventory table ───────────────────────────► Inventory Service (future)

PHASE 2: Extract User Service (months 3-4)
┌─────────────────────────────────────────────────────────────┐
│ New: User Service (PostgreSQL + Redis)                       │
│ GET /users/{id} → returns user data from new DB             │
└──────────────────────────┬──────────────────────────────────┘
                           │ dual-write during transition
┌──────────────────────────▼──────────────────────────────────┐
│ Old: Monolith PostgreSQL (still has users table)             │
│ All other services still READ from old DB                    │
└─────────────────────────────────────────────────────────────┘
Traffic split:
  User reads:  100% → New User Service
  User writes: 100% → New User Service (with dual-write to old DB)
  Orders:      100% → Monolith (unchanged)

PHASE 3: Validate + cutover (month 5)
  Shadow read comparison: New DB result == Old DB result? (automated)
  Mismatch rate target: < 0.001%
  Cutover: stop dual-write to old DB, old users table archived

PHASE 4: Repeat for Products, Orders, Inventory (months 6-18)
```

```python
# Dual-write implementation with comparison
class UserServiceProxy:
    def get_user(self, user_id: str):
        new_result = new_user_service.get(user_id)

        if SHADOW_READ_ENABLED:
            old_result = legacy_db.query_user(user_id)
            if normalize(new_result) != normalize(old_result):
                metrics.increment('user_service.shadow_mismatch')
                logger.warning(f"Shadow mismatch for {user_id}: {diff(new_result, old_result)}")
                # Use old result during shadow period (old DB is source of truth)
                return old_result

        return new_result

    def create_user(self, user_data: dict):
        # Write to new system first
        user_id = new_user_service.create(user_data)
        # Dual-write to old system (for rollback safety)
        try:
            legacy_db.insert_user(user_id, user_data)
        except Exception as e:
            logger.error(f"Legacy dual-write failed: {e}")
            # Alert but don't fail — new system is source of truth
        return user_id
```

---

## Analogy 5: Event Sourcing — The Accounting Ledger

**The Story:**
A traditional accountant records CURRENT balances in ledger accounts: "Checking Account:
$5,432.00." If you want to know why the balance is that amount, you must audit the files.

A sophisticated accountant records every TRANSACTION: "$5,000 deposit on Jan 1," "$1,000
withdrawal on Jan 15," "$432 interest on Jan 31." The current balance is DERIVED by summing
the transactions. You can reconstruct the balance at any date. You can audit every change
without a separate audit trail.

This is event sourcing. The events (transactions) are immutable facts. The current state
(balance) is a computed view of those facts.

**Connection to the database:**

```python
# Event sourced order management
EVENTS_TIMELINE = [
    {"type": "OrderCreated",    "time": "T+0", "data": {"items": ["SKU-1","SKU-2"], "total": 150.00}},
    {"type": "PaymentReceived", "time": "T+5m","data": {"amount": 150.00, "method": "Visa"}},
    {"type": "ItemShipped",     "time": "T+2d","data": {"item": "SKU-1", "tracking": "UPS-123"}},
    {"type": "ItemShipped",     "time": "T+2d","data": {"item": "SKU-2", "tracking": "UPS-456"}},
    {"type": "ItemDelivered",   "time": "T+5d","data": {"item": "SKU-1"}},
    {"type": "ItemReturned",    "time": "T+10d","data": {"item": "SKU-2", "reason": "Wrong size"}},
    {"type": "RefundIssued",    "time": "T+12d","data": {"amount": 75.00}},
]

# CURRENT STATE: reconstructed by replaying all events
def get_current_order_state(order_id: str) -> dict:
    state = {"status": None, "items": {}, "balance_due": 0}

    for event in event_store.get_events(order_id):
        if event.type == "OrderCreated":
            state["status"] = "pending_payment"
            state["items"] = {item: "pending" for item in event.data["items"]}
            state["balance_due"] = event.data["total"]

        elif event.type == "PaymentReceived":
            state["status"] = "paid"
            state["balance_due"] -= event.data["amount"]

        elif event.type == "ItemShipped":
            state["items"][event.data["item"]] = "shipped"
            if all(v == "shipped" for v in state["items"].values()):
                state["status"] = "shipped"

        elif event.type == "ItemDelivered":
            state["items"][event.data["item"]] = "delivered"

        elif event.type == "ItemReturned":
            state["items"][event.data["item"]] = "returned"

        elif event.type == "RefundIssued":
            state["balance_due"] -= event.data["amount"]
            state["status"] = "partial_refund"

    return state  # Current truth derived from immutable history

# POINT-IN-TIME QUERY: What was the order status 3 days after creation?
# Just replay events up to T+3d — no "time travel" query language needed
def get_order_state_at(order_id: str, as_of: datetime) -> dict:
    state = {}
    for event in event_store.get_events_before(order_id, as_of):
        apply_event(state, event)
    return state
```

---

## Analogy 6: CQRS — The Restaurant Order and Kitchen Board

**The Story:**
In a busy restaurant, a waiter takes your order (Command). The waiter doesn't go to the
kitchen, cook your food, and bring it back personally — that would serialize all service.
Instead, the waiter writes the order on a ticket and puts it on the kitchen board (the
Write Model). The kitchen cooks all orders from the board. A separate expediter monitors
the ready meals board (the Read Model) and calls orders when ready.

The "order board" (write model) and the "ready board" (read model) are separate. They're
synchronized, but not identical. When your food is ready, the expediter's board is updated.
If you asked "is order 42 ready?" before the board was updated (while the cook is still
plating), you'd get a slightly stale answer — but that's usually fine.

**Connection to the database:**

```
CQRS PATTERN: E-commerce Order Service

COMMAND SIDE (Write Model):
┌─────────────────────────────────────────────────────────────────┐
│ Commands: PlaceOrder, CancelOrder, UpdateShipping               │
│ Validated against business rules                                │
│ Stored in: PostgreSQL (normalized, ACID, append-only events)    │
│ Optimized for: write throughput, consistency, audit trail       │
└──────────────────────────────┬──────────────────────────────────┘
                               │ Events published to message bus
                               ▼
QUERY SIDE (Read Model):
┌─────────────────────────────────────────────────────────────────┐
│ Projections updated by event handlers:                          │
│   "Order dashboard" → denormalized summary in Redis             │
│   "Order history"   → materialized view in PostgreSQL read DB   │
│   "Shipping status" → DynamoDB for per-order tracking           │
│   "Analytics"       → ClickHouse for reporting                  │
│ Optimized for: read throughput, specific query shapes           │
└─────────────────────────────────────────────────────────────────┘

SYNCHRONIZATION:
Event: OrderPlaced → Update ALL read models asynchronously
(Read models may lag write model by milliseconds to seconds)
```

```python
# Application-level read-your-writes for CQRS (avoid stale read after write)
class OrderService:
    def place_order(self, user_id: str, items: list) -> str:
        order_id = command_handler.place_order(user_id, items)

        # Force read-your-writes by waiting for event propagation
        # OR: bypass read model and read from write model for "just created" resources
        return order_id

    def get_order(self, order_id: str, bypass_cache: bool = False) -> dict:
        if bypass_cache:
            # Read from command store directly (consistent but slower)
            return command_store.reconstruct_order(order_id)
        # Read from read model (fast but potentially stale by <1 second)
        return read_model.get_order(order_id)
```

CQRS adds value when read patterns are VERY different from write patterns — when the
optimal read storage (pre-aggregated, denormalized, cached) would be terrible for writes
(normalized, consistent, ACID). If your read and write models are similar, CQRS adds
complexity without benefit.
