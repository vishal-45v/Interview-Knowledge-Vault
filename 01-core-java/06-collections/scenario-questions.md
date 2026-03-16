# Collections — Scenario Questions

> 30 real-world scenario questions about collections usage, pitfalls, and design choices.

---

## Scenario 1: HashMap Key Mutation Bug

**Context:**
```java
Map<List<String>, Integer> map = new HashMap<>();
List<String> key = new ArrayList<>(Arrays.asList("a", "b"));
map.put(key, 1);

key.add("c");  // mutate the key!
System.out.println(map.get(key));  // null!
System.out.println(map.get(Arrays.asList("a", "b")));  // null!
```
**Question:** Explain this behavior. What is the fix?

**Answer:**
`List.hashCode()` is computed from its contents. After `key.add("c")`, the list's hash changes. The entry is now in the "wrong" bucket based on its new hash. Both lookups return null because the key is no longer findable.

**Fix:** Never use mutable objects as HashMap keys. Use immutable objects (String, Integer, custom immutable record):
```java
Map<List<String>, Integer> map = new HashMap<>();
List<String> key = List.of("a", "b");  // immutable — hashCode never changes
map.put(key, 1);
System.out.println(map.get(List.of("a", "b")));  // 1 — correct
```

---

## Scenario 2: ArrayList vs LinkedList Performance Test

**Context:** A developer switches from ArrayList to LinkedList thinking that frequent insertions in the middle will be faster.

**Question:** Is this always correct? When might ArrayList still win?

**Answer:**
LinkedList insertion at a known node position is O(1). But finding the node at index `i` is O(n). So unless you use an iterator (which positions the cursor), `list.add(i, element)` on LinkedList is still O(n).

ArrayList's O(n) shift is actually faster in practice for small-to-medium sizes due to System.arraycopy using native memory operations (memcpy) which is extremely cache-friendly.

For most use cases, ArrayList wins. LinkedList advantages are narrow:
- Queue operations (`addFirst/removeFirst`) — O(1) vs ArrayList's O(n) for removeFirst
- When you hold an iterator cursor and insert at that position

```java
// Benchmark shows ArrayList is usually faster even for frequent middle insertions
// due to CPU cache effects. Profile before assuming LinkedList is better.
```

---

## Scenario 3: ConcurrentHashMap vs synchronized HashMap

**Context:** A service uses `Collections.synchronizedMap(new HashMap<>())` under high concurrency and experiences contention.

**Question:** Explain the performance difference and migrate to ConcurrentHashMap.

**Answer:**
`Collections.synchronizedMap` wraps every method with `synchronized(this.mutex)` — a single lock for all operations. Under high concurrency, all threads compete for one lock.

`ConcurrentHashMap` in Java 8+ uses CAS + per-bucket locking. Multiple threads can operate on different buckets simultaneously.

```java
// Before — single global lock:
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
// All reads AND writes compete for the same lock

// After — fine-grained locking:
ConcurrentHashMap<String, Integer> concurrentMap = new ConcurrentHashMap<>();
// Reads: lock-free (volatile reads)
// Writes: synchronized only at bucket level

// Important: iteration must be done differently:
// syncMap requires external sync during iteration:
synchronized (syncMap) {
    for (Map.Entry<String, Integer> e : syncMap.entrySet()) { ... }
}

// ConcurrentHashMap iterator is weakly consistent — safe without external lock:
for (Map.Entry<String, Integer> e : concurrentMap.entrySet()) { ... }
```

---

## Scenario 4: PriorityQueue with Custom Ordering

**Context:** A task scheduler needs to process tasks in order: CRITICAL first, then HIGH, then NORMAL, then LOW. Within each priority, oldest tasks first (by submission time).

**Question:** Implement the correct ordering.

**Answer:**
```java
enum Priority { CRITICAL, HIGH, NORMAL, LOW }

record Task(String id, Priority priority, long submittedAt) {}

PriorityQueue<Task> scheduler = new PriorityQueue<>(
    Comparator.comparingInt((Task t) -> t.priority().ordinal())
              .thenComparingLong(Task::submittedAt)
);

scheduler.offer(new Task("t1", Priority.NORMAL, 1000));
scheduler.offer(new Task("t2", Priority.CRITICAL, 2000));
scheduler.offer(new Task("t3", Priority.HIGH, 1500));
scheduler.offer(new Task("t4", Priority.CRITICAL, 500));

Task next = scheduler.poll();  // t4 — CRITICAL + oldest (time 500)
```

