# Low-Level Design Questions

---

## Design a Thread-Safe Singleton

```java
// Modern Java — enum singleton (best approach)
public enum AppConfig {
    INSTANCE;
    
    private final Map<String, String> config = new ConcurrentHashMap<>();
    
    public void load(Properties props) {
        props.forEach((k, v) -> config.put(k.toString(), v.toString()));
    }
    
    public String get(String key) { return config.get(key); }
}

// Double-checked locking (acceptable for complex initialization)
public class DatabaseConnectionPool {
    private static volatile DatabaseConnectionPool instance;  // volatile required!
    
    private DatabaseConnectionPool() {
        // Initialize connection pool
    }
    
    public static DatabaseConnectionPool getInstance() {
        if (instance == null) {
            synchronized (DatabaseConnectionPool.class) {
                if (instance == null) {  // Second check
                    instance = new DatabaseConnectionPool();
                }
            }
        }
        return instance;
    }
}
```

---

## Design a Rate Limiter

```java
public class TokenBucketRateLimiter {
    
    private final long maxTokens;
    private final long refillRatePerSecond;
    private long currentTokens;
    private long lastRefillTimestamp;
    
    public TokenBucketRateLimiter(long maxTokens, long refillRatePerSecond) {
        this.maxTokens = maxTokens;
        this.refillRatePerSecond = refillRatePerSecond;
        this.currentTokens = maxTokens;
        this.lastRefillTimestamp = System.nanoTime();
    }
    
    public synchronized boolean tryConsume() {
        refill();
        if (currentTokens > 0) {
            currentTokens--;
            return true;
        }
        return false;
    }
    
    private void refill() {
        long now = System.nanoTime();
        long elapsed = now - lastRefillTimestamp;
        long tokensToAdd = (long)(elapsed / 1e9 * refillRatePerSecond);
        currentTokens = Math.min(maxTokens, currentTokens + tokensToAdd);
        lastRefillTimestamp = now;
    }
}
```

---

## Design an LRU Cache

```java
public class LRUCache<K, V> {
    
    private final int capacity;
    private final Map<K, V> cache;  // LinkedHashMap maintains insertion/access order
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new LinkedHashMap<K, V>(capacity, 0.75f, true) {
            // removeEldestEntry called after put — return true to evict
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > capacity;
            }
        };
    }
    
    public synchronized V get(K key) {
        return cache.getOrDefault(key, null);
        // LinkedHashMap(accessOrder=true) moves accessed entry to end
    }
    
    public synchronized void put(K key, V value) {
        cache.put(key, value);
        // If over capacity, eldest (least recently accessed) is removed
    }
    
    // O(1) get and put
}
```

---

## Design a Producer-Consumer Queue

```java
public class BoundedQueue<T> {
    
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public BoundedQueue(int capacity) {
        this.capacity = capacity;
    }
    
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // Wait if full
            }
            queue.offer(item);
            notEmpty.signal();  // Wake one consumer
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait if empty
            }
            T item = queue.poll();
            notFull.signal();  // Wake one producer
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```
