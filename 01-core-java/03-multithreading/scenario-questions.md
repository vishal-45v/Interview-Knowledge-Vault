# Multithreading — Scenario Questions

> 30 scenario-based questions simulating real concurrency challenges in production Java applications.

---

## Scenario 1: Bank Transfer Deadlock

**Context:** A banking system has transfer logic:

```java
class BankAccount {
    private double balance;

    synchronized void transfer(BankAccount to, double amount) {
        synchronized (to) {
            this.balance -= amount;
            to.balance += amount;
        }
    }
}
```

**Problem:** Under load, the system occasionally freezes.

**Question:** Identify the deadlock and fix it.

**Answer:**

Thread A calls `accountA.transfer(accountB, 100)` → acquires lock on `accountA`, waits for lock on `accountB`.
Thread B calls `accountB.transfer(accountA, 50)` → acquires lock on `accountB`, waits for lock on `accountA`.
Deadlock!

**Fix — consistent lock ordering using account ID:**
```java
class BankAccount {
    private final long id;
    private double balance;

    void transfer(BankAccount to, double amount) {
        BankAccount first = this.id < to.id ? this : to;
        BankAccount second = this.id < to.id ? to : this;

        synchronized (first) {
            synchronized (second) {
                this.balance -= amount;
                to.balance += amount;
            }
        }
    }
}
```

**Fix 2 — ReentrantLock with timeout:**
```java
void transfer(BankAccount to, double amount) throws InterruptedException {
    while (true) {
        if (this.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
            try {
                if (to.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                    try {
                        this.balance -= amount;
                        to.balance += amount;
                        return;
                    } finally { to.lock.unlock(); }
                }
            } finally { this.lock.unlock(); }
        }
        Thread.sleep(1);  // back off before retry
    }
}
```

---

## Scenario 2: Thread Pool Exhaustion in Web Application

**Context:** A Spring Boot REST API starts returning 503 errors under moderate load. Thread dump shows 200 Tomcat threads all in WAITING state.

**Question:** Diagnose and fix.

**Answer:**

The Tomcat threads are likely blocked waiting for a downstream dependency (database, external API). If the downstream is slow, all threads fill up waiting, and new requests get rejected.

**Diagnosis:**
1. Take thread dump: `jstack <PID>`
2. Look for threads in WAITING or TIMED_WAITING with same stack trace pattern
3. Identify the blocking call (e.g., database query, HTTP call)

**Fix options:**

```java
// Option 1 — Add timeouts to all external calls
RestTemplate restTemplate = new RestTemplateBuilder()
    .setConnectTimeout(Duration.ofSeconds(2))
    .setReadTimeout(Duration.ofSeconds(5))
    .build();

// Option 2 — Use async/reactive for I/O-heavy endpoints
@GetMapping("/orders")
public CompletableFuture<List<Order>> getOrders() {
    return CompletableFuture.supplyAsync(() -> orderService.findAll(), asyncExecutor);
}

// Option 3 — Circuit breaker to fail fast when downstream is slow
@CircuitBreaker(name = "inventoryService", fallbackMethod = "fallback")
public Inventory checkInventory(String productId) {
    return inventoryClient.check(productId);
}

// Option 4 — Increase Tomcat thread pool (band-aid, not a fix)
server.tomcat.max-threads=400
```

---

## Scenario 3: Visibility Bug with Stop Flag

**Context:**

```java
class Worker {
    private boolean running = true;

    void stop() {
        running = false;
    }

    void run() {
        while (running) {
            doWork();
        }
    }
}
```

**Problem:** After calling `stop()`, the worker thread sometimes doesn't stop.

**Question:** Explain and fix.

**Answer:**

`running` is not `volatile`. Each thread may cache it in CPU registers or L1/L2 cache. The worker thread may never see the update from the main thread because there is no happens-before relationship.

**Fix:**
```java
class Worker {
    private volatile boolean running = true;  // volatile ensures visibility

    void stop() {
        running = false;  // flush to main memory
    }

    void run() {
        while (running) {  // reads from main memory each time
            doWork();
        }
    }
}
```

**Alternative — use interrupt:**
```java
class Worker implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                doWork();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
}

Thread t = new Thread(worker);
t.start();
t.interrupt();  // signal to stop
```

