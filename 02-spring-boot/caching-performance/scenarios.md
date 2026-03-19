# Caching & Performance — Scenarios

> 15+ real-world caching scenarios covering @Cacheable, Redis, eviction, and cache-aside pattern.

---

## Scenario 1: @Cacheable for Expensive Lookups

**Problem:** The `getProductById()` method hits the database on every call, and products rarely change.

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public ProductDTO getProductById(Long id) {
        log.debug("Cache miss — querying DB for product {}", id);
        return productRepository.findById(id)
            .map(productMapper::toDTO)
            .orElseThrow(() -> new ResourceNotFoundException("Product", id));
    }

    @CachePut(value = "products", key = "#result.id")
    public ProductDTO updateProduct(Long id, UpdateProductRequest request) {
        Product product = productRepository.findById(id).orElseThrow();
        productMapper.updateFromRequest(product, request);
        Product saved = productRepository.save(product);
        return productMapper.toDTO(saved);
        // @CachePut: Updates the cache entry with the new value
        // Unlike @Cacheable: always executes the method
    }

    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
        // @CacheEvict: Removes the cache entry
    }
}
```

**Enable caching:**
```java
@SpringBootApplication
@EnableCaching  // Required!
public class MyApplication { }
```

---

## Scenario 2: Redis Caching Configuration

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration redisCacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))  // Global TTL
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer()))
            .disableCachingNullValues();  // Don't cache null values
    }

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        // Per-cache TTL configuration
        Map<String, RedisCacheConfiguration> configs = new HashMap<>();
        configs.put("products", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1)));  // Products cached for 1 hour
        configs.put("users", RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(5)));  // Users cached for 5 minutes

        return RedisCacheManager.builder(factory)
            .cacheDefaults(redisCacheConfiguration())
            .withInitialCacheConfigurations(configs)
            .build();
    }
}
```

---

## Scenario 3: Cache Stampede Problem

**Problem:** A popular cached item expires. Multiple threads simultaneously find a cache miss and all try to load from DB simultaneously, overwhelming the database.

```java
// Problem: Cache stampede on popular product
@Cacheable("products")
public ProductDTO getProduct(Long id) {
    return db.findProduct(id);  // All threads hit DB simultaneously on expiry!
}

// Fix 1: Lock-based (prevents concurrent cache misses)
@Service
public class ProductService {

    private final ConcurrentHashMap<Long, CompletableFuture<ProductDTO>> inFlight = new ConcurrentHashMap<>();

    public ProductDTO getProduct(Long id) {
        // Check cache first
        ProductDTO cached = cacheManager.getCache("products").get(id, ProductDTO.class);
        if (cached != null) return cached;

        // Only one thread loads from DB per ID
        CompletableFuture<ProductDTO> future = inFlight.computeIfAbsent(id,
            k -> CompletableFuture.supplyAsync(() -> {
                ProductDTO dto = loadFromDb(k);
                cacheManager.getCache("products").put(k, dto);
                return dto;
            }).whenComplete((r, e) -> inFlight.remove(k)));

        try {
            return future.get();
        } catch (InterruptedException | ExecutionException e) {
            throw new RuntimeException(e);
        }
    }
}

// Fix 2: Probabilistic early expiration (refresh before expiry)
// Set cache TTL = 30 min but refresh when 25 min have passed
```

---

## Scenario 4: Cache-Aside Pattern

```java
// Cache-aside (lazy loading): Application manages the cache manually
@Service
public class UserService {

    @Autowired
    private RedisTemplate<String, UserDTO> redisTemplate;

    @Autowired
    private UserRepository userRepository;

    private static final Duration TTL = Duration.ofMinutes(15);

    public UserDTO getUserById(Long id) {
        String key = "user:" + id;

        // 1. Check cache
        UserDTO cached = redisTemplate.opsForValue().get(key);
        if (cached != null) {
            return cached;
        }

        // 2. Cache miss — load from DB
        UserDTO user = userRepository.findById(id)
            .map(userMapper::toDTO)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));

        // 3. Populate cache
        redisTemplate.opsForValue().set(key, user, TTL);

        return user;
    }

    public void updateUser(Long id, UpdateUserRequest request) {
        userRepository.update(id, request);

        // 4. Invalidate cache on update
        redisTemplate.delete("user:" + id);
    }
}
```

---

## Scenario 5: @Caching for Multiple Operations

