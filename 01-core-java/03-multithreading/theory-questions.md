# Multithreading — Theory Questions

> 35 theory questions covering thread lifecycle, synchronization, locks, concurrent utilities, and the Java Memory Model.

---

## Q1: What are the states in the Java thread lifecycle?

**Answer:**

A Java thread can be in one of six states defined in `Thread.State`:

1. **NEW:** Thread created but `start()` not called yet
2. **RUNNABLE:** Thread is running or ready to run (waiting for CPU scheduler)
3. **BLOCKED:** Thread waiting to acquire a monitor lock (synchronized block/method)
4. **WAITING:** Thread waiting indefinitely until another thread signals it (`Object.wait()`, `Thread.join()`, `LockSupport.park()`)
5. **TIMED_WAITING:** Thread waiting for a specified time (`Thread.sleep(n)`, `Object.wait(n)`, `Thread.join(n)`, `LockSupport.parkNanos()`)
6. **TERMINATED:** Thread has completed execution

```
     NEW
      │
      │ start()
      ▼
  RUNNABLE ◄──────────────────────────────────────────────────┐
      │                                                        │
      │ synchronized block         │ sleep(n) / wait(n)       │
      ▼ (lock held by other)       ▼                          │
  BLOCKED                    TIMED_WAITING ─── time expires ──┘
      │ lock acquired             │
      └──────────────────────────►┘  notify() / notifyAll() / interrupt()
      │
      │ wait() / join() / park()
      ▼
   WAITING ─── notified/joined/unparked ──► RUNNABLE
      │
      │ run() completes
      ▼
  TERMINATED
```

---

## Q2: What is the difference between `Runnable` and `Thread`?

**Answer:**

Both allow defining task code to run in a thread, but with different approaches:

**Using Thread (extend):**
```java
class MyTask extends Thread {
    @Override
    public void run() {
        System.out.println("Running in " + Thread.currentThread().getName());
    }
}
new MyTask().start();
```

**Using Runnable (implement):**
```java
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Running in " + Thread.currentThread().getName());
    }
}
new Thread(new MyTask()).start();
// Or with lambda:
new Thread(() -> System.out.println("Lambda task")).start();
```

**Prefer Runnable because:**
1. Java allows only single inheritance — using Runnable keeps the class free to extend others
2. Runnable separates the task (what to do) from the execution mechanism (how to run it)
3. Runnable tasks can be submitted to ExecutorService — Thread cannot
4. Functional interface — works with lambdas
5. Better OOP — a task IS-NOT a thread, it HAS-A thread (composition over inheritance)

---

## Q3: What is `synchronized` and what does it guarantee?

**Answer:**

`synchronized` provides mutual exclusion (only one thread executes synchronized code at a time) and visibility (changes made by one thread are visible to the next thread that acquires the same lock).

**On a method:**
```java
public synchronized void transfer(int amount) {
    // Only one thread can execute this at a time per instance
    this.balance -= amount;
}
```
Equivalent to acquiring `this` as the lock.

**On a static method:**
```java
public static synchronized void init() { ... }
```
Acquires the `Class` object lock — all instances share it.

**On a block (preferred for finer control):**
```java
private final Object lock = new Object();

public void transfer(int amount) {
    synchronized (lock) {
        this.balance -= amount;
    }
    // Other non-critical work outside lock
}
```

**What synchronized guarantees:**
1. **Atomicity:** The synchronized block runs without interruption from other threads on the same lock
2. **Visibility:** All changes made inside synchronized block are visible to threads that subsequently acquire the same lock
3. **Ordering:** No reordering within the synchronized block across threads

**What synchronized does NOT guarantee:**
- Performance — blocking all threads except one
- Fairness — threads don't get turns in order of waiting

---

## Q4: What is the `volatile` keyword in multithreading?

**Answer:**

`volatile` ensures that reads and writes to a variable go directly to main memory, not a thread's CPU cache. This provides:

1. **Visibility:** A write by Thread A to a volatile variable is immediately visible to Thread B when it reads it
2. **Happens-before guarantee:** All writes before a volatile write are visible to the thread doing the volatile read
3. **Prevents instruction reordering** around the volatile access

```java
class StopTask {
    private volatile boolean stopped = false;

    void stop() {
        stopped = true;  // immediately visible to other threads
    }

    void run() {
        while (!stopped) {  // reads fresh value from main memory each time
            doWork();
        }
    }
}
```

**What volatile does NOT provide:**
- **Atomicity for compound operations:** `volatile int count; count++` is still NOT atomic (read-modify-write is three operations)
- Use `AtomicInteger` for atomic increment

