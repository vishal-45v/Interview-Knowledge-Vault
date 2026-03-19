# Exception Handling — Follow-Up Traps & Tricky Interview Questions

---

## Trap 1: "Does the finally block always run?"

**What most people say:** Yes, finally always runs — that is its whole point.

**Correct answer:** Almost always, but there are well-defined exceptions:

1. `System.exit()` — the JVM terminates immediately; shutdown hooks run but finally does not.
2. JVM crash (native code segfault, `kill -9` from the OS, OOM in a non-recoverable state during GC).
3. Infinite loop or deadlock inside the `try` or `catch` block — the thread never reaches `finally`.
4. The thread executing the try block is killed via `Thread.stop()` (deprecated, but technically possible).

```java
try {
    System.out.println("In try");
    System.exit(0); // JVM exits here
} finally {
    System.out.println("In finally"); // NEVER printed
}

// Output: "In try"
// The finally block is completely skipped
```

The correct answer in an interview: "finally always runs *unless* the JVM is forcibly terminated or the code never reaches the finally boundary due to an infinite loop or thread kill."

---

## Trap 2: "Can you catch multiple exception types in one catch block?"

**What most people say:** No, you need a separate catch block for each exception type.

**Correct answer:** Yes, since Java 7 you can use multi-catch with `|` (pipe). However, there is a subtle restriction: the caught types must not be in an inheritance relationship with each other.

```java
// Java 7+ multi-catch — both types handled identically
try {
    riskyOperation();
} catch (IOException | SQLException e) {
    log.error("Operation failed", e);
    throw new ServiceException("Failed", e);
}

// COMPILE ERROR: cannot catch FileNotFoundException | IOException
// because FileNotFoundException IS-A IOException (redundant)
try {
    readFile();
} catch (FileNotFoundException | IOException e) { // COMPILE ERROR
    // This is illegal — FileNotFoundException is a subclass of IOException
}

// Hidden trap with multi-catch: the variable `e` is implicitly final
try {
    riskyOperation();
} catch (IOException | SQLException e) {
    e = new IOException("modified"); // COMPILE ERROR — e is effectively final
}
```

---

## Trap 3: "What happens if both the catch block and the finally block throw exceptions?"

**What most people say:** The second exception is thrown and the first is lost.

**Correct answer:** This is correct but dangerous and a well-known pitfall. If both `catch` and `finally` throw, **the exception from `finally` completely replaces the exception from `catch`**, and the original exception is silently lost with no indication in the stack trace. This is why try-with-resources was introduced — it uses `addSuppressed()` to preserve both.

```java
void dangerous() throws Exception {
    try {
        throw new RuntimeException("Original problem"); // step 1
    } catch (RuntimeException e) {
        System.out.println("Caught: " + e.getMessage());
        throw new IOException("Problem in catch block"); // step 2
    } finally {
        throw new IllegalStateException("Problem in finally"); // step 3
    }
}

// Caller sees ONLY: IllegalStateException("Problem in finally")
// The RuntimeException and IOException are COMPLETELY LOST
// This is a real debugging nightmare

// Safe pattern: don't throw from finally, do this instead
void safer() throws Exception {
    Exception primary = null;
    try {
        riskyOperation();
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
}
```

---

## Trap 4: "What is the return value if the finally block has a return statement?"

**What most people say:** The try block's return value is used.

**Correct answer:** The `finally` block's `return` **overwrites** the try block's `return`. This is one of the most dangerous anti-patterns in Java — it silently discards the try block's return value and even suppresses any pending exception.

```java
int tricky() {
    try {
        return 1; // step 1: return value 1 is staged
    } finally {
        return 2; // step 2: return value 2 OVERWRITES 1
    }
}
// tricky() returns 2 — the 1 is silently discarded

// Even worse — finally return swallows exceptions
int silentException() {
    try {
        throw new RuntimeException("I will be lost!");
    } finally {
        return 42; // exception is SILENTLY SWALLOWED
    }
}
// Returns 42, the RuntimeException is completely gone
// This is extremely dangerous in production code

// Rule: NEVER put a return statement in a finally block
// Modern IDEs (IntelliJ, SonarQube) flag this as a warning
```

---

## Trap 5: "Can you catch Error in Java?"

**What most people say:** No, Errors represent JVM-level problems and cannot be caught.

**Correct answer:** Technically you **can** catch `Error` — it compiles and runs. The question is whether you **should**, and the answer is almost never. However, there are legitimate narrow use cases:

```java
// You CAN do this (compiles and runs)
try {
    recursiveMethod();
} catch (StackOverflowError e) {
    System.out.println("Stack overflow caught — this is actually OK in controlled tests");
}

// Legitimate use: test frameworks catch AssertionError
try {
    assertEquals(expected, actual);
} catch (AssertionError e) {
    // Test framework catches this to record test failure (not re-throw)
}

// Legitimate use: top-level server loop to stay alive
public void serverLoop() {
    while (true) {
        try {
            processNextRequest();
        } catch (OutOfMemoryError e) {
            // Log it, trigger GC, alert ops — then decide to continue or exit
            log.error("OOM encountered", e);
            System.gc(); // hint, not guaranteed
            // Maybe: System.exit(1) is actually safer here
        } catch (Throwable t) {
            // Catch-all at top level to prevent silent thread death
            log.error("Unexpected error in server loop", t);
        }
    }
}
```

