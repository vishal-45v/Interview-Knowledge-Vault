# Exception Handling — Structured Interview Answers

---

## Q1: What is the difference between checked and unchecked exceptions?

**Answer:** The distinction is enforced at compile time and reflects a design philosophy about who is responsible for handling a given failure.

**Checked exceptions** (`Exception` and its non-`RuntimeException` subclasses) must either be caught or declared in the method's `throws` clause. The compiler enforces this. They represent anticipated, recoverable external conditions — file not found, database connection lost, network timeout.

**Unchecked exceptions** (`RuntimeException` and its subclasses, plus `Error`) are not enforced by the compiler. They typically represent programming errors (bugs) that should not occur in correct code — null pointer dereference, array index out of bounds, illegal argument.

```java
// Checked: compiler forces handling
public String readFile(String path) throws IOException { // must declare
    return Files.readString(Path.of(path)); // IOException is checked
}

// Must be handled by caller:
try {
    String content = readFile("/etc/config.txt");
} catch (IOException e) {
    log.warn("Config not found, using defaults");
}

// Unchecked: no declaration or handling required
public int divide(int a, int b) {
    if (b == 0) throw new ArithmeticException("Division by zero"); // unchecked
    return a / b;
}

// Caller can call without try-catch (but should validate input):
int result = divide(10, 2);
```

**Design controversy:** Modern frameworks (Spring, Hibernate) prefer unchecked exceptions because checked exceptions force every intermediate layer to either handle or declare them, polluting method signatures. Josh Bloch (Effective Java) recommends checked only when the caller can realistically recover. For library/framework code, unchecked is almost always preferable.

---

## Q2: How does try-with-resources work internally?

**Answer:** try-with-resources is syntactic sugar introduced in Java 7 that the compiler desugars into nested try-finally blocks with a critical improvement: it uses `addSuppressed()` to preserve exceptions thrown during `close()`.

The resource must implement `java.lang.AutoCloseable` (or its subinterface `java.io.Closeable`). Resources are closed in reverse order of declaration — the last opened is the first closed.

```java
// What you write:
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql)) {
    stmt.setLong(1, userId);
    return stmt.executeQuery();
}

// What the compiler generates (approximately):
Connection conn = dataSource.getConnection();
Throwable primaryException = null;
try {
    PreparedStatement stmt = conn.prepareStatement(sql);
    Throwable stmtException = null;
    try {
        stmt.setLong(1, userId);
        return stmt.executeQuery();
    } catch (Throwable t) {
        stmtException = t;
        throw t;
    } finally {
        if (stmtException != null) {
            try { stmt.close(); }
            catch (Throwable t) { stmtException.addSuppressed(t); }
        } else {
            stmt.close();
        }
    }
} catch (Throwable t) {
    primaryException = t;
    throw t;
} finally {
    if (primaryException != null) {
        try { conn.close(); }
        catch (Throwable t) { primaryException.addSuppressed(t); }
    } else {
        conn.close();
    }
}
```

**Key behavioural guarantees:**
1. Resources are always closed regardless of whether an exception occurred.
2. Close order is LIFO (last declared = first closed).
3. If the body throws and `close()` also throws, the body exception is primary; the close exception is suppressed (not lost).
4. If the body completes normally but `close()` throws, that exception propagates normally.

**Custom AutoCloseable:**
```java
class ManagedTransaction implements AutoCloseable {
    private final Connection connection;
    private boolean committed = false;

    ManagedTransaction(DataSource ds) throws SQLException {
        this.connection = ds.getConnection();
        this.connection.setAutoCommit(false);
    }

    public void commit() throws SQLException {
        connection.commit();
        committed = true;
    }

    @Override
    public void close() throws SQLException {
        try {
            if (!committed) {
                connection.rollback(); // rollback if not explicitly committed
            }
        } finally {
            connection.close();
        }
    }
}

// Usage
try (ManagedTransaction tx = new ManagedTransaction(dataSource)) {
    userDao.save(user);
    orderDao.save(order);
    tx.commit(); // only reached if no exception
} // close() called — rolls back if commit() was never called
```

---

## Q3: What are suppressed exceptions?

**Answer:** Suppressed exceptions are secondary exceptions that occurred during cleanup (in `close()` methods) while a primary exception was already in flight. Java 7 introduced `Throwable.addSuppressed()` and `getSuppressed()` to preserve these so they are not silently lost.