**When to use volatile:**
- Simple flag variables (stop flags, initialization flags)
- Status indicators updated by one thread, read by many
- Double-checked locking (see Singleton pattern)

---

## Q5: What is a deadlock? How do you detect and prevent it?

**Answer:**

A deadlock occurs when two or more threads are each waiting for a lock held by the other, creating a circular wait.

**Classic example:**
```java
Object lockA = new Object();
Object lockB = new Object();

// Thread 1                    // Thread 2
synchronized (lockA) {         synchronized (lockB) {
    Thread.sleep(100);             Thread.sleep(100);
    synchronized (lockB) {         synchronized (lockA) {
        // DEADLOCK                    // DEADLOCK
    }                              }
}                              }
```

**Four conditions for deadlock (Coffman conditions):**
1. **Mutual exclusion:** Resources cannot be shared
2. **Hold and wait:** Thread holds one resource while waiting for another
3. **No preemption:** Resources cannot be forcibly taken
4. **Circular wait:** Thread A waits for B, B waits for A

**Prevention strategies:**
1. **Lock ordering:** Always acquire locks in the same order across all threads
```java
// Always lock A before B — consistently
synchronized (lockA) {
    synchronized (lockB) {
        // safe — consistent ordering
    }
}
```

2. **Lock timeout:** Use `ReentrantLock.tryLock(timeout)` instead of blocking indefinitely
```java
if (lockA.tryLock(1, TimeUnit.SECONDS) && lockB.tryLock(1, TimeUnit.SECONDS)) {
    try { doWork(); }
    finally { lockA.unlock(); lockB.unlock(); }
}
```

3. **Avoid nested locks:** Design systems so you never hold one lock while acquiring another

**Detection:** `jstack <PID>` generates a thread dump showing "DEADLOCK DETECTED" with the involved threads and locks.

---

## Q6: What is the difference between `wait()`, `sleep()`, and `yield()`?

**Answer:**

| | `wait()` | `sleep()` | `yield()` |
|---|----------|-----------|-----------|
| Defined in | `Object` | `Thread` | `Thread` |
| Must hold lock? | Yes | No | No |
| Releases lock? | Yes | No | No |
| Can be interrupted? | Yes | Yes | N/A |
| Woken by | `notify()`/`notifyAll()` | Timeout expiry | Hint to scheduler |

```java
// wait() — must be in synchronized block
synchronized (obj) {
    while (condition) {
        obj.wait();  // releases lock, waits until notified
    }
    // do work with condition satisfied
}

// sleep() — pauses thread without releasing lock
Thread.sleep(1000);  // pauses for 1 second

// yield() — hints scheduler to give up CPU
Thread.yield();  // may or may not actually yield
```

**Critical:** `wait()` must always be called in a loop (not an if-statement) because of **spurious wakeups** — the JVM may wake a waiting thread without notification.

---

## Q7: What is an ExecutorService and why use it over raw threads?

**Answer:**

`ExecutorService` is a thread pool abstraction that decouples task submission from thread lifecycle management.

```java
// Raw threads — poor man's concurrency
Thread t1 = new Thread(task1);
Thread t2 = new Thread(task2);
t1.start();
t2.start();
t1.join();
t2.join();
// Problems: creating threads is expensive, no limit on thread count,
// no result return, no exception handling, no lifecycle control

// ExecutorService — proper way
ExecutorService executor = Executors.newFixedThreadPool(10);

Future<String> future1 = executor.submit(() -> task1());
Future<String> future2 = executor.submit(() -> task2());

String result1 = future1.get(5, TimeUnit.SECONDS);  // with timeout
String result2 = future2.get(5, TimeUnit.SECONDS);

executor.shutdown();
executor.awaitTermination(60, TimeUnit.SECONDS);
```

**Common thread pool types:**
```java
// Fixed pool — N threads, unlimited queue
Executors.newFixedThreadPool(4);

// Cached pool — grows/shrinks, 60s idle timeout, SynchronousQueue
Executors.newCachedThreadPool();

// Single thread — guarantees sequential execution
Executors.newSingleThreadExecutor();

// Scheduled — for periodic/delayed tasks
ScheduledExecutorService scheduled = Executors.newScheduledThreadPool(2);
scheduled.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);

// Custom — best for production
ThreadPoolExecutor custom = new ThreadPoolExecutor(
    4,                          // core pool size
    10,                         // max pool size
    60L, TimeUnit.SECONDS,      // idle thread keepalive
    new LinkedBlockingQueue<>(100),  // bounded work queue
    new ThreadPoolExecutor.CallerRunsPolicy()  // rejection policy
);
```

---

## Q8: What is CompletableFuture? How does it differ from Future?

