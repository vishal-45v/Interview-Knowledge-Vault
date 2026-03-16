# System Design — Diagram Explanations

---

## Diagram 1: Three-Tier Architecture

```
  ┌─────────────────────────────────────────────────────────────┐
  │  Tier 1: Presentation                                       │
  │  Browser / Mobile App / CLI                                 │
  └──────────────────────────┬──────────────────────────────────┘
                              │ HTTP/HTTPS
  ┌──────────────────────────▼──────────────────────────────────┐
  │  Tier 2: Application                                        │
  │  Load Balancer → Spring Boot Services                       │
  │  (stateless, horizontally scalable)                         │
  └──────────────────────────┬──────────────────────────────────┘
                              │ JDBC/Redis/Kafka
  ┌──────────────────────────▼──────────────────────────────────┐
  │  Tier 3: Data                                               │
  │  PostgreSQL (primary/replica) + Redis + S3                  │
  └─────────────────────────────────────────────────────────────┘
```

---

## Diagram 2: Microservices Architecture

```
  Client → API Gateway → ┌── Order Service     → Order DB
                         ├── User Service       → User DB
                         ├── Product Service    → Product DB
                         ├── Payment Service    → Payment DB
                         └── Notification Svc   → Message Queue

  Inter-service communication:
  Sync:  HTTP/gRPC (request-response)
  Async: Kafka events (fire-and-forget, eventual consistency)

  Cross-cutting concerns (API Gateway):
  - Authentication/JWT validation
  - Rate limiting
  - Request logging
  - SSL termination
```

---

## Diagram 3: CQRS Pattern

```
  Write side:                    Read side:
  ┌──────────────────┐          ┌──────────────────────────┐
  │  Command          │          │  Query Model              │
  │  (normalized DB)  │          │  (denormalized, optimized)│
  │                   │          │                           │
  │  POST /orders     │          │  GET /orders/summary      │
  │  PUT /orders/{id} │          │  GET /dashboard/stats     │
  └────────┬──────────┘          └──────────┬────────────────┘
           │                                │
           │ Events published               │ Queries
           ▼                                ▼
  ┌────────────────────────────────────────────────────────┐
  │                  Event Bus (Kafka)                      │
  │  OrderCreated, PaymentProcessed, OrderShipped events   │
  └────────────────────────────────────────────────────────┘
           │
           ▼ Consumers build read models
  ┌────────────────────────────────────────────────────────┐
  │  Read Model Store (PostgreSQL view / Elasticsearch)    │
  │  Pre-computed, denormalized for specific queries       │
  └────────────────────────────────────────────────────────┘
```
