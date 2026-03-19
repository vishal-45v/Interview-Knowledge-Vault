# Exception Handling — Theory Questions

> 22 theory questions covering the Java exception hierarchy, checked vs unchecked exceptions, try-with-resources, multi-catch, and best practices for Senior Java Backend Engineers.

---

## Exception Hierarchy

**Q1. Describe the Java exception class hierarchy.**

```
java.lang.Object
  └── java.lang.Throwable
        ├── java.lang.Error
        │     ├── OutOfMemoryError
        │     ├── StackOverflowError
        │     ├── AssertionError
        │     └── VirtualMachineError
        └── java.lang.Exception
              ├── RuntimeException  (unchecked)
              │     ├── NullPointerException
              │     ├── IllegalArgumentException
              │     ├── IllegalStateException
              │     ├── IndexOutOfBoundsException
              │     ├── ClassCastException
              │     └── UnsupportedOperationException
              ├── IOException  (checked)
              ├── SQLException  (checked)
              └── CloneNotSupportedException  (checked)
```

**Throwable**: the root. Has `getMessage()`, `getCause()`, `getStackTrace()`, `printStackTrace()`.
**Error**: JVM-level problems — do not catch (OOM, StackOverflow).
**Exception**: application-level problems.

---

**Q2. What is the difference between checked and unchecked exceptions?**

| Feature | Checked | Unchecked |
|---------|---------|-----------|
| Extends | `Exception` (not RuntimeException) | `RuntimeException` or `Error` |
| Compiler enforcement | Yes — must declare or handle | No compiler requirement |
| Represents | Expected failure conditions | Programming bugs or JVM errors |
| Examples | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |

**Checked exceptions** represent conditions the caller reasonably might want to handle: file not found, network timeout, SQL constraint violation.

**Unchecked exceptions** represent bugs (NPE, array out of bounds) or unrecoverable conditions. The caller cannot reasonably recover — fix the code.

**The debate:** checked exceptions were a Java design decision that many now consider a mistake. Modern Java APIs (Stream, CompletableFuture) don't support checked exceptions well. Spring converts most checked exceptions to unchecked (e.g., `DataAccessException`).

---

**Q3. When should you create a custom exception? What should it extend?**

**Create custom exceptions when:**
- You need domain-specific error semantics that standard exceptions don't convey
- You want to carry additional context (error codes, field names, severity)
- Callers need to distinguish your exception from others programmatically

```java
// Domain exception hierarchy
public class OrderException extends RuntimeException {  // Extend RuntimeException (unchecked)
    private final String orderId;
    private final ErrorCode errorCode;

    public OrderException(String orderId, ErrorCode errorCode, String message) {
        super(message);
        this.orderId = orderId;
        this.errorCode = errorCode;
    }

    public OrderException(String orderId, ErrorCode errorCode, String message, Throwable cause) {
        super(message, cause);  // ALWAYS pass cause to preserve stack trace chain
        this.orderId = orderId;
        this.errorCode = errorCode;
    }
}

public class OrderNotFoundException extends OrderException {
    public OrderNotFoundException(String orderId) {
        super(orderId, ErrorCode.ORDER_NOT_FOUND,
              "Order not found: " + orderId);
    }
}

public class InsufficientInventoryException extends OrderException {
    private final int requested;
    private final int available;

    public InsufficientInventoryException(String productId, int requested, int available) {
        super(productId, ErrorCode.INSUFFICIENT_INVENTORY,
              String.format("Insufficient inventory: requested=%d, available=%d",
                            requested, available));
        this.requested = requested;
        this.available = available;
    }
}
```

**Prefer unchecked** for most custom exceptions in modern Java — checked exceptions hurt API usability (lambdas, streams).

---

**Q4. What is try-with-resources and how does it work?**

Try-with-resources automatically closes resources that implement `AutoCloseable` (or `Closeable`).

```java
// Without try-with-resources (verbose, bug-prone)
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("file.txt"));
    return reader.readLine();
} catch (IOException e) {
    throw new RuntimeException(e);
} finally {
    if (reader != null) {
        try {
            reader.close();  // close() itself can throw!
        } catch (IOException e) {
            // silently ignored — the original exception is lost if both throw
        }
    }
}

// With try-with-resources (clean, safe)
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    return reader.readLine();
}
// reader.close() called automatically in REVERSE order of declaration
```

**Multiple resources:**
```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(SQL);
     ResultSet rs = stmt.executeQuery()) {
    // Closed in reverse order: rs, then stmt, then conn
}
```

