# System Design Patterns — Diagram Library

---

## 1. Load Balancer

**Purpose:** Distributes incoming traffic across multiple backend servers to ensure no single server is overwhelmed.

```
                         ┌─────────────────────────────────────────────┐
                         │              LOAD BALANCER                  │
      ┌──────┐           │                                             │
      │Client│──────────►│  Algorithm: Round Robin / Least Conn / IP  │
      └──────┘           │  Health checks every 30s                   │
                         └──────────────┬──────────────────────────────┘
                                        │
               ┌────────────────────────┼────────────────────────┐
               │                        │                        │
               ▼                        ▼                        ▼
    ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
    │  App Server #1   │    │  App Server #2   │    │  App Server #3   │
    │  192.168.1.10    │    │  192.168.1.11    │    │  192.168.1.12    │
    │  Port: 8080      │    │  Port: 8080      │    │  Port: 8080      │
    │  [Healthy ✓]     │    │  [Healthy ✓]     │    │  [Healthy ✓]     │
    └──────────────────┘    └──────────────────┘    └──────────────────┘
               │                        │                        │
               └────────────────────────┴────────────────────────┘
                                        │
                                        ▼
                            ┌─────────────────────┐
                            │     Database         │
                            │   Primary Instance   │
                            └─────────────────────┘
```

**Key components:**
1. **Health checks:** Load balancer polls each backend at a configured interval. Unhealthy backends are removed from rotation.
2. **Session affinity (sticky sessions):** Routes the same client to the same backend — useful when session state is local.
3. **Algorithms:** Round-robin (equal distribution), Least Connections (routes to least busy server), IP Hash (consistent routing per client).

**How to explain in an interview:**
> "A load balancer sits between clients and backend servers, acting as a traffic cop. It continuously health-checks backends and removes unhealthy ones from the pool. The choice of algorithm matters — round-robin works for stateless services, while IP hash or sticky sessions are needed when session state lives on the server."

---

## 2. Microservices Architecture

**Purpose:** Decompose a large application into small, independently deployable services organized around business capabilities.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENT LAYER                                │
│  ┌───────────┐    ┌───────────┐    ┌───────────┐                   │
│  │  Browser  │    │Mobile App │    │ 3rd Party │                   │
│  └─────┬─────┘    └─────┬─────┘    └─────┬─────┘                   │
└────────┼────────────────┼────────────────┼────────────────────────┘
         │                │                │
         └────────────────┴────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY                                │
│  • Authentication / JWT validation                                  │
│  • Rate limiting                                                    │
│  • Request routing                                                  │
│  • SSL termination                                                  │
│  • Response aggregation                                             │
└──────────┬──────────────┬──────────────┬───────────────────────────┘
           │              │              │
           ▼              ▼              ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ User Service │  │Order Service │  │Product Service│
│   :8081      │  │   :8082      │  │    :8083      │
│              │  │              │  │               │
│ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐  │
│ │  Users   │ │  │ │  Orders  │ │  │ │ Products │  │
│ │    DB    │ │  │ │    DB    │ │  │ │    DB    │  │
│ └──────────┘ │  │ └──────────┘ │  │ └──────────┘  │
└──────────────┘  └──────────────┘  └──────────────┘
       │                  │                  │
       └──────────────────┴──────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │   Message Broker    │
              │  (Kafka / RabbitMQ) │
              │                     │
              │  order.created ──►  │
              │  user.updated  ──►  │
              └─────────────────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │ Notification Service│
              │    Email / SMS      │
              └─────────────────────┘
