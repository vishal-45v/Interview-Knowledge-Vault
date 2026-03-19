# API Latency Debugging

---

## Systematic Latency Investigation

```
Request comes in → high latency observed

Decompose the time:
  Total = Network in + Processing + DB + Network out

  Processing = Authentication + Business logic + Serialization
  DB = Connection wait + Query execution + Result transfer

Measure each component:
  - Distributed tracing (Jaeger, Zipkin, AWS X-Ray)
  - Database query logs with execution time
  - Spring Boot Actuator /actuator/metrics/http.server.requests
```

---

## Tools for Latency Analysis

```bash
# Application metrics
curl http://localhost:8080/actuator/metrics/http.server.requests

# Database slow query log (PostgreSQL)
# In postgresql.conf:
log_min_duration_statement = 500  # Log queries taking > 500ms
log_statement = 'none'

# MySQL slow query log:
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.5;

# Spring Boot query logging
logging.level.org.hibernate.SQL=DEBUG
spring.jpa.properties.hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS=100
```

---

## Common Latency Culprits

### 1. Missing Database Index
```sql
-- Before index: EXPLAIN shows Seq Scan, 5+ seconds
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;
-- Seq Scan on orders (rows=5000000) → 5.2s

-- After index:
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
-- Index Scan → 0.003s
```

### 2. N+1 Query Problem
```
GET /orders → 1 query for 100 orders + 100 queries for customers = 101 queries
Expected: 1 query with JOIN
Actual: 5 seconds

Fix: JOIN FETCH or @EntityGraph
After fix: 50ms
```

### 3. Slow External Service Call
```java
// Identify with distributed tracing
// Span "call-payment-service" shows 3000ms

// Fix: Timeout + Circuit Breaker
RestTemplate restTemplate = new RestTemplate();
restTemplate.setRequestFactory(new HttpComponentsClientHttpRequestFactory());
((HttpComponentsClientHttpRequestFactory)restTemplate.getRequestFactory())
    .setConnectTimeout(2000);
restTemplate.getRequestFactory().setReadTimeout(3000);
```

### 4. Large Response Payload
```java
// GET /users returns 10MB JSON with all user fields
// Fix: Pagination + field selection

@GetMapping("/users")
public ResponseEntity<Page<UserSummary>> getUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "20") int size) {
    return ResponseEntity.ok(userService.getSummaries(PageRequest.of(page, size)));
}
// UserSummary: only id, name, email (not full profile)
```

---

## P99 vs P50 Latency

```
P50 (median): 50% of requests are faster than this value
P95: 95% of requests are faster than this
P99: 99% of requests are faster than this
P99.9: 99.9% of requests are faster

A high P99 with low P50 indicates:
  - A small % of requests hit a slow path
  - Cache misses on occasional requests
  - DB queries hitting an occasional slow replica
  - GC pauses affecting a small % of requests

Monitor: Alert on P95 and P99, not just average
Average can hide terrible performance for some users
```

---

## Response Time Budget

```
Target: 200ms API response time budget

Components:
  Network in/out:      10ms
  Authentication:      5ms  (JWT validation — fast)
  Cache lookup:        5ms  (Redis — fast)
  DB query:            50ms (indexed query — target)
  Business logic:      30ms
  Serialization:       10ms
  Network overhead:    10ms
  Buffer:              80ms
  ─────────────────────────
  Total:               200ms

If any component exceeds its budget → overall SLA missed
```