```java
class BrokenResource implements AutoCloseable {
    public void use() throws Exception {
        throw new Exception("Primary: resource operation failed");
    }

    @Override
    public void close() throws Exception {
        throw new Exception("Suppressed: resource close failed");
    }
}

try {
    try (BrokenResource r = new BrokenResource()) {
        r.use(); // throws primary
    }   // close() throws suppressed — added to primary via addSuppressed()
} catch (Exception e) {
    System.out.println("Primary: " + e.getMessage());
    // Primary: resource operation failed

    for (Throwable suppressed : e.getSuppressed()) {
        System.out.println("Suppressed: " + suppressed.getMessage());
    }
    // Suppressed: resource close failed
}

// Manually adding suppressed exceptions (e.g. in your own cleanup code)
Exception primary = null;
try {
    doWork();
} catch (Exception e) {
    primary = e;
    throw e;
} finally {
    try {
        cleanup();
    } catch (Exception cleanupEx) {
        if (primary != null) {
            primary.addSuppressed(cleanupEx); // preserve both
        } else {
            throw cleanupEx;
        }
    }
}
```

**Why it matters:** Before Java 7, the old `finally` pattern had a critical bug — if both the try body and the finally block threw exceptions, the finally exception replaced the body exception and the root cause was permanently lost. Many debugging sessions were wasted because the exception reported was the cleanup failure, not the actual original problem. `addSuppressed()` solved this once and for all.

---

## Q4: How do you create a good custom exception hierarchy?

**Answer:** A well-designed exception hierarchy communicates domain intent, carries necessary context, and allows callers to catch at the right level of specificity.

```java
// Base domain exception — all service exceptions extend this
public class AppException extends RuntimeException {
    private final String errorCode;
    private final Map<String, Object> context;

    public AppException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.context = new HashMap<>();
    }

    public AppException(String errorCode, String message, Throwable cause) {
        super(message, cause);
        this.errorCode = errorCode;
        this.context = new HashMap<>();
    }

    public AppException withContext(String key, Object value) {
        this.context.put(key, value);
        return this;
    }

    public String getErrorCode() { return errorCode; }
    public Map<String, Object> getContext() { return Collections.unmodifiableMap(context); }
}

// Domain-specific exceptions
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String resourceType, Object id) {
        super("RESOURCE_NOT_FOUND",
              resourceType + " not found with id: " + id);
        withContext("resourceType", resourceType);
        withContext("id", id);
    }
}

public class ValidationException extends AppException {
    private final List<FieldError> fieldErrors;

    public ValidationException(List<FieldError> errors) {
        super("VALIDATION_FAILED", "Request validation failed");
        this.fieldErrors = List.copyOf(errors);
    }

    public List<FieldError> getFieldErrors() { return fieldErrors; }
}

public class InsufficientFundsException extends AppException {
    public InsufficientFundsException(BigDecimal required, BigDecimal available) {
        super("INSUFFICIENT_FUNDS",
              "Required: " + required + ", Available: " + available);
        withContext("required", required);
        withContext("available", available);
    }
}

// Usage in service layer
public Order placeOrder(Long userId, OrderRequest request) {
    User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User", userId));

    if (user.getBalance().compareTo(request.getTotal()) < 0) {
        throw new InsufficientFundsException(request.getTotal(), user.getBalance());
    }

    return orderRepository.save(new Order(user, request));
}

// Spring exception handler — catches at the right level
@ExceptionHandler(ResourceNotFoundException.class)
public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
    return ResponseEntity.status(404)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
}

@ExceptionHandler(AppException.class) // catches all domain exceptions
public ResponseEntity<ErrorResponse> handleAppException(AppException ex) {
    return ResponseEntity.status(400)
            .body(new ErrorResponse(ex.getErrorCode(), ex.getMessage()));
}
```

**Design principles:**
1. Start with one base unchecked exception for your domain.
2. Create specific subclasses only when callers need to distinguish between them.
3. Carry meaningful context (IDs, values) not just messages — makes debugging faster.
4. Use error codes for machine-readable classification (useful for APIs, monitoring).
5. Avoid exception hierarchies deeper than 3 levels — they become hard to maintain.

---