**Answer:**

`Future<T>` (Java 5) represents an async computation result but has limitations: you can only block and wait for it (`get()`), cannot chain operations, cannot handle exceptions cleanly.

`CompletableFuture<T>` (Java 8) adds: non-blocking callbacks, chaining, combining multiple futures, exception handling.

```java
// Future — must block to get result
Future<String> f = executor.submit(() -> fetchData());
String result = f.get();  // blocks until done

// CompletableFuture — non-blocking chains
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> fetchData())               // runs async
    .thenApply(data -> process(data))             // transform result
    .thenApply(processed -> format(processed))   // chain another transform
    .exceptionally(e -> "fallback")              // handle exceptions
    .whenComplete((result, ex) -> log(result));  // side effect

// Combining multiple futures:
CompletableFuture<String> userFuture = CompletableFuture.supplyAsync(() -> fetchUser());
CompletableFuture<String> orderFuture = CompletableFuture.supplyAsync(() -> fetchOrders());

CompletableFuture<String> combined = userFuture.thenCombine(orderFuture,
    (user, orders) -> user + " has orders: " + orders);

// Wait for all:
CompletableFuture.allOf(cf1, cf2, cf3).thenRun(() -> System.out.println("All done"));

// Wait for first:
CompletableFuture.anyOf(cf1, cf2, cf3).thenAccept(result -> System.out.println("First: " + result));
```

**thenApply vs thenCompose:**
- `thenApply(Function<T,U>)` — maps result to another value (like Stream.map)
- `thenCompose(Function<T,CompletableFuture<U>>)` — flatMap — for when the function itself returns a future (avoids nested `CompletableFuture<CompletableFuture<T>>`)

---

## Q9: What is the Java Memory Model (JMM) and the happens-before relationship?

**Answer:**

The Java Memory Model defines the rules for how threads interact through memory and what values can be observed. Without it, the JVM and CPU are free to reorder operations for performance.

**Happens-before (HB) relationship:**
If action A happens-before action B, then all effects of A are visible to B. HB is established by:
1. **Program order:** Within a single thread, each action HB the next one
2. **Monitor lock:** An `unlock()` HB every subsequent `lock()` on the same monitor
3. **Volatile variable:** A write to volatile HB every subsequent read of that variable
4. **Thread start:** `Thread.start()` HB any action in the started thread
5. **Thread join:** All actions in thread T HB `T.join()` returning
6. **Transitivity:** if A HB B and B HB C, then A HB C

```java
int x = 0;
volatile boolean ready = false;

// Thread A:
x = 42;
ready = true;  // volatile write

// Thread B:
if (ready) {  // volatile read — HB guaranteed
    System.out.println(x);  // guaranteed to see 42
}
```

Without `volatile` on `ready`, Thread B might see `ready = true` but `x = 0` due to reordering.

---

## Q10: What are CountDownLatch, CyclicBarrier, and Semaphore?

**Answer:**

**CountDownLatch:** One or more threads wait until a countdown reaches zero. One-shot — cannot be reset.
```java
CountDownLatch latch = new CountDownLatch(3);

// 3 worker threads, each call latch.countDown() when done
for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        doWork();
        latch.countDown();  // decrements from 3 → 0
    });
}

latch.await();  // main thread waits until count = 0
System.out.println("All 3 workers done");
```

**CyclicBarrier:** Multiple threads wait for each other at a barrier point. Reusable — can be reset.
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("All at barrier"));

// Each thread calls await() — waits until all 3 reach the barrier
executor.submit(() -> {
    doPhase1();
    barrier.await();  // waits for 2 others
    doPhase2();
    barrier.await();  // waits for 2 others again (reusable!)
});
```

**Semaphore:** Controls access to a resource with N permits.
```java
Semaphore semaphore = new Semaphore(5);  // max 5 concurrent connections

// Each thread must acquire a permit before accessing resource
semaphore.acquire();
try {
    useResource();
} finally {
    semaphore.release();
}
```

---

## Q11: What is ReentrantLock and when should you use it over `synchronized`?

**Answer:**

`ReentrantLock` is an explicit lock that provides more features than `synchronized`:

```java
ReentrantLock lock = new ReentrantLock();

// Basic usage (equivalent to synchronized):
lock.lock();
try {
    // critical section
} finally {
    lock.unlock();  // must be in finally to avoid lock leak!
}

// tryLock — non-blocking attempt:
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { doWork(); }
    finally { lock.unlock(); }
} else {
    handleLockNotAcquired();
}

// Interruptible lock acquisition:
lock.lockInterruptibly();  // throws InterruptedException if thread interrupted while waiting