---

## Scenario 4: Counter Race Condition

**Context:**

```java
class HitCounter {
    private int count = 0;

    void increment() { count++; }
    int getCount() { return count; }
}
```

**Problem:** After 1000 concurrent increments, count is less than 1000.

**Question:** Why and how to fix?

**Answer:**

`count++` is NOT atomic. It compiles to: (1) read count, (2) add 1, (3) write back. Two threads can read the same value, both increment, and write the same incremented value — losing one increment.

**Fix options:**
```java
// Fix 1 — synchronized:
synchronized void increment() { count++; }

// Fix 2 — AtomicInteger (lock-free, fastest):
private final AtomicInteger count = new AtomicInteger(0);
void increment() { count.incrementAndGet(); }
int getCount() { return count.get(); }

// Fix 3 — LongAdder (even faster under high contention):
private final LongAdder count = new LongAdder();
void increment() { count.increment(); }
long getCount() { return count.sum(); }

// Fix 4 — ReentrantLock:
private final ReentrantLock lock = new ReentrantLock();
void increment() {
    lock.lock();
    try { count++; }
    finally { lock.unlock(); }
}
```

---

## Scenario 5: Memory Leak via ThreadLocal

**Context:** A web application running on Tomcat starts leaking memory over time. Heap dumps show many `UserContext` objects accumulating.

```java
public class SecurityContext {
    private static ThreadLocal<UserContext> context = new ThreadLocal<>();

    public static void setUser(UserContext user) {
        context.set(user);
    }

    public static UserContext getUser() {
        return context.get();
    }
}
```

**Question:** Why is memory leaking? How to fix?

**Answer:**

Tomcat reuses threads from its thread pool. After request processing completes, the `ThreadLocal` is not cleared. The `UserContext` object stays attached to the thread for the lifetime of the server.

Since `ThreadLocal` holds a weak reference to the key but a strong reference to the value, the `UserContext` objects are never garbage collected.

**Fix — always clean up ThreadLocal in a filter/interceptor:**
```java
public class SecurityContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        try {
            chain.doFilter(req, res);
        } finally {
            SecurityContext.clear();  // ALWAYS clear in finally block
        }
    }
}

// Add clear method:
public class SecurityContext {
    private static ThreadLocal<UserContext> context = new ThreadLocal<>();

    public static void clear() {
        context.remove();  // removes from current thread's map
    }
}
```

---

## Scenario 6: Incorrect Double-Checked Locking

**Context:**

```java
public class ExpensiveCache {
    private static ExpensiveCache instance;
    private Map<String, Object> data;

    private ExpensiveCache() {
        this.data = loadExpensiveData();  // takes 10 seconds
    }

    public static ExpensiveCache getInstance() {
        if (instance == null) {               // First check
            synchronized (ExpensiveCache.class) {
                if (instance == null) {        // Second check
                    instance = new ExpensiveCache();
                }
            }
        }
        return instance;
    }
}
```

**Question:** Is this implementation correct? What's missing?

**Answer:**

`instance` must be `volatile`. Without it, the JVM can publish a partially-constructed object:
1. Thread A allocates memory for `ExpensiveCache`
2. Thread A assigns the reference to `instance` (before constructor finishes!)
3. Thread B sees `instance != null` in the first check
4. Thread B returns a partially-constructed object → crashes

**Fix:**
```java
private static volatile ExpensiveCache instance;  // volatile is required
```

**Better fix — Initialization-on-demand holder:**
```java
public class ExpensiveCache {
    private ExpensiveCache() { ... }

    private static class Holder {
        // Loaded lazily when Holder class is first accessed
        // Class initialization is thread-safe by JVM spec
        static final ExpensiveCache INSTANCE = new ExpensiveCache();
    }

    public static ExpensiveCache getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

## Scenario 7: Concurrent Modification Exception

**Context:**

```java
List<Order> orders = getActiveOrders();
for (Order order : orders) {
    if (order.isExpired()) {
        orders.remove(order);  // ConcurrentModificationException!
    }
}
```

**Question:** Why does this fail? Provide multiple solutions.

**Answer:**

The enhanced for-loop uses an iterator internally. Removing from the collection while iterating invalidates the iterator's internal state, triggering `ConcurrentModificationException`.

**Fix 1 — Use iterator's remove():**
```java
Iterator<Order> iterator = orders.iterator();
while (iterator.hasNext()) {
    if (iterator.next().isExpired()) {
        iterator.remove();  // safe — removes via iterator
    }
}
```

**Fix 2 — removeIf (Java 8+):**
```java
orders.removeIf(Order::isExpired);
```

**Fix 3 — Collect and remove:**
```java
orders.removeAll(orders.stream()
    .filter(Order::isExpired)
    .collect(Collectors.toList()));
