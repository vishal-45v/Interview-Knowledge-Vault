# REST API Design — Scenarios

> 20+ real-world REST API design scenarios covering versioning, HATEOAS, error handling, and production patterns.

---

## Scenario 1: API Versioning Strategy

**Problem:** Your REST API is used by 50+ clients. You need to introduce breaking changes to the `/products` endpoint. Which versioning strategy do you use?

**Option A: URL Path Versioning (Most Common)**

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductControllerV1 {

    @GetMapping("/{id}")
    public ProductV1Response getProduct(@PathVariable Long id) {
        // Old response format
    }
}

@RestController
@RequestMapping("/api/v2/products")
public class ProductControllerV2 {

    @GetMapping("/{id}")
    public ProductV2Response getProduct(@PathVariable Long id) {
        // New response format with additional fields
    }
}

// Clients call: GET /api/v1/products/123 OR GET /api/v2/products/123
```

**Option B: Header Versioning**

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping(value = "/{id}", headers = "API-Version=1")
    public ProductV1Response getProductV1(@PathVariable Long id) { ... }

    @GetMapping(value = "/{id}", headers = "API-Version=2")
    public ProductV2Response getProductV2(@PathVariable Long id) { ... }
}

// Client sends: GET /api/products/123 with header: API-Version: 2
```

**Option C: Content Negotiation (Media Type)**

```java
@GetMapping(value = "/{id}", produces = "application/vnd.company.product-v1+json")
public ProductV1Response getProductV1(@PathVariable Long id) { ... }

@GetMapping(value = "/{id}", produces = "application/vnd.company.product-v2+json")
public ProductV2Response getProductV2(@PathVariable Long id) { ... }

// Client sends: Accept: application/vnd.company.product-v2+json
```

**Recommendation:** URL path versioning is most visible, cacheable, and debuggable. Use it unless you have a strong reason to prefer header versioning.

---

## Scenario 2: Proper HTTP Status Codes for Different Outcomes

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        Order order = orderService.create(request);
        URI location = URI.create("/api/orders/" + order.getId());
        return ResponseEntity
            .created(location)        // 201 Created with Location header
            .body(new OrderResponse(order));
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(order -> ResponseEntity.ok(new OrderResponse(order)))  // 200 OK
            .orElse(ResponseEntity.notFound().build());  // 404 Not Found
    }

    @PutMapping("/{id}")
    public ResponseEntity<OrderResponse> updateOrder(
            @PathVariable Long id,
            @Valid @RequestBody UpdateOrderRequest request) {
        if (!orderService.exists(id)) {
            return ResponseEntity.notFound().build();  // 404
        }
        Order updated = orderService.update(id, request);
        return ResponseEntity.ok(new OrderResponse(updated));  // 200 OK
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        if (!orderService.exists(id)) {
            return ResponseEntity.notFound().build();  // 404
        }
        orderService.delete(id);
        return ResponseEntity.noContent().build();  // 204 No Content
    }

    @PatchMapping("/{id}/status")
    public ResponseEntity<OrderResponse> updateStatus(
            @PathVariable Long id,
            @RequestBody UpdateStatusRequest request) {
        try {
            Order updated = orderService.updateStatus(id, request.getStatus());
            return ResponseEntity.ok(new OrderResponse(updated));
        } catch (InvalidStatusTransitionException e) {
            return ResponseEntity.unprocessableEntity().build();  // 422
        }
    }
}
```

---

## Scenario 3: Consistent Error Response Format

**Problem:** Your API returns inconsistent error formats. Some endpoints return plain strings, others return JSON with different structures.

**Solution — Standardized error response:**

```java
// Error response DTO
@JsonInclude(JsonInclude.Include.NON_NULL)
public class ApiError {

    private final int status;
    private final String error;
    private final String message;
    private final String path;
    private final Instant timestamp;
    private List<FieldError> fieldErrors;

    // For validation errors
    @Data
    public static class FieldError {
        private final String field;
        private final String rejectedValue;
        private final String message;
    }
}

@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex,
                                                    HttpServletRequest request) {
        ApiError error = ApiError.builder()
            .status(404)
            .error("Not Found")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .build();
        return ResponseEntity.status(404).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex,
                                                      HttpServletRequest request) {
        List<ApiError.FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fe -> new ApiError.FieldError(
                fe.getField(),
                String.valueOf(fe.getRejectedValue()),
                fe.getDefaultMessage()))
            .collect(Collectors.toList());

        ApiError error = ApiError.builder()
            .status(400)
            .error("Validation Failed")
            .message("Request validation failed")
            .path(request.getRequestURI())
            .timestamp(Instant.now())
            .fieldErrors(fieldErrors)
            .build();
        return ResponseEntity.badRequest().body(error);
    }
}
```

---

## Scenario 4: Pagination and Sorting

**Problem:** The `/products` endpoint returns all products, causing timeouts for large datasets.

```java
@GetMapping("/products")
public ResponseEntity<Page<ProductResponse>> getProducts(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id") String sortBy,
        @RequestParam(defaultValue = "asc") String direction) {

    // Validate size to prevent abuse
    if (size > 100) {
        throw new BadRequestException("Maximum page size is 100");
    }

    Sort sort = direction.equalsIgnoreCase("desc")
        ? Sort.by(sortBy).descending()
        : Sort.by(sortBy).ascending();

    Pageable pageable = PageRequest.of(page, size, sort);
    Page<Product> products = productRepository.findAll(pageable);

    return ResponseEntity.ok(products.map(ProductResponse::from));
}