## Q5: What is exception chaining and why use it?

**Answer:** Exception chaining (wrapping) captures the original causing exception when translating between abstraction layers. The cause is accessible via `getCause()` and appears in the full stack trace.

```java
// Without chaining — loses all diagnostic information
public User findUser(long id) {
    try {
        return jdbcTemplate.queryForObject("SELECT * FROM users WHERE id = ?",
                USER_MAPPER, id);
    } catch (DataAccessException e) {
        throw new ServiceException("User not found"); // original cause LOST
    }
}

// With chaining — full causal chain preserved
public User findUser(long id) {
    try {
        return jdbcTemplate.queryForObject("SELECT * FROM users WHERE id = ?",
                USER_MAPPER, id);
    } catch (DataAccessException e) {
        throw new ServiceException("Failed to find user with id: " + id, e); // cause preserved
    }
}

// Stack trace for chained version:
// ServiceException: Failed to find user with id: 42
//     at UserService.findUser(UserService.java:28)
//     at UserController.getUser(UserController.java:45)
// Caused by: org.springframework.dao.EmptyResultDataAccessException: ...
//     at JdbcTemplate.queryForObject(...)
// Caused by: java.sql.SQLException: Connection refused
//     at ...

// Why chaining matters:
// - Layer A (DAO) knows about SQL errors
// - Layer B (Service) should not expose SQL details to callers
// - Layer C (Controller) gets a domain exception with full diagnostic trail
// - Operations team sees the full stack trace in logs

// Standard Java constructors for chaining:
// Throwable(String message, Throwable cause)
// Throwable(Throwable cause)
// Or post-construction: exception.initCause(cause)

// Correct pattern for wrapping checked as unchecked:
try {
    Files.readString(Path.of(configPath));
} catch (IOException e) {
    throw new ConfigurationException("Cannot read config: " + configPath, e);
    // ConfigurationException extends RuntimeException
}
```

---

## Q6: Why should you NOT catch Exception or Throwable broadly?

**Answer:** Catching `Exception` or `Throwable` indiscriminately violates the principle that exceptions should be handled at the level where they can be meaningfully addressed. It causes several specific problems:

```java
// PROBLEM 1: Swallowing exceptions you should propagate
try {
    processOrder(order);
} catch (Exception e) {
    log.error("Error"); // loses stack trace, message, type
    // continues executing as if nothing happened — dangerous
}

// PROBLEM 2: Catching InterruptedException and losing interrupt status
try {
    Thread.sleep(1000);
} catch (Exception e) { // catches InterruptedException too!
    // You've swallowed the interrupt signal — the thread's interrupted
    // flag is now cleared and the calling code that interrupted you
    // will never know — deadlocks, hangs, incorrect shutdown behaviour
}

// CORRECT: handle InterruptedException properly
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // restore the interrupt flag
    throw new TaskAbortedException("Task interrupted", e);
}

// PROBLEM 3: Catching OutOfMemoryError via Throwable
try {
    processHugeDataset();
} catch (Throwable t) {
    log.error("Error processing"); // JVM might be in broken state
    // continue running — catastrophically wrong
}

// ACCEPTABLE broad catch: at top-level boundaries only
// (e.g. servlet filter, Kafka consumer loop, scheduled task)
@Scheduled(fixedDelay = 5000)
public void scheduledTask() {
    try {
        doWork();
    } catch (Exception e) {
        // Acceptable here: prevent the scheduler from dying permanently
        // But always log with full stack trace
        log.error("Scheduled task failed — will retry in 5s", e);
        // Metrics: counter("task.errors").increment();
    }
}
```

**The correct approach is to catch the most specific exception type possible** at the level that can meaningfully handle it. Unhandled exceptions should propagate to a boundary handler (Spring's `@ExceptionHandler`, a Servlet filter, a thread's `UncaughtExceptionHandler`).

---

## Q7: How do you handle checked exceptions in Java 8 streams and lambdas?

**Answer:** Java's built-in functional interfaces (`Function`, `Consumer`, `Predicate`, etc.) do not declare checked exceptions, so the compiler rejects lambda bodies that throw checked exceptions. The solutions range from wrapping to custom functional interfaces.

