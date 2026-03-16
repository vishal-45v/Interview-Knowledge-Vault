# Caching & Performance — Trap Questions

---

## Trap 1: @Cacheable on Private Methods

**Question:** Why doesn't this cache work?

```java
@Service
public class ProductService {
    @Cacheable("products")
    private ProductDTO getProduct(Long id) { ... }  // private!
}
```

**Answer:** Same as @Transactional — Spring AOP creates a CGLIB proxy that can only intercept public methods. @Cacheable on private methods is silently ignored.

---

## Trap 2: Self-Invocation and @Cacheable

**Question:** Why doesn't caching work when called from the same class?

```java
@Service
public class ProductService {
    public void process() {
        getProduct(1L);  // self-invocation — bypasses proxy!
    }
    
    @Cacheable("products")
    public ProductDTO getProduct(Long id) { ... }
}
```

**Answer:** Same as @Transactional. Self-invocations bypass the AOP proxy. Cache check never happens. Fix by injecting self or extracting to a separate bean.

---

## Trap 3: @CacheEvict with allEntries=true Performance

**Question:** Is allEntries=true on @CacheEvict always safe?

**Answer:** Safe but potentially expensive. allEntries=true removes EVERY entry in that cache region. For large caches, this causes a sudden flood of DB queries as all clients find cache misses simultaneously — essentially a cache stampede.

For production, prefer evicting specific keys. Only use allEntries when the cache is small or the operation is rare.

---

## Trap 4: ConcurrentModificationException with @Cacheable and Collections

**Question:** You cache a List<Product> and a caller modifies the returned list. The cached entry is also modified. Why?

**Answer:** Spring's @Cacheable (with in-memory cache like ConcurrentMapCache) stores a reference to the returned object. If the caller modifies the returned List, the cached entry is also modified — they share the same reference.

Fix: Return an immutable list, or copy the list before returning, or use a cache that serializes/deserializes (like Redis — always returns a new copy).

---

## Trap 5: Caching null Values

**Question:** @Cacheable caches null values by default — why is this a problem?

**Answer:** If getProduct() returns null (product not found), @Cacheable stores null in the cache. All subsequent calls return the cached null without hitting the DB — even after the product is created.

Fix: Use unless = "#result == null" on @Cacheable, or use disableCachingNullValues() in RedisCacheConfiguration.
