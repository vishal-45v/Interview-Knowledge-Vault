# Caching & Performance — Structured Answers

---

## Q1: @Cacheable vs @CachePut vs @CacheEvict

@Cacheable: Check cache first. If hit, return cached value. If miss, execute method and cache result.

@CachePut: ALWAYS execute method and update/insert cache with result. Does not check cache first.
Use case: When you update data and want to refresh the cache with the new value.

@CacheEvict: ALWAYS execute method and remove from cache.
Use case: When you delete data or want to force re-load on next access.

---

## Q2: Cache Eviction Strategies

Time-based (TTL): Entry expires after a fixed time.
Size-based (LRU/LFU): When cache is full, evict least recently used or least frequently used.
Explicit eviction: Application code evicts entries on update/delete.
Write-through: Cache updated on every write (strong consistency, more writes).
Write-behind (write-back): Cache updated first, DB updated asynchronously.

---

## Q3: Cache-Aside vs Read-Through vs Write-Through

Cache-aside (lazy loading): Application manages cache manually. Check cache, if miss load from DB and populate cache.
- Application controls caching logic
- Resilient: cache failure degrades gracefully (goes to DB)
- Initial cache miss for cold start

Read-through: Cache layer automatically loads from DB on miss (transparent to application).
- Simpler application code
- Only available with specialized cache providers

Write-through: Every write goes to both cache and DB synchronously.
- Strong consistency between cache and DB
- Higher write latency

---

## Q4: What Is a Cache Stampede?

When a popular cached item expires and multiple concurrent requests all find a cache miss simultaneously. They all query the DB at the same time, overwhelming it.

Solutions:
1. Probabilistic early expiration: Refresh before expiry
2. Mutex/lock: Only one thread loads, others wait
3. Stale-while-revalidate: Serve stale content while refreshing in background

---

## Q5: Redis vs Local Cache (Caffeine)

| Aspect | Caffeine (Local) | Redis (Distributed) |
|--------|-----------------|---------------------|
| Speed | ~ns (in-process) | ~ms (network) |
| Consistency | Per-JVM instance | Shared across instances |
| Persistence | None (lost on restart) | Configurable |
| Serialization | No | Required |
| Capacity | JVM heap | RAM of Redis server |
| Use when | Single instance, speed critical | Multi-instance, shared state |

Common pattern: Two-level cache — Caffeine L1 for hot data, Redis L2 for warm data.