// Response includes pagination metadata:
// {
//   "content": [...products...],
//   "totalElements": 523,
//   "totalPages": 27,
//   "number": 0,         ← current page
//   "size": 20,
//   "first": true,
//   "last": false
// }
```

---

## Scenario 5: Idempotency for POST Requests

**Problem:** A client retries a payment request due to network timeout. You get duplicate payments.

**Solution — Idempotency Key:**

```java
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> createPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @Valid @RequestBody CreatePaymentRequest request) {

    // Check if we've seen this idempotency key before
    Optional<PaymentResponse> existingResult = idempotencyStore.get(idempotencyKey);
    if (existingResult.isPresent()) {
        // Return the same response as the original request
        return ResponseEntity.ok(existingResult.get());
    }

    // Process the payment
    Payment payment = paymentService.process(request);
    PaymentResponse response = PaymentResponse.from(payment);

    // Store the result keyed by idempotency key (with TTL, e.g., 24 hours)
    idempotencyStore.put(idempotencyKey, response, Duration.ofHours(24));

    return ResponseEntity
        .created(URI.create("/api/payments/" + payment.getId()))
        .body(response);
}
```

---

## Scenario 6: Filtering with Query Parameters

**Problem:** Clients need flexible filtering of the product catalog.

```java
@GetMapping("/products")
public ResponseEntity<Page<ProductResponse>> searchProducts(
        @RequestParam(required = false) String name,
        @RequestParam(required = false) String category,
        @RequestParam(required = false) BigDecimal minPrice,
        @RequestParam(required = false) BigDecimal maxPrice,
        @RequestParam(required = false) Boolean inStock,
        Pageable pageable) {

    ProductFilter filter = ProductFilter.builder()
        .nameContains(name)
        .category(category)
        .minPrice(minPrice)
        .maxPrice(maxPrice)
        .inStock(inStock)
        .build();

    return ResponseEntity.ok(productService.search(filter, pageable));
}

// GET /products?category=electronics&minPrice=100&maxPrice=500&inStock=true&page=0&size=20&sort=price,asc
```

---

## Scenario 7: HATEOAS — Hypermedia-Driven API

**Problem:** Clients hardcode URLs. When you change URL structure, clients break.

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public EntityModel<OrderResponse> getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id).orElseThrow();
        OrderResponse response = OrderResponse.from(order);

        EntityModel<OrderResponse> model = EntityModel.of(response);

        // Add hypermedia links
        model.add(linkTo(methodOn(OrderController.class).getOrder(id)).withSelfRel());

        if (order.getStatus() == PENDING) {
            model.add(linkTo(methodOn(OrderController.class)
                .cancelOrder(id)).withRel("cancel"));
            model.add(linkTo(methodOn(OrderController.class)
                .confirmOrder(id)).withRel("confirm"));
        }

        if (order.getStatus() == CONFIRMED) {
            model.add(linkTo(methodOn(OrderController.class)
                .shipOrder(id)).withRel("ship"));
        }

        return model;
    }
}

// Response:
// {
//   "orderId": 123,
//   "status": "PENDING",
//   "_links": {
//     "self": { "href": "/api/orders/123" },
//     "cancel": { "href": "/api/orders/123/cancel" },
//     "confirm": { "href": "/api/orders/123/confirm" }
//   }
// }
```

---

## Scenario 8: Content Negotiation

**Problem:** Different clients need different response formats (JSON vs XML vs CSV).

```java
@GetMapping(value = "/products/{id}",
    produces = {
        MediaType.APPLICATION_JSON_VALUE,
        MediaType.APPLICATION_XML_VALUE,
        "text/csv"
    })
public ResponseEntity<?> getProduct(
        @PathVariable Long id,
        HttpServletRequest request) {

    Product product = productService.findById(id).orElseThrow();
    String accept = request.getHeader("Accept");

    if ("text/csv".equals(accept)) {
        String csv = product.getId() + "," + product.getName() + "," + product.getPrice();
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("text/csv"))
            .body(csv);
    }

    // Spring MVC handles JSON/XML via message converters
    return ResponseEntity.ok(ProductResponse.from(product));
}
// Client: GET /products/1 with Accept: application/xml → XML response
// Client: GET /products/1 with Accept: application/json → JSON response
```