---

## Scenario 5: LRU Cache Implementation

**Context:** Implement an LRU cache with O(1) get and put.

**Answer:**
```java
public class LRUCache<K, V> {
    private final int capacity;
    private final LinkedHashMap<K, V> cache;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {  // accessOrder=true
            @Override
            protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                return size() > capacity;
            }
        };
    }

    public synchronized V get(K key) {
        return cache.getOrDefault(key, null);  // moves to MRU position
    }

    public synchronized void put(K key, V value) {
        cache.put(key, value);  // removes LRU if over capacity
    }

    public synchronized int size() { return cache.size(); }
}
```

---

## Scenario 6: Deduplication While Preserving Order

**Context:** Given a list of user events with possible duplicates, return a deduplicated list preserving first occurrence order.

**Answer:**
```java
public List<Event> deduplicate(List<Event> events) {
    // LinkedHashSet: O(1) lookup + insertion order preserved
    Set<Event> seen = new LinkedHashSet<>(events);
    return new ArrayList<>(seen);
    // Events must implement equals() and hashCode() correctly
}

// Alternative without modifying equals/hashCode — dedup by event ID:
public List<Event> deduplicateById(List<Event> events) {
    Set<String> seen = new LinkedHashSet<>();
    return events.stream()
        .filter(e -> seen.add(e.getId()))  // add returns false for duplicates
        .collect(Collectors.toList());
}
```

---

## Scenario 7: Sorting a List of Objects by Multiple Fields

**Context:** Sort a list of orders: by status (PENDING first, COMPLETED last), then by amount descending, then by creation date ascending.

**Answer:**
```java
List<Order> sorted = orders.stream()
    .sorted(Comparator
        .<Order, Integer>comparing(o -> statusOrder(o.getStatus()))
        .thenComparing(Comparator.comparingDouble(Order::getAmount).reversed())
        .thenComparing(Order::getCreatedAt))
    .collect(Collectors.toList());

private int statusOrder(OrderStatus status) {
    return switch (status) {
        case PENDING -> 0;
        case PROCESSING -> 1;
        case COMPLETED -> 2;
    };
}
```

---

## Scenario 8: Frequency Count with Streams and Collections

**Context:** Given a list of transactions, count how many transactions exist per customer.

**Answer:**
```java
// Using Collectors.groupingBy + counting:
Map<String, Long> countPerCustomer = transactions.stream()
    .collect(Collectors.groupingBy(
        Transaction::getCustomerId,
        Collectors.counting()
    ));

// Using merge (when building incrementally):
Map<String, Long> counts = new HashMap<>();
for (Transaction t : transactions) {
    counts.merge(t.getCustomerId(), 1L, Long::sum);
}

// Top 10 customers by transaction count:
countPerCustomer.entrySet().stream()
    .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
    .limit(10)
    .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
```

---

## Scenario 9: ThreadLocal vs Synchronized for Per-Thread Data

**Context:** Each HTTP request needs its own request context (trace ID, user ID, etc.) accessible throughout the call stack without passing it explicitly.

**Question:** Should you use ThreadLocal or synchronized?

**Answer:**
ThreadLocal is the correct choice here — each thread (request) needs its own independent state, not shared state.

```java
public class RequestContext {
    private static final ThreadLocal<RequestContext> CONTEXT = new ThreadLocal<>();

    private final String traceId;
    private final String userId;

    public static void set(RequestContext ctx) { CONTEXT.set(ctx); }
    public static RequestContext get() { return CONTEXT.get(); }
    public static void clear() { CONTEXT.remove(); }
}

// Spring filter:
@Component
public class ContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        try {
            RequestContext.set(new RequestContext(generateTraceId(), extractUserId(req)));
            chain.doFilter(req, res);
        } finally {
            RequestContext.clear();  // essential — thread pool reuse!
        }
    }
}
```

---

## Scenario 10: Collections Performance Anti-Pattern

**Context:** A code review finds:
```java
List<User> users = userRepository.findAll();  // 100,000 users
for (int i = 0; i < users.size(); i++) {  // users.size() called each iteration
    User user = users.get(i);
    // process...
}
```

**Question:** Identify the performance concerns.

**Answer:**
1. `users.size()` for ArrayList is O(1) — not an issue, JIT inlines it
2. The real concern: `findAll()` loads 100,000 entities into memory
3. `users.get(i)` on ArrayList is O(1) — fine

