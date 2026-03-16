# Database Scaling Patterns — Diagram Library

---

## 1. Primary-Replica Replication

**Purpose:** Improve read throughput and provide high availability by replicating data from a primary to one or more replicas.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    PRIMARY-REPLICA REPLICATION                           │
│                                                                          │
│     Application                                                          │
│         │                                                                │
│    ┌────┴────────────────────┐                                           │
│    │     Read/Write Router   │                                           │
│    │   (Application / proxy) │                                           │
│    └────┬───────────────┬────┘                                           │
│         │               │                                                │
│    WRITES only      READS only                                           │
│         │               │                                                │
│         ▼               ▼                                                │
│  ┌──────────────┐  ┌──────────────┐   ┌──────────────┐                 │
│  │   PRIMARY    │  │   REPLICA 1  │   │   REPLICA 2  │                 │
│  │              │  │              │   │              │                 │
│  │  Read/Write  │  │  Read Only   │   │  Read Only   │                 │
│  │  All INSERT  │  │              │   │              │                 │
│  │  UPDATE      │  │  Slightly    │   │  Slightly    │                 │
│  │  DELETE      │  │  behind      │   │  behind      │                 │
│  │              │  │  primary     │   │  primary     │                 │
│  └──────┬───────┘  └──────▲───────┘   └──────▲───────┘                 │
│         │                 │                   │                         │
│         │    Replication  │  Replication      │                         │
│         └─────────────────┴───────────────────┘                         │
│              (binary log / WAL streaming)                                │
│                                                                          │
│  Replication lag: Replicas may be 100ms–seconds behind primary          │
│  Failover: If primary dies, promote replica to primary                  │
└──────────────────────────────────────────────────────────────────────────┘
```

**Key considerations:**
- **Replication lag:** Reads from replica may return stale data
- **Read-your-writes consistency:** After a write, route the next read to primary
- **Automatic failover:** Tools like Patroni (Postgres) or MHA (MySQL) automate promotion

---

## 2. Database Sharding

**Purpose:** Horizontally partition data across multiple database instances to handle data volumes beyond a single machine.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         DATABASE SHARDING                                │
│                                                                          │
│  Application                                                             │
│       │                                                                  │
│       ▼                                                                  │
│  ┌─────────────────────────────────────────────────────────┐            │
│  │                    SHARD ROUTER                          │            │
│  │   Determines which shard based on shard key              │            │
│  │                                                         │            │
│  │   user_id % 3 == 0 → Shard A                           │            │
│  │   user_id % 3 == 1 → Shard B                           │            │
│  │   user_id % 3 == 2 → Shard C                           │            │
│  └──────────────────┬────────────────┬──────────────────┬─┘            │
│                     │                │                  │               │
│                     ▼                ▼                  ▼               │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐ │
│  │     SHARD A        │  │     SHARD B        │  │     SHARD C        │ │
│  │  user_ids 0,3,6,9  │  │  user_ids 1,4,7,10 │  │  user_ids 2,5,8,11 │ │
│  │                    │  │                    │  │                    │ │
│  │  PostgreSQL        │  │  PostgreSQL        │  │  PostgreSQL        │ │
│  │  instance A        │  │  instance B        │  │  instance C        │ │
│  └────────────────────┘  └────────────────────┘  └────────────────────┘ │
│                                                                          │
│  Shard key choices:                                                      │
│  • user_id (range or hash)                                               │
│  • tenant_id (for multi-tenant SaaS)                                     │
│  • geographic region                                                     │
│                                                                          │
│  Problems:                                                               │
│  • Cross-shard queries are expensive                                     │
│  • Hotspots if shard key not evenly distributed                          │
│  • Resharding is painful — consistent hashing helps                      │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Read Replicas with Caching Layer

**Purpose:** Combine read replicas with a cache tier to serve most reads from memory.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                  READ REPLICAS + CACHING LAYER                           │
│                                                                          │
│                      ┌────────────┐                                      │
│                      │Application │                                      │
│                      └─────┬──────┘                                      │
│                            │                                             │
│                  ┌─────────┴──────────┐                                  │
│                  │                    │                                  │
│             READ │                    │ WRITE                            │
│                  ▼                    ▼                                  │
│        ┌──────────────────┐  ┌──────────────────┐                       │
│        │   CACHE LAYER    │  │     PRIMARY DB   │                       │
│        │                  │  │                  │                       │
│        │      Redis       │  │   All writes     │                       │
│        │                  │  │   go here        │                       │
│        │  Cache miss  ──┐ │  └────────┬─────────┘                       │
│        │  Cache hit  ◄──┘ │           │                                  │
│        └──────────────────┘           │ replication                     │
│                 │                     │                                  │
│           (cache miss)                ▼                                  │
│                 │         ┌─────────────────────────────────┐           │
│                 │         │         READ REPLICAS           │           │
│                 │         │   ┌──────────┐  ┌──────────┐   │           │
│                 └────────►│   │ Replica 1│  │ Replica 2│   │           │
│                           │   └──────────┘  └──────────┘   │           │
│                           └─────────────────────────────────┘           │
│                                                                          │
│  Cache-aside pattern:                                                    │
│  1. Check cache → cache hit? → return cached data                       │
│  2. Cache miss → query replica                                           │
│  3. Store result in cache with TTL                                       │
│  4. Return data                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Caching Layer Architecture

**Purpose:** Reduce database load and latency by serving frequently accessed data from fast in-memory stores.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        CACHING ARCHITECTURE                              │
│                                                                          │
│  L1: In-Process Cache (Caffeine)                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  JVM Heap Memory                                                  │   │
│  │  Size: ~100MB    Hit Rate: ~70%    Latency: <1ms                  │   │
│  │  Eviction: LRU / Size-based                                       │   │
│  └────────────────────────────┬─────────────────────────────────────┘   │
│                               │ Cache miss                              │
│                               ▼                                         │
│  L2: Distributed Cache (Redis Cluster)                                   │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐                  │   │
│  │  │  Master 1  │  │  Master 2  │  │  Master 3  │                  │   │
│  │  │ Slots 0-   │  │Slots 5461- │  │Slots 10922-│                  │   │
│  │  │  5460      │  │  10921     │  │  16383     │                  │   │
│  │  └────┬───────┘  └────┬───────┘  └────┬───────┘                  │   │
│  │       │               │               │                           │   │
│  │  ┌────▼───────┐  ┌────▼───────┐  ┌────▼───────┐                  │   │
│  │  │  Replica 1 │  │  Replica 2 │  │  Replica 3 │                  │   │
│  │  └────────────┘  └────────────┘  └────────────┘                  │   │
│  │  Size: ~10GB    Hit Rate: ~25%    Latency: ~1-5ms                 │   │
│  └────────────────────────────┬─────────────────────────────────────┘   │
│                               │ Cache miss                              │
│                               ▼                                         │
│  L3: Database                                                            │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  PostgreSQL / MySQL                                               │   │
│  │  Latency: ~10-100ms                                               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Connection Pooling

**Purpose:** Reuse database connections to avoid the overhead of creating a new connection for each request.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       CONNECTION POOLING (HikariCP)                      │
│                                                                          │
│  Application Threads                  Connection Pool                   │
│                                                                          │
│  ┌──────────────┐                    ┌────────────────────────────────┐ │
│  │  Thread 1    │──acquire──────────►│  ┌──────┐ ┌──────┐ ┌──────┐   │ │
│  └──────────────┘                   │  │ Con 1│ │ Con 2│ │ Con 3│   │ │
│                                      │  │(idle)│ │(used)│ │(used)│   │ │
│  ┌──────────────┐                    │  └──────┘ └──────┘ └──────┘   │ │
│  │  Thread 2    │──waiting──────────►│  ┌──────┐ ┌──────┐            │ │
│  └──────────────┘   (pool full)      │  │ Con 4│ │ Con 5│            │ │
│                                      │  │(used)│ │(idle)│            │ │
│  ┌──────────────┐                    │  └──────┘ └──────┘            │ │
│  │  Thread 3    │──acquire──────────►│                                │ │
│  └──────────────┘                   │  Max pool size: 10              │ │
│                                      │  Min idle: 5                   │ │
│                                      │  Connection timeout: 30s        │ │
│                                      │  Idle timeout: 10 min           │ │
│                                      │  Max lifetime: 30 min           │ │
│                                      └────────────────┬───────────────┘ │
│                                                        │                │
│                                      ┌─────────────────▼───────────────┐ │
│                                      │          DATABASE                │ │
│                                      │  TCP connections: 5-10          │ │
│                                      │  (not one per thread!)          │ │
│                                      └─────────────────────────────────┘ │
│                                                                          │
│  HikariCP tuning:                                                        │
│  spring.datasource.hikari.maximum-pool-size=10                          │
│  spring.datasource.hikari.minimum-idle=5                                │
│  spring.datasource.hikari.connection-timeout=30000                      │
│  spring.datasource.hikari.idle-timeout=600000                           │
└──────────────────────────────────────────────────────────────────────────┘
```

**Connection pool sizing formula (rough guide):**
- `pool size = (core count * 2) + effective_spindle_count`
- For a 4-core machine with SSDs: `4 * 2 + 1 = 9 ≈ 10 connections`
- Larger pool ≠ better performance. Too many connections causes context-switching overhead.