```

**Fix 4 — For concurrent use, CopyOnWriteArrayList:**
```java
CopyOnWriteArrayList<Order> orders = new CopyOnWriteArrayList<>(getActiveOrders());
// iteration uses a snapshot — safe to modify original during iteration
for (Order order : orders) {
    if (order.isExpired()) orders.remove(order);
}
```

---

## Scenario 8: CompletableFuture Exception Handling

**Context:**

```java
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> callExternalAPI())
    .thenApply(response -> parseResponse(response))
    .thenApply(data -> enrichData(data));

result.get();  // What happens if callExternalAPI() throws?
```

**Question:** How does CompletableFuture propagate exceptions? How would you add proper error handling?

**Answer:**

If any stage throws, the exception is wrapped in `CompletionException` and propagated to all subsequent `thenApply` stages. `get()` throws `ExecutionException` wrapping the original.

**Proper error handling:**
```java
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> callExternalAPI())
    .thenApply(response -> parseResponse(response))
    .thenApply(data -> enrichData(data))
    .exceptionally(ex -> {
        log.error("Pipeline failed", ex.getCause());
        return "fallback-value";  // recover with default
    });

// Or handle provides access to both result and exception:
CompletableFuture<String> handled = result.handle((value, ex) -> {
    if (ex != null) {
        log.error("Failed", ex);
        return "fallback";
    }
    return value;
});

// With timeout (Java 9+):
CompletableFuture<String> withTimeout = result
    .orTimeout(5, TimeUnit.SECONDS)
    .exceptionally(ex -> {
        if (ex instanceof TimeoutException) return "timeout-fallback";
        return "error-fallback";
    });
```

---

## Scenario 9: High CPU from Spinning Thread

**Context:** A service that polls a queue shows 100% CPU on one core.

```java
class QueuePoller {
    private final Queue<Task> queue;

    void poll() {
        while (true) {
            Task task = queue.poll();
            if (task != null) {
                process(task);
            }
            // No sleep — spinning!
        }
    }
}
```

**Question:** Fix the busy-wait loop.

**Answer:**

Spinning (busy-waiting) wastes CPU checking an empty queue repeatedly.

**Fix 1 — Use BlockingQueue:**
```java
private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>();

void poll() {
    while (!Thread.interrupted()) {
        try {
            Task task = queue.take();  // blocks until task available — no CPU wasted
            process(task);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }
    }
}
```

**Fix 2 — Add sleep (temporary workaround):**
```java
void poll() {
    while (true) {
        Task task = queue.poll();
        if (task != null) {
            process(task);
        } else {
            Thread.sleep(100);  // sleep 100ms when queue is empty
        }
    }
}
```

`BlockingQueue` is the correct solution — it uses OS-level wait/notify, consuming no CPU when empty.

---

## Scenario 10: Lost Wake-Up Bug

**Context:**

```java
boolean ready = false;

// Thread A:
ready = true;
// Thread B:
while (!ready) {
    obj.wait();  // Thread A may have set ready=true BEFORE B calls wait!
}
```

**Question:** What is the "lost wake-up" problem here?

**Answer:**

1. Thread B checks `while (!ready)` → false, doesn't call wait
2. Thread A sets `ready = true` and calls `notify()` — nobody is waiting, notification lost
3. Thread B now calls `wait()` — waits forever

The check and wait must be atomic (inside synchronized block):

**Fix:**
```java
synchronized (obj) {
    while (!ready) {   // check inside synchronized — atomically
        obj.wait();    // releases lock while waiting
    }
}