// Fairness:
ReentrantLock fairLock = new ReentrantLock(true);  // FIFO order for waiting threads

// Condition variables (more flexible than wait/notify):
Condition notEmpty = lock.newCondition();
Condition notFull = lock.newCondition();

// Producer signals notEmpty, consumer waits on notEmpty
notEmpty.signal();
notEmpty.await();
```

**Use ReentrantLock when:**
- You need `tryLock()` (non-blocking)
- You need interruptible lock acquisition
- You need multiple `Condition` variables per lock
- You need fair locking
- You need to query lock state (`isLocked()`, `getQueueLength()`)

**Use `synchronized` when:**
- Simple mutual exclusion
- No advanced features needed
- Code clarity is more important
- JVM can optimize `synchronized` better in some cases

---

## Q12: What is the difference between livelock, deadlock, and starvation?

**Answer:**

**Deadlock:** Threads are blocked forever, each waiting for a resource held by the other.
```
Thread A: holds L1, waiting for L2
Thread B: holds L2, waiting for L1
→ Neither can proceed
```

**Livelock:** Threads are not blocked but are too busy responding to each other to make progress.
```
Thread A: detects conflict, backs off
Thread B: detects conflict, backs off
Thread A: retries → detects conflict → backs off
Thread B: retries → detects conflict → backs off
→ Both active but neither progressing
```
(Like two people in a hallway, each stepping aside to let the other pass, then both stepping back.)

**Starvation:** A thread is perpetually denied access to a resource because other threads are always favored.
```
High-priority threads always get the lock
Low-priority thread waits indefinitely
```

**Solutions:**
- Deadlock: Lock ordering, tryLock with timeout, avoid nested locks
- Livelock: Add randomness to retry delays, use ordered resource acquisition
- Starvation: Use fair locks (`new ReentrantLock(true)`), use `LinkedBlockingQueue` with priority

---

## Q13: What is thread-local storage (`ThreadLocal`)?

**Answer:**

`ThreadLocal<T>` provides each thread its own independent copy of a variable. Common use cases: per-thread state without synchronization, user session in web applications, database connections, formatting objects.

```java
// Not thread-safe — shared SimpleDateFormat
static SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");

// Thread-safe via ThreadLocal — each thread gets its own instance
static ThreadLocal<SimpleDateFormat> sdfLocal =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

public String formatDate(Date date) {
    return sdfLocal.get().format(date);  // no synchronization needed
}

// Always clean up in web/thread-pool contexts to avoid memory leaks:
try {
    return sdfLocal.get().format(date);
} finally {
    sdfLocal.remove();  // critical in thread pools — threads are reused!
}
```

**Memory leak warning:** In thread pool scenarios (Tomcat, Spring), threads are reused. If you put objects in ThreadLocal and don't call `remove()`, the objects persist for the lifetime of the thread (potentially forever), causing memory leaks and data leakage across requests.

---

## Q14: What are atomic classes in `java.util.concurrent.atomic`?

**Answer:**

Atomic classes provide lock-free, thread-safe operations on single variables using CAS (Compare-And-Swap) CPU instructions, which are more efficient than `synchronized` for simple counters and flags.

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();          // thread-safe ++
counter.getAndIncrement();          // returns old value, then ++
counter.compareAndSet(5, 10);       // if current == 5, set to 10 (CAS)
counter.addAndGet(5);               // thread-safe += 5

AtomicBoolean flag = new AtomicBoolean(false);
flag.compareAndSet(false, true);   // if false, set to true atomically

AtomicLong total = new AtomicLong(0);
total.accumulateAndGet(100, Long::sum);  // apply accumulator function

// AtomicReference for object references:
AtomicReference<String> atomicRef = new AtomicReference<>("initial");
atomicRef.compareAndSet("initial", "updated");  // CAS on reference

// For high-contention counters, LongAdder is better than AtomicLong:
LongAdder adder = new LongAdder();
adder.increment();    // lower contention — uses multiple cells
adder.sum();          // returns sum
```

---

## Q15: What is the Fork/Join framework?

**Answer:**

The Fork/Join framework (Java 7) is designed for divide-and-conquer parallel algorithms. It splits a large task into smaller tasks (fork), executes them in parallel, then combines results (join). Uses work-stealing to balance load across threads.

```java
class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 1000;
    private final long[] array;
    private final int start, end;

    SumTask(long[] array, int start, int end) {
        this.array = array; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // Base case — compute directly
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        // Divide
        int mid = (start + end) / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        left.fork();                // submit left to pool
        long rightResult = right.compute();  // compute right in current thread
        long leftResult = left.join();       // wait for left result
        return leftResult + rightResult;
    }
}

ForkJoinPool pool = ForkJoinPool.commonPool();
long sum = pool.invoke(new SumTask(array, 0, array.length));
```