```java
@Service
public class ProductService {

    // Multiple cache operations on one method
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#id"),
        @CacheEvict(value = "productsByCategory", allEntries = true),
        @CacheEvict(value = "featuredProducts", allEntries = true)
    })
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
        // Evicts from product cache, category cache, and featured cache
    }

    // Conditional caching — only cache if product is active
    @Cacheable(value = "products", key = "#id",
               condition = "#result != null && #result.active")
    public ProductDTO getProduct(Long id) {
        return productRepository.findById(id).map(productMapper::toDTO).orElse(null);
    }

    // Unless — cache unless product is out of stock (changes frequently)
    @Cacheable(value = "products", key = "#id",
               unless = "#result?.stockQuantity == 0")
    public ProductDTO getProductForDisplay(Long id) {
        return productRepository.findById(id).map(productMapper::toDTO).orElseThrow();
    }
}
```

---

## Scenario 6: @CacheEvict allEntries for Collection Caches

```java
@Service
public class ProductService {

    @Cacheable(value = "productsByCategory", key = "#category")
    public List<ProductDTO> getProductsByCategory(String category) {
        return productRepository.findByCategory(category)
            .stream().map(productMapper::toDTO).collect(Collectors.toList());
    }

    @CacheEvict(value = "productsByCategory", allEntries = true)
    public void createProduct(CreateProductRequest request) {
        productRepository.save(productMapper.toEntity(request));
        // allEntries = true: Clear ALL entries in productsByCategory cache
        // Because we don't know which category was affected
    }
}
```

---

## Scenario 7: Cache Key Design

```java
@Service
public class SearchService {

    // Default key: method name + all params
    @Cacheable("searches")
    public List<ProductDTO> search(String query, String category, int page) {
        // Key = "search::query#category#page" (all params concatenated)
        return searchRepository.search(query, category, page);
    }

    // Custom SpEL key expression
    @Cacheable(value = "searches", key = "#query.toLowerCase() + ':' + #category + ':' + #page")
    public List<ProductDTO> searchNormalized(String query, String category, int page) {
        // Key normalized: "laptop:electronics:0" (lowercase query)
    }

    // Key from complex object
    @Cacheable(value = "searches", key = "#request.query + ':' + #request.category")
    public SearchResult search(SearchRequest request) {
        return searchRepository.search(request);
    }
}
```

---

## Scenario 8: Cache Warming

**Problem:** After a deployment, all caches are cold. The first requests are slow until the cache is populated.

```java
@Component
public class CacheWarmUpService {

    @Autowired
    private ProductService productService;

    @Autowired
    private ProductRepository productRepository;

    @EventListener(ApplicationReadyEvent.class)
    @Async  // Don't block startup
    public void warmUpProductCache() {
        log.info("Warming up product cache...");

        // Load top 100 most-viewed products
        List<Long> popularProductIds = productRepository.findTop100ByViewCountDesc()
            .stream().map(Product::getId).collect(Collectors.toList());

        for (Long id : popularProductIds) {
            try {
                productService.getProductById(id);  // Loads and caches
            } catch (Exception e) {
                log.warn("Failed to warm cache for product {}", id, e);
            }
        }

        log.info("Cache warmed with {} products", popularProductIds.size());
    }
}
```

---

## Scenario 9: Distributed Cache Consistency

**Problem:** You have multiple instances of your service. One instance updates a product and evicts its cache. The other instances still have the old cached value.

```java
// Solution: Use a shared Redis cache (all instances connect to same Redis)
@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        // All instances share the same Redis — evictions are global ✓
        return RedisCacheManager.builder(redisConnectionFactory).build();
    }
}

// In-memory cache (Caffeine) — NOT shared between instances:
// @Bean CacheManager cacheManager() {
//     return new CaffeineCacheManager("products");
//     // Each instance has its own cache — consistency issues!
// }
```

---

## Scenario 10: Cache Statistics and Monitoring

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(30, TimeUnit.MINUTES)
            .recordStats());  // Enable statistics recording
        return manager;
    }
}

// Expose metrics to Micrometer (Spring Boot Actuator)
@Component
public class CacheMetricsExporter {

    @Autowired
    private CacheManager cacheManager;

    @Autowired
    private MeterRegistry meterRegistry;

    @PostConstruct
    public void registerMetrics() {
        CaffeineCache caffeineCache = (CaffeineCache) cacheManager.getCache("products");
        Cache<Object, Object> nativeCache = caffeineCache.getNativeCache();

        // Register hit/miss counters with Prometheus/Grafana
        Gauge.builder("cache.hit.rate", nativeCache, cache -> cache.stats().hitRate())
            .tag("cache", "products")
            .register(meterRegistry);
    }
}
```
