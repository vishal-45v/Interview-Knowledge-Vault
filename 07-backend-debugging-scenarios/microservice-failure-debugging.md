# Microservice Failure Debugging

---

## Distributed Tracing Setup

```yaml
# Enable distributed tracing
management:
  tracing:
    sampling:
      probability: 1.0  # 100% sampling (reduce in prod)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

Every request gets a `traceparent` header propagated through all services:
```
POST /orders  → Order Service
  traceId=abc, spanId=001
    → Payment Service
       traceId=abc, spanId=002 (same trace, new span)
      → Inventory Service
         traceId=abc, spanId=003
```

---

## Circuit Breaker Pattern

```java
// Without circuit breaker:
// Inventory service down → every order request waits 30s → threads exhausted → cascading failure

// With Resilience4j circuit breaker:
@Service
public class InventoryClient {

    @CircuitBreaker(name = "inventory", fallbackMethod = "fallbackInventoryCheck")
    @TimeLimiter(name = "inventory")
    @Retry(name = "inventory")
    public CompletableFuture<Boolean> checkInventory(Long productId, int quantity) {
        return CompletableFuture.supplyAsync(() ->
            inventoryService.check(productId, quantity));
    }

    // Fallback: degrade gracefully when circuit is open
    public CompletableFuture<Boolean> fallbackInventoryCheck(
            Long productId, int quantity, Exception e) {
        log.warn("Inventory service unavailable, using fallback: {}", e.getMessage());
        return CompletableFuture.completedFuture(true);  // Optimistic: assume in stock
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      inventory:
        sliding-window-size: 10
        failure-rate-threshold: 50      # Open if 50% of last 10 calls fail
        wait-duration-in-open-state: 30s # Stay open for 30s before half-open
        permitted-number-of-calls-in-half-open-state: 3
  timelimiter:
    instances:
      inventory:
        timeout-duration: 2s  # 2 second timeout
  retry:
    instances:
      inventory:
        max-attempts: 3
        wait-duration: 500ms
```

---

## Distributed Transaction Debugging

```
Problem: Order created but inventory not decremented
  → Partial failure in distributed transaction

Investigation:
  1. Check distributed trace: which service failed?
  2. Check each service's logs with the traceId
  3. Check saga state machine: what step was reached?

Common causes:
  - Timeout in one service during saga execution
  - Network partition
  - Bug in compensating transaction

Remediation:
  - Replay the saga from the failed step (if idempotent)
  - Manually compensate: decrement inventory, or cancel order
  - Fix the bug, create migration script for inconsistent data
  - Add idempotency checks to prevent duplicate processing
```

---

## Service Discovery and Load Balancing Issues

```bash
# Spring Cloud with Kubernetes service discovery
# Check if service is registered
kubectl get endpoints myapp-service

# If endpoints are empty → pods not ready (failing health check)
kubectl describe pods -l app=myapp | grep -A10 "Liveness\|Readiness"

# Check service health via actuator
curl http://pod-ip:8080/actuator/health

# Common causes of missing endpoints:
# 1. Container not started yet (startupProbe not passing)
# 2. App crashed (check logs: kubectl logs pod-name)
# 3. Readiness probe failing (check /actuator/health/readiness)
```