**Suppressed exceptions:** If both the try block and close() throw, the close() exception is suppressed (not lost) — accessible via `e.getSuppressed()`.

```java
try (Resource r = new Resource()) {
    throw new RuntimeException("original");
    // r.close() also throws — attached as suppressed, not replacing original
}
```

---

**Q5. What is the difference between `throw` and `throws`?**

```java
// throws — declaration that a method may throw a checked exception
// Part of the method signature, tells callers they must handle it
public void readFile(String path) throws IOException {
    // ...
}

// throw — actually throws an exception instance
public void validateAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative: " + age);
    }
}
```

You can declare `throws RuntimeException` but it has no compiler enforcement — purely informational.

---

**Q6. What is exception chaining and why is it important?**

Exception chaining preserves the original cause when re-throwing or wrapping exceptions.

```java
// WRONG — original exception lost, stack trace gone
try {
    userRepository.save(user);
} catch (DataAccessException e) {
    throw new UserCreationException("Failed to create user");  // cause lost!
}

// CORRECT — chain the cause
try {
    userRepository.save(user);
} catch (DataAccessException e) {
    throw new UserCreationException("Failed to create user", e);  // cause preserved
}

// Constructor must accept cause:
public UserCreationException(String message, Throwable cause) {
    super(message, cause);  // Throwable's constructor accepts cause
}
```

**Why it matters:**
- Full stack trace shows BOTH where the problem manifested AND the root cause
- Without chaining, you see only the outer exception — debugging becomes much harder
- log.error("message", e) includes the full cause chain

---

**Q7. What is multi-catch syntax (Java 7+)?**

```java
// Before Java 7 — duplicate code
try {
    parseAndSave(data);
} catch (ParseException e) {
    log.error("Parse error", e);
    throw new ProcessingException("Failed to parse", e);
} catch (IOException e) {
    log.error("IO error", e);
    throw new ProcessingException("Failed to process", e);
}

// Java 7+ multi-catch — cleaner
try {
    parseAndSave(data);
} catch (ParseException | IOException e) {
    log.error("Processing error", e);
    throw new ProcessingException("Failed to process", e);
}
```

**Limitation:** Cannot catch related exception types in multi-catch.
```java
// COMPILE ERROR — IOException is a supertype of FileNotFoundException
catch (IOException | FileNotFoundException e) { ... }
```

The variable `e` in a multi-catch is effectively final — you cannot reassign it.

---

**Q8. What happens when an exception is thrown in a `finally` block?**

```java
public int riskyMethod() {
    try {
        throw new RuntimeException("from try");
    } finally {
        throw new RuntimeException("from finally");  // This REPLACES the try exception!
    }
    // The "from try" exception is completely lost — no suppression, just discarded
}
```

This is a major difference from try-with-resources (which uses suppression).

**In finally, never throw — just clean up:**
```java
finally {
    try {
        connection.close();
    } catch (Exception e) {
        log.warn("Error closing connection", e);  // Log but don't rethrow
    }
}
```

---

**Q9. What is the `Throwable.addSuppressed()` mechanism?**

Java 7 introduced suppressed exceptions for try-with-resources:

```java
// Manually use suppression
Exception primary = new RuntimeException("primary");
try {
    // cleanup that throws
} catch (Exception suppressed) {
    primary.addSuppressed(suppressed);
}
throw primary;

// Access suppressed exceptions
try {
    riskyOperation();
} catch (Exception e) {
    for (Throwable suppressed : e.getSuppressed()) {
        log.error("Suppressed: {}", suppressed.getMessage());
    }
}
```

Try-with-resources uses this automatically: if the try block throws AND close() throws, the close() exception is suppressed (attached to the primary exception) rather than discarding the primary.

---

**Q10. How does exception handling interact with Java streams and lambdas?**

Functional interfaces (`Function`, `Predicate`, `Consumer`) don't declare checked exceptions:

```java
// COMPILE ERROR — readFile throws IOException (checked)
List<String> lines = files.stream()
    .map(file -> readFile(file))  // IOException not allowed in Function
    .collect(Collectors.toList());

// Option 1: Wrap in unchecked
List<String> lines = files.stream()
    .map(file -> {
        try {
            return readFile(file);
        } catch (IOException e) {
            throw new UncheckedIOException(e);  // UncheckedIOException wraps IOException
        }
    })
    .collect(Collectors.toList());

// Option 2: Utility method — ThrowingFunction wrapper
@FunctionalInterface
public interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;

    static <T, R> Function<T, R> wrap(ThrowingFunction<T, R> f) {
        return t -> {
            try {
                return f.apply(t);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        };
    }
}

List<String> lines = files.stream()
    .map(ThrowingFunction.wrap(file -> readFile(file)))
    .collect(Collectors.toList());
```

