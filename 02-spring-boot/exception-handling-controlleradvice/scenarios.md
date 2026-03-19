# Exception Handling & @ControllerAdvice — Scenarios

> 18+ real-world scenarios covering @ControllerAdvice, custom exceptions, and error response design.

---

## Scenario 1: Global Exception Handler with @ControllerAdvice

**Problem:** You need a single place to handle all exceptions thrown by any controller in your Spring Boot application.

```java
@ControllerAdvice  // or @RestControllerAdvice (adds @ResponseBody)
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // Handle specific custom exception
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex,
                                                    HttpServletRequest request) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ApiError.of(404, "Not Found", ex.getMessage(), request.getRequestURI()));
    }

    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex,
                                                      HttpServletRequest request) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .collect(Collectors.toList());

        ApiError error = ApiError.builder()
            .status(400)
            .error("Validation Failed")
            .message("One or more fields are invalid")
            .path(request.getRequestURI())
            .fieldErrors(errors)
            .build();

        return ResponseEntity.badRequest().body(error);
    }

    // Handle all other exceptions (fallback)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleAll(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error on {}", request.getRequestURI(), ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiError.of(500, "Internal Server Error",
                "An unexpected error occurred", request.getRequestURI()));
    }
}
```

---

## Scenario 2: Custom Exception Hierarchy

**Problem:** You need a well-structured exception hierarchy that carries HTTP status codes and error details:

```java
// Base exception
public abstract class ApiException extends RuntimeException {

    private final HttpStatus status;
    private final String errorCode;

    protected ApiException(String message, HttpStatus status, String errorCode) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
    }

    public HttpStatus getStatus() { return status; }
    public String getErrorCode() { return errorCode; }
}

// 404
public class ResourceNotFoundException extends ApiException {
    public ResourceNotFoundException(String resourceType, Object id) {
        super(resourceType + " not found with id: " + id,
              HttpStatus.NOT_FOUND, "RESOURCE_NOT_FOUND");
    }
}

// 409
public class DuplicateResourceException extends ApiException {
    public DuplicateResourceException(String message) {
        super(message, HttpStatus.CONFLICT, "DUPLICATE_RESOURCE");
    }
}

// 422
public class BusinessRuleViolationException extends ApiException {
    public BusinessRuleViolationException(String message) {
        super(message, HttpStatus.UNPROCESSABLE_ENTITY, "BUSINESS_RULE_VIOLATION");
    }
}

// Usage in service
public Order placeOrder(OrderRequest request) {
    if (inventoryService.getStock(request.getProductId()) < request.getQuantity()) {
        throw new BusinessRuleViolationException(
            "Insufficient stock for product: " + request.getProductId());
    }
    // ...
}

// Single handler for all ApiException subclasses
@ExceptionHandler(ApiException.class)
public ResponseEntity<ApiError> handleApiException(ApiException ex,
                                                    HttpServletRequest request) {
    return ResponseEntity.status(ex.getStatus())
        .body(ApiError.of(ex.getStatus().value(), ex.getErrorCode(),
                         ex.getMessage(), request.getRequestURI()));
}
```

---

## Scenario 3: Handling Constraint Violations (@Validated on Service Layer)

**Problem:** You use `@Validated` on a service method, and Bean Validation throws `ConstraintViolationException` (different from `MethodArgumentNotValidException`):

```java
// MethodArgumentNotValidException: @Valid in @RequestBody, @ModelAttribute
// ConstraintViolationException: @Validated on @Service methods or @PathVariable

@ExceptionHandler(ConstraintViolationException.class)
public ResponseEntity<ApiError> handleConstraintViolation(ConstraintViolationException ex,
                                                           HttpServletRequest request) {
    List<FieldError> errors = ex.getConstraintViolations()
        .stream()
        .map(cv -> {
            String field = cv.getPropertyPath().toString();
            // Remove method name prefix: "methodName.paramName" → "paramName"
            if (field.contains(".")) {
                field = field.substring(field.lastIndexOf(".") + 1);
            }
            return new FieldError(field, cv.getMessage());
        })
        .collect(Collectors.toList());

    return ResponseEntity.badRequest()
        .body(ApiError.builder()
            .status(400)
            .error("Validation Failed")
            .fieldErrors(errors)
            .build());
}
```

---

## Scenario 4: Problem Details Standard (RFC 7807)

**Problem:** Your API needs to follow the RFC 7807 "Problem Details for HTTP APIs" standard for interoperability.

```java
// Spring 6 / Spring Boot 3 includes ProblemDetail support natively

@ExceptionHandler(ResourceNotFoundException.class)
public ProblemDetail handleNotFound(ResourceNotFoundException ex,
                                    HttpServletRequest request) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(
        HttpStatus.NOT_FOUND, ex.getMessage());
    problem.setTitle("Resource Not Found");
    problem.setType(URI.create("https://api.example.com/errors/resource-not-found"));
    problem.setProperty("timestamp", Instant.now());
    problem.setProperty("requestId", MDC.get("requestId"));
    return problem;
}

// Response:
// {
//   "type": "https://api.example.com/errors/resource-not-found",
//   "title": "Resource Not Found",
//   "status": 404,
//   "detail": "Order not found with id: 123",
//   "timestamp": "2024-01-15T10:30:00Z",
//   "requestId": "abc-123-xyz"
// }
```

---

## Scenario 5: Exception Handler Ordering

**Problem:** You have two `@ControllerAdvice` classes — one generic and one specific to a module. The generic one must handle `RuntimeException` but the module-specific one must handle it first for that module:

