# Exception Handling — Trap Questions

---

## Trap 1: @ExceptionHandler for Checked Exceptions Without throws Declaration

**Question:** Does this work?

```java
@ExceptionHandler(IOException.class)
public ResponseEntity<ApiError> handleIO(IOException ex) { ... }
```

**Answer:** Yes, @ExceptionHandler can handle any exception type, checked or unchecked, regardless of method signatures. The exception doesn't need to be declared in the handler method signature. Spring routes exceptions to handlers by type matching.

---

## Trap 2: @ControllerAdvice Not Catching Exceptions from @Service

**Question:** @ControllerAdvice doesn't catch exceptions from my @Service. Why?

**Answer:** @ControllerAdvice only intercepts exceptions that propagate to the DispatcherServlet from controllers. If an exception is thrown in a @Service but caught inside the @Controller (in a try-catch), @ControllerAdvice never sees it.

Also, @ControllerAdvice does NOT catch exceptions from:
- @Async methods (different thread)
- Filter/Interceptor exceptions (before/after DispatcherServlet)
- Scheduler (@Scheduled) methods

---

## Trap 3: Filter Exceptions vs ControllerAdvice

**Question:** You throw an exception in a Spring Security filter. Will @ControllerAdvice handle it?

**Answer:** No. Filters run before the DispatcherServlet. @ControllerAdvice is inside the DispatcherServlet. Exceptions from filters bypass @ControllerAdvice entirely.

Handle filter exceptions by writing directly to HttpServletResponse, or delegate to Spring Security's AuthenticationEntryPoint/AccessDeniedHandler.

---

## Trap 4: Returning void from @ExceptionHandler

**Question:** Can an @ExceptionHandler method return void?

**Answer:** Yes — if you write directly to the HttpServletResponse. But this bypasses HttpMessageConverters and is harder to test. Prefer returning ResponseEntity or ProblemDetail.

---

## Trap 5: @ResponseStatus on Custom Exception

**Question:** What does this do?

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ProductNotFoundException extends RuntimeException { }
```

**Answer:** Spring's ResponseStatusExceptionResolver detects @ResponseStatus on thrown exceptions and uses that status code. No @ExceptionHandler method is needed.

Limitation: Cannot customize the response body — only status code and reason phrase. For custom response body, use @ExceptionHandler in @ControllerAdvice.