---

## Best Practices

**Q11. What are the key "don't" rules for exception handling?**

**1. Never catch Exception or Throwable broadly without re-throwing:**
```java
// WRONG — swallows all exceptions silently
try {
    processOrder(order);
} catch (Exception e) {
    // silent swallow — nobody knows the error occurred
}

// ACCEPTABLE — if you log and handle appropriately
try {
    processOrder(order);
} catch (Exception e) {
    log.error("Failed to process order {}", order.getId(), e);
    throw new OrderProcessingException("Order processing failed", e);
}
```

**2. Never use exceptions for flow control:**
```java
// WRONG — expensive stack trace creation for normal flow
public boolean isValidUser(String id) {
    try {
        userRepository.findById(id);
        return true;
    } catch (UserNotFoundException e) {
        return false;
    }
}

// CORRECT
public boolean isValidUser(String id) {
    return userRepository.existsById(id);
}
```

**3. Never log and re-throw (double logging):**
```java
// WRONG — exception logged twice (here and by caller)
catch (DataAccessException e) {
    log.error("DB error", e);   // logged here
    throw e;                     // and again by whoever catches this
}

// CORRECT — either log OR rethrow, not both
// (let top-level exception handler log it)
catch (DataAccessException e) {
    throw new ServiceException("Failed to save order", e);
}
```

**4. Never throw generic exceptions:**
```java
// WRONG
throw new Exception("something went wrong");

// CORRECT
throw new IllegalArgumentException("userId cannot be null");
throw new IllegalStateException("Order already shipped; cannot cancel");
```

---

**Q12. What is the difference between `RuntimeException` and `Exception` from an API design perspective?**

**Checked (`Exception`):**
- Forces callers to acknowledge the possibility of failure
- Appropriate when the caller CAN and SHOULD handle it: `FileNotFoundException`, `InterruptedException`
- Creates coupling in API signatures (propagates through call stack)
- Doesn't work with lambdas/streams without wrapping

**Unchecked (`RuntimeException`):**
- Caller decides whether to handle it
- Appropriate for programming errors (NPE, bad arguments) and domain exceptions where most callers will let it propagate
- Cleaner API signatures
- Works naturally with lambdas

**Modern Java trend:** Prefer unchecked. Spring's exception hierarchy (DataAccessException) is all unchecked. Many developers argue checked exceptions were a mistake.

---

**Q13. Explain `NullPointerException` prevention strategies.**

```java
// 1. Objects.requireNonNull — fail-fast with clear message
public OrderService(OrderRepository repository) {
    this.repository = Objects.requireNonNull(repository, "repository must not be null");
}

// 2. Optional — for return values that might be absent
public Optional<User> findUser(Long id) {
    return userRepository.findById(id);
}

// Caller uses Optional properly:
findUser(id)
    .map(user -> user.getName())
    .orElse("Unknown");

// 3. @NonNull / @Nullable annotations (Lombok, JSR-305, JetBrains)
public void processUser(@NonNull User user) { ... }

// 4. Java 14+ — helpful NullPointerMessages
// NPE now says: "Cannot invoke 'String.length()' because 'user.address.street' is null"
// Instead of just: "NullPointerException"

// 5. Defensive null checks at boundaries (API entry points)
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(@RequestBody @Valid OrderRequest request) {
    // @Valid ensures non-null fields via Bean Validation
}
```

---

**Q14. How does `StackOverflowError` occur and how do you diagnose it?**

Occurs when the call stack exceeds the maximum depth (JVM default: ~500-1000 frames).

**Common causes:**
```java
// 1. Infinite recursion
public int factorial(int n) {
    return n * factorial(n - 1);  // Missing base case!
}

// 2. Circular toString / equals / hashCode
@Entity
public class Order {
    private Customer customer;

    @Override
    public String toString() {
        return "Order{customer=" + customer + "}";  // Calls Customer.toString()
    }
}

@Entity
public class Customer {
    private List<Order> orders;

    @Override
    public String toString() {
        return "Customer{orders=" + orders + "}";  // Calls Order.toString() → infinite!
    }
}

// Fix: use IDs in toString, or @Exclude Lombok's @ToString
@ToString.Exclude  // Lombok — exclude from toString to break cycle
private List<Order> orders;
```

