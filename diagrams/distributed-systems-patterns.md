# Distributed Systems Patterns — Diagram Library

---

## 1. Message Queue

**Purpose:** Decouples producers and consumers, enabling asynchronous processing and load leveling.

```
   PRODUCER SIDE                                        CONSUMER SIDE

   ┌─────────────┐                                   ┌───────────────┐
   │  Order      │                                   │  Email        │
   │  Service    │──┐                         ┌─────►│  Service      │
   └─────────────┘  │                         │      └───────────────┘
                    │                         │
   ┌─────────────┐  │    ┌─────────────────┐  │      ┌───────────────┐
   │  Payment    │  │    │   MESSAGE QUEUE  │  ├─────►│  Analytics    │
   │  Service    │──┼───►│                 │  │      │  Service      │
   └─────────────┘  │    │  ┌───────────┐  │  │      └───────────────┘
                    │    │  │ Message 1 │  │  │
   ┌─────────────┐  │    │  │ Message 2 │  │  │      ┌───────────────┐
   │  Inventory  │  │    │  │ Message 3 │  │  └─────►│  Inventory    │
   │  Service    │──┘    │  │ Message N │  │         │  Consumer     │
   └─────────────┘       │  └───────────┘  │         └───────────────┘
                         │                 │
                         │  Queue: order.  │
                         │  created        │
                         └─────────────────┘

   Properties:
   • Durability: messages persisted to disk
   • Acknowledgment: consumer acks after processing
   • Dead Letter Queue: failed messages → DLQ
   • Visibility timeout: prevents duplicate processing
```

**Key concepts:**
1. **At-least-once delivery:** Message delivered until ACKed. Consumers must be idempotent.
2. **At-most-once delivery:** Fire-and-forget. Message may be lost but never duplicated.
3. **Exactly-once delivery:** Most expensive. Requires transactions (Kafka exactly-once semantics).

---

## 2. Event-Driven Architecture

**Purpose:** Services react to events rather than calling each other directly, achieving loose coupling.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      EVENT-DRIVEN ARCHITECTURE                           │
│                                                                          │
│  ┌──────────────┐     publishes      ┌──────────────────────────────┐   │
│  │              │    "order.placed"  │                              │   │
│  │    Order     │───────────────────►│         EVENT BUS            │   │
│  │   Service    │                   │   (Kafka / EventBridge)       │   │
│  │  (Producer)  │    publishes       │                              │   │
│  │              │    "payment.done"  │  Topics:                     │   │
│  └──────────────┘───────────────────►│  • order.placed             │   │
│                                      │  • payment.completed         │   │
│  ┌──────────────┐     publishes      │  • inventory.updated         │   │
│  │   Payment    │    "payment.done"  │  • notification.requested    │   │
│  │   Service    │───────────────────►│                              │   │
│  │  (Producer)  │                   └──────────────┬───────────────┘   │
│  └──────────────┘                                  │                   │
│                                                     │                   │
│         ┌──────────────────────┬────────────────────┤                   │
│         │                      │                    │                   │
│         ▼                      ▼                    ▼                   │
│  ┌──────────────┐   ┌──────────────────┐  ┌──────────────────┐        │
│  │  Inventory   │   │  Notification    │  │   Analytics      │        │
│  │   Service    │   │   Service        │  │   Service        │        │
│  │  (Consumer)  │   │  (Consumer)      │  │  (Consumer)      │        │
│  │              │   │                  │  │                  │        │
│  │ Updates stock│   │ Sends email/SMS  │  │ Records metrics  │        │
│  └──────────────┘   └──────────────────┘  └──────────────────┘        │
└──────────────────────────────────────────────────────────────────────────┘
```

**Benefits:**
- Services don't know about each other (loose coupling)
- New consumers can be added without changing producers
- Natural audit log of everything that happened

**Challenges:**
- Eventual consistency — data may be stale briefly
- Debugging is harder (events flow asynchronously)
- Handling out-of-order events

---

## 3. Saga Pattern

**Purpose:** Manages distributed transactions across multiple services using a sequence of local transactions with compensating transactions on failure.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     SAGA PATTERN (Choreography)                         │
│                                                                         │
│  Happy Path:                                                            │
│                                                                         │
│  ┌───────────┐  order.placed  ┌───────────┐  payment.done              │
│  │  Order    │───────────────►│  Payment  │─────────────────┐           │
│  │  Service  │                │  Service  │                 │           │
│  └───────────┘                └───────────┘                 │           │
│                                                             ▼           │
│                                              ┌─────────────────────┐   │
│                                              │  Inventory Service  │   │
│                                              │  (reserve stock)    │   │
│                                              └──────────┬──────────┘   │
│                                                         │               │
│                                              inventory.reserved         │
│                                                         │               │
│                                                         ▼               │
│                                              ┌─────────────────────┐   │
│                                              │  Shipping Service   │   │
│                                              │  (create shipment)  │   │
│                                              └─────────────────────┘   │
│                                                                         │
│  Failure Path (compensation):                                           │
│                                                                         │
│  ┌───────────┐ payment.failed ┌───────────┐                            │
│  │  Order    │◄───────────────│  Payment  │                            │
│  │  Service  │                │  Service  │                            │
│  │  CANCEL   │                │  REFUND   │                            │
│  └───────────┘                └───────────┘                            │
│       ▲                                                                 │
│       │ order.cancelled                                                  │
│       │                                                                 │
│  ┌────┴──────────────────┐                                              │
│  │  Inventory Service    │ ← releases reserved stock                   │
│  └───────────────────────┘                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

**Saga variants:**
1. **Choreography:** Each service listens for events and reacts. No central coordinator.
2. **Orchestration:** A central saga orchestrator tells each service what to do.

---

## 4. CQRS (Command Query Responsibility Segregation)

**Purpose:** Separate the read model from the write model to allow independent scaling and optimization.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                 CQRS                                     │
│                                                                          │
│  CLIENT                                                                  │
│  ┌──────────┐                                                            │
│  │  Web UI  │                                                            │
│  └────┬─────┘                                                            │
│       │                                                                  │
│    ┌──┴──────────────────────────────────┐                               │
│    │             API LAYER               │                               │
│    └──┬──────────────────────────────────┘                               │
│       │                   │                                              │
│       ▼ Commands          ▼ Queries                                      │
│  ┌─────────────┐     ┌──────────────┐                                   │
│  │  WRITE SIDE │     │  READ SIDE   │                                   │
│  │             │     │              │                                   │
│  │ ┌─────────┐ │     │ ┌──────────┐ │                                   │
│  │ │ Command │ │     │ │  Query   │ │                                   │
│  │ │ Handler │ │     │ │  Handler │ │                                   │
│  │ └────┬────┘ │     │ └────┬─────┘ │                                   │
│  │      │      │     │      │       │                                   │
│  │      ▼      │     │      ▼       │                                   │
│  │ ┌─────────┐ │     │ ┌──────────┐ │                                   │
│  │ │  Write  │ │     │ │  Read    │ │                                   │
│  │ │   DB    │ │     │ │   DB     │ │                                   │
│  │ │(Postgres│ │     │ │(Elastic- │ │                                   │
│  │ │normalized│     │ │ search / │ │                                   │
│  │ │  tables)│ │     │ │ Redis /  │ │                                   │
│  │ └─────────┘ │     │ │ denorm.) │ │                                   │
│  └─────────────┘     │ └──────────┘ │                                   │
│         │            └──────────────┘                                   │
│         │ Domain Events                                                  │
│         └─────────────────────────►  Event Bus ────► Projection         │
│                                                       (updates read DB)  │
└──────────────────────────────────────────────────────────────────────────┘
```