**Real fixes:**
```java
// Fix 1 — Use iterator (slightly cleaner):
for (User user : users) { process(user); }  // enhanced for loop uses iterator

// Fix 2 — Stream with batching (avoid loading all into memory):
userRepository.findAllInBatches(1000)  // returns Stream<List<User>>
    .flatMap(Collection::stream)
    .forEach(this::process);

// Fix 3 — Database-level pagination:
int page = 0;
Page<User> batch;
do {
    batch = userRepository.findAll(PageRequest.of(page++, 1000));
    batch.forEach(this::process);
} while (batch.hasNext());
```

---

## Scenario 11: Grouping and Transforming with Collectors

**Context:** Given a list of order items, produce a map of `productId → total revenue`.

**Answer:**
```java
Map<String, BigDecimal> revenueByProduct = orderItems.stream()
    .collect(Collectors.groupingBy(
        OrderItem::getProductId,
        Collectors.mapping(
            item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())),
            Collectors.reducing(BigDecimal.ZERO, BigDecimal::add)
        )
    ));

// Alternative with toMap:
Map<String, BigDecimal> revenue = orderItems.stream()
    .collect(Collectors.toMap(
        OrderItem::getProductId,
        item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())),
        BigDecimal::add  // merge function for duplicate keys
    ));
```

---

## Scenario 12: Safe Map Iteration with Removal

**Context:** Remove all expired sessions from a concurrent map.

**Answer:**
```java
ConcurrentHashMap<String, Session> sessions = new ConcurrentHashMap<>();

// Safe — ConcurrentHashMap entrySet iterator is weakly consistent and safe for removal:
sessions.entrySet().removeIf(entry -> entry.getValue().isExpired());

// Or explicitly:
sessions.forEach((sessionId, session) -> {
    if (session.isExpired()) {
        sessions.remove(sessionId, session);  // atomic check-and-remove
    }
});
```

---

## Scenario 13: Efficient Multi-Key Grouping

**Context:** Group employees by department and role simultaneously.

**Answer:**
```java
record GroupKey(String department, String role) {}

Map<GroupKey, List<Employee>> grouped = employees.stream()
    .collect(Collectors.groupingBy(
        emp -> new GroupKey(emp.getDepartment(), emp.getRole())
    ));

// Access:
List<Employee> engineersInBackend = grouped.get(new GroupKey("Engineering", "Backend"));

// Nested map alternative:
Map<String, Map<String, List<Employee>>> nested = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getRole)
    ));
```

---

## Scenario 14: Choosing Between `contains()` Performance

**Context:** A service needs to check if a user ID is in a blocklist of 1 million IDs, on every request.

**Question:** What collection would you use?

**Answer:**
```java
// Don't use List — O(n) contains is too slow for 1M elements on each request
List<String> blocklist = new ArrayList<>(blocked);
blocklist.contains(userId);  // O(n) — bad

// Use HashSet — O(1) contains:
Set<String> blocklist = new HashSet<>(blocked);
blocklist.contains(userId);  // O(1) — ideal

// If blocklist is static/rarely-changing, load once at startup:
@Component
public class BlocklistService {
    private volatile Set<String> blocklist = Collections.emptySet();

    @Scheduled(fixedRate = 60000)
    public void refresh() {
        Set<String> fresh = new HashSet<>(loadFromDatabase());
        this.blocklist = Collections.unmodifiableSet(fresh);  // atomic reference swap
    }

    public boolean isBlocked(String userId) {
        return blocklist.contains(userId);  // O(1) — no lock needed (volatile + unmodifiable)
    }
}
```

---

## Scenario 15: Sorted Deduplication

**Context:** Given a list of integers with duplicates, return a sorted list of unique values efficiently.

**Answer:**
```java
// Using TreeSet (sorted + deduplicated):
List<Integer> nums = Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5);
List<Integer> sortedUnique = new ArrayList<>(new TreeSet<>(nums));
// [1, 2, 3, 4, 5, 6, 9]

// Using stream:
List<Integer> sortedUnique2 = nums.stream()
    .distinct()
    .sorted()
    .collect(Collectors.toList());

// Note: TreeSet approach is generally faster for large inputs
// because it sorts+deduplicates in one O(n log n) pass
```

---

## Scenario 16: Flatten Nested Collections

**Context:** Given a list of orders where each order has a list of items, get all unique product IDs.