---

## Scenario 9: Rate Limiting

**Problem:** A client is hammering your API with 10,000 requests per second, causing degraded service for others.

```java
@Component
public class RateLimitingFilter implements Filter {

    private final Map<String, RateLimiter> rateLimiters = new ConcurrentHashMap<>();
    private final int requestsPerSecond = 100;

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;

        String clientId = extractClientId(request);  // API key or IP
        RateLimiter limiter = rateLimiters.computeIfAbsent(
            clientId, k -> RateLimiter.create(requestsPerSecond));

        if (!limiter.tryAcquire()) {
            response.setStatus(429);  // 429 Too Many Requests
            response.setHeader("Retry-After", "1");
            response.getWriter().write("{\"error\":\"Rate limit exceeded\"}");
            return;
        }

        chain.doFilter(req, res);
    }
}
```

---

## Scenario 10: API Key Authentication

```java
@Component
public class ApiKeyAuthenticationFilter extends OncePerRequestFilter {

    @Autowired
    private ApiKeyRepository apiKeyRepository;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        String apiKey = request.getHeader("X-API-Key");

        if (apiKey == null || apiKey.isBlank()) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\":\"API key required\"}");
            return;
        }

        Optional<ApiKeyEntity> keyEntity = apiKeyRepository.findByKey(apiKey);
        if (keyEntity.isEmpty() || keyEntity.get().isExpired()) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\":\"Invalid or expired API key\"}");
            return;
        }

        // Set authentication context
        SecurityContextHolder.getContext().setAuthentication(
            new ApiKeyAuthenticationToken(keyEntity.get().getClientId(),
                                          keyEntity.get().getPermissions()));

        filterChain.doFilter(request, response);
    }
}
```

---

## Scenario 11: Async REST with DeferredResult

**Problem:** A report generation endpoint takes 30-60 seconds. Clients time out waiting.

```java
@PostMapping("/reports")
public ResponseEntity<ReportJobResponse> requestReport(@RequestBody ReportRequest request) {
    String jobId = UUID.randomUUID().toString();

    // Start async processing
    reportService.generateAsync(jobId, request);

    // Immediately return job ID
    return ResponseEntity
        .accepted()  // 202 Accepted
        .body(new ReportJobResponse(jobId, "PENDING",
              "/api/reports/" + jobId));
}

@GetMapping("/reports/{jobId}")
public ResponseEntity<ReportResponse> getReport(@PathVariable String jobId) {
    ReportJob job = reportService.findJob(jobId).orElseThrow();

    return switch (job.getStatus()) {
        case PENDING   -> ResponseEntity.accepted().body(new ReportResponse(job));
        case COMPLETED -> ResponseEntity.ok(new ReportResponse(job));
        case FAILED    -> ResponseEntity.status(500).body(new ReportResponse(job));
    };
}
```

---

## Scenario 12: Validation with Bean Validation

```java
@PostMapping("/products")
public ResponseEntity<ProductResponse> createProduct(
        @Valid @RequestBody CreateProductRequest request) {
    // @Valid triggers Bean Validation
    // Invalid fields → MethodArgumentNotValidException → 400 Bad Request
    Product product = productService.create(request);
    return ResponseEntity.created(...).body(ProductResponse.from(product));
}

// Request DTO with validation
public class CreateProductRequest {

    @NotBlank(message = "Product name is required")
    @Size(max = 255, message = "Name must not exceed 255 characters")
    private String name;

    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01", message = "Price must be greater than 0")
    @Digits(integer = 10, fraction = 2)
    private BigDecimal price;

    @NotNull
    @Min(0)
    private Integer stockQuantity;

    @NotBlank
    @Pattern(regexp = "^[A-Z]{2,10}$", message = "Category must be 2-10 uppercase letters")
    private String category;
}
```

---

## Scenario 13: ETag and Conditional Requests for Caching

```java
@GetMapping("/products/{id}")
public ResponseEntity<ProductResponse> getProduct(@PathVariable Long id,
                                                   HttpServletRequest request) {
    Product product = productService.findById(id).orElseThrow();

    String etag = "\"" + product.getVersion() + "\"";
    String ifNoneMatch = request.getHeader("If-None-Match");

    if (etag.equals(ifNoneMatch)) {
        return ResponseEntity.status(HttpStatus.NOT_MODIFIED).build();  // 304
    }

    return ResponseEntity.ok()
        .eTag(etag)
        .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
        .body(ProductResponse.from(product));
}
// Client: GET /products/1 → 200 OK, ETag: "42"
// Client: GET /products/1, If-None-Match: "42" → 304 Not Modified (no body)
```
