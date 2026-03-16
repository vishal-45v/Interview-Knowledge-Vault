# Multithreading — Structured Answers

---

## Topic 1: Java Memory Model — Complete Guide

The JMM defines how threads interact through memory. Without it, CPUs and JVMs can reorder operations unpredictably.

### Happens-Before Rules (Complete List)
1. Program order rule: each action in a thread HB the next action
2. Monitor lock rule: unlock HB subsequent lock on same monitor
3. Volatile variable rule: volatile write HB subsequent volatile read
4. Thread start rule: start() HB all actions in started thread
5. Thread termination rule: all thread actions HB join() returning
6. Thread interruption rule: interrupt() HB interrupted thread detecting interruption
7. Finalizer rule: constructor completion HB start of finalizer
8. Transitivity: A HB B and B HB C → A HB C

### Safe Publication Patterns
An object is safely published when it's made visible to other threads after full construction:
1. Storing in a static field initialized by a static initializer
2. Storing in a volatile field or AtomicReference
3. Storing in a final field (immutable)
4. Storing in a field guarded by a lock

---

## Topic 2: Thread Pool Sizing Formula

```
CPU-bound tasks:
  pool_size = N_cpu (Runtime.getRuntime().availableProcessors())

I/O-bound tasks:
  pool_size = N_cpu * (1 + wait_time / compute_time)

  Example: 8 CPUs, task spends 90% waiting:
  pool_size = 8 * (1 + 9) = 80 threads

Mixed workloads:
  Separate pools for CPU-bound and I/O-bound tasks
  CPU pool: N_cpu threads
  IO pool: N_cpu * W threads (W = wait ratio)
```

### Common Pool Configurations
```java
// For HTTP request handling:
int httpThreads = Runtime.getRuntime().availableProcessors() * 10;

// For background jobs (CPU-bound):
int jobThreads = Runtime.getRuntime().availableProcessors();

// For database calls (I/O-bound):
int dbThreads = hikariConfig.getMaximumPoolSize();  // match DB pool size
```

---

## Topic 3: Deadlock Prevention — Comprehensive Strategy

### Detection
```bash
# Generate thread dump:
jstack -l <PID> > thread-dump.txt
# Look for: "Found one Java-level deadlock:"

# Via JMX programmatically:
ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = mxBean.findDeadlockedThreads();
if (deadlockedThreads != null) {
    ThreadInfo[] info = mxBean.getThreadInfo(deadlockedThreads, true, true);
    // Log and alert
}
```

### Prevention Strategies (ranked by effectiveness)
1. **Lock ordering:** Always acquire in same order (most reliable)
2. **Avoid nested locks:** Single lock per operation when possible
3. **Lock timeout:** Use tryLock with timeout (Resilience4j / ReentrantLock)
4. **Use higher-level constructs:** ConcurrentHashMap, BlockingQueue (no explicit locking)
5. **Design review:** Deadlock-prone code is usually a design smell