---

## Q16: What is the difference between `notify()` and `notifyAll()`?

**Answer:**

Both wake up threads waiting on an object's monitor (via `wait()`).

- **`notify()`:** Wakes exactly one waiting thread (chosen arbitrarily by the JVM). Other waiting threads remain waiting.
- **`notifyAll()`:** Wakes ALL waiting threads. They all compete for the lock, but only one proceeds at a time.

**When to use which:**
- Use `notifyAll()` as the safe default — `notify()` can cause starvation (the same thread might always be picked, others wait forever)
- Use `notify()` only when: there is exactly one waiting thread, or all waiting threads are functionally identical (waiting for the same condition)

```java
// Producer-Consumer with notifyAll:
synchronized (queue) {
    queue.add(item);
    queue.notifyAll();  // wake all consumers
}

// Consumer:
synchronized (queue) {
    while (queue.isEmpty()) {
        queue.wait();  // always in a loop!
    }
    return queue.remove();
}
```

---

## Q17: What is a race condition and how do you prevent it?

**Answer:**

A race condition occurs when the program's behavior depends on the relative timing of thread execution. The "check-then-act" pattern is the most common form:

```java
// Race condition — check-then-act is not atomic
if (balance >= amount) {      // Thread A checks: balance = 100, amount = 100 → true
    balance -= amount;        // Thread B also checks at same time → also true
}                             // Both deduct: balance goes negative!

// Fix 1 — synchronized:
synchronized void withdraw(int amount) {
    if (balance >= amount) {
        balance -= amount;
    }
}

// Fix 2 — AtomicInteger with CAS loop:
AtomicInteger atomicBalance = new AtomicInteger(100);
boolean withdrawn = false;
while (!withdrawn) {
    int current = atomicBalance.get();
    if (current >= amount) {
        withdrawn = atomicBalance.compareAndSet(current, current - amount);
    } else {
        break;  // insufficient funds
    }
}

// Fix 3 — Database-level: use transactions with proper isolation
```

---

## Q18: What is a thread dump and what information does it provide?

**Answer:**

A thread dump is a snapshot of all threads in a JVM at a specific moment, showing each thread's state, stack trace, and any locks held or waited for.

**How to generate:**
```bash
# Method 1 — jstack
jstack <PID> > thread-dump.txt

# Method 2 — kill signal (Unix)
kill -3 <PID>

# Method 3 — via JMX
jconsole  # GUI tool

# Method 4 — programmatically
Map<Thread, StackTraceElement[]> threads = Thread.getAllStackTraces();
```

**Thread dump reading guide:**
```
"http-nio-8080-exec-1" #17 daemon prio=5 os_prio=0 tid=0x... nid=0x... waiting on condition
   java.lang.Thread.State: WAITING (parking)
        at sun.misc.Unsafe.park(Native Method)
        - parking to wait for  <0x0000000712a3f440> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
        at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
        at com.example.OrderService.processOrder(OrderService.java:42)
```

Useful for diagnosing: deadlocks, thread pool exhaustion, CPU-spinning threads, blocked threads.

---

## Q19: What is the `synchronized` visibility guarantee?

**Answer:**

When a thread exits a `synchronized` block, it flushes all changes to main memory. When a thread enters a `synchronized` block, it reads fresh values from main memory. This means:

1. All writes made inside a synchronized block are visible to the next thread that acquires the same lock
2. This is the same guarantee as `volatile`, but also adds mutual exclusion

```java
class SharedData {
    private int value = 0;
    private final Object lock = new Object();

    void set(int v) {
        synchronized (lock) {
            this.value = v;  // flushed to main memory on exit
        }
    }

    int get() {
        synchronized (lock) {
            return value;    // reads from main memory on entry
        }
    }
}
```

---

## Q20: What is the ThreadPoolExecutor rejection policy?

**Answer:**