// Thread A:
synchronized (obj) {
    ready = true;
    obj.notifyAll();  // safe — sets ready before notifying
}
```

---

## Scenario 11: ConcurrentHashMap Compute Atomically

**Context:** A word frequency counter needs to be thread-safe:

```java
Map<String, Integer> frequency = new ConcurrentHashMap<>();

void increment(String word) {
    Integer count = frequency.get(word);  // Not atomic with the next line!
    frequency.put(word, count == null ? 1 : count + 1);
}
```

**Question:** Fix the race condition.

**Answer:**

```java
// Fix 1 — merge (Java 8):
void increment(String word) {
    frequency.merge(word, 1, Integer::sum);
    // If key absent → put(word, 1)
    // If key present → put(word, oldValue + 1)
    // Atomic per entry
}

// Fix 2 — compute:
void increment(String word) {
    frequency.compute(word, (k, v) -> v == null ? 1 : v + 1);
}

// Fix 3 — AtomicInteger values:
Map<String, AtomicInteger> frequency = new ConcurrentHashMap<>();

void increment(String word) {
    frequency.computeIfAbsent(word, k -> new AtomicInteger(0))
             .incrementAndGet();
}
```

---

## Scenario 12: Thread Local Database Connection

**Context:** Each request needs its own database connection held for the request duration:

**Question:** Design a connection-per-request mechanism using ThreadLocal.

**Answer:**

```java
public class TransactionManager {
    private static final ThreadLocal<Connection> connectionHolder = new ThreadLocal<>();
    private final DataSource dataSource;

    public Connection getConnection() throws SQLException {
        Connection conn = connectionHolder.get();
        if (conn == null || conn.isClosed()) {
            conn = dataSource.getConnection();
            connectionHolder.set(conn);
        }
        return conn;
    }

    public void beginTransaction() throws SQLException {
        getConnection().setAutoCommit(false);
    }

    public void commit() throws SQLException {
        Connection conn = connectionHolder.get();
        if (conn != null) conn.commit();
    }

    public void rollback() {
        Connection conn = connectionHolder.get();
        if (conn != null) {
            try { conn.rollback(); } catch (SQLException ignored) {}
        }
    }

    public void release() {
        Connection conn = connectionHolder.get();
        if (conn != null) {
            try {
                conn.close();
            } catch (SQLException e) {
                log.error("Error closing connection", e);
            } finally {
                connectionHolder.remove();  // critical — clean up!
            }
        }
    }
}
```

---

## Scenario 13: Parallel Processing with CompletableFuture

**Context:** An order fulfillment service needs to call 3 microservices in parallel and combine results:

**Question:** Implement parallel calls with a 5-second overall timeout.

**Answer:**

```java
@Service
public class FulfillmentService {
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    private final ShippingClient shippingClient;
    private final Executor executor;

    public FulfillmentResult fulfill(Order order) {
        CompletableFuture<InventoryResult> inventoryFuture =
            CompletableFuture.supplyAsync(() -> inventoryClient.check(order), executor);

        CompletableFuture<PaymentResult> paymentFuture =
            CompletableFuture.supplyAsync(() -> paymentClient.charge(order), executor);

        CompletableFuture<ShippingResult> shippingFuture =
            CompletableFuture.supplyAsync(() -> shippingClient.schedule(order), executor);

        try {
            return CompletableFuture.allOf(inventoryFuture, paymentFuture, shippingFuture)
                .orTimeout(5, TimeUnit.SECONDS)  // Java 9+
                .thenApply(v -> new FulfillmentResult(
                    inventoryFuture.join(),
                    paymentFuture.join(),
                    shippingFuture.join()
                ))
                .exceptionally(ex -> FulfillmentResult.failed(ex.getMessage()))
                .get();
        } catch (Exception e) {
            return FulfillmentResult.failed(e.getMessage());
        }
    }
}
```

---

## Scenario 14: Semaphore for Rate Limiting

**Context:** An API client must not exceed 10 concurrent requests to an external service.

**Question:** Implement using Semaphore.

**Answer:**

```java
public class RateLimitedClient {
    private final Semaphore semaphore = new Semaphore(10);  // max 10 concurrent
    private final HttpClient httpClient;