The key point: catching `OutOfMemoryError` is risky because the JVM state is now unknown — other threads may be in inconsistent states, class loading may be broken, etc. The safest response to `OutOfMemoryError` is usually to log and exit. `StackOverflowError` is safer to catch because it is localized to one thread's stack and does not indicate broader JVM instability.

---

## Trap 6: "How do you handle checked exceptions inside Java 8 lambda expressions?"

**What most people say:** You can't use checked exceptions in lambdas; wrap them in try-catch inside.

**Correct answer:** Java's `Function`, `Consumer`, `Supplier` etc. do not declare checked exceptions, so the compiler rejects checked exceptions in lambdas. You have several options:

```java
// Option 1: Inline try-catch (verbose but explicit)
List<String> urls = List.of("http://a.com", "http://b.com");
urls.stream()
    .map(url -> {
        try {
            return fetchContent(url); // throws IOException
        } catch (IOException e) {
            throw new UncheckedIOException(e); // wrap in unchecked
        }
    })
    .forEach(System.out::println);

// Option 2: Utility wrapper method — cleaner at call site
@FunctionalInterface
interface CheckedFunction<T, R> {
    R apply(T t) throws Exception;
}

static <T, R> Function<T, R> wrap(CheckedFunction<T, R> f) {
    return t -> {
        try {
            return f.apply(t);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    };
}

// Usage:
urls.stream()
    .map(wrap(url -> fetchContent(url))) // clean
    .forEach(System.out::println);

// Option 3: Sneaky throws (Lombok @SneakyThrows) — controversial
// Bypasses checked exception compilation check using type erasure
// The exception IS still thrown; the compiler just doesn't see it
// Not recommended for library code — surprises callers
```

The standard library's `UncheckedIOException` (Java 8+) is the idiomatic wrapper for `IOException` in streams.

---

## Trap 7: "What happens if an exception is thrown in a static initializer?"

**What most people say:** The class fails to load and you get a ClassCastException or similar.

**Correct answer:** If a static initializer throws an unchecked exception, the JVM wraps it in `ExceptionInInitializerError` on the first load attempt. All subsequent attempts to use that class throw `NoClassDefFoundError`, **not** the original exception. This is a debugging nightmare because the root cause is hidden.

```java
class BrokenClass {
    static {
        int x = 1 / 0; // throws ArithmeticException
    }
}

// First access:
try {
    new BrokenClass();
} catch (ExceptionInInitializerError e) {
    System.out.println("Cause: " + e.getCause()); // ArithmeticException: / by zero
}

// All SUBSEQUENT accesses (even in a different try block):
try {
    new BrokenClass();
} catch (NoClassDefFoundError e) {
    // Class is now in a failed state — can never be re-initialised in this JVM
    System.out.println("Class permanently broken: " + e.getMessage());
    // Root cause is NOT in this exception — you need to find the original EIIE
}
```

**Practical impact:** This affects Spring beans with expensive static initialisation (database drivers registering via `DriverManager`, static `Pattern.compile()` calls with invalid regex, static `Logger` with misconfigured appenders). Always use lazy initialisation and avoid throwing from static blocks.

---

## Trap 8: "Can you recover from StackOverflowError?"

**What most people say:** No, the JVM is done — nothing you can do.

**Correct answer:** `StackOverflowError` is actually one of the safer errors to catch because it only affects the thread's stack — the JVM heap and other threads are unaffected. After catching it, the stack frames from the recursive call are unwound, freeing stack space, so the thread can continue. However, you must be careful not to call deeply recursive code in the catch handler.

```java
void attemptRecursion(int depth) {
    try {
        infiniteRecurse(); // will eventually throw StackOverflowError
    } catch (StackOverflowError e) {
        // Stack is now partially unwound — safe to do simple operations
        System.out.println("Stack overflow caught at depth: " + depth);
        // DO NOT recurse here again
        // DO keep handlers simple and non-recursive
    }
}

// Real use: test that your recursive algorithm handles deep input
@Test
void testDeepRecursion() {
    try {
        deepParse(generateDeeplyNestedInput(100_000));
    } catch (StackOverflowError e) {
        fail("Parser is not tail-recursive or iterative — blew the stack");
    }
}
```

The fix for production code is always to convert recursive algorithms to iterative ones using an explicit stack (`Deque`), or to use trampoline/continuation patterns.

---

## Trap 9: "How do suppressed exceptions work in try-with-resources?"

**What most people say:** There is no suppression — the last exception thrown wins.

**Correct answer:** try-with-resources uses `Throwable.addSuppressed()` to preserve exceptions thrown during `close()` when the try body also threw an exception. This was specifically designed to solve the "exception swallowing" problem of the old `finally` pattern.

