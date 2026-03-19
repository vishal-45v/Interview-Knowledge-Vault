# Production Failure: Thread Pool Exhaustion

---

## Scenario: All Request Threads Blocked

**Symptom:** New requests to the API return immediately with connection refused or 503. Existing requests are stuck.

```
Normal:
  Tomcat thread pool: 200 threads
  Average request: 50ms → thread free quickly
  Throughput: 4,000 RPS

Problem: Downstream service (inventory API) slows to 10s response time
  200 threads × 10s = all threads blocked waiting for inventory API
  New requests: No threads available → Connection refused or queue overflow
  503 Service Unavailable
```

---

## Root Cause Pattern

```java
// Without timeout — thread blocked indefinitely
@GetMapping("/products/{id}")
public ResponseEntity<Product> getProduct(@PathVariable Long id) {
    ProductDetail detail = productRepository.findById(id).orElseThrow();
    InventoryStatus inventory = inventoryClient.getStatus(id);  // BLOCKS THREAD!
    // If inventoryClient takes 30s, this thread is occupied for 30s
    return ResponseEntity.ok(new Product(detail, inventory));
}
```

---

## Solutions

### 1. Timeouts on HTTP Clients

```java
@Bean
public RestTemplate restTemplate() {
    HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(2000);   // 2s connect timeout
    factory.setReadTimeout(3000);      // 3s read timeout
    return new RestTemplate(factory);
}

@Bean
public WebClient webClient() {
    return WebClient.builder()
        .baseUrl("http://inventory-service")
        .clientConnector(new ReactorClientHttpConnector(
            HttpClient.create()
                .responseTimeout(Duration.ofSeconds(3))
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)))
        .build();
}
```

### 2. Circuit Breaker

```java
@CircuitBreaker(name = "inventory")
@TimeLimiter(name = "inventory")
public CompletableFuture<InventoryStatus> getInventory(Long productId) {
    return CompletableFuture.supplyAsync(() -> inventoryClient.getStatus(productId));
}
```

### 3. Thread Pool Isolation (Bulkhead)

```java
// Give each downstream service its own thread pool
// If inventory service is slow, only INVENTORY threads are exhausted
// ORDER processing continues on separate thread pool

@Bulkhead(name = "inventory", type = Bulkhead.Type.THREADPOOL)
public CompletableFuture<InventoryStatus> getInventory(Long productId) {
    return CompletableFuture.supplyAsync(() -> inventoryClient.getStatus(productId));
}
```

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      inventory:
        max-thread-pool-size: 10   # Only 10 threads for inventory calls
        core-thread-pool-size: 5
        queue-capacity: 20
```

---

## Monitoring Thread Pool Health

```java
// Expose Tomcat thread pool metrics
management:
  endpoint:
    metrics:
      enabled: true

# Metrics available:
# tomcat.threads.busy: Currently processing requests
# tomcat.threads.current: Total threads in pool
# tomcat.threads.config.max: Configured max
```

Alert when `tomcat.threads.busy / tomcat.threads.config.max > 0.8` (80% utilization).