**Diagnosis:**
- Stack trace shows repeated pattern of method calls
- Use jstack to capture current stack state
- Look for circular dependencies in serialization/toString

---

**Q15. What is `OutOfMemoryError` and what are its different forms?**

OOM occurs when the JVM cannot allocate memory. Different forms indicate different causes:

| Error | Cause |
|-------|-------|
| `OutOfMemoryError: Java heap space` | Heap full — memory leak or heap too small |
| `OutOfMemoryError: GC overhead limit exceeded` | GC running >98% of time, recovering <2% heap |
| `OutOfMemoryError: Metaspace` | Class metadata area full (too many classes/classloaders) |
| `OutOfMemoryError: Direct buffer memory` | NIO direct buffers exhausted |
| `OutOfMemoryError: unable to create new native thread` | Too many threads (OS limit) |

**Heap space diagnosis:**
```bash
# Capture heap dump on OOM
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heapdump.hprof

# Analyze with Eclipse Memory Analyzer (MAT)
# Look for: large object trees, unexpected retained objects
```

---

**Q16. What is `InterruptedException` and how should you handle it?**

`InterruptedException` is thrown when a thread waiting on `sleep()`, `wait()`, `join()`, or blocking I/O is interrupted.

**The interrupted flag:** Each thread has an interrupt flag. `interrupt()` sets it. `sleep()` etc. check it and throw `InterruptedException` (clearing the flag).

```java
// WRONG — swallowing InterruptedException
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    // Do nothing — thread's interrupt status cleared, caller can't know thread was interrupted
}

// CORRECT Option 1: Re-interrupt the thread (restore interrupt status)
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();  // Restore interrupt flag
    // Then return or handle gracefully
    return;
}

// CORRECT Option 2: If you can declare checked exception, propagate it
public void process() throws InterruptedException {
    Thread.sleep(1000);
}

// CORRECT Option 3: Wrap if you must hide it
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    throw new RuntimeException("Thread interrupted during processing", e);
}
```

**Why it matters:** If you swallow `InterruptedException` without re-interrupting, the thread ignores shutdown signals — leading to threads that won't stop during graceful shutdown.

---

**Q17. What is the `UncaughtExceptionHandler` and when would you use it?**

```java
// Set handler for uncaught exceptions in a thread
Thread thread = new Thread(() -> {
    throw new RuntimeException("Unexpected error");
});

thread.setUncaughtExceptionHandler((t, e) -> {
    log.error("Uncaught exception in thread {}: {}", t.getName(), e.getMessage(), e);
    // Send alert, update metrics, etc.
});

// Set default handler for all threads
Thread.setDefaultUncaughtExceptionHandler((t, e) -> {
    log.error("Uncaught exception in thread {}", t.getName(), e);
    metrics.increment("uncaught.exceptions");
});

// For thread pools (ExecutorService doesn't use UncaughtExceptionHandler for submitted tasks!)
ExecutorService executor = Executors.newFixedThreadPool(4, runnable -> {
    Thread t = new Thread(runnable);
    t.setUncaughtExceptionHandler((thread, e) -> log.error("Task failed", e));
    return t;
});
```

**Note:** For tasks submitted to `ExecutorService` via `submit()`, exceptions are captured in the `Future` — they don't trigger the uncaught exception handler. Only `execute()` triggers it.

---

**Q18. What are `Error` subclasses that you should never catch? Why?**

| Error | Why Not Catch |
|-------|--------------|
| `OutOfMemoryError` | If caught, very little can be done; next allocation will fail again |
| `StackOverflowError` | Program is in inconsistent state; bug must be fixed |
| `VirtualMachineError` | JVM integrity compromised |
| `ThreadDeath` | Thrown when `Thread.stop()` is called; catching it prevents thread from dying |

```java
// Generally acceptable (but be careful):
try {
    riskyOperation();
} catch (OutOfMemoryError e) {
    // Release large caches, log, then rethrow or exit
    cache.clear();
    log.error("OOM — clearing cache", e);
    throw e;  // Still propagate
}

// NEVER catch Error silently
try {
    ...
} catch (Error e) {
    // Pretend nothing happened — WRONG, program is in broken state
}
```

`AssertionError` is debatable — thrown by `assert` statements (disabled in production by default).

---

**Q19. How does exception handling work in a multi-threaded context?**

