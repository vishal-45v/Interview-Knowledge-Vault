# Cache Invalidation Scenarios

> The two hardest things in computer science: naming things and cache invalidation.

---

## Scenario 1: Stale Cache After Update

**Problem:** User updates their profile. Some requests still see the old profile because different servers have different cache entries.

**Solution:** Cache-aside with explicit eviction:
```java
@CacheEvict(value = "user-profiles", key = "#userId")
public void updateUserProfile(Long userId, UpdateProfileRequest request) {
    userRepository.update(userId, request);
}
```

---

## Scenario 2: Cache Stampede After Eviction

**Problem:** Product cache expires at 3am. First 100 requests all miss and query DB simultaneously.

**Solutions:**
1. Stale-while-revalidate: Serve stale content, refresh in background
2. Probabilistic early expiration: Start refreshing before expiry
3. Lock: First thread locks, others wait for refresh

---

## Scenario 3: Distributed Cache Consistency

**Problem:** Service A updates a user record and evicts the cache. Service B (different process) queries and re-populates cache with old data from its read replica (lag!).

**Solution:** 
1. Use a write-through cache (update cache and DB together in same operation)
2. Route cache invalidation through the single DB writer, not readers
3. Use cache versioning: include a version token in the cache key

---

## Scenario 4: Cache Warm-Up After Deployment

**Problem:** After deployment, all caches are empty. First requests are slow.

**Solution:**
1. Pre-warm on ApplicationReadyEvent before accepting traffic
2. Use readiness probe: don't mark as ready until cache is warm
3. Canary deployment: warm on 5% of traffic before full rollout

---

## Scenario 5: Cache Poisoning

**Problem:** A bug causes incorrect data to be cached. Now all users see wrong data until TTL expires.

**Solution:**
1. Low TTL for frequently-changing data
2. Manual cache eviction endpoint: POST /admin/cache/evict/{key}
3. Cache versioning: change cache name prefix on deployment (forces full miss)
4. Circuit breaker on cache layer: if too many errors, skip cache

---

## Cache Invalidation by Event (Preferred for Microservices)

```java
// Product service publishes event
@Service
public class ProductService {
    
    @Autowired
    private ApplicationEventPublisher eventPublisher;
    
    @Transactional
    public void updateProduct(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id).orElseThrow();
        productMapper.updateFrom(product, request);
        productRepository.save(product);
        
        // After commit, publish event
        eventPublisher.publishEvent(new ProductUpdatedEvent(id));
    }
}

// Cache service listens and evicts
@Component
public class ProductCacheInvalidator {
    
    @Autowired
    private CacheManager cacheManager;
    
    @EventListener
    @Async
    public void onProductUpdated(ProductUpdatedEvent event) {
        cacheManager.getCache("products").evict(event.getProductId());
        cacheManager.getCache("product-categories").clear();
    }
}
```