    public String call(String url) throws Exception {
        semaphore.acquire();  // blocks if 10 permits already in use
        try {
            return httpClient.send(HttpRequest.newBuilder(URI.create(url)).build(),
                HttpResponse.BodyHandlers.ofString()).body();
        } finally {
            semaphore.release();  // always release in finally
        }
    }

    // Non-blocking with timeout:
    public Optional<String> callWithTimeout(String url, long timeoutMs) throws Exception {
        if (!semaphore.tryAcquire(timeoutMs, TimeUnit.MILLISECONDS)) {
            return Optional.empty();  // couldn't acquire in time
        }
        try {
            return Optional.of(httpClient.send(...).body());
        } finally {
            semaphore.release();
        }
    }
}
```

---

## Scenario 15: CountDownLatch for Application Startup

**Context:** An application needs to wait for all components to initialize before serving requests.

**Answer:**

```java
public class ApplicationStartup {
    private final CountDownLatch startupLatch = new CountDownLatch(3);  // 3 components

    public void start() {
        initializeDatabase(startupLatch);
        initializeCache(startupLatch);
        initializeMessageQueue(startupLatch);

        try {
            boolean ready = startupLatch.await(30, TimeUnit.SECONDS);
            if (ready) {
                System.out.println("All components ready — accepting traffic");
            } else {
                System.out.println("Startup timed out — some components not ready");
                System.exit(1);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void initializeDatabase(CountDownLatch latch) {
        executor.submit(() -> {
            try {
                db.connect();
                db.runMigrations();
                System.out.println("Database ready");
            } catch (Exception e) {
                System.out.println("Database failed: " + e.getMessage());
            } finally {
                latch.countDown();  // always countdown, even on failure
            }
        });
    }
}
```

---

## Scenario 16: Atomic Reference for Lock-Free Cache

**Context:** A high-performance read-heavy cache that occasionally needs full replacement.

**Answer:**

```java
public class AtomicCache<K, V> {
    private final AtomicReference<Map<K, V>> cacheRef =
        new AtomicReference<>(Collections.emptyMap());

    public V get(K key) {
        return cacheRef.get().get(key);  // no lock needed for reads
    }

    public void refresh(Map<K, V> newData) {
        // Atomically replace entire cache
        Map<K, V> immutableCopy = Map.copyOf(newData);
        cacheRef.set(immutableCopy);  // atomic reference swap
        // Old map will be GC'd when no threads are reading it
    }

    // CAS update — only update if cache hasn't changed since we read it:
    public boolean updateIfUnchanged(Map<K, V> expected, Map<K, V> newData) {
        return cacheRef.compareAndSet(expected, Map.copyOf(newData));
    }
}
```

---

## Scenario 17: ExecutorService Shutdown Correctly

**Context:** A service that needs graceful shutdown.

**Answer:**

```java
public class WorkerService {
    private final ExecutorService executor = Executors.newFixedThreadPool(10);

    public void shutdown() {
        executor.shutdown();  // stop accepting new tasks
        try {
            // Wait up to 60 seconds for running tasks to complete
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();  // cancel running tasks
                // Wait again for interrupted tasks to respond
                if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                    log.error("ExecutorService did not terminate");
                }
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    // Register shutdown hook for JVM shutdown:
    public WorkerService() {
        Runtime.getRuntime().addShutdownHook(new Thread(this::shutdown));
    }
}
```

---

## Scenario 18: Reentrant Lock with Condition Variables

**Context:** A bounded buffer (producer-consumer) using explicit locks.

**Answer:**

```java
public class BoundedBuffer<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();   // separate from notEmpty
    private final Condition notEmpty = lock.newCondition();  // finer-grained signaling
    private final Queue<T> buffer;
    private final int capacity;

    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
        this.buffer = new ArrayDeque<>(capacity);
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (buffer.size() == capacity) {
                notFull.await();  // release lock, wait for space
            }
            buffer.add(item);
            notEmpty.signal();  // signal one consumer
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (buffer.isEmpty()) {
                notEmpty.await();  // release lock, wait for item
            }
            T item = buffer.poll();
            notFull.signal();  // signal one producer
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## Scenario 19: Thread Pool Sizing Decision

**Context:** You're deploying a service that processes uploaded files. Processing is CPU-intensive (image resizing). How do you determine thread pool size?

**Answer:**

For CPU-bound tasks, the optimal thread pool size is approximately equal to the number of available processors:

```java
// CPU-bound work — N processors → N threads
int cpuThreads = Runtime.getRuntime().availableProcessors();
ExecutorService cpuBound = Executors.newFixedThreadPool(cpuThreads);

// I/O-bound work (waiting for network/disk) — can have more threads
// Because threads block waiting for I/O, others can use the CPU
// Rule of thumb: N / (1 - blocking fraction)
// If 90% of time is blocking: 8 cores / (1-0.9) = 80 threads
ExecutorService ioBound = Executors.newFixedThreadPool(cpuThreads * 10);

// For this scenario (CPU-intensive image processing):
int processors = Runtime.getRuntime().availableProcessors();
ExecutorService imageProcessor = new ThreadPoolExecutor(
    processors,           // core pool size = num CPUs
    processors,           // max = core (fixed pool — no benefit growing for CPU-bound)
    0L, TimeUnit.MILLISECONDS,
    new LinkedBlockingQueue<>(500),   // queue up to 500 tasks
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // backpressure
);
```

---

## Scenario 20: CyclicBarrier for Batch Processing

**Context:** Monthly batch job with 4 phases: extract → transform → validate → load. Each phase must complete across all workers before the next begins.

**Answer:**

```java
public class BatchJob {
    private final int workers = 4;
    private final CyclicBarrier barrier = new CyclicBarrier(workers, () -> {
        log.info("Phase complete — all workers synchronized");
    });

    public void run(List<Record> records) {
        List<List<Record>> partitions = partition(records, workers);

        for (int i = 0; i < workers; i++) {
            final int partition = i;
            executor.submit(() -> {
                try {
                    // Phase 1: Extract
                    List<Record> extracted = extract(partitions.get(partition));
                    barrier.await();  // wait for all workers to finish extract

                    // Phase 2: Transform
                    List<Record> transformed = transform(extracted);
                    barrier.await();  // wait for all to finish transform

                    // Phase 3: Validate
                    List<Record> validated = validate(transformed);
                    barrier.await();  // wait for all to finish validate

                    // Phase 4: Load
                    load(validated);
                    barrier.await();  // all done

                } catch (Exception e) {
                    log.error("Worker {} failed", partition, e);
                }
            });
        }
    }
}
```

---

## Scenario 21: ForkJoin for Recursive Aggregation

**Context:** Aggregate statistics across a large dataset using all available CPU cores.

**Answer:**

```java
class AggregateTask extends RecursiveTask<Statistics> {
    private static final int THRESHOLD = 5000;
    private final List<Transaction> transactions;
    private final int start, end;

    @Override
    protected Statistics compute() {
        if (end - start <= THRESHOLD) {
            // Sequential computation for small enough chunk
            return computeSequentially();
        }

        int mid = (start + end) / 2;
        AggregateTask left = new AggregateTask(transactions, start, mid);
        AggregateTask right = new AggregateTask(transactions, mid, end);

        left.fork();                         // async execution
        Statistics rightStats = right.compute();  // execute right in current thread
        Statistics leftStats = left.join();   // wait for left

        return Statistics.merge(leftStats, rightStats);
    }

    private Statistics computeSequentially() {
        return transactions.subList(start, end).stream()
            .collect(Statistics.collector());
    }
}

ForkJoinPool pool = ForkJoinPool.commonPool();
Statistics result = pool.invoke(new AggregateTask(transactions, 0, transactions.size()));
```

---

## Scenario 22: Detecting and Logging Slow Tasks

**Context:** Wrap task execution to detect and alert on tasks taking more than 500ms.

**Answer:**

```java
public class TimedExecutorService extends AbstractExecutorService {
    private final ExecutorService delegate;
    private final long thresholdMs;

    @Override
    public <T> Future<T> submit(Callable<T> task) {
        Callable<T> timed = () -> {
            long start = System.currentTimeMillis();
            try {
                return task.call();
            } finally {
                long elapsed = System.currentTimeMillis() - start;
                if (elapsed > thresholdMs) {
                    log.warn("Slow task detected: {}ms > {}ms threshold. Task: {}",
                        elapsed, thresholdMs, task.getClass().getName());
                    metrics.recordSlowTask(task.getClass().getName(), elapsed);
                }
            }
        };
        return delegate.submit(timed);
    }
}
```

---

## Scenario 23: Thread Safety of SimpleDateFormat

**Context:** A legacy service has:
```java
private static final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

public String formatDate(Date date) {
    return sdf.format(date);  // not thread-safe!
}
```

**Question:** Fix this.

**Answer:**

`SimpleDateFormat` is NOT thread-safe. Multiple threads calling `format()` concurrently can produce incorrect results or throw exceptions.

**Fix options:**
```java
// Option 1 — ThreadLocal (good for high-frequency use):
private static final ThreadLocal<SimpleDateFormat> sdf =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String formatDate(Date date) {
    return sdf.get().format(date);
}

// Option 2 — DateTimeFormatter (Java 8+, thread-safe and immutable):
private static final DateTimeFormatter formatter =
    DateTimeFormatter.ofPattern("yyyy-MM-dd");

public String formatDate(LocalDate date) {
    return formatter.format(date);  // DateTimeFormatter is thread-safe
}

// Option 3 — Synchronize (poor performance):
public synchronized String formatDate(Date date) {
    return sdf.format(date);
}
```

---

## Scenario 24: Preventing OOM with Bounded Queue

**Context:** A task submission system uses unbounded queue, causing OOM under load.

**Question:** Fix by implementing backpressure.

**Answer:**

```java
// Bad — unbounded queue, tasks accumulate until OOM:
ExecutorService executor = Executors.newFixedThreadPool(10);  // uses LinkedBlockingQueue internally
// Can hold infinite tasks

// Fix — bounded queue with CallerRunsPolicy:
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4,                              // core threads
    8,                              // max threads
    60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),  // max 200 queued tasks
    Executors.defaultThreadFactory(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // submitter runs task if full
);

// CallerRunsPolicy acts as natural backpressure:
// When queue and max threads are full, the calling thread (e.g., HTTP handler)
// runs the task directly — this slows down the caller, providing backpressure
// rather than throwing an exception or losing the task.
```

---

## Scenario 25: Read-Write Lock for Config Cache

**Context:** A configuration cache read by hundreds of threads, updated every 5 minutes.

**Answer:**

```java
public class ConfigCache {
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private volatile Map<String, String> config = Collections.emptyMap();

    // Called by many threads — concurrent reads allowed
    public String get(String key) {
        lock.readLock().lock();
        try {
            return config.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }

    // Called by one thread every 5 minutes — exclusive write
    public void refresh(Map<String, String> newConfig) {
        lock.writeLock().lock();
        try {
            this.config = Map.copyOf(newConfig);  // immutable copy
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

## Scenario 26: Async Event Processing Pipeline

**Context:** Build an async event processing pipeline: receive → enrich → persist → notify.

**Answer:**

```java
public class EventPipeline {
    private final EnrichmentService enricher;
    private final EventRepository repo;
    private final NotificationService notifier;
    private final Executor executor = Executors.newFixedThreadPool(8);

    public CompletableFuture<Void> process(RawEvent event) {
        return CompletableFuture
            .supplyAsync(() -> enricher.enrich(event), executor)
            .thenApplyAsync(enriched -> repo.save(enriched), executor)
            .thenAcceptAsync(saved -> notifier.notify(saved), executor)
            .exceptionally(ex -> {
                log.error("Event processing failed for event {}", event.getId(), ex);
                deadLetterQueue.publish(event);
                return null;
            });
    }

    // Process batch:
    public CompletableFuture<Void> processBatch(List<RawEvent> events) {
        List<CompletableFuture<Void>> futures = events.stream()
            .map(this::process)
            .collect(Collectors.toList());
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
    }
}
```

---

## Scenario 27: Priority Task Execution

**Context:** Tasks have three priorities (HIGH, MEDIUM, LOW) and must be processed in priority order.

**Answer:**

```java
public class PriorityTaskExecutor {
    private final PriorityBlockingQueue<PrioritizedTask> queue =
        new PriorityBlockingQueue<>();
    private final ExecutorService workers;

    enum Priority { HIGH(0), MEDIUM(1), LOW(2);
        final int order;
        Priority(int order) { this.order = order; }
    }

    record PrioritizedTask(Runnable task, Priority priority) implements Comparable<PrioritizedTask> {
        @Override
        public int compareTo(PrioritizedTask other) {
            return Integer.compare(this.priority.order, other.priority.order);
        }
    }

    public PriorityTaskExecutor(int threads) {
        this.workers = Executors.newFixedThreadPool(threads);
        for (int i = 0; i < threads; i++) {
            workers.submit(this::workerLoop);
        }
    }

    private void workerLoop() {
        while (!Thread.interrupted()) {
            try {
                PrioritizedTask task = queue.take();  // blocks if empty
                task.task().run();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }

    public void submit(Runnable task, Priority priority) {
        queue.offer(new PrioritizedTask(task, priority));
    }
}
```

---

## Scenario 28: Detecting Thread Pool Health

**Context:** How do you monitor the health of a ThreadPoolExecutor in production?

**Answer:**

```java
@Configuration
public class ExecutorMonitoring {
    @Bean
    public ThreadPoolExecutor monitoredExecutor(MeterRegistry registry) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            8, 16, 60, TimeUnit.SECONDS, new LinkedBlockingQueue<>(1000)
        );

        // Register Micrometer gauges
        Gauge.builder("executor.pool.size", executor, ThreadPoolExecutor::getPoolSize)
            .register(registry);
        Gauge.builder("executor.active.tasks", executor, ThreadPoolExecutor::getActiveCount)
            .register(registry);
        Gauge.builder("executor.queue.size", executor, e -> e.getQueue().size())
            .register(registry);
        Gauge.builder("executor.completed.tasks", executor, ThreadPoolExecutor::getCompletedTaskCount)
            .register(registry);

        return executor;
    }
}

// Alert rules:
// - queue.size > 800 (80% full) → warning
// - active.tasks == max.pool.size → all threads busy
// - rejected tasks count > 0 → tasks being dropped
```

---

## Scenario 29: Parallel Stream Thread Safety

**Context:**

```java
List<String> results = new ArrayList<>();
Stream.of(data).parallel().forEach(item -> {
    results.add(process(item));  // ArrayList is not thread-safe!
});
```

**Question:** Fix this.

**Answer:**

`ArrayList.add()` is not thread-safe. Parallel streams use the ForkJoin common pool. Multiple threads calling `add()` concurrently leads to data corruption.

```java
// Fix 1 — Collect to list (recommended):
List<String> results = Stream.of(data)
    .parallel()
    .map(item -> process(item))
    .collect(Collectors.toList());  // collect is thread-safe

// Fix 2 — ConcurrentLinkedQueue:
ConcurrentLinkedQueue<String> results = new ConcurrentLinkedQueue<>();
Stream.of(data).parallel().forEach(item -> results.add(process(item)));

// Fix 3 — toList (Java 16+):
List<String> results = Stream.of(data).parallel().map(this::process).toList();
```

---

## Scenario 30: Preventing Reordering with Happens-Before

**Context:** A developer claims this code is thread-safe without volatile:

```java
class DataPublisher {
    private Object data;
    private boolean published = false;  // not volatile

    void publish(Object data) {
        this.data = data;
        published = true;
    }

    Object get() {
        return published ? data : null;
    }
}
```

**Question:** Is it thread-safe? What can go wrong?

**Answer:**

**Not thread-safe.** Two issues:

1. **Visibility:** Thread B may cache `published = false` and never see the update.

2. **Reordering:** JVM/CPU may reorder `this.data = data` and `published = true`. Thread B could see `published = true` but `data = null` because the assignments were reordered.

**Fix 1 — volatile on published (establishes happens-before):**
```java
private volatile boolean published = false;
// write to data before volatile write → guaranteed visible when published read
```

**Fix 2 — synchronized:**
```java
synchronized void publish(Object data) {
    this.data = data;
    published = true;
}

synchronized Object get() {
    return published ? data : null;
}
```

**Fix 3 — Use AtomicReference (clean solution):**
```java
private final AtomicReference<Object> dataRef = new AtomicReference<>();

void publish(Object data) {
    dataRef.set(data);  // single atomic publication
}

Object get() {
    return dataRef.get();
}
```
