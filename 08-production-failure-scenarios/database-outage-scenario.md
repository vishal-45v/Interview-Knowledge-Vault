# Production Failure: Database Outage

> Post-mortem style walkthrough of a database outage and response.

---

## Scenario: Primary Database Fails During Peak Traffic

**Timeline:**
```
09:00 AM — Traffic at normal peak levels (10K RPM)
09:15 AM — Primary PostgreSQL instance fails (hardware failure)
09:15 AM — PagerDuty alert: "Database connection pool exhausted"
09:16 AM — Alert: "Error rate > 50% on /api/orders"
09:17 AM — On-call engineer receives page
09:25 AM — Root cause identified: DB primary down
09:30 AM — Automatic failover to read replica completes
09:35 AM — Write traffic restored to new primary
09:45 AM — Error rate back to normal
Total downtime: 30 minutes
```

---

## What Happened

```
09:15: Primary DB hardware failed
         │
         ▼
  Spring Boot HikariCP:
  - Connections to primary start failing
  - Pool waits for timeout (30s) per connection attempt
  - 10 pool connections × 30s timeout = pool exhausted for 5 minutes
  - New requests: "Unable to acquire JDBC Connection"

  Meanwhile:
  - Health check: /actuator/health → DOWN (DB not reachable)
  - Kubernetes readiness probe fails
  - Load balancer stops sending traffic to unhealthy pods

  But: All pods fail readiness check simultaneously!
  → No healthy pods → traffic dropped entirely
```

---

## Immediate Response Steps

```bash
# 1. Assess scope
kubectl get pods -n production | grep -v Running  # Any failing pods?
curl https://api.mycompany.com/health              # Is the API responding?

# 2. Check database
psql -h primary.db.internal -U admin -c "SELECT 1;"  # Can we connect?
# Connection refused or timeout → DB is down

# 3. Check AWS RDS events
aws rds describe-events --source-identifier myapp-prod-db

# 4. Check if replica is available
psql -h replica.db.internal -U admin -c "SELECT 1;"  # Replica responding?

# 5. Promote replica to primary (if auto-failover not configured)
aws rds failover-db-cluster --db-cluster-identifier myapp-prod

# 6. Update connection string to point to new primary
# (In AWS RDS Multi-AZ: this happens automatically with 60-90s cutover)
```

---

## Application-Level Resilience Fixes

### Fix 1: Shorter Timeouts

```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 5000   # 5 seconds (not 30)
      validation-timeout: 2000   # 2 seconds
      # Fail fast → pods fail readiness → traffic rerouted to healthy pods
```

### Fix 2: Retry Logic

```java
@Retryable(
    value = {DataAccessException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2))
@Transactional
public Order createOrder(OrderRequest request) {
    return orderRepository.save(new Order(request));
}
```

### Fix 3: Read/Write Split

```java
// Route reads to replica, writes to primary
// If primary is down, reads still work from replica
@Transactional(readOnly = true)  // → Routed to replica
public List<Order> getOrders() { ... }

@Transactional  // → Routed to primary
public Order createOrder(OrderRequest request) { ... }
```

### Fix 4: Circuit Breaker on DB Operations

```java
@CircuitBreaker(name = "database", fallbackMethod = "dbFallback")
public Order findOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

public Order dbFallback(Long id, DataAccessException e) {
    // Return from cache, or return partial data, or throw appropriate error
    return orderCache.get(id);
}
```

---

## Architecture Improvements After Outage

1. **RDS Multi-AZ:** Automatic failover within 60-90 seconds
2. **Read replicas:** Separate read/write traffic (reads survive primary failure)
3. **Connection pooling tuning:** Short timeouts for fast failure detection
4. **Readiness probe improvement:** Use separate circuit breaker per component
5. **Graceful degradation:** Serve cached data during DB outage
6. **Runbook:** Document manual failover steps for on-call engineers

---

## Post-Mortem Template

```
INCIDENT: Database Primary Failure
DATE: 2024-01-15
DURATION: 30 minutes (09:15 - 09:45)
IMPACT: 100% error rate on write operations, 30% on reads

TIMELINE: (detailed above)

ROOT CAUSE:
  Hardware failure on primary DB instance.
  Slow failover due to long connection timeouts.

CONTRIBUTING FACTORS:
  - No automatic failover configured
  - Connection timeouts too long (30s)
  - All pods failed readiness checks simultaneously

ACTION ITEMS:
  [P0] Enable RDS Multi-AZ for automatic failover (Due: this week)
  [P1] Reduce HikariCP connection timeout to 5 seconds
  [P1] Add read replica with read/write splitting
  [P2] Implement graceful degradation for read operations
  [P3] Add runbook for manual DB failover
  [P3] Implement canary releases to limit blast radius

LESSONS LEARNED:
  - Fast failure detection > long retry waits
  - Separate read/write traffic for better resilience
  - Practice runbook execution before incidents
```
