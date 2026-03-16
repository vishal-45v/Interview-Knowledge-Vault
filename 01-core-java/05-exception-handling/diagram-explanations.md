# Exception Handling — ASCII Diagram Explanations

---

## 1. Java Exception Hierarchy

```
                        java.lang.Object
                               │
                        java.lang.Throwable
                        ┌──────┴───────────┐
                        │                  │
                   java.lang.Error    java.lang.Exception
                        │                  │
          ┌─────────────┤          ┌───────┴──────────────────────┐
          │             │          │                              │
   OutOfMemory-    StackOverflow-  RuntimeException         (Checked Exceptions)
   Error           Error               │                          │
                                  ┌────┴───────────────┐    IOException
          AssertionError          │          │          │         │
                                  │          │          │    FileNotFound-
                          NullPointer-  IllegalArg-  ClassCast-   Exception
                          Exception    umentException Exception
                                                               SQLException
                          IndexOutOf-  ArithmeticEx- (needs throws or try-catch)
                          BoundsExcep- ception
                          tion
                                  UnsupportedOp-
                          Concurrent-  erationExcep-
                          Modification  tion
                          Exception

  Legend:
  ┌──────────────────────────────────────────────────────────────┐
  │  Error              → Do NOT catch (JVM-level, unrecoverable)│
  │  RuntimeException   → Unchecked (compiler does not enforce)  │
  │  Exception          → Checked (compiler enforces handling)   │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. try-catch-finally Execution Flow

```
  ┌─────────────────────────────────────────────────────────────┐
  │  try block starts                                           │
  │                                                             │
  │  Code line 1  ──────────────────────► executes OK          │
  │  Code line 2  ──► EXCEPTION THROWN                         │
  │  Code line 3  ←── (SKIPPED — never reached)                │
  └──────────────────────┬──────────────────────────────────────┘
                         │ exception object
                         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  catch (SpecificException e)                                 │
  │    Does exception type match?                                │
  │    YES ──► run catch block ──────────────────────────────►   │
  │    NO  ──► try next catch block                              │
  │            No match found? ──► exception propagates up stack │
  └──────────────────────────────────────────────────────────────┘
                         │ (after catch block OR no catch matched)
                         ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  finally block                                               │
  │  ALWAYS runs (cleanup: close streams, release locks)         │
  │  Runs even if:                                               │
  │    - try completed normally                                  │
  │    - catch handled the exception                             │
  │    - exception propagated (no matching catch)                │
  │    - catch block itself threw another exception              │
  │    - return statement in try or catch                        │
  └──────────────────────────────────────────────────────────────┘
                         │
                         ▼
  Normal flow continues (if exception was caught and not rethrown)
  OR exception propagates to caller (if not caught)

  ┌──────────────────────────────────────────────────────────────┐
  │  EXCEPTION: finally does NOT run when:                       │
  │  1. System.exit() called inside try or catch                 │
  │  2. JVM crashes (SIGSEGV, OutOfMemoryError during GC)        │
  │  3. Infinite loop or deadlock inside try/catch               │
  │  4. Thread.stop() called (deprecated, dangerous)             │
  └──────────────────────────────────────────────────────────────┘
```

---

## 3. try-with-resources Cleanup Flow

```
  try (Resource r1 = new R1();    ← resources opened in order
       Resource r2 = new R2()) {
      // use r1 and r2
  }

  ┌─────────────────────────────────────────────────────────────┐
  │  Equivalent expanded bytecode (what compiler generates):    │
  │                                                             │
  │  Resource r1 = new R1();                                    │
  │  Throwable primaryEx = null;                                │
  │  try {                                                      │
  │      Resource r2 = new R2();                                │
  │      try {                                                  │
  │          // body                                            │
  │      } catch (Throwable t) {                                │
  │          primaryEx = t;                                     │
  │          throw t;                                           │
  │      } finally {                                            │
  │          if (primaryEx != null) {                           │
  │              try { r2.close(); }                            │
  │              catch (Throwable t) {                          │
  │                  primaryEx.addSuppressed(t); ← key!         │
  │              }                                              │
  │          } else { r2.close(); }                             │
  │      }                                                      │
  │  } finally {                                                │
  │      // same pattern for r1.close()                         │
  │  }                                                          │
  └─────────────────────────────────────────────────────────────┘

  Closing order:
  ┌─────────────────────────────────────────────────────────────┐
  │  Opened:  R1 → R2 → R3                                      │
  │  Closed:  R3 → R2 → R1  (reverse order, like a stack)       │
  └─────────────────────────────────────────────────────────────┘

  Exception handling in close():
  ┌─────────────────────────────────────────────────────────────┐
  │  try body throws ExA                                        │
  │  r2.close() throws ExB                                      │
  │                                                             │
  │  Result: ExA is PRIMARY exception (thrown to caller)        │
  │          ExB is SUPPRESSED: ExA.getSuppressed() → [ExB]     │
  │                                                             │
  │  Without try-with-resources (old finally{} approach):       │
  │  ExB would REPLACE ExA — original cause LOST forever        │
  └─────────────────────────────────────────────────────────────┘
