# Memory Leak Debugging

> Detecting and resolving Java memory leaks in Spring Boot applications.

---

## Common Memory Leak Sources in Java

1. **Static collections holding references:**
```java
public class UserCache {
    private static final Map<Long, User> cache = new HashMap<>();
    // If you add users but never remove them, heap grows unboundedly
    public static void add(Long id, User user) {
        cache.put(id, user);  // Never evicted!
    }
}
```

2. **ThreadLocal not cleaned up:**
```java
private static final ThreadLocal<Connection> connectionHolder = new ThreadLocal<>();
// Thread pools reuse threads. If ThreadLocal is not removed,
// old Connection objects accumulate in the thread pool.
connectionHolder.remove();  // MUST call this!
```

3. **Listener/Observer not unregistered:**
```java
eventBus.subscribe(handler);  // Registered
// Never unregistered → eventBus holds reference to handler
// Handler cannot be GC'd even if you "release" it
```

4. **Inner class holding outer class reference:**
```java
class OuterService {
    class InnerTask implements Runnable {
        // Implicit reference to OuterService.this
        // If InnerTask is long-lived, OuterService is never GC'd
    }
}
// Fix: Use static nested class or don't keep inner instances
```

---

## Detecting Memory Leaks

### Step 1: Monitor Heap Usage

```
Symptom: Heap grows over time, GC happens frequently, eventually OOM

Monitor with:
  - Spring Boot Actuator: /actuator/metrics/jvm.memory.used
  - JVisualVM: Heap trend over time
  - Prometheus: jvm_memory_used_bytes with Grafana dashboard

If heap grows but doesn't decrease after GC → memory leak
```

### Step 2: Heap Dump Analysis

```bash
# Generate heap dump while running
jmap -dump:format=b,file=/tmp/heapdump.hprof <PID>

# Or trigger on OOM (add to JVM flags):
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/var/log/heapdumps/

# Analyze with:
# Eclipse Memory Analyzer (MAT) — best tool, free
# JVisualVM
# jhat (basic, built-in)
```

### Step 3: Analyze with Eclipse MAT

```
Open heapdump.hprof in Eclipse MAT:
1. "Leak Suspects" report → MAT identifies likely leaks automatically
2. "Dominator Tree" → shows objects by retained heap size
3. "OQL" queries: SELECT * FROM java.util.HashMap

Look for:
  - HashMap/ArrayList that's much larger than expected
  - Many instances of a class that should only have a few
  - Long chains of references keeping objects alive
```

---

## Heap Dump Common Findings

```
Finding: 500MB retained by HashMap in com.example.UserCache
  → Static cache never evicted
  → Fix: Use Caffeine/Guava cache with size limit and TTL

Finding: 200MB retained by ThreadLocal
  → ThreadPool threads holding UserContext objects
  → Fix: Add cleanup in filter: try { ... } finally { MDC.clear(); context.remove(); }

Finding: 50,000 HttpClient instances
  → Creating new HttpClient per request instead of sharing singleton
  → Fix: @Bean HttpClient httpClient singleton injected into services

Finding: 100MB retained by Spring session data
  → In-memory session store accumulating expired sessions
  → Fix: Configure session expiry, use Redis for session storage
```

---

## Production Memory Leak Response

```
Step 1: Immediate
  → Scale up instances to handle load while investigating
  → Enable heap dump on OOM on one instance for analysis
  → Set JVM restart trigger if heap > 90%: -XX:+ExitOnOutOfMemoryError

Step 2: Data collection
  → Enable memory metrics (Micrometer → Prometheus)
  → Take heap dump from affected instance
  → Compare heap at t=0 vs t=6h to see growth pattern

Step 3: Analysis
  → Open heap dump in Eclipse MAT
  → "Leak Suspects" report
  → Identify the object type and retention chain

Step 4: Fix and validate
  → Fix in dev, deploy to staging with memory pressure test
  → Run for 24h and verify heap stabilizes
  → Deploy to production with monitoring
```

---

## WeakReference for Cache Implementation

```java
// WeakReference: allows GC to collect the object when memory is needed
// WeakHashMap: entries automatically removed when key is GC'd
// Useful for caches where you don't want to prevent GC

Map<Object, Data> cache = Collections.synchronizedMap(new WeakHashMap<>());
// Entry removed when key has no other strong references
// Good for: metadata associated with objects you don't own

// Caffeine cache with size limit and TTL (better for most cases):
Cache<Long, User> userCache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(30, TimeUnit.MINUTES)
    .recordStats()
    .build();
```
