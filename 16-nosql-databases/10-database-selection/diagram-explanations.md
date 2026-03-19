# Chapter 10: Database Selection — Diagram Explanations

## Diagram 1: Database Selection Decision Framework

```
DATABASE SELECTION DECISION TREE
═══════════════════════════════════════════════════════════════════════════

Start: What are your PRIMARY access patterns?

                     ┌─────────────────────────────┐
                     │  What is the dominant        │
                     │  query shape?                │
                     └──────────────┬──────────────┘
             ┌──────────────────────┴─────────────────────┐
             │                                             │
   ┌─────────▼──────────┐                      ┌──────────▼─────────┐
   │ Structured, known  │                      │ Variable, sparse,  │
   │ schema, complex    │                      │ document-shaped,   │
   │ JOINs needed?      │                      │ nested data?       │
   └─────────┬──────────┘                      └──────────┬─────────┘
   Yes ──────┘                                            │ Yes
             ▼                                            ▼
   ┌─────────────────────┐                    ┌──────────────────────┐
   │  Relational         │                    │  Document Store      │
   │  PostgreSQL, MySQL  │                    │  MongoDB, Firestore  │
   │  CockroachDB        │                    │  DynamoDB (document) │
   └─────────────────────┘                    └──────────────────────┘

             ┌──────────────────────────────────────┐
             │  Are you traversing RELATIONSHIPS     │
             │  (friends of friends, supply chain)?  │
             └────────────────┬─────────────────────┘
                              │ Yes
                              ▼
                   ┌──────────────────────┐
                   │  Graph Database      │
                   │  Neo4j, Amazon       │
                   │  Neptune             │
                   └──────────────────────┘

             ┌──────────────────────────────────────┐
             │  Is data TIME-ORDERED with high       │
             │  write throughput (metrics/IoT)?      │
             └────────────────┬─────────────────────┘
                              │ Yes
                              ▼
                   ┌──────────────────────┐
                   │  Time-Series DB      │
                   │  TimescaleDB,        │
                   │  InfluxDB,           │
                   │  ClickHouse (OLAP)   │
                   └──────────────────────┘

             ┌──────────────────────────────────────┐
             │  Do you need FULL-TEXT SEARCH,        │
             │  faceting, or relevance ranking?      │
             └────────────────┬─────────────────────┘
                              │ Yes
                              ▼
                   ┌──────────────────────┐
                   │  Search Engine       │
                   │  Elasticsearch,      │
                   │  OpenSearch,         │
                   │  Meilisearch         │
                   └──────────────────────┘

             ┌──────────────────────────────────────┐
             │  Simple key lookup, sub-millisecond, │
             │  or session/cache data?              │
             └────────────────┬─────────────────────┘
                              │ Yes
                              ▼
                   ┌──────────────────────┐
                   │  Key-Value Store     │
                   │  Redis, Memcached,   │
                   │  DynamoDB            │
                   └──────────────────────┘

SECONDARY CHECK: Scale and consistency
  < 10K writes/sec + < 1TB → Try PostgreSQL first (simpler, proven)
  > 10K writes/sec OR  > 1TB → Evaluate distributed options
  Need ACID across entities → Relational or distributed SQL
  Tolerate eventual consistency → NoSQL options open up
```

**Explanation:**
The decision tree starts with access patterns, not with scale or technology popularity.
For most applications, PostgreSQL handles the load for the first 2-3 years. The question
"should we use NoSQL?" should be answered by the query shape and consistency requirements,
not by aspirational scale targets. Add specialized databases only when PostgreSQL
demonstrably cannot serve the specific access pattern.

---

## Diagram 2: Polyglot Persistence Architecture — E-commerce System