When a thread pool cannot accept a new task (queue full and max threads reached), the rejection policy determines what happens:

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    4, 8, 60, TimeUnit.SECONDS, new LinkedBlockingQueue<>(100),
    new ThreadPoolExecutor.AbortPolicy()  // default
);
```

**Four built-in rejection policies:**

1. **AbortPolicy (default):** Throws `RejectedExecutionException`
2. **CallerRunsPolicy:** The submitting thread runs the task itself (natural backpressure)
3. **DiscardPolicy:** Silently drops the new task
4. **DiscardOldestPolicy:** Drops the oldest queued task and retries submission

```java
// Custom rejection policy:
executor.setRejectedExecutionHandler((task, executor) -> {
    if (!executor.isShutdown()) {
        log.warn("Task rejected, queuing to fallback");
        fallbackQueue.offer(task);
    }
});
```

**CallerRunsPolicy is often the best choice** for web applications — it slows down the caller naturally, providing backpressure without losing tasks.

---

## Q21: What is the difference between `submit()` and `execute()` in ExecutorService?

**Answer:**

- **`execute(Runnable)`:** Fire-and-forget. No return value. Exceptions are uncaught (handled by UncaughtExceptionHandler or lost).
- **`submit(Callable<T>)` or `submit(Runnable)`:** Returns a `Future<T>`. Exceptions are captured and rethrown when `future.get()` is called.

```java
// execute — no result, exceptions swallowed
executor.execute(() -> {
    throw new RuntimeException("ignored!");  // UncaughtExceptionHandler handles it
});

// submit — Future captures result and exceptions
Future<String> f = executor.submit(() -> {
    throw new RuntimeException("captured!");
    return "result";
});

try {
    String result = f.get();
} catch (ExecutionException e) {
    Throwable cause = e.getCause();  // "captured!" is here
    log.error("Task failed", cause);
}
```

**Important:** If you use `submit()` but never call `get()`, exceptions are silently swallowed. Always handle `Future.get()` exceptions.

---

## Q22: What are `ReadWriteLock` and `StampedLock`?

**Answer:**

**ReadWriteLock:** Multiple threads can read concurrently, but write requires exclusive access.

```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();
Lock readLock = rwLock.readLock();
Lock writeLock = rwLock.writeLock();

// Multiple readers can hold readLock simultaneously
readLock.lock();
try { return data; }
finally { readLock.unlock(); }

// Writer gets exclusive access
writeLock.lock();
try { data = newValue; }
finally { writeLock.unlock(); }
```

**StampedLock (Java 8):** More flexible — supports optimistic reads (no lock acquisition for read, just validate afterward).

```java
StampedLock lock = new StampedLock();

// Optimistic read — no lock acquisition
long stamp = lock.tryOptimisticRead();
double result = computeFromData();   // read data
if (!lock.validate(stamp)) {         // if data changed, fall back to read lock
    stamp = lock.readLock();
    try { result = computeFromData(); }
    finally { lock.unlockRead(stamp); }
}
```

**Use case:** High read-heavy workloads (caches, read-mostly data structures) where ReadWriteLock would have too much contention on the lock itself.

---

## Q23: What is the `Exchanger` class?

**Answer:**

`Exchanger<V>` allows two threads to swap objects at a synchronization point.

```java
Exchanger<List<String>> exchanger = new Exchanger<>();

// Thread A (producer):
List<String> fullBuffer = new ArrayList<>();
// ... fill the buffer
List<String> emptyBuffer = exchanger.exchange(fullBuffer);  // blocks until Thread B arrives
// Now Thread A has empty buffer

// Thread B (consumer):
List<String> emptyBuffer = new ArrayList<>();
List<String> fullBuffer = exchanger.exchange(emptyBuffer);  // blocks until Thread A arrives
// Now Thread B has full buffer to consume
```

---

## Q24: What is the `Phaser` class?

**Answer:**

`Phaser` is a flexible barrier that supports multiple phases and dynamic party registration. More flexible than CyclicBarrier.

```java
Phaser phaser = new Phaser(3);  // 3 parties

for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        doPhase1();
        phaser.arriveAndAwaitAdvance();  // barrier — wait for all phase 1 completions

        doPhase2();
        phaser.arriveAndAwaitAdvance();  // barrier for phase 2

        doPhase3();
        phaser.arriveAndDeregister();    // this party won't participate in future phases
    });
}
```

---

## Q25: What is the `java.util.concurrent.locks` package, and what advantages do explicit locks offer?

**Answer:**

The `java.util.concurrent.locks` package provides:
- `Lock` interface: `lock()`, `unlock()`, `tryLock()`, `lockInterruptibly()`
- `ReentrantLock`: Reentrant implementation
- `ReadWriteLock` / `ReentrantReadWriteLock`
- `StampedLock`
- `Condition`: More flexible `wait()`/`notify()` replacement

**Advantages over `synchronized`:**
1. **Interruptible:** `lockInterruptibly()` allows thread to be interrupted while waiting
2. **Timed:** `tryLock(time, unit)` — give up if lock not acquired in time
3. **Non-blocking:** `tryLock()` returns immediately without blocking
4. **Multiple conditions:** Per-lock condition variables (multiple `wait/notify` queues)
5. **Fairness:** `new ReentrantLock(true)` guarantees FIFO ordering
6. **Lock monitoring:** `getQueueLength()`, `isLocked()`, `isHeldByCurrentThread()`

---

## Q26: What is false sharing in multithreading and how do you prevent it?

**Answer:**

Modern CPUs cache memory in cache lines (~64 bytes). False sharing occurs when two threads access different variables that happen to be in the same cache line. When one thread modifies its variable, the entire cache line is invalidated on all CPUs, forcing the other thread to reload — even though it's accessing a different variable.

```java
// False sharing — x and y likely in same cache line:
class Counters {
    volatile long x;  // Thread A writes x
    volatile long y;  // Thread B writes y
    // x and y are adjacent in memory → same cache line → false sharing!
}