```java
// PROBLEM: Files.readString throws IOException (checked)
List<Path> paths = List.of(Path.of("a.txt"), Path.of("b.txt"));

// COMPILE ERROR:
paths.stream()
     .map(p -> Files.readString(p)) // IOException not handled — compile error
     .collect(Collectors.toList());

// SOLUTION 1: Wrap in unchecked inside the lambda
paths.stream()
     .map(p -> {
         try {
             return Files.readString(p);
         } catch (IOException e) {
             throw new UncheckedIOException(e); // standard Java wrapper
         }
     })
     .collect(Collectors.toList());

// SOLUTION 2: Generic wrapper utility (call-site clean)
@FunctionalInterface
interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;
}

public static <T, R> Function<T, R> sneaky(ThrowingFunction<T, R> fn) {
    return t -> {
        try {
            return fn.apply(t);
        } catch (RuntimeException e) {
            throw e;
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    };
}

// Clean usage:
paths.stream()
     .map(sneaky(Files::readString)) // no try-catch clutter
     .collect(Collectors.toList());

// SOLUTION 3: Return Optional to model failure without exception
paths.stream()
     .map(p -> {
         try { return Optional.of(Files.readString(p)); }
         catch (IOException e) { return Optional.<String>empty(); }
     })
     .filter(Optional::isPresent)
     .map(Optional::get)
     .collect(Collectors.toList());

// SOLUTION 4: Collect failures separately (useful for batch processing)
record Result<T>(T value, Throwable error) {
    static <T> Result<T> of(T value) { return new Result<>(value, null); }
    static <T> Result<T> failure(Throwable e) { return new Result<>(null, e); }
    boolean isSuccess() { return error == null; }
}

List<Result<String>> results = paths.stream()
        .map(p -> {
            try { return Result.of(Files.readString(p)); }
            catch (IOException e) { return Result.<String>failure(e); }
        })
        .collect(Collectors.toList());

List<String> successes = results.stream().filter(Result::isSuccess)
        .map(Result::value).collect(Collectors.toList());
List<Throwable> failures = results.stream().filter(r -> !r.isSuccess())
        .map(Result::error).collect(Collectors.toList());
```

---

## Q8: What is the performance overhead of exceptions in Java?

**Answer:** Exception performance is nuanced. The expensive operation is creating the exception object, not throwing or catching it.

```java
// Benchmarking the cost components:

// 1. new Exception() — creates object AND captures stack trace
// fillInStackTrace() walks the call stack frame-by-frame
// Cost: proportional to stack depth; roughly 1-10 microseconds for typical stacks

// 2. throw e — very cheap, just changes program flow
// Cost: comparable to a method call

// 3. catch (Exception e) — cheap if exception is thrown (branch prediction miss)
// Cost: a few nanoseconds

// Expensive pattern — creating exception objects in a loop:
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    try {
        Integer.parseInt("not-a-number"); // creates + throws + catches every iteration
    } catch (NumberFormatException e) {
        // each iteration: creates new exception object, captures stack trace
    }
}
// Duration: ~5-10 seconds (microseconds per iteration)

// Fast alternative — validate first:
long start2 = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    String s = "not-a-number";
    if (s.matches("-?\\d+")) { // no exception path
        Integer.parseInt(s);
    }
}
// Duration: ~50 milliseconds — 100x faster

// Trick: pre-created static exception (loses per-call stack trace)
class FastException extends RuntimeException {
    static final FastException INSTANCE = new FastException();

    private FastException() { super("fast path signal", null, true, false); }
    // (message, cause, enableSuppression, writableStackTrace)
    // writableStackTrace = false: NO stack trace captured = cheap
}

// Disable stack trace for control-flow exceptions
class ControlFlowSignal extends RuntimeException {
    ControlFlowSignal(String msg) {
        super(msg, null, true, false); // no stack trace
    }
}
```

**JIT optimisation:** The JVM can sometimes optimise frequently-executed try-catch blocks after profiling (the JIT may eliminate the overhead in hot paths), but this is implementation-dependent and not something to rely on.

**Rule of thumb:** Exceptions should be exceptional. Using them for expected control flow (parsing user input, checking "does this item exist") in hot paths is a design smell and a performance problem. Use `Optional`, return codes, or pre-validation instead.

---

## Q9: What happens to the return value if the finally block has a return statement?