```
E-COMMERCE POLYGLOT PERSISTENCE ARCHITECTURE
═══════════════════════════════════════════════════════════════════════════

                         USERS / CLIENTS
                               │
                    ┌──────────▼──────────┐
                    │    API Gateway       │
                    │  (Auth, Rate Limit)  │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
          ▼                    ▼                    ▼
   ┌─────────────┐    ┌─────────────┐     ┌─────────────┐
   │  User       │    │  Product    │     │  Order      │
   │  Service    │    │  Service    │     │  Service    │
   └──────┬──────┘    └──────┬──────┘     └──────┬──────┘
          │                  │                    │
          ▼                  │                    ▼
   ┌──────────────┐          │            ┌──────────────┐
   │ PostgreSQL   │          │            │ PostgreSQL   │
   │ (users,      │          │            │ (orders,     │
   │  profiles,   │          │            │  payments,   │
   │  auth)       │          │            │  ACID txns)  │
   └──────┬───────┘          │            └──────┬───────┘
          │                  ▼                   │
          │           ┌──────────────┐           │
          │           │ PostgreSQL   │           │
          │           │ (products,   │           │
          │           │  inventory,  │           │
          │           │  categories) │           │
          │           └──────┬───────┘           │
          │                  │                   │
          │         ┌────────┼────────┐          │
          │         │        │        │          │
          │         ▼        ▼        ▼          │
          │  ┌──────────┐ ┌──────┐ ┌────────┐   │
          │  │Elasticsearch│Redis│ │Cassandra│  │
          │  │(product  │ │(cart,│ │(activity│  │
          │  │ search,  │ │token,│ │ feed,   │  │
          │  │ facets)  │ │cache)│ │ events) │  │
          │  └──────────┘ └──────┘ └────────┘   │
          │                                      │
          └──────────────┬───────────────────────┘
                         │
                   ┌─────▼──────┐
                   │   Kafka    │ ← Event Bus (synchronizes all services)
                   │  (Events)  │
                   └─────┬──────┘
                         │
          ┌──────────────┴─────────────────────┐
          ▼                                    ▼
   ┌─────────────┐                    ┌────────────────┐
   │ Outbox      │                    │ Event Consumer │
   │ Processor   │                    │ (ES indexer,   │
   │ (per service│                    │  Cassandra     │
   │  DB outbox) │                    │  writer, etc.) │
   └─────────────┘                    └────────────────┘

CONSISTENCY MODEL:
PostgreSQL ← Synchronous writes (ACID, source of truth)
Elasticsearch ← Async update via events (eventual, search index)
Redis ← Async update via events (eventual, cache) + TTL expiry
Cassandra ← Async append via events (eventual, activity log)
```

**Explanation:**
Each service owns its database — no service accesses another service's database directly.
Cross-service data sharing happens only through APIs or events (Kafka). The outbox pattern
ensures that when a service commits a database write, the corresponding event is reliably
published to Kafka — the two operations are atomic within the service's database transaction.
This avoids the "write to DB but fail to publish event" inconsistency.

---

## Diagram 3: CAP/PACELC Database Positioning Map

```
CAP/PACELC POSITIONING OF MAJOR DATABASES
═══════════════════════════════════════════════════════════════════════════

CONSISTENCY AXIS (strong ←────────────────────► eventual)

                Strong                              Eventual
                  │                                    │
HIGH         ┌────┴──────────────────────────────────┐│
AVAILABILITY │ DynamoDB*  MongoDB   Cassandra(tunable)││
             │ (strong    (primary  ↕ tunable         ││
             │  reads)    reads)    │                  ││
             │                    ╔╪═══════════════╗  ││
             │                    ║Cassandra(default)  ││
             │                    ║Elasticsearch   ╚══╗│
             │                    ╚════════════════╗  ││
             │ CockroachDB        DynamoDB(eventual)║  ││
             │ PostgreSQL(primary)                  ╚══╝│
LOW          │ Spanner                                   │
AVAILABILITY │                                           │
             └───────────────────────────────────────────┘

PACELC MAP (During normal operation: Latency vs Consistency):
              Low Latency                    High Consistency
                  │                                │
EL           ┌────┴──────────────────────────────┐│EC
(favor        │Redis                              ││
latency)      │Cassandra(local reads)             ││
              │DynamoDB(eventual reads)           ││ DynamoDB(strong)
              │                                   ││ CockroachDB
              │MongoDB(secondaries)               ││ Spanner
              │                                   ││ PostgreSQL(sync rep.)
              │Elasticsearch                      ││
              └───────────────────────────────────┘│

DETAILED PACELC:
┌─────────────────┬──────────┬──────────┬───────────────────────────────────┐
│ Database        │ CAP (P)  │ PACELC   │ Notes                             │
├─────────────────┼──────────┼──────────┼───────────────────────────────────┤
│ PostgreSQL      │ CP       │ ELC      │ Single node = no partition;       │
│ (single)        │          │          │ Read replicas = EL trade-off      │
├─────────────────┼──────────┼──────────┼───────────────────────────────────┤
│ CockroachDB     │ CP       │ ELC      │ Raft consensus; stale reads opt-in│
├─────────────────┼──────────┼──────────┼───────────────────────────────────┤
│ Cassandra       │ AP       │ EL       │ Default eventual; QUORUM = EPC    │
│                 │          │          │ LOCAL_QUORUM = EPC per datacenter │
├─────────────────┼──────────┼──────────┼───────────────────────────────────┤
│ DynamoDB        │ AP/CP*   │ EL/EC*   │ *Per-request choice; txns = CP    │
├─────────────────┼──────────┼──────────┼───────────────────────────────────┤
│ MongoDB         │ CP*      │ ELC/EL*  │ *Default unsafe write = AP-ish;   │
│                 │          │          │ w:majority j:true = CP            │
├─────────────────┼──────────┼──────────┼───────────────────────────────────┤
│ Redis           │ CP*      │ EL       │ *Cluster = AP; Sentinel = CP;     │
│                 │          │          │ AOF = higher C, RDB = lower C     │
├─────────────────┼──────────┼──────────┼───────────────────────────────────┤
│ Elasticsearch   │ AP       │ EL       │ Near-real-time (1s visibility)    │
│                 │          │          │ Splits: each half available        │
└─────────────────┴──────────┴──────────┴───────────────────────────────────┘
```