```java
// 1. Callable captures exceptions — retrieve via Future.get()
ExecutorService executor = Executors.newFixedThreadPool(4);

Future<Order> future = executor.submit(() -> {
    return processOrder(orderId);  // May throw checked or unchecked exception
});

try {
    Order order = future.get();  // Blocks
} catch (ExecutionException e) {
    Throwable cause = e.getCause();  // The actual exception from the task
    if (cause instanceof OrderNotFoundException) {
        // Handle
    }
    throw new RuntimeException("Task failed", cause);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}

// 2. CompletableFuture exception handling
CompletableFuture.supplyAsync(() -> processOrder(orderId))
    .exceptionally(e -> {
        log.error("Order processing failed", e);
        return Order.failed();  // Fallback
    })
    .thenApply(order -> buildResponse(order));

// 3. Global exception handler in Spring (for async @Async methods)
@Bean
public AsyncUncaughtExceptionHandler asyncUncaughtExceptionHandler() {
    return (ex, method, params) -> {
        log.error("Exception in @Async method {}: {}", method.getName(), ex.getMessage(), ex);
    };
}
```

---

**Q20. What is the cost of exception handling in Java?**

Creating an exception involves:
1. **Allocating the Exception object** — relatively cheap
2. **Capturing the stack trace** (`Throwable` constructor calls `fillInStackTrace()`) — **expensive**, especially for deep call stacks. Can take microseconds to milliseconds.

**Performance implications:**
- Throwing 1 exception: ~1-5 μs (depending on stack depth)
- Throwing 1000 exceptions/second: 1-5 ms overhead
- At 100K exceptions/second: could become a bottleneck

**When not to use exceptions:**
```java
// WRONG — exception for flow control in hot path
public boolean tryGetFromCache(String key) {
    try {
        cache.get(key);  // throws on miss
        return true;
    } catch (CacheException e) {
        return false;
    }
}

// CORRECT — return null or Optional for expected absence
Optional<String> result = cache.getIfPresent(key);
```

**Optimization:** You can override `fillInStackTrace()` to return `this` for performance-critical exception types used for flow control (e.g., sentinel values):

```java
public class ControlFlowException extends RuntimeException {
    @Override
    public synchronized Throwable fillInStackTrace() {
        return this;  // Skip expensive stack capture
    }
}
```

---

**Q21. What are Java's built-in exception types and when do you use each?**

```java
// IllegalArgumentException — invalid parameter value
if (amount.compareTo(BigDecimal.ZERO) <= 0) {
    throw new IllegalArgumentException("Amount must be positive, got: " + amount);
}

// IllegalStateException — object in wrong state for this operation
if (order.isShipped()) {
    throw new IllegalStateException("Cannot cancel a shipped order: " + orderId);
}

// NullPointerException — can throw explicitly for null validation (but prefer IAE)
Objects.requireNonNull(userId, "userId");  // Throws NPE with message

// UnsupportedOperationException — method not implemented
// (e.g., immutable collection's add())
@Override
public void remove() {
    throw new UnsupportedOperationException("Read-only iterator");
}

// IndexOutOfBoundsException — invalid index
if (index < 0 || index >= size) {
    throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
}

// ConcurrentModificationException — collection modified during iteration
// (automatically thrown by fail-fast iterators)

// NoSuchElementException — iterator has no more elements
if (!iterator.hasNext()) {
    throw new NoSuchElementException();
}

// ArithmeticException — integer division by zero (thrown automatically)
// ClassCastException — invalid cast (thrown automatically)
// NumberFormatException — invalid string-to-number conversion
```

---

**Q22. How does Spring's exception handling hierarchy work?**

Spring wraps checked exceptions into unchecked exceptions to prevent them from polluting service layer code.

**DataAccessException hierarchy:**
```
DataAccessException (unchecked)
  ├── DataIntegrityViolationException  (constraint violation)
  │     └── DuplicateKeyException
  ├── DataAccessResourceFailureException (DB unreachable)
  ├── QueryTimeoutException
  ├── EmptyResultDataAccessException   (expected 1, got 0)
  ├── IncorrectResultSizeDataAccessException
  └── OptimisticLockingFailureException
```

**PersistenceExceptionTranslation:** Spring's `@Repository` annotation enables `PersistenceExceptionTranslationPostProcessor`, which wraps JPA/Hibernate exceptions (like `javax.persistence.NoResultException`) into Spring's `DataAccessException` hierarchy.

```java
@Repository  // Without this, JPA exceptions bubble up unwrapped
public class OrderRepositoryImpl {
    // JPA's EntityNotFoundException → wrapped to Spring's EmptyResultDataAccessException
}
```

This decouples your service layer from the specific ORM/database technology — you handle `DataAccessException`, not `HibernateException` or `SQLException`.
