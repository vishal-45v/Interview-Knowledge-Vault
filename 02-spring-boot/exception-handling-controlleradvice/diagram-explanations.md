# Exception Handling — Diagram Explanations

---

## Diagram 1: Exception Flow Through Spring MVC

```
  @RestController method throws Exception
           │
           ▼
  DispatcherServlet.processHandlerException()
           │
           ▼
  ┌──────────────────────────────────────────────────────────┐
  │  HandlerExceptionResolverComposite                        │
  │                                                           │
  │  1. ExceptionHandlerExceptionResolver                     │
  │     └── Check controller @ExceptionHandler first          │
  │     └── Check @ControllerAdvice @ExceptionHandler         │
  │     └── Match by most specific exception type             │
  │                         │                                 │
  │  2. ResponseStatusExceptionResolver                        │
  │     └── Check @ResponseStatus on exception class          │
  │                         │                                 │
  │  3. DefaultHandlerExceptionResolver                        │
  │     └── Handle standard Spring MVC exceptions             │
  │         (TypeMismatch, MethodNotAllowed, etc.)             │
  └──────────────────────────────────────────────────────────┘
           │
           ▼
  Response written to client
```

---

## Diagram 2: Exception Hierarchy Design

```
  Throwable
  ├── Error  (don't catch — JVM failures)
  │   ├── OutOfMemoryError
  │   └── StackOverflowError
  │
  └── Exception
      ├── IOException (checked — must handle)
      │
      └── RuntimeException (unchecked — Spring style)
          │
          └── ApiException (base for all business exceptions)
              ├── ResourceNotFoundException → 404 Not Found
              ├── DuplicateResourceException → 409 Conflict
              ├── BusinessRuleViolationException → 422 Unprocessable Entity
              ├── UnauthorizedException → 401 Unauthorized
              └── ForbiddenException → 403 Forbidden
```

---

## Diagram 3: @ControllerAdvice Scope

```
  @ControllerAdvice Scope Options:

  @ControllerAdvice
  → Applies to ALL controllers

  @ControllerAdvice(basePackages = "com.example.api")
  → Only controllers in that package

  @ControllerAdvice(assignableTypes = {ProductController.class, OrderController.class})
  → Only those specific controllers

  @ControllerAdvice(annotations = RestController.class)
  → Only @RestController classes

  Priority: Local @ExceptionHandler > @ControllerAdvice (high @Order) > @ControllerAdvice (low @Order)
```