```java
class LeakyResource implements AutoCloseable {
    @Override
    public void close() throws Exception {
        throw new Exception("Error during close");
    }
}

try (LeakyResource r = new LeakyResource()) {
    throw new RuntimeException("Primary error in body");
}

// What you see in the stack trace:
// RuntimeException: Primary error in body       ← PRIMARY (body exception)
//     at ...
//     Suppressed: java.lang.Exception: Error during close  ← SUPPRESSED (close exception)
//         at LeakyResource.close(...)

// Accessing suppressed exceptions programmatically:
try {
    // ...
} catch (RuntimeException e) {
    Throwable[] suppressed = e.getSuppressed(); // array of suppressed exceptions
    for (Throwable s : suppressed) {
        log.warn("Suppressed during cleanup: {}", s.getMessage());
    }
    throw e;
}
```

The body exception is primary because it represents the original failure. The close exception is secondary — it happened as a consequence of the cleanup during an already-failed operation.

---

## Trap 10: "What is the performance cost of exceptions in Java?"

**What most people say:** Exceptions are expensive to throw so you should avoid them in hot paths.

**Correct answer:** The expensive part is **not** throwing the exception — it is **creating** the exception object, specifically the call to `fillInStackTrace()` which walks the entire call stack and captures it. In tight loops, this can be 100x slower than normal control flow.

```java
// SLOW: creating exception in a hot loop (captures stack trace every time)
for (int i = 0; i < 1_000_000; i++) {
    try {
        Integer.parseInt(input[i]); // throws NumberFormatException for bad input
    } catch (NumberFormatException e) {
        // even though caught immediately, stack trace was captured
        badInputCount++;
    }
}

// FASTER: pre-validate instead of relying on exception-driven control flow
for (int i = 0; i < 1_000_000; i++) {
    if (isValidInteger(input[i])) {
        int val = Integer.parseInt(input[i]);
    } else {
        badInputCount++;
    }
}

// TRICK: override fillInStackTrace() for exceptions used in control flow
class FastControlFlowException extends RuntimeException {
    FastControlFlowException(String msg) { super(msg); }

    @Override
    public synchronized Throwable fillInStackTrace() {
        return this; // do nothing — no stack trace captured
    }
}
// Creating this exception is ~10-50x cheaper than a standard exception
// Used internally by some frameworks (e.g. Spring's StopIteration-style signals)
```

**JIT optimisation note:** The JVM can sometimes detect frequently-thrown exceptions and optimise them, but this is JVM-version dependent and not something you should rely on. The general rule: exceptions should be exceptional — not used for flow control in performance-sensitive code.

---

## Trap 11: "What happens to exception information when you do: throw new RuntimeException(e.getMessage())?"

**What most people say:** It rethrows the exception with the original message — that is fine.

**Correct answer:** This is an anti-pattern that permanently loses the original exception's stack trace, class type, and all context. You only preserve the message string. The correct pattern is to pass the original exception as the cause.

```java
// WRONG — loses stack trace, original type, full context
try {
    dao.findUser(id);
} catch (SQLException e) {
    throw new ServiceException(e.getMessage()); // throws away everything useful
}

// CORRECT — full causal chain preserved
try {
    dao.findUser(id);
} catch (SQLException e) {
    throw new ServiceException("Failed to find user with id: " + id, e); // cause preserved
}

// Also wrong — re-throwing as String
} catch (Exception e) {
    throw new RuntimeException("Error: " + e.toString()); // still loses stack trace
}

// Logging and rethrowing — also a common mistake
} catch (Exception e) {
    log.error("Error occurred: " + e.getMessage()); // only logs message, not trace
    throw e; // stack trace IS preserved here, but log was incomplete
}

// Correct logging
} catch (Exception e) {
    log.error("Failed to find user with id: {}", id, e); // SLF4J logs full stack trace
    throw new ServiceException("User lookup failed", e);
}
```

---

## Trap 12: "In a catch block, if you call a method that throws an unchecked exception, does it propagate correctly?"

**What most people say:** Yes, unchecked exceptions always propagate.

**Correct answer:** Yes, but the real trap is about **exception masking** when you log-and-rethrow the original exception but then code in the catch block or finally throws a second exception — and both need to be preserved. A subtler version: if your `catch` block calls a method that throws, the original caught exception is replaced.

```java
void process() {
    try {
        riskyWork();
    } catch (IOException e) {
        cleanupThatMightFail(); // throws NullPointerException!
        // ^ if this throws, IOException 'e' is LOST
        // caller sees NullPointerException, not IOException
        throw new ProcessingException("Work failed", e); // never reached
    }
}

// Correct: handle potential secondary exceptions
void processSafe() {
    IOException primaryCause = null;
    try {
        riskyWork();
    } catch (IOException e) {
        primaryCause = e;
    }

    if (primaryCause != null) {
        try {
            cleanupThatMightFail();
        } catch (Exception cleanupEx) {
            primaryCause.addSuppressed(cleanupEx);
        }
        throw new ProcessingException("Work failed", primaryCause);
    }
}
```