**Explanation:**
The "it depends" asterisks on most databases reveal why CAP theorem is often overly
simplified. Most modern databases offer TUNABLE consistency — you choose your position
on the spectrum per operation. DynamoDB is AP by default (eventual reads) but CP on
demand (strong reads, transactions). Cassandra is AP by default but can be configured
to CP with QUORUM read/write consistency. The practical takeaway: design your application
to request the minimum consistency level it actually requires for each operation. Using
strong consistency for everything when only some operations need it wastes throughput and
increases latency unnecessarily.

---

## Diagram 4: Database Migration Strategies — Strangler Fig vs Dual-Write vs Shadow Read

```
THREE DATABASE MIGRATION STRATEGIES COMPARED
═══════════════════════════════════════════════════════════════════════════

STRATEGY 1: STRANGLER FIG (gradual routing)
Month 1-2: All traffic → Old DB
Month 3:   User service migrated; user reads/writes → New DB
           Everything else → Old DB
Month 4-6: Product, order, inventory services migrated one by one
Month 7:   Old DB decommissioned

Traffic routing via feature flag:
┌─────────┐          ┌─────────────┐
│ Request │──────────►Feature Flag │
└─────────┘          └──────┬──────┘
                            │ Flag: USE_NEW_USER_DB=true
                    ┌───────▼───────────────┐
                    │ New User DB (Postgres) │
                    └───────────────────────┘
                            │ Flag: USE_NEW_USER_DB=false
                    ┌───────▼───────────────┐
                    │ Old Monolith Postgres  │
                    └───────────────────────┘

STRATEGY 2: DUAL-WRITE (write to both, read from old)
┌─────────┐     ┌─────────────┐     ┌──────────────┐
│ Write   │────►│ Application │────►│ Old DB       │ ← Source of truth
│ Request │     │ Layer       │     └──────────────┘
└─────────┘     │             │
                │             │────►┌──────────────┐
                └─────────────┘     │ New DB       │ ← Being populated
                                    └──────────────┘
Risk: Write-1 succeeds, Write-2 fails → databases diverge
Fix: Outbox pattern to guarantee dual-write consistency

STRATEGY 3: SHADOW READ (write to new, compare reads)
┌─────────┐     ┌─────────────┐     ┌──────────────┐
│ Read    │────►│ Application │────►│ Old DB       │ ← Serves real response
│ Request │     │ Layer       │     └──────────────┘
└─────────┘     │             │
                │   ┌─────────┘
                │   │ Shadow read (async, does NOT affect user response)
                │   ▼
                │  ┌──────────────┐
                └──►New DB       │ ← Compared silently
                   └──────────────┘
                   Comparison result → metrics/logs
                   If identical rate > 99.9% → safe to switch reads

MIGRATION TIMELINE WITH ALL THREE STRATEGIES:
Week 1-4:   SHADOW READ only (validate new DB returns same results)
Week 5-6:   DUAL-WRITE (write to both; read from old; validate write consistency)
Week 7:     TRAFFIC SPLIT: 5% reads → New DB (canary)
Week 8:     Monitor: error rate? latency? consistency mismatches?
Week 9:     Ramp: 25% → 50% → 75% → 100% reads to New DB
Week 10:    Stop writes to Old DB (new DB is source of truth)
Week 11-12: Old DB in read-only mode (rollback safety net)
Week 13:    Decommission Old DB
```