```

**Key components:**
1. **API Gateway:** Single entry point. Handles cross-cutting concerns.
2. **Services:** Each owns its data store — no shared databases.
3. **Message broker:** Async communication decouples services.
4. **Service discovery:** Services register themselves and discover peers (Eureka, Consul).

---

## 3. API Gateway

**Purpose:** Single entry point that handles authentication, routing, rate limiting, and protocol translation.

```
                    ┌────────────────────────────────────────────────┐
                    │                  API GATEWAY                   │
                    │                                                │
                    │  Step 1: SSL Termination (HTTPS → HTTP)        │
                    │       ↓                                        │
                    │  Step 2: Authentication (Validate JWT)         │
                    │       ↓                                        │
                    │  Step 3: Rate Limiting (100 req/min/user)      │
                    │       ↓                                        │
                    │  Step 4: Request Routing (path → service)      │
                    │       ↓                                        │
                    │  Step 5: Load Balancing (to service replicas)  │
                    │       ↓                                        │
                    │  Step 6: Response Caching (optional)           │
                    │       ↓                                        │
                    │  Step 7: Response Transformation               │
                    │                                                │
                    └─────────────────┬──────────────────────────────┘
                                      │
             ┌────────────────────────┼────────────────────────────┐
             │                        │                            │
             ▼                        ▼                            ▼
   ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐
   │   /api/users/**  │    │  /api/orders/**  │    │/api/products/**  │
   │  → User Service  │    │ → Order Service  │    │→ Product Service │
   └──────────────────┘    └──────────────────┘    └──────────────────┘
```

**Rate limiting strategies:**
- **Token bucket:** Tokens accumulate up to a max; each request consumes one
- **Sliding window:** Count requests in a rolling time window
- **Fixed window:** Count requests per fixed time interval

---

## 4. Service Mesh

**Purpose:** Handles all service-to-service communication concerns (retries, mTLS, circuit breaking, observability) transparently via sidecar proxies.

```
┌─────────────────────────────────────────────────────────────────┐
│                      SERVICE MESH (Istio)                       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                     CONTROL PLANE                       │   │
│  │   ┌───────────┐  ┌───────────┐  ┌───────────────────┐  │   │
│  │   │   Pilot   │  │  Mixer   │  │     Citadel        │  │   │
│  │   │ (routing) │  │(telemetry│  │  (certificates/    │  │   │
│  │   │           │  │ /policy) │  │     mTLS)          │  │   │
│  │   └─────┬─────┘  └─────┬────┘  └────────┬──────────┘  │   │
│  └─────────┼──────────────┼────────────────┼─────────────┘   │
│            │              │                │                   │
│     ┌──────┴──────────────┴────────────────┴──────┐           │
│     │              DATA PLANE                      │           │
│     │                                              │           │
│     │  ┌─────────────────────────────────────────┐│           │
│     │  │  Pod A                                  ││           │
│     │  │  ┌──────────────┐  ┌──────────────────┐ ││           │
│     │  │  │  App Container│  │ Envoy Sidecar    │ ││           │
│     │  │  │  (port 8080)  │  │ (port 15001)     │ ││           │
│     │  │  └──────────────┘  └────────┬─────────┘ ││           │
│     │  └────────────────────────────-┼────────────┘│           │
│     │                                │              │           │
│     │  ┌─────────────────────────────┼─────────────┐│           │
│     │  │  Pod B                      │             ││           │
│     │  │  ┌──────────────┐  ┌────────▼─────────┐   ││           │
│     │  │  │  App Container│  │ Envoy Sidecar    │   ││           │
│     │  │  │  (port 8080)  │  │ mTLS encrypted   │   ││           │
│     │  │  └──────────────┘  └──────────────────┘   ││           │
│     │  └────────────────────────────────────────────┘│           │
│     └──────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

**Benefits of Service Mesh:**
1. **mTLS everywhere:** Automatic mutual TLS between all services
2. **Observability:** Distributed tracing, metrics, logs without code changes
3. **Traffic management:** Canary deployments, A/B testing, retries, timeouts
4. **Circuit breaking:** Automatic failure isolation

---

## 5. Circuit Breaker

**Purpose:** Prevents cascading failures by stopping calls to a failing service and allowing it to recover.

```
                    ┌──────────────────────────────────────────────┐
                    │              CIRCUIT BREAKER STATES           │
                    │                                              │
                    │         ┌──────────────────┐                │
                    │         │      CLOSED       │                │
                    │   ┌────►│  (Normal state)   │◄────┐         │
                    │   │     │  All calls pass   │     │         │
                    │   │     └────────┬──────────┘     │         │
                    │   │              │                 │         │
                    │   │   Failure threshold exceeded   │         │
                    │   │   (e.g., 50% fail in 10s)     │         │
                    │   │              │                 │         │
                    │   │              ▼                 │         │
                    │   │     ┌──────────────────┐       │         │
                    │   │     │       OPEN        │       │         │
                    │   │     │  (Tripped state)  │       │         │
                    │   │     │  Calls fail fast  │       │         │
                    │   │     └────────┬──────────┘       │         │
                    │   │              │                   │         │
                    │   │    After timeout (e.g., 30s)     │         │
                    │   │              │                   │         │
                    │   │              ▼                   │         │
                    │   │     ┌──────────────────┐         │         │
                    │   │     │   HALF-OPEN       │         │         │
                    │   │     │  (Testing state)  │         │         │
                    │   │     │  Limited calls    │         │         │
                    │   │     └────────┬──────────┘         │         │
                    │   │              │                     │         │
                    │   │    Success ──┼─────────────────────┘         │
                    │   │              │                               │
                    │   └── Failure ───┘ (re-opens)                    │
                    └──────────────────────────────────────────────────┘

     ┌────────────┐    request    ┌─────────────────┐    forward    ┌─────────────┐
     │   Caller   │──────────────►│ Circuit Breaker │──────────────►│  Service B  │
     │  Service A │               │   (Resilience4j │               │  (may fail) │
     └────────────┘               │   / Hystrix)    │               └─────────────┘
                                  └─────────────────┘
                                         │
                                    (if OPEN)
                                         │
                                         ▼
                                  ┌─────────────┐
                                  │  Fallback   │
                                  │  Response   │
                                  │  (cached or │
                                  │   default)  │
                                  └─────────────┘
```

**Resilience4j configuration example:**

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentServiceClient.pay(request);
}

public PaymentResponse paymentFallback(PaymentRequest request, Exception ex) {
    return PaymentResponse.queued("Payment queued for retry");
}
```

**Key metrics to monitor:**
- Failure rate percentage
- Slow call rate percentage
- Number of buffered calls
- State transitions (CLOSED → OPEN → HALF-OPEN)
