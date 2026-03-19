# Redis Caching Strategies

---

## Redis Data Structures for Caching

```java
// String — simple key/value
redisTemplate.opsForValue().set("product:123", productJson, Duration.ofHours(1));

// Hash — object fields
redisTemplate.opsForHash().putAll("user:456", Map.of(
    "name", "Alice",
    "email", "alice@example.com"
));

// List — recent items, queues
redisTemplate.opsForList().leftPush("recent-products:user:789", "productId123");
redisTemplate.opsForList().trim("recent-products:user:789", 0, 9);  // Keep 10

// Set — unique items, tags
redisTemplate.opsForSet().add("product:tags:123", "electronics", "portable", "sale");

// Sorted Set — leaderboards, rate limiting
redisTemplate.opsForZSet().add("product-views", "product:123", 9500.0);
List<String> topProducts = redisTemplate.opsForZSet().reverseRange("product-views", 0, 9);
```

---

## Cache Key Naming Conventions

```
Pattern: {domain}:{type}:{identifier}

Examples:
  product:detail:123              ← product detail by ID
  user:profile:alice@company.com  ← user profile by email
  search:results:electronics:page1 ← paginated search
  session:token:abc123            ← session token
  rate-limit:ip:192.168.1.1       ← rate limiting counter
  lock:order:processing:456       ← distributed lock
```

---

## Distributed Lock with Redis

```java
@Service
public class DistributedLockService {

    @Autowired
    private StringRedisTemplate redisTemplate;

    public <T> T executeWithLock(String lockKey, Duration timeout, Supplier<T> operation) {
        String lockValue = UUID.randomUUID().toString();
        String fullKey = "lock:" + lockKey;

        // Acquire lock: SET key value NX PX timeout
        Boolean acquired = redisTemplate.opsForValue()
            .setIfAbsent(fullKey, lockValue, timeout);

        if (!Boolean.TRUE.equals(acquired)) {
            throw new LockNotAvailableException("Could not acquire lock: " + lockKey);
        }

        try {
            return operation.get();
        } finally {
            // Release lock ONLY if we own it (compare-and-delete)
            String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                            "return redis.call('del', KEYS[1]) else return 0 end";
            redisTemplate.execute(
                new DefaultRedisScript<>(script, Long.class),
                List.of(fullKey), lockValue);
        }
    }
}
```

---

## Redis Cache Invalidation Patterns

```java
// Strategy 1: TTL-based (simplest, accepts eventual consistency)
redisTemplate.opsForValue().set("product:123", json, Duration.ofMinutes(30));
// Stale for up to 30 minutes after DB update — acceptable for product catalog

// Strategy 2: Explicit delete on update (write-through consistency)
@CacheEvict(value = "products", key = "#product.id")
public void updateProduct(Product product) {
    productRepository.save(product);
    // Cache evicted immediately — next read from DB ✓
}

// Strategy 3: Event-driven invalidation (microservices)
// ProductService publishes event → CacheInvalidationConsumer subscribes
@KafkaListener(topics = "product-updated")
public void invalidateProductCache(ProductUpdatedEvent event) {
    redisTemplate.delete("product:" + event.getProductId());
}
```

---

## Redis Pub/Sub for Real-Time Updates

```java
@Configuration
public class RedisMessageConfig {

    @Bean
    public RedisMessageListenerContainer messageListenerContainer(
            RedisConnectionFactory connectionFactory) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(priceUpdateListener(),
            new PatternTopic("product:price:*"));
        return container;
    }

    @Bean
    public MessageListenerAdapter priceUpdateListener() {
        return new MessageListenerAdapter(new PriceUpdateHandler());
    }
}

@Component
public class PriceUpdateHandler implements MessageListener {
    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());  // "product:price:123"
        String body = new String(message.getBody());
        // Broadcast price update to connected WebSocket clients
        webSocketService.broadcast(channel, body);
    }
}
```