**When to use CQRS:**
- High read-to-write ratio (reads can be scaled independently)
- Read model needs different structure than write model
- Complex domain with many business rules on the write side

---

## 5. Event Sourcing

**Purpose:** Store the sequence of events that led to the current state, rather than storing the current state directly.

```
┌───────────────────────────────────────────────────────────────────────┐
│                        EVENT SOURCING                                 │
│                                                                       │
│  TRADITIONAL (store current state):                                   │
│  ┌────────────────────────────────────────────────────┐               │
│  │  accounts table                                    │               │
│  │  id | balance | updated_at                         │               │
│  │  1  | $500    | 2024-01-15                         │               │
│  └────────────────────────────────────────────────────┘               │
│                                                                       │
│  EVENT SOURCED (store all events):                                    │
│  ┌────────────────────────────────────────────────────┐               │
│  │  account_events table (append-only)                │               │
│  │  id | event_type    | amount | timestamp           │               │
│  │  1  | AccountOpened | $0     | 2024-01-01          │               │
│  │  2  | MoneyDeposited| $1000  | 2024-01-05          │               │
│  │  3  | MoneyWithdrawn| $300   | 2024-01-10          │               │
│  │  4  | MoneyDeposited| $200   | 2024-01-12          │               │
│  │  5  | MoneyWithdrawn| $400   | 2024-01-15          │               │
│  │     → Current state = 0+1000-300+200-400 = $500    │               │
│  └────────────────────────────────────────────────────┘               │
│                                                                       │
│  SNAPSHOT + EVENTS (performance optimization):                        │
│                                                                       │
│  ┌──────────────────┐    ┌─────────────────────────────────────────┐  │
│  │  Snapshot        │    │  Events after snapshot                  │  │
│  │  balance: $300   │    │  MoneyDeposited: $100                   │  │
│  │  at event #100   │ +  │  MoneyWithdrawn: $50                    │  │
│  └──────────────────┘    │  MoneyDeposited: $150                   │  │
│                           └─────────────────────────────────────────┘  │
│       Current state = $300 + $100 - $50 + $150 = $500 (faster!)        │
│                                                                       │
│  PROJECTIONS (read models built from events):                         │
│                                                                       │
│  Event Stream ──► Projection A ──► Account Balance View               │
│                └► Projection B ──► Transaction History View           │
│                └► Projection C ──► Fraud Detection Model              │
└───────────────────────────────────────────────────────────────────────┘
```

**Benefits of Event Sourcing:**
1. Complete audit trail — you know exactly what happened and when
2. Time travel — rebuild state at any point in history
3. Event replay — rebuild read models from scratch
4. Temporal decoupling — process events at different rates

**Challenges:**
1. Event schema evolution is complex
2. Querying current state requires replaying or snapshots
3. High storage usage over time