**Answer:**
```java
Set<String> allProductIds = orders.stream()
    .flatMap(order -> order.getItems().stream())  // flatten
    .map(OrderItem::getProductId)
    .collect(Collectors.toSet());  // toSet deduplicates

// With statistics:
Map<String, Long> productFrequency = orders.stream()
    .flatMap(order -> order.getItems().stream())
    .collect(Collectors.groupingBy(OrderItem::getProductId, Collectors.counting()));
```

---

## Scenario 17: Fixed-Size Queue (Sliding Window)

**Context:** Implement a sliding window that keeps the last N measurements.

**Answer:**
```java
public class SlidingWindow<T> {
    private final int maxSize;
    private final Deque<T> window = new ArrayDeque<>();

    public SlidingWindow(int maxSize) { this.maxSize = maxSize; }

    public void add(T measurement) {
        if (window.size() == maxSize) window.pollFirst();  // evict oldest
        window.addLast(measurement);
    }

    public List<T> getWindow() { return new ArrayList<>(window); }

    // Compute average (for numeric windows):
    public OptionalDouble average(List<Double> values) {
        return values.stream().mapToDouble(Double::doubleValue).average();
    }
}
```

---

## Scenario 18: Partitioning with Collectors

**Context:** Split a list of orders into fulfilled and unfulfilled.

**Answer:**
```java
Map<Boolean, List<Order>> partitioned = orders.stream()
    .collect(Collectors.partitioningBy(Order::isFulfilled));

List<Order> fulfilled = partitioned.get(true);
List<Order> unfulfilled = partitioned.get(false);

// With additional aggregation:
Map<Boolean, Long> countByFulfillment = orders.stream()
    .collect(Collectors.partitioningBy(Order::isFulfilled, Collectors.counting()));
```

---

## Scenario 19: Immutable Collections in API Design

**Context:** A service's public API returns a configuration map. Internal code modifies this map. External callers are breaking the service by modifying the returned map.

**Question:** Fix the API design.

**Answer:**
```java
// Bad — returns mutable internal state:
private Map<String, String> config = new HashMap<>();

public Map<String, String> getConfig() {
    return config;  // caller can modify!
}

// Fix 1 — return unmodifiable view:
public Map<String, String> getConfig() {
    return Collections.unmodifiableMap(config);  // still reflects internal changes
}

// Fix 2 — return defensive copy:
public Map<String, String> getConfig() {
    return new HashMap<>(config);  // snapshot, callers can't affect internal state
}

// Fix 3 — return immutable copy (Java 10+):
public Map<String, String> getConfig() {
    return Map.copyOf(config);  // immutable + defensive copy
}

// Fix 4 — store as unmodifiable from the start (if config is set once):
private final Map<String, String> config = Map.copyOf(loadConfig());
public Map<String, String> getConfig() { return config; }  // safe — already immutable
```

---

## Scenario 20: EnumMap for State Machine Transitions

**Context:** Model a state machine for order status transitions.

**Answer:**
```java
enum OrderStatus { PENDING, PAYMENT_RECEIVED, PROCESSING, SHIPPED, DELIVERED, CANCELLED }

// Use EnumMap for O(1) transition lookups:
private static final EnumMap<OrderStatus, Set<OrderStatus>> TRANSITIONS =
    new EnumMap<>(OrderStatus.class);

static {
    TRANSITIONS.put(PENDING, EnumSet.of(PAYMENT_RECEIVED, CANCELLED));
    TRANSITIONS.put(PAYMENT_RECEIVED, EnumSet.of(PROCESSING, CANCELLED));
    TRANSITIONS.put(PROCESSING, EnumSet.of(SHIPPED, CANCELLED));
    TRANSITIONS.put(SHIPPED, EnumSet.of(DELIVERED));
    TRANSITIONS.put(DELIVERED, EnumSet.noneOf(OrderStatus.class));
    TRANSITIONS.put(CANCELLED, EnumSet.noneOf(OrderStatus.class));
}

public boolean canTransition(OrderStatus from, OrderStatus to) {
    return TRANSITIONS.get(from).contains(to);
}

public void transition(Order order, OrderStatus newStatus) {
    if (!canTransition(order.getStatus(), newStatus)) {
        throw new IllegalStateException(
            "Cannot transition from " + order.getStatus() + " to " + newStatus);
    }
    order.setStatus(newStatus);
}
```

---