```

---

## 4. Exception Propagation Up the Call Stack

```
  Thread call stack (grows downward):
  ────────────────────────────────────────────────────────────

  main()
    └─ calls serviceA()
         └─ calls serviceB()
              └─ calls dao.findById()
                   └─ calls connection.executeQuery()
                        │
                        │  SQL network error occurs here
                        ▼
                   SQLException thrown
                        │
                        │  dao.findById() has no try-catch for SQLException
                        ▼
              Exception propagates up to serviceB()
                        │
                        │  serviceB() catches it:
                        │  catch (SQLException e) {
                        │      throw new DataAccessException("DB error", e);
                        │  }
                        ▼
         DataAccessException thrown (wraps original cause)
                        │
                        │  serviceA() has no handler — propagates
                        ▼
  main() receives DataAccessException
    └─ catch (DataAccessException e) {
           log.error("Failed", e); // full chain visible in stack trace
           return ResponseEntity.status(500).build();
       }

  ────────────────────────────────────────────────────────────
  Stack trace printed by e.printStackTrace():

  DataAccessException: DB error
      at ServiceB.findUser(ServiceB.java:42)
      at ServiceA.getProfile(ServiceA.java:28)
      at Main.main(Main.java:15)
  Caused by: java.sql.SQLException: Connection timed out
      at Dao.findById(Dao.java:77)
      at ServiceB.findUser(ServiceB.java:39)
      ... 2 more
```

---

## 5. Checked vs Unchecked Decision Tree

```
  You are designing a new exception class. Which should it be?
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Is this caused by a programming bug / misuse of API?           │
  │  (null passed where non-null required, index out of bounds,      │
  │   illegal state, contract violation)                             │
  │                                                                  │
  │        YES ──────────────────────────────────────────────►      │
  │                                               RuntimeException   │
  │                                               (UNCHECKED)        │
  │        NO                                     e.g. NPE,          │
  │         │                                     IllegalArgEx,      │
  │         │                                     IllegalStateEx     │
  │         ▼                                                        │
  │  Is recovery possible and MEANINGFUL for the caller?             │
  │  Can the caller realistically do something useful with it?       │
  │                                                                  │
  │        NO ───────────────────────────────────────────────►       │
  │                                               RuntimeException   │
  │                                               (UNCHECKED)        │
  │        YES                                    e.g. config error  │
  │         │                                     on startup         │
  │         ▼                                                        │
  │  Is this a well-defined, anticipated external condition          │
  │  (file not found, network error, database unavailable)?          │
  │                                                                  │
  │        YES ──────────────────────────────────────────────►       │
  │                                               Exception          │
  │                                               (CHECKED)          │
  │        NO                                     e.g. IOException,  │
  │         │                                     SQLException       │
  │         │                                                        │
  │         ▼                                                        │
  │  Default: use RuntimeException (unchecked)                       │
  └──────────────────────────────────────────────────────────────────┘

  Practical modern guidance (Effective Java, Item 71):
  ┌──────────────────────────────────────────────────────────────────┐
  │  Use CHECKED when:                                               │
  │    - Caller can and should handle the condition                  │
  │    - External resources (files, DB, network) that can fail       │
  │    - The failure is recoverable (retry, fallback, user prompt)   │
  │                                                                  │
  │  Use UNCHECKED when:                                             │
  │    - It is a programming error (bug)                             │
  │    - You are writing a framework/library (checked forces users   │
  │      to handle what they often cannot meaningfully handle)       │
  │    - The exception will propagate many layers without being      │
  │      handled (checked forces ugly throws declarations everywhere)│
  └──────────────────────────────────────────────────────────────────┘
```
