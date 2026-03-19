# Thread Deadlock Debugging

> Step-by-step guide to detecting, diagnosing, and resolving thread deadlocks in Java.

---

## What Is a Deadlock?

Two or more threads are blocked forever, each waiting for a resource held by the other.

```
Thread 1:              Thread 2:
LOCK account A         LOCK account B
  wait for B...          wait for A...
   → DEADLOCK ← ───────────────────────
```

---

## Classic Deadlock Example

```java
// Transfer money — naive implementation (DEADLOCK prone)
public class Bank {

    public void transfer(Account from, Account to, BigDecimal amount) {
        synchronized(from) {          // Lock account A
            synchronized(to) {        // Try to lock account B
                from.debit(amount);
                to.credit(amount);
            }
        }
    }
}

// Thread 1: transfer(accountA, accountB, 100)
// Thread 2: transfer(accountB, accountA, 50)
// T1 locks A, T2 locks B, T1 waits for B, T2 waits for A → DEADLOCK
```

---

## Detecting a Deadlock

### Method 1: Thread Dump

```bash
# Generate thread dump
jstack <PID>
# Or: kill -3 <PID>  (sends SIGQUIT — triggers thread dump without killing)

# Look for "Found one Java-level deadlock" in output:
# Found one Java-level deadlock:
# =============================
# "Thread-1":
#   waiting to lock monitor 0x00007f5b8000d008 (object of type 'Account'),
#   which is held by "Thread-0"
# "Thread-0":
#   waiting to lock monitor 0x00007f5b8000c8a0 (object of type 'Account'),
#   which is held by "Thread-1"
```

### Method 2: JMX / VisualVM

```java
// Programmatic deadlock detection
ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreadIds = threadBean.findDeadlockedThreads();

if (deadlockedThreadIds != null) {
    ThreadInfo[] threadInfos = threadBean.getThreadInfo(deadlockedThreadIds);
    for (ThreadInfo info : threadInfos) {
        log.error("Deadlocked thread: {}", info.getThreadName());
        log.error("Waiting for: {}", info.getLockName());
        log.error("Lock held by: {}", info.getLockOwnerName());
    }
}
```

---

## Fixing Deadlocks

### Fix 1: Consistent Lock Ordering

```java
// Fix: Always lock in the same order (lower ID first)
public void transfer(Account from, Account to, BigDecimal amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;

    synchronized(first) {
        synchronized(second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
// Now both threads lock in the same order → no deadlock
```

### Fix 2: tryLock with Timeout

```java
public void transfer(Account from, Account to, BigDecimal amount)
        throws InterruptedException {

    ReentrantLock lockA = from.getLock();
    ReentrantLock lockB = to.getLock();

    while (true) {
        if (lockA.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                if (lockB.tryLock(100, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        return;  // Success!
                    } finally {
                        lockB.unlock();
                    }
                }
            } finally {
                lockA.unlock();
            }
        }
        // Could not acquire both locks — wait and retry
        Thread.sleep(10 + ThreadLocalRandom.current().nextInt(50));
    }
}
```

### Fix 3: Use Higher-Level Abstractions

```java
// Use @Transactional with database-level locking instead of Java locks
@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    // Database handles locking — no Java-level synchronization needed
    Account from = accountRepository.findByIdForUpdate(fromId);  // SELECT FOR UPDATE
    Account to = accountRepository.findByIdForUpdate(toId);

    from.debit(amount);
    to.credit(amount);
}
```

---

## Preventing Deadlocks — Best Practices

1. **Lock ordering:** Always acquire locks in the same global order
2. **Lock timeout:** Use `tryLock(timeout)` instead of `lock()`
3. **Minimize lock scope:** Hold locks for the shortest time possible
4. **Use database transactions:** Let the DB handle locking when possible
5. **Use concurrent utilities:** `ConcurrentHashMap`, `AtomicReference`, `BlockingQueue`
6. **Avoid nested locks:** Try not to acquire a lock while holding another
7. **Use lock-free algorithms:** CAS operations via `AtomicInteger`, `compareAndSet()`

---

## Production Deadlock Response

```
Step 1: Confirm deadlock
  → Check app logs for "waiting to acquire lock" or timeout errors
  → Take thread dump: jstack <PID>
  → Look for "Found one Java-level deadlock"

Step 2: Immediate mitigation
  → Restart the affected service (breaks the deadlock)
  → Route traffic to healthy instances

Step 3: Root cause analysis
  → Analyze thread dump to identify which locks and which code path
  → Look at git history for recent changes that modified synchronized code
  → Reproduce in staging environment

Step 4: Fix
  → Implement lock ordering or use tryLock
  → Add monitoring for thread contention: jstack analysis, JMX metrics

Step 5: Prevention
  → Add deadlock detection metrics (findDeadlockedThreads())
  → Alert when deadlock detected
  → Code review checklist: check lock ordering in all synchronized code
```
