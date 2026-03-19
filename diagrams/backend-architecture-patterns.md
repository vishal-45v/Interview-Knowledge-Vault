# Backend Architecture Patterns — Diagram Library

---

## 1. Monolith vs Microservices

```
MONOLITHIC ARCHITECTURE:                    MICROSERVICES ARCHITECTURE:

┌─────────────────────────────────┐        ┌──────────┐  ┌──────────┐  ┌──────────┐
│         MONOLITH                │        │  User    │  │  Order   │  │ Product  │
│                                 │        │ Service  │  │ Service  │  │ Service  │
│  ┌──────────┐  ┌──────────────┐ │        └────┬─────┘  └────┬─────┘  └────┬─────┘
│  │   Web    │  │   Business   │ │             │              │              │
│  │  Layer   │  │    Logic     │ │        ┌────▼─────┐  ┌────▼─────┐  ┌────▼─────┐
│  └──────────┘  └──────────────┘ │        │  Users   │  │  Orders  │  │ Products │
│  ┌──────────┐  ┌──────────────┐ │        │    DB    │  │    DB    │  │    DB    │
│  │   Data   │  │  Auth Logic  │ │        └──────────┘  └──────────┘  └──────────┘
│  │  Access  │  │              │ │
│  └──────────┘  └──────────────┘ │        ┌──────────────────────────────────────┐
│  ┌──────────┐  ┌──────────────┐ │        │              API GATEWAY             │
│  │  Order   │  │  Reporting   │ │        └──────────────────────────────────────┘
│  │  Module  │  │   Module     │ │
│  └──────────┘  └──────────────┘ │        Pros: Independent scaling, fault isolation
│                                 │        Cons: Network latency, operational complexity
│         ONE DEPLOY              │
│      ONE SINGLE DB              │        Monolith Pros: Simple deploy, no network hops
└─────────────────────────────────┘        Monolith Cons: Scaling entire app for one hot path
```

---

## 2. Layered (N-Tier) Architecture

**Purpose:** Separate concerns by organizing code into horizontal layers, each with a specific responsibility.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    LAYERED ARCHITECTURE                             │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    PRESENTATION LAYER                        │   │
│  │  @RestController, @Controller                               │   │
│  │  • Handles HTTP requests/responses                          │   │
│  │  • Input validation (@Valid, @NotNull)                      │   │
│  │  • DTO serialization (JSON ↔ Java objects)                  │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │ calls (DTOs flow downward)           │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     SERVICE LAYER                            │   │
│  │  @Service, @Transactional                                   │   │
│  │  • Business logic lives here                                │   │
│  │  • Orchestrates repository calls                            │   │
│  │  • Transaction boundaries defined here                      │   │
│  │  • Converts between domain objects and DTOs                 │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │ calls (domain objects)               │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   REPOSITORY LAYER                           │   │
│  │  @Repository, JpaRepository, CrudRepository                 │   │
│  │  • Data access abstraction                                  │   │
│  │  • JPQL / native SQL queries                                │   │
│  │  • Hides persistence technology from service layer          │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
│                             │ SQL / ORM operations                 │
│                             ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     DATA LAYER                               │   │
│  │  PostgreSQL / MySQL / MongoDB / Redis                       │   │
│  │  • Stores persistent data                                   │   │
│  │  • Indexes, constraints, transactions                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘

  Rule: Dependencies flow downward only.
  Higher layers depend on lower layers — never the reverse.
