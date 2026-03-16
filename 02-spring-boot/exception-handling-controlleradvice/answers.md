# Exception Handling — Structured Answers

---

## Q1: @ControllerAdvice vs @RestControllerAdvice

@RestControllerAdvice = @ControllerAdvice + @ResponseBody

Use @RestControllerAdvice when all your @ExceptionHandler methods return response body objects (typical for REST APIs).

Use @ControllerAdvice when some handlers might return ModelAndView (traditional MVC).

---

## Q2: @ExceptionHandler Scope

An @ExceptionHandler method in a @Controller class only handles exceptions from THAT controller.

An @ExceptionHandler method in a @ControllerAdvice class handles exceptions from ALL controllers (globally).

If both exist for the same exception type, the controller-local handler takes priority over @ControllerAdvice.

---

## Q3: What Is ResponseEntityExceptionHandler?

Spring provides ResponseEntityExceptionHandler as a convenient base class for @ControllerAdvice. It handles all standard Spring MVC exceptions (MethodArgumentNotValidException, MissingServletRequestParameterException, etc.) and returns appropriate HTTP status codes.

Extend it to add your custom exception handlers while keeping the default Spring exception handling.

---

## Q4: Exception vs Error — Which Should You Catch?

Never catch Error (OutOfMemoryError, StackOverflowError) — these represent JVM-level failures that you cannot recover from. Let them propagate.

Catch Exception (or specific RuntimeException subclasses) for application-level error handling.

In @ControllerAdvice, use @ExceptionHandler(Exception.class) as a fallback, but handle specific exceptions first.

---

## Q5: Checked vs Unchecked Exceptions in Spring Services

Spring best practice: Use unchecked exceptions (RuntimeException subclasses) in service layers. Don't force callers to handle checked exceptions with try/catch — that clutters the code.

Custom business exceptions (ResourceNotFoundException, InsufficientStockException) should extend RuntimeException.

The service layer translates data access exceptions (DataAccessException is already unchecked in Spring) into domain exceptions.
