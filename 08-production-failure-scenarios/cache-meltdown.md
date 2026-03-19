# Production Failure: Cache Meltdown

---

## Scenario: Redis Cache Failure Under High Traffic

**Situation:** Redis cluster goes down. 100% of traffic falls through to database. Database overwhelmed. Complete outage.

**This is called a "Cache Stampede" or "Cache Meltdown":**
```
Normal: 100K RPM → 95% Cache Hits → 5K RPM to DB (manageable)
Cache Down: 100K RPM → 0% Cache Hits → 100K RPM to DB → DB overwhelmed → full outage
```

---

## Prevention Strategies

### Circuit Breaker on Cache

```java
@CircuitBreaker(name = "cache", fallbackMethod = "fallbackGet")
public User getUser(Long id) {
    return redisTemplate.opsForValue().get("user:" + id);
}

// If Redis is down, circuit opens → directly query DB
public User fallbackGet(Long id, Exception ex) {
    log.warn("Cache unavailable, hitting DB directly: {}", ex.getMessage());
    return userRepository.findById(id).orElseThrow();
}
```

### Cache-Aside with DB Fallback

```java
public User getUser(Long id) {
    try {
        User cached = redisTemplate.opsForValue().get("user:" + id);
        if (cached != null) return cached;
    } catch (RedisException e) {
        log.warn("Cache read failed, falling back to DB: {}", e.getMessage());
        // Don't rethrow — fall through to DB
    }

    User user = userRepository.findById(id).orElseThrow();

    try {
        redisTemplate.opsForValue().set("user:" + id, user, Duration.ofMinutes(30));
    } catch (RedisException e) {
        log.warn("Cache write failed: {}", e.getMessage());
        // Don't fail the request — DB read succeeded
    }

    return user;
}
```

### Dog-Pile Protection (Mutex)

```java
// Only one thread populates cache after miss
// Others wait for the first thread to complete
private final ConcurrentHashMap<String, CompletableFuture<User>> inFlight = new ConcurrentHashMap<>();

public User getUser(Long id) {
    String key = "user:" + id;
    User cached = cache.get(key);
    if (cached != null) return cached;

    CompletableFuture<User> future = inFlight.computeIfAbsent(key,
        k -> CompletableFuture.supplyAsync(() -> {
            User user = userRepository.findById(id).orElseThrow();
            cache.put(key, user);
            return user;
        }).whenComplete((r, e) -> inFlight.remove(key)));

    return future.join();
}
```

---

## Recovery Plan

```
When cache comes back online:
1. Cache is empty (cold start)
2. Requests hit DB → populate cache (gradual warm-up)
3. DB under stress during warm-up period

Mitigation:
  - Rate limit requests during recovery
  - Pre-warm critical keys from DB before opening traffic
  - Use feature flag to enable read-through mode during recovery

Monitoring to add:
  - Cache hit rate metric alert (if hits < 80% → page on-call)
  - Redis connectivity check in health probe
  - DB query rate alert (if > 2x normal → potential cache issue)
```