**Explanation:**
The three strategies are complementary and typically used in sequence: shadow reading
first validates correctness without any user impact; dual-write ensures the new database
is fully populated; gradual traffic shifting (strangler fig) manages risk during cutover.
The most dangerous phase is stopping dual-write — once you stop syncing to the old
database, rollback requires re-syncing all writes that happened in the new system,
which may be days or weeks of data. Running the old DB in read-only mode for a safety
window (1-2 weeks) before full decommissioning is cheap insurance.

---

## Diagram 5: CQRS + Event Sourcing — Complete Data Flow

```
CQRS + EVENT SOURCING ARCHITECTURE
═══════════════════════════════════════════════════════════════════════════

COMMAND SIDE (Write Path):                READ SIDE (Query Path):
                                          ┌─────────────────────────────────┐
User → API → Command Handler              │ Multiple read model projections │
               │                          │                                 │
               │ Validate business rules  │ ┌──────────────────────────┐   │
               │ (current state from      │ │ "Order Summary" (Redis)  │   │
               │  snapshot + events)      │ │ Hot data, fast access    │   │
               ▼                          │ └──────────────────────────┘   │
┌──────────────────────────────────────┐  │                                 │
│         EVENT STORE                   │  │ ┌──────────────────────────┐   │
│  (Append-only, Immutable)            │  │ │ "Order History" (PostgreSQL)│ │
│                                      │  │ │ Normalized, query-friendly │  │
│  stream: order-456                   │  │ └──────────────────────────┘   │
│  v1: OrderCreated   T+0  {items,...} │  │                                 │
│  v2: PaymentReceived T+5m {amount}   │  │ ┌──────────────────────────┐   │
│  v3: ItemShipped    T+2d {SKU-1}     │  │ │ "Analytics" (ClickHouse) │   │
│  v4: ItemDelivered  T+5d {SKU-1}     │──┤►│ Aggregated, OLAP queries │   │
│  v5: ItemReturned   T+10d{SKU-2}    │  │ └──────────────────────────┘   │
│  v6: RefundIssued   T+12d{$75}      │  └─────────────────────────────────┘
└──────────────────────────────────────┘
               │
               │ Events published to message bus
               ▼
         ┌──────────┐
         │  Kafka   │
         └────┬─────┘
              │
    ┌─────────┴────────┐
    ▼                   ▼
┌─────────┐        ┌─────────┐
│Projection│        │Projection│
│Handler  │        │Handler  │
│(Redis)  │        │(ClickHouse)
└────┬────┘        └────┬────┘
     │                  │
     ▼                  ▼
   Redis           ClickHouse

SNAPSHOT STRATEGY (prevents O(n) event replay):
                                   Rebuild cost:
Events: 1-999                   ──►  All 999 events replayed (slow for large n)
Snapshot at v1000:              ──►  Store current state snapshot
Events: 1001-1999               ──►  Load snapshot + replay 999 events
Snapshot at v2000               ──►  Store new snapshot

Without snapshot: rebuild = O(total_events)   = hours for millions of events
With snapshot:    rebuild = O(events_since_snapshot) = seconds for recent events
```

**Explanation:**
CQRS with event sourcing creates completely separate optimized storage for reads and writes.
The event store is append-only and provides perfect auditability — every state change is
recorded immutably. The read models are pre-computed, denormalized views optimized for
specific query patterns. The key operational challenge is managing projection lag: if the
event consumer falls behind, read models show stale data. For critical read-after-write
scenarios (user expects to see their just-placed order immediately), the application must
either wait for projection update confirmation or read directly from the event store.
