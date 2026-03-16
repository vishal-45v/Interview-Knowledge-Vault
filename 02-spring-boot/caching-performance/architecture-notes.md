# Caching & Performance — Architecture Notes

---

## Spring Cache Abstraction Layers

```
Application Code
@Cacheable("products")
     │
     ▼
Spring Cache Abstraction (CacheManager, Cache interfaces)
     │
     ├── SimpleCacheManager (ConcurrentHashMap — dev/test)
     ├── CaffeineCacheManager (Caffeine — high-performance local)
     ├── RedisCacheManager (Redis — distributed)
     ├── EhCacheCacheManager (EhCache)
     └── CompositeCacheManager (multiple providers)

Application code is decoupled from cache implementation.
Change CacheManager bean → change cache backend without modifying service code.
```

---

## @Cacheable Execution Flow

```
Client calls productService.getProduct(1L)
         │
         ▼
[Spring Cache Interceptor proxy intercepts]
         │
         ▼
Generate cache key: "products::1"
         │
         ▼
cacheManager.getCache("products").get("products::1")
  ├── HIT → return cached value (method NOT called)
  └── MISS ─► Execute real method
                     │
                     ▼
              Return value obtained
                     │
                     ▼
         cache.put("products::1", returnValue)
                     │
                     ▼
         Return value to caller
```

---

## Cache Key Generation

```
Default KeyGenerator (SimpleKeyGenerator):
  No params:    SimpleKey.EMPTY
  One param:    param.toString()
  Multi params: SimpleKey([param1, param2, ...])

Custom key via SpEL:
  @Cacheable(key = "#id")                    → id value
  @Cacheable(key = "#user.id")               → user.id field
  @Cacheable(key = "#root.method.name + #id") → "getProduct1"
  @Cacheable(key = "T(java.util.Objects).hash(#a, #b)")  → hash of params
```