## Scenario 21: Map.getOrDefault vs computeIfAbsent

**Context:** Building a multi-value map (Map<String, List<String>>).

**Answer:**
```java
Map<String, List<String>> multimap = new HashMap<>();

// getOrDefault — doesn't put the default if key is absent:
List<String> items = multimap.getOrDefault("key", new ArrayList<>());
items.add("value");
// BUG: the new list is not in the map if key was absent!

// computeIfAbsent — creates and puts default if absent (correct):
multimap.computeIfAbsent("key", k -> new ArrayList<>()).add("value");

// Or more explicitly:
multimap.computeIfAbsent("key", k -> new ArrayList<>());
multimap.get("key").add("value");
```

---

## Scenario 22: Collections in Spring Service Layer

**Context:** A Spring service builds a response from multiple sources. How do you combine the results efficiently?

**Answer:**
```java
@Service
public class DashboardService {
    public DashboardResponse buildDashboard(String userId) {
        // Parallel data fetching with CompletableFuture
        CompletableFuture<List<Order>> ordersFuture =
            CompletableFuture.supplyAsync(() -> orderRepo.findByUserId(userId));
        CompletableFuture<List<Notification>> notifFuture =
            CompletableFuture.supplyAsync(() -> notifRepo.findUnread(userId));

        List<Order> orders = ordersFuture.join();
        List<Notification> notifications = notifFuture.join();

        // Efficient aggregation using collectors:
        Map<OrderStatus, Long> statusCounts = orders.stream()
            .collect(Collectors.groupingBy(Order::getStatus, Collectors.counting()));

        Map<String, BigDecimal> revenueByMonth = orders.stream()
            .filter(o -> o.getStatus() == OrderStatus.COMPLETED)
            .collect(Collectors.groupingBy(
                o -> o.getCompletedAt().getMonth().name(),
                Collectors.mapping(Order::getTotal,
                    Collectors.reducing(BigDecimal.ZERO, BigDecimal::add))
            ));

        return new DashboardResponse(statusCounts, revenueByMonth, notifications);
    }
}
```

---

## Scenario 23: Finding Duplicates in a List

**Context:** Given a list of transaction IDs, find duplicates efficiently.

**Answer:**
```java
public Set<String> findDuplicates(List<String> ids) {
    Set<String> seen = new HashSet<>();
    Set<String> duplicates = new LinkedHashSet<>();  // preserve order of first duplicate occurrence

    for (String id : ids) {
        if (!seen.add(id)) {  // add returns false if already present
            duplicates.add(id);
        }
    }
    return duplicates;
}

// With streams:
public Set<String> findDuplicatesStream(List<String> ids) {
    Map<String, Long> frequency = ids.stream()
        .collect(Collectors.groupingBy(id -> id, Collectors.counting()));
    return frequency.entrySet().stream()
        .filter(e -> e.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toSet());
}
```

---

## Scenario 24: Merging Multiple Maps

**Context:** Merge multiple configuration maps, where later maps override earlier ones.

**Answer:**
```java
public Map<String, String> mergeConfigs(List<Map<String, String>> configs) {
    // Simple fold — later values override earlier:
    Map<String, String> merged = new HashMap<>();
    configs.forEach(merged::putAll);
    return merged;

    // With conflict resolution (e.g., concatenate values):
    Map<String, String> merged2 = configs.stream()
        .flatMap(m -> m.entrySet().stream())
        .collect(Collectors.toMap(
            Map.Entry::getKey,
            Map.Entry::getValue,
            (v1, v2) -> v1 + "," + v2  // merge: concatenate on conflict
        ));
}
```

---

## Scenario 25: Efficient Contains Check for Sorted List

**Context:** Given a sorted list of 1 million integers, you need to check if a value exists.

**Answer:**
```java
List<Integer> sortedList = getSortedList();  // 1 million integers

// Bad — O(n) linear search:
sortedList.contains(target);  // scans entire list

// Good — O(log n) binary search:
int index = Collections.binarySearch(sortedList, target);
boolean found = index >= 0;

// Best for frequent lookups — convert to HashSet:
Set<Integer> lookupSet = new HashSet<>(sortedList);  // O(n) one-time cost
lookupSet.contains(target);  // O(1) per lookup — pay once, benefit many times
```

---

## Scenario 26: Checking Collection Null Safety

**Context:** A method receives a `Collection<T>` that might be null or empty.