**Answer:** The finally block's `return` silently overwrites any pending return value or exception. This is one of the most dangerous anti-patterns in Java and modern IDEs flag it as a warning.

```java
// Case 1: finally return overwrites try return
int demo1() {
    try {
        return 1; // staged but not yet returned
    } finally {
        return 2; // OVERWRITES the 1
    }
}
System.out.println(demo1()); // prints 2

// Case 2: finally return swallows pending exception
int demo2() {
    try {
        throw new RuntimeException("I will be lost!");
    } finally {
        return 42; // exception is silently consumed — returns 42
    }
}
System.out.println(demo2()); // prints 42, exception gone

// Case 3: the staged return value is captured before finally executes
int x = 0;
int demo3() {
    try {
        x = 1;
        return x; // return value is captured as 1
    } finally {
        x = 2; // modifies x but does NOT affect the already-captured return value
    }
}
System.out.println(demo3()); // prints 1
System.out.println(x);       // prints 2

// RULE: never use return in a finally block
// Correct pattern: compute return value into a variable, return outside try
int correct(String input) {
    String result = null;
    try {
        result = expensiveOperation(input);
    } catch (ProcessingException e) {
        result = "default";
        log.warn("Using default due to: {}", e.getMessage());
    } finally {
        cleanup(); // no return here
    }
    return result.length(); // return is here, after try-finally
}
```

**Why compilers allow it:** The JVM specification defines this behaviour. Java does not prohibit `return` in `finally` because there are rare, contrived cases where it is intentional. But virtually all production code style guides (Google Java Style, Sun conventions) and static analysis tools (SonarQube, FindBugs/SpotBugs) flag it as a defect.

---

## Q10: How does Spring's @ExceptionHandler relate to Java exception handling?

**Answer:** `@ExceptionHandler` is Spring MVC's mechanism for centralising exception-to-HTTP-response translation. It sits at the boundary between Java exception semantics and HTTP response semantics. Under the hood, exceptions propagate normally through Java's exception mechanism until they reach the DispatcherServlet, where Spring's `HandlerExceptionResolverComposite` matches them against registered `@ExceptionHandler` methods.

```java
// Method-level: handles exceptions only for this controller
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id); // throws UserNotFoundException if missing
    }

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("USER_NOT_FOUND", ex.getMessage()));
    }
}

// Global handler: handles exceptions from ALL controllers
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Most specific handlers first — Spring picks the most specific match
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex,
                                         WebRequest request) {
        log.warn("Resource not found: {}", ex.getMessage());
        return new ErrorResponse(
                ex.getErrorCode(),
                ex.getMessage(),
                request.getDescription(false)
        );
    }

    @ExceptionHandler(ValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(ValidationException ex) {
        return new ErrorResponse("VALIDATION_FAILED", ex.getFieldErrors());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleBindingErrors(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
                .collect(Collectors.toList());
        return new ErrorResponse("VALIDATION_FAILED", errors);
    }

    // Catch-all: log unexpected exceptions, return 500
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleUnexpected(Exception ex, WebRequest request) {
        // Generate a correlation ID to link logs to user-visible error
        String correlationId = UUID.randomUUID().toString();
        log.error("Unexpected error [correlationId={}]", correlationId, ex);
        return new ErrorResponse("INTERNAL_ERROR",
                "An unexpected error occurred. Reference: " + correlationId);
    }
}

// Error response DTO
record ErrorResponse(String code, String message) {
    ErrorResponse(String code, List<FieldError> fieldErrors) {
        this(code, fieldErrors.stream()
                .map(fe -> fe.field() + ": " + fe.message())
                .collect(Collectors.joining(", ")));
    }
}
```

**How Spring selects the handler:**
1. Spring scans for `@ExceptionHandler` methods in the current controller, then in `@ControllerAdvice` classes.
2. It picks the most specific matching type (closest in the inheritance hierarchy to the thrown exception).
3. If two handlers match at the same specificity level, the order of `@ControllerAdvice` beans (controlled by `@Order`) determines precedence.

**Integration with Java exception chaining:** The `@ExceptionHandler` method receives the exception as thrown — including its cause chain. You can call `ex.getCause()` to access the wrapped original exception for logging or decision-making. Spring's `ResponseEntityExceptionHandler` base class provides default handlers for all standard Spring MVC exceptions that you can override.