// Fix 1 — padding:
class Counters {
    volatile long x;
    long p1, p2, p3, p4, p5, p6, p7;  // 56 bytes of padding
    volatile long y;  // now in different cache line
}

// Fix 2 — @Contended annotation (Java 8+):
class Counters {
    @sun.misc.Contended volatile long x;  // JVM adds padding automatically
    @sun.misc.Contended volatile long y;
}
// Requires -XX:-RestrictContended JVM flag

// Fix 3 — LongAdder (designed to avoid false sharing):
LongAdder counterA = new LongAdder();
LongAdder counterB = new LongAdder();
```

---

## Q27: What is the difference between `Thread.interrupt()` and `Thread.stop()`?

**Answer:**

`Thread.stop()` is **deprecated** and dangerous — it abruptly terminates a thread by throwing `ThreadDeath`, potentially leaving shared data in an inconsistent state (locks not released, transactions not rolled back).

`Thread.interrupt()` is the **cooperative** approach — it sets the thread's interrupted status, which the thread can check and handle gracefully.

```java
// Cooperative interruption pattern:
class Worker implements Runnable {
    @Override
    public void run() {
        while (!Thread.currentThread().isInterrupted()) {
            try {
                doWork();
                Thread.sleep(100);  // throws InterruptedException when interrupted
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();  // restore interrupted status
                break;  // or log and exit loop
            }
        }
        cleanup();  // can run cleanup safely
    }
}

Worker worker = new Worker();
Thread thread = new Thread(worker);
thread.start();
// ...
thread.interrupt();  // signal to stop
thread.join();       // wait for graceful shutdown
```

---

## Q28: What is `FutureTask`?

**Answer:**

`FutureTask<V>` implements both `Runnable` and `Future<V>`. It can be submitted to an executor or run directly in a thread, while still allowing `get()` to retrieve the result.

```java
Callable<String> task = () -> {
    Thread.sleep(1000);
    return "result";
};

FutureTask<String> futureTask = new FutureTask<>(task);

Thread thread = new Thread(futureTask);  // can run in plain Thread
thread.start();

// Or submit to executor:
executor.submit(futureTask);

String result = futureTask.get();  // blocks until complete
boolean isDone = futureTask.isDone();
futureTask.cancel(true);  // interrupt if running
```

---

## Q29: What is the `java.util.concurrent.atomic.LongAdder` and when is it better than `AtomicLong`?

**Answer:**

`LongAdder` and `LongAccumulator` are designed for high-contention counters. Under high contention, `AtomicLong.incrementAndGet()` causes threads to spin on CAS failures. `LongAdder` uses an array of cells — each thread updates its own cell, reducing contention. The final sum is computed by summing all cells.

```java
// AtomicLong — OK for low contention
AtomicLong atomicCounter = new AtomicLong(0);
atomicCounter.incrementAndGet();

// LongAdder — better for high-contention counting
LongAdder adder = new LongAdder();
adder.increment();          // low contention — uses thread-local cells
long total = adder.sum();   // sums all cells

// LongAdder vs AtomicLong tradeoffs:
// LongAdder: higher throughput under contention, but sum() is not atomic
// AtomicLong: lower throughput under contention, but get() is always current
```

---

## Q30: What is the `java.util.concurrent.BlockingQueue` and its implementations?

**Answer:**

`BlockingQueue` is a thread-safe queue that blocks producers when full and blocks consumers when empty.

```java
// ArrayBlockingQueue — bounded, array-backed, single lock
BlockingQueue<String> bounded = new ArrayBlockingQueue<>(100);

// LinkedBlockingQueue — optionally bounded, two locks (higher throughput)
BlockingQueue<String> linked = new LinkedBlockingQueue<>(1000);

// PriorityBlockingQueue — unbounded, priority-ordered (elements must be Comparable)
BlockingQueue<Task> priority = new PriorityBlockingQueue<>();