**Answer:**
```java
// Null-safe utility:
public <T> boolean isNullOrEmpty(Collection<T> collection) {
    return collection == null || collection.isEmpty();
}

// Apache Commons CollectionUtils:
CollectionUtils.isEmpty(collection);

// Return safe empty collection from methods:
public List<Order> getOrders(String userId) {
    List<Order> result = repository.findByUserId(userId);
    return result != null ? result : Collections.emptyList();  // never return null!
}

// Java 9+ — Optional with orElse:
public List<Order> getOrders(String userId) {
    return Optional.ofNullable(repository.findByUserId(userId))
        .orElse(List.of());
}
```

---

## Scenario 27: Batch Processing with Sublists

**Context:** Process a list of 100,000 records in batches of 1,000 to avoid OOM.

**Answer:**
```java
public void processBatch(List<Record> allRecords) {
    int batchSize = 1000;
    for (int i = 0; i < allRecords.size(); i += batchSize) {
        List<Record> batch = allRecords.subList(i, Math.min(i + batchSize, allRecords.size()));
        // subList returns a view — no copy! memory efficient
        processBatch(batch);
    }
}

// With streams partition (Guava):
Lists.partition(allRecords, 1000).forEach(this::processBatch);

// Custom collector:
public <T> List<List<T>> partition(List<T> list, int batchSize) {
    return IntStream.range(0, (list.size() + batchSize - 1) / batchSize)
        .mapToObj(i -> list.subList(i * batchSize, Math.min((i + 1) * batchSize, list.size())))
        .collect(Collectors.toList());
}
```

---

## Scenario 28: Multi-Set (Frequency Map) Pattern

**Context:** Count occurrences of each word in a document.

**Answer:**
```java
String text = "the quick brown fox jumps over the lazy dog";
String[] words = text.split("\\s+");

// Using HashMap:
Map<String, Integer> freq = new HashMap<>();
for (String word : words) {
    freq.merge(word, 1, Integer::sum);
}

// Using Streams:
Map<String, Long> freq2 = Arrays.stream(words)
    .collect(Collectors.groupingBy(w -> w, Collectors.counting()));

// Using Guava Multiset:
Multiset<String> multiset = HashMultiset.create(Arrays.asList(words));
multiset.count("the");  // 2

// Top 5 most frequent words:
freq2.entrySet().stream()
    .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
    .limit(5)
    .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
```

---

## Scenario 29: Converting Between Collection Types

**Context:** You receive a `Set<String>` but need a sorted `List<String>`.

**Answer:**
```java
Set<String> input = new HashSet<>(Arrays.asList("banana", "apple", "cherry"));

// To sorted list:
List<String> sorted = new ArrayList<>(input);
Collections.sort(sorted);  // in-place sort

// Or using TreeSet intermediary:
List<String> sorted2 = new ArrayList<>(new TreeSet<>(input));

// Or streams:
List<String> sorted3 = input.stream().sorted().collect(Collectors.toList());

// To sorted list by custom comparator:
List<String> byLength = input.stream()
    .sorted(Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder()))
    .collect(Collectors.toList());
```

---

## Scenario 30: Detecting the Most Appropriate Collection

**Context:** You need a collection for a word game dictionary with 100,000 words. Operations: check if word exists (frequent), iterate all words (occasional), maintain sorted order (required for display).

**Question:** Which collection would you choose and why?

**Answer:**

If BOTH O(1) lookup AND sorted iteration are needed:
```java
// Option 1 — TreeSet: O(log n) lookup, sorted iteration, 1 collection
TreeSet<String> dictionary = new TreeSet<>();
dictionary.contains("word");  // O(log n) — acceptable for 100K words
dictionary.forEach(System.out::println);  // sorted naturally

// Option 2 — HashSet + sort when needed:
Set<String> dictionary = new HashSet<>();  // O(1) lookup
// When display needed:
List<String> sorted = dictionary.stream().sorted().collect(Collectors.toList());

// Option 3 — Both (if both operations are very frequent):
Set<String> hashDictionary = new HashSet<>();    // O(1) lookup
TreeSet<String> sortedDictionary = new TreeSet<>();  // sorted iteration
// Keep both in sync on modification

// Best answer for interview:
// For 100K words, TreeSet's O(log n) lookup is still fast (~17 comparisons)
// and avoids maintaining dual structures. Use TreeSet unless profiling shows
// the log(n) lookup is a bottleneck.
```
