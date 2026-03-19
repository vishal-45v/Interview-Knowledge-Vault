# Multithreading — Follow-Up Trap Questions

---

## Trap 1: "You said synchronized provides atomicity. Does it work across JVMs?"
**Answer:** No. `synchronized` uses JVM-level monitors which are per-JVM. In a distributed system (multiple JVMs), you need distributed locks: Redis SETNX, ZooKeeper, database-level SELECT FOR UPDATE, or distributed lock libraries like Redisson.

## Trap 2: "Can a thread hold two locks simultaneously?"
**Answer:** Yes, and this is exactly how nested synchronized blocks work. The risk is deadlock if another thread acquires the locks in a different order. ReentrantLock's `tryLock()` makes this safer.

## Trap 3: "What happens to the stack if a thread is interrupted while sleeping?"
**Answer:** `Thread.sleep()` throws `InterruptedException` and clears the interrupted flag. The `catch` block should either re-interrupt (`Thread.currentThread().interrupt()`) or re-throw. Swallowing InterruptedException is a common mistake that causes threads to never stop.

## Trap 4: "Is volatile enough to implement a thread-safe counter?"
**Answer:** No. `volatile int count; count++` is still a race condition. `volatile` provides visibility and prevents reordering but does NOT provide atomicity for compound operations (read-modify-write). Use `AtomicInteger` or `synchronized`.

## Trap 5: "What is the difference between the ForkJoin common pool and a custom pool?"
**Answer:** `ForkJoinPool.commonPool()` is a shared JVM-level pool. All parallel streams and CompletableFuture without explicit executor use it. Blocking tasks in the common pool can starve other parts of the application. For blocking I/O, use a dedicated pool. Custom pools also allow tuning parallelism level.

## Trap 6: "If CompletableFuture.supplyAsync() has no executor, which thread runs it?"
**Answer:** The ForkJoin common pool by default. If all common pool threads are busy, it may execute in the calling thread. For production code, always provide an explicit executor to control which thread pool is used.

## Trap 7: "Can a daemon thread prevent JVM shutdown?"
**Answer:** No — daemon threads do NOT prevent JVM shutdown. When all non-daemon threads complete, the JVM exits and kills all daemon threads, even if they're in the middle of execution. Use daemon threads for background housekeeping (GC, logging) but not for critical work.

## Trap 8: "What does ConcurrentHashMap.computeIfAbsent() guarantee?"
**Answer:** It guarantees that the function is called at most once for each key, and the put is atomic. However, the function itself must not modify the map (deadlock risk). In Java 8, `computeIfAbsent` on ConcurrentHashMap can deadlock if the mapping function tries to update the same map. Fixed in Java 9.

## Trap 9: "What is the difference between synchronized(this) and synchronized(MyClass.class)?"
**Answer:** `synchronized(this)` acquires the instance monitor — different instances don't block each other. `synchronized(MyClass.class)` acquires the class-level monitor — all instances are blocked for the duration. Use class-level lock for protecting static state; instance-level for instance state.

## Trap 10: "What is the memory visibility guarantee when a new thread is started?"
**Answer:** The JVM guarantees that all actions in the thread that calls `Thread.start()` happen-before any action in the started thread. Similarly, all actions in thread T happen-before `T.join()` returns. This means you don't need volatile to pass initial data to a new thread — just set the fields before calling `start()`.