```

---

## 3. Hexagonal Architecture (Ports and Adapters)

**Purpose:** Isolate the core domain from external systems using ports (interfaces) and adapters (implementations).

```
┌──────────────────────────────────────────────────────────────────────────┐
│                       HEXAGONAL ARCHITECTURE                             │
│                                                                          │
│   External World                Core Domain             External World   │
│                                                                          │
│  ┌────────────┐       ┌────────────────────────────┐   ┌────────────┐   │
│  │  REST API  │──────►│     «Primary Port»          │   │  Database  │   │
│  │  (Driving  │       │  OrderService interface     │   │  (Driven   │   │
│  │   Adapter) │       │                             │   │  Adapter)  │   │
│  └────────────┘       │  ┌──────────────────────┐  │   └─────▲──────┘   │
│                       │  │   DOMAIN CORE         │  │         │          │
│  ┌────────────┐       │  │                       │  │         │          │
│  │  GraphQL   │──────►│  │  Order entity         │  │  ┌──────┴──────┐   │
│  │  (Driving  │       │  │  OrderService         │  │  │«Secondary   │   │
│  │   Adapter) │       │  │  PlaceOrderUseCase    │  │──│  Port»      │   │
│  └────────────┘       │  │                       │  │  │OrderRepo    │   │
│                       │  │  No framework deps!   │  │  │interface    │   │
│  ┌────────────┐       │  │  Pure Java / domain   │  │  └─────────────┘   │
│  │  CLI / Job │──────►│  │  logic only           │  │                    │
│  │  (Driving) │       │  └──────────────────────┘  │  ┌────────────┐   │
│  └────────────┘       │                             │  │  Kafka     │   │
│                       │     «Primary Port»          │  │  (Driven   │   │
│                       │  EventPublisher interface   │──│  Adapter)  │   │
│                       └────────────────────────────┘  └────────────┘   │
│                                                                          │
│  KEY RULE: The domain core has ZERO knowledge of how it is called       │
│            or what external systems it talks to.                        │
└──────────────────────────────────────────────────────────────────────────┘
```

**Benefits:**
1. **Testability:** Core domain tested with mock adapters — no database needed
2. **Replaceability:** Swap PostgreSQL for MongoDB by writing a new adapter
3. **Framework independence:** Domain logic compiles without Spring on the classpath

---

## 4. Clean Architecture

**Purpose:** Enforce strict dependency rules so the most important business rules have zero dependencies on infrastructure.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         CLEAN ARCHITECTURE                               │
│                                                                          │
│     ╔═══════════════════════════════════════════════════════════════╗    │
│     ║                    FRAMEWORKS & DRIVERS                       ║    │
│     ║   Spring Boot, JPA, Jackson, Redis, REST controllers          ║    │
│     ║                                                               ║    │
│     ║   ┌───────────────────────────────────────────────────────┐  ║    │
│     ║   │                INTERFACE ADAPTERS                     │  ║    │
│     ║   │  Controllers, Presenters, Gateways                    │  ║    │
│     ║   │  Converts data between use cases and external format  │  ║    │
│     ║   │                                                       │  ║    │
│     ║   │  ┌───────────────────────────────────────────────┐   │  ║    │
│     ║   │  │              APPLICATION LAYER                │   │  ║    │
│     ║   │  │  Use Cases (Interactors)                      │   │  ║    │
│     ║   │  │  • PlaceOrderUseCase                          │   │  ║    │
│     ║   │  │  • ProcessPaymentUseCase                      │   │  ║    │
│     ║   │  │  • CancelOrderUseCase                         │   │  ║    │
│     ║   │  │  Contains application-specific business rules │   │  ║    │
│     ║   │  │                                               │   │  ║    │
│     ║   │  │  ┌─────────────────────────────────────┐     │   │  ║    │
│     ║   │  │  │          DOMAIN LAYER               │     │   │  ║    │
│     ║   │  │  │  Entities: Order, Customer, Product  │     │   │  ║    │
│     ║   │  │  │  Value Objects: Money, Address       │     │   │  ║    │
│     ║   │  │  │  Domain Services: PricingService     │     │   │  ║    │
│     ║   │  │  │  Domain Events: OrderPlacedEvent     │     │   │  ║    │
│     ║   │  │  │                                      │     │   │  ║    │
│     ║   │  │  │  NO external dependencies.           │     │   │  ║    │
│     ║   │  │  │  This is the stable core.            │     │   │  ║    │
│     ║   │  │  └─────────────────────────────────────┘     │   │  ║    │
│     ║   │  └───────────────────────────────────────────────┘   │  ║    │
│     ║   └───────────────────────────────────────────────────────┘  ║    │
│     ╚═══════════════════════════════════════════════════════════════╝    │
│                                                                          │
│   DEPENDENCY RULE: Arrows point inward only.                            │
│   Inner layers know NOTHING about outer layers.                         │
└──────────────────────────────────────────────────────────────────────────┘

Package structure in Java:
  com.example.app/
  ├── domain/           ← no Spring, no JPA annotations
  │   ├── model/
  │   ├── service/
  │   └── repository/  ← interfaces only
  ├── application/      ← use cases, orchestration
  │   └── usecase/
  ├── infrastructure/   ← JPA repositories, REST controllers
  │   ├── persistence/
  │   └── web/
  └── config/           ← Spring configuration
```