// SynchronousQueue — zero capacity — each put() blocks until a take() happens
BlockingQueue<String> sync = new SynchronousQueue<>();  // used in Executors.newCachedThreadPool()

// DelayQueue — elements only available after delay
BlockingQueue<DelayedTask> delayed = new DelayQueue<>();

// Usage pattern:
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

// Producer (blocks if full):
queue.put("task");            // blocks
queue.offer("task", 1, SECONDS); // times out

// Consumer (blocks if empty):
String task = queue.take();   // blocks
String task = queue.poll(1, SECONDS);  // times out
```

---

## Q31: What is thread starvation and how does a fair lock help?

**Answer:**

Thread starvation occurs when a thread is perpetually unable to acquire a resource because other threads are always preferred. This can happen with non-fair locks where any waiting thread can be chosen.

```java
// Non-fair lock (default) — any waiting thread may be selected:
ReentrantLock nonFair = new ReentrantLock(false);
// High-priority threads may keep getting the lock, low-priority threads wait forever

// Fair lock — FIFO order:
ReentrantLock fairLock = new ReentrantLock(true);
// Threads acquire the lock in the order they requested it
// Prevents starvation but reduces throughput (lock hand-off overhead)
```

**Fair lock drawbacks:** Lower throughput because threads must wait in order even if the next thread in queue hasn't been scheduled by the OS yet.

---

## Q32: What is the producer-consumer pattern and how is it implemented?

**Answer:**

```java
class ProducerConsumer {
    private final BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

    class Producer implements Runnable {
        @Override public void run() {
            try {
                while (!Thread.interrupted()) {
                    String item = produce();
                    queue.put(item);  // blocks if queue is full
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    class Consumer implements Runnable {
        @Override public void run() {
            try {
                while (!Thread.interrupted()) {
                    String item = queue.take();  // blocks if queue is empty
                    consume(item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

---

## Q33: What happens if an exception is thrown in a thread?

**Answer:**

Uncaught exceptions in threads do not propagate to the parent thread. They invoke the thread's `UncaughtExceptionHandler`, or the default handler if none is set.

```java
// Set per-thread handler:
Thread t = new Thread(() -> { throw new RuntimeException("oops"); });
t.setUncaughtExceptionHandler((thread, exception) -> {
    log.error("Thread {} threw exception", thread.getName(), exception);
    // alert, metrics, etc.
});
t.start();

// Set default handler for all threads:
Thread.setDefaultUncaughtExceptionHandler((thread, exception) -> {
    log.error("Unhandled exception in thread {}", thread.getName(), exception);
});

// With ExecutorService — exceptions captured in Future:
Future<?> f = executor.submit(() -> { throw new RuntimeException("task failed"); });
try {
    f.get();  // throws ExecutionException
} catch (ExecutionException e) {
    log.error("Task failed", e.getCause());
}
```

---

## Q34: What is the `synchronized` reentrant property?

**Answer:**

`synchronized` in Java is **reentrant** — a thread that already holds a lock can re-acquire it without blocking itself. The JVM tracks the thread that holds each monitor and a count of how many times it has been acquired.

```java
class ReentrantExample {
    synchronized void methodA() {
        methodB();  // calls another synchronized method on same object
    }

    synchronized void methodB() {
        // This works! Same thread already holds 'this' lock
        // Lock count incremented: 1 → 2
    }
    // Lock count decremented on each return: 2 → 1 → 0 (released)
}
```

Without reentrancy, `methodA` calling `methodB` would deadlock — the thread already holds the lock and would wait forever to acquire it again.

---

## Q35: What is `ScheduledExecutorService` and how is it different from `Timer`?

**Answer:**

Both schedule tasks for future or repeated execution, but `ScheduledExecutorService` is strictly superior:

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// One-time delayed task:
scheduler.schedule(() -> sendReminder(), 10, TimeUnit.MINUTES);

// Fixed rate — fires every N milliseconds, regardless of task duration
// If task takes longer than period, tasks may overlap
scheduler.scheduleAtFixedRate(() -> collectMetrics(), 0, 30, TimeUnit.SECONDS);

// Fixed delay — N milliseconds after previous task completes
// Prevents overlap — next run starts N ms after last run finishes
scheduler.scheduleWithFixedDelay(() -> pollDatabase(), 0, 5, TimeUnit.SECONDS);
```

**Why Timer is obsolete:**
1. `Timer` uses a single thread — one slow/failing task blocks all others
2. `Timer` exceptions kill the timer thread permanently — all future tasks silently not run
3. `ScheduledExecutorService` uses a thread pool, isolates task failures, continues after exceptions
4. Better integration with modern concurrency APIs