```java
// Higher @Order value = lower priority (runs later)
@ControllerAdvice
@Order(Ordered.LOWEST_PRECEDENCE)  // Low priority — fallback handler
public class GlobalExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<ApiError> handleRuntime(RuntimeException ex, ...) {
        // Generic fallback
    }
}

@ControllerAdvice(basePackages = "com.example.payments")
@Order(1)  // High priority — handles payments module first
public class PaymentExceptionHandler {

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<ApiError> handlePaymentRuntime(RuntimeException ex, ...) {
        // Payment-specific error handling
    }
}
```

**How handler resolution works:**
1. Spring finds all `@ExceptionHandler` methods that match the exception type
2. Applies ordering to select the most specific/highest priority handler
3. More specific exception types beat more general ones
4. `@Order` / `Ordered` breaks ties

---

## Scenario 6: Async Exceptions and CompletableFuture

**Problem:** Your `@ControllerAdvice` doesn't catch exceptions from `@Async` methods or `CompletableFuture`:

```java
// This does NOT work for async exceptions:
@GetMapping("/async-report")
public CompletableFuture<ResponseEntity<Report>> getReport() {
    return reportService.generateAsync()
        .thenApply(report -> ResponseEntity.ok(report));
    // If generateAsync() throws, @ControllerAdvice won't catch it!
}

// Fix: Handle exceptions in the CompletableFuture chain
@GetMapping("/async-report")
public CompletableFuture<ResponseEntity<Report>> getReport(HttpServletRequest request) {
    return reportService.generateAsync()
        .thenApply(report -> ResponseEntity.ok(report))
        .exceptionally(throwable -> {
            Throwable cause = throwable.getCause() != null ? throwable.getCause() : throwable;
            if (cause instanceof ResourceNotFoundException) {
                return ResponseEntity.notFound().<Report>build();
            }
            log.error("Async report generation failed", cause);
            return ResponseEntity.status(500).<Report>build();
        });
}
```

---

## Scenario 7: Logging with Correlation IDs

**Problem:** When an error occurs, logs from different services don't correlate. You need to track a request across multiple service calls.

```java
@Component
public class CorrelationIdFilter implements Filter {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String MDC_KEY = "correlationId";

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String correlationId = request.getHeader(CORRELATION_ID_HEADER);
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }

        MDC.put(MDC_KEY, correlationId);
        response.setHeader(CORRELATION_ID_HEADER, correlationId);

        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove(MDC_KEY);  // Clean up to prevent memory leaks with thread pools
        }
    }
}

// In exception handler, include the correlation ID:
@ExceptionHandler(Exception.class)
public ResponseEntity<ApiError> handleAll(Exception ex, HttpServletRequest request) {
    log.error("Error processing request {}: {}", MDC.get("correlationId"), request.getRequestURI(), ex);

    return ResponseEntity.status(500)
        .body(ApiError.builder()
            .status(500)
            .message("An unexpected error occurred")
            .correlationId(MDC.get("correlationId"))  // Include in response
            .build());
}
```

---

## Scenario 8: Hiding Internal Exception Details

**Problem:** Your 500 errors leak stack traces and internal implementation details to clients:

```java
// BAD: Exposes internal details
@ExceptionHandler(Exception.class)
public ResponseEntity<Map<String, String>> handleAll(Exception ex) {
    return ResponseEntity.status(500).body(Map.of(
        "error", ex.getMessage(),           // May expose SQL, class names, etc.
        "exception", ex.getClass().getName() // Exposes implementation!
    ));
}

// GOOD: Generic message, internal details only in logs
@ExceptionHandler(Exception.class)
public ResponseEntity<ApiError> handleAll(Exception ex, HttpServletRequest request) {
    String correlationId = MDC.get("correlationId");

    // Log full details internally
    log.error("Unhandled exception [correlationId={}] on {}: {}",
              correlationId, request.getRequestURI(), ex.getMessage(), ex);

    // Return safe message to client
    return ResponseEntity.status(500).body(ApiError.builder()
        .status(500)
        .error("Internal Server Error")
        .message("An unexpected error occurred. Reference: " + correlationId)
        // No stack trace, no SQL, no class names
        .build());
}
```

---

## Scenario 9: @ExceptionHandler Method Resolution

**Problem:** An exception extends `IllegalArgumentException` which extends `RuntimeException`. You have handlers for both. Which runs?

```java
@ControllerAdvice
public class Handler {

    @ExceptionHandler(RuntimeException.class)
    public ResponseEntity<?> handleRuntime(RuntimeException ex) {
        return ResponseEntity.status(500).build();
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<?> handleIllegalArg(IllegalArgumentException ex) {
        return ResponseEntity.badRequest().build();
    }
}

// Exception thrown: new ProductNotFoundException("Product not found")
// ProductNotFoundException extends IllegalArgumentException
```

**Answer:** The `IllegalArgumentException` handler runs because Spring picks the most specific matching handler. The exception hierarchy is checked and the handler for the most-specific matching type wins.

Order of resolution:
1. Exact type match
2. Nearest parent match in class hierarchy
3. `@Order` / priority breaks ties

---

## Scenario 10: Testing Exception Handlers

```java
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @Test
    void shouldReturn404WhenProductNotFound() throws Exception {
        when(productService.findById(999L))
            .thenThrow(new ResourceNotFoundException("Product", 999L));

        mockMvc.perform(get("/api/products/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.status").value(404))
            .andExpect(jsonPath("$.error").value("Not Found"))
            .andExpect(jsonPath("$.message").value("Product not found with id: 999"));
    }

    @Test
    void shouldReturn400ForInvalidRequest() throws Exception {
        String invalidJson = """
            {"name": "", "price": -1}
            """;

        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidJson))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.status").value(400))
            .andExpect(jsonPath("$.fieldErrors[?(@.field=='name')]").exists())
            .andExpect(jsonPath("$.fieldErrors[?(@.field=='price')]").exists());
    }
}
```
