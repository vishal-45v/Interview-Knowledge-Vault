# Web Development — Theory Questions

> 18 theory questions on Servlet API, Spring MVC, HTTP, WebSockets, and REST API development for Senior Java Backend Engineers.

---

**Q1. What is the Servlet API and how does Spring MVC build on it?**

A Servlet is a Java class that handles HTTP requests/responses. The Servlet container (Tomcat, Jetty) manages servlet lifecycle.

```java
// Basic Servlet
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp)
            throws IOException {
        resp.setContentType("application/json");
        resp.getWriter().write("{\"message\": \"Hello World\"}");
    }
}

// Spring MVC builds on Servlets:
// DispatcherServlet (one servlet) → routes all requests → @Controller methods
// Spring maps URL patterns → Controller → MethodHandler via HandlerMapping
```

**Spring MVC request flow:**
```
HTTP Request
    → Tomcat/Jetty (Servlet Container)
    → DispatcherServlet (doDispatch)
    → HandlerMapping (find @RequestMapping match)
    → HandlerAdapter (convert params, call @Controller method)
    → MessageConverter (serialize response to JSON)
    → HTTP Response
```

---

**Q2. How does `@RequestMapping` and its variants work?**

```java
// Method-level mapping
@RestController
@RequestMapping("/api/v1/orders")  // Class-level prefix
public class OrderController {

    @GetMapping("/{id}")              // GET /api/v1/orders/{id}
    @PostMapping                      // POST /api/v1/orders
    @PutMapping("/{id}")              // PUT /api/v1/orders/{id}
    @PatchMapping("/{id}")            // PATCH /api/v1/orders/{id}
    @DeleteMapping("/{id}")           // DELETE /api/v1/orders/{id}

    // Full form with conditions
    @RequestMapping(
        path = "/search",
        method = RequestMethod.GET,
        produces = MediaType.APPLICATION_JSON_VALUE,
        consumes = MediaType.APPLICATION_JSON_VALUE,
        headers = "X-API-Version=2",
        params = "version=2"
    )
    public ResponseEntity<List<Order>> searchOrders() { ... }
}
```

---

**Q3. What are the different ways to extract data from HTTP requests?**

```java
// Path variable
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id) { ... }

// Query parameter
@GetMapping("/orders")
public Page<Order> getOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(required = false) String status) { ... }

// Request body (JSON → POJO)
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(@RequestBody @Valid CreateOrderRequest request) { ... }

// Request header
@GetMapping("/orders")
public List<Order> getOrders(@RequestHeader("X-Client-Version") String clientVersion) { ... }

// Cookie
@GetMapping("/profile")
public User getProfile(@CookieValue("session-id") String sessionId) { ... }

// Model attribute (form data)
@PostMapping("/orders/form")
public String submitForm(@ModelAttribute OrderForm form) { ... }

// Entire request/response
@GetMapping("/download")
public void download(HttpServletRequest request, HttpServletResponse response) { ... }
```

---

**Q4. What is `ResponseEntity` and when do you use it vs returning POJO directly?**

```java
// Return POJO directly — Spring uses @ResponseBody, defaults to 200 OK
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderService.findById(id);
}

// Return ResponseEntity — full control over status, headers, body
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(@RequestBody @Valid CreateOrderRequest req) {
    OrderResponse order = orderService.create(req);
    URI location = URI.create("/api/orders/" + order.id());
    return ResponseEntity
        .created(location)             // 201 + Location header
        .header("X-Order-Id", order.id())
        .body(order);
}

// No body (204)
@DeleteMapping("/orders/{id}")
public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
    orderService.delete(id);
    return ResponseEntity.noContent().build();  // 204
}

// Conditional response (304)
@GetMapping("/orders/{id}")
public ResponseEntity<Order> getOrderConditional(
        @PathVariable Long id, WebRequest request) {
    Order order = orderService.findById(id);
    if (request.checkNotModified(order.getVersion().toString())) {
        return ResponseEntity.status(HttpStatus.NOT_MODIFIED).build();
    }
    return ResponseEntity.ok()
        .eTag(order.getVersion().toString())
        .body(order);
}
```

---

**Q5. What is the filter chain in Servlet/Spring? How do filters differ from interceptors?**

```
HTTP Request
    → Filter 1 (javax.servlet.Filter) — runs before DispatcherServlet
    → Filter 2 (CorsFilter, SecurityFilter, etc.)
    → DispatcherServlet
    → Interceptor 1 (HandlerInterceptor) — runs inside Spring MVC
    → Interceptor 2
    → Controller Method
    → Interceptor 2 (postHandle)
    → Interceptor 1 (postHandle)
    → Filters (reverse)
    → HTTP Response
```

**Filter** — Servlet-level, runs for all requests (including static resources):
```java
@Component
@Order(1)
public class CorrelationIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        MDC.put("correlationId", UUID.randomUUID().toString());
        chain.doFilter(req, res);
        MDC.remove("correlationId");
    }
}
```

**Interceptor** — Spring MVC level, only for requests handled by DispatcherServlet:
```java
public class AuthorizationInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        // Return false to abort request processing
        return jwtTokenProvider.isValidToken(req.getHeader("Authorization"));
    }
}
```

---

**Q6. What is content negotiation and how does Spring handle it?**

```java
// Spring selects response format based on Accept header, URL extension, or param
// Accept: application/json  →  Jackson  →  JSON response
// Accept: application/xml   →  JAXB/Jackson-XML  →  XML response

// Configure content negotiation
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer config) {
        config
            .favorParameter(true)            // ?format=json
            .parameterName("format")
            .ignoreAcceptHeader(false)       // Use Accept header
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}

// Add XML support (includes Jackson XML module or JAXB)
// pom.xml: com.fasterxml.jackson.dataformat:jackson-dataformat-xml
```

---

**Q7. How does multipart file upload work in Spring?**

```java
@RestController
public class FileController {

    @PostMapping("/upload")
    public ResponseEntity<UploadResponse> upload(
            @RequestPart("file") MultipartFile file,
            @RequestPart("metadata") UploadMetadata metadata) throws IOException {

        // Stream directly — don't load into memory for large files
        String path = storageService.store(
            file.getInputStream(),
            file.getOriginalFilename(),
            file.getContentType());

        return ResponseEntity.ok(new UploadResponse(path, file.getSize()));
    }

    // Multiple files
    @PostMapping("/upload-multiple")
    public List<String> uploadMultiple(
            @RequestParam("files") List<MultipartFile> files) {
        return files.stream()
            .map(f -> storageService.store(f))
            .collect(Collectors.toList());
    }
}

# application.yml
spring:
  servlet:
    multipart:
      max-file-size: 100MB
      max-request-size: 100MB
      location: /tmp/uploads   # Temp directory for upload staging
```

---

**Q8. What is Server-Sent Events (SSE) and how do you implement it in Spring?**

SSE is a one-directional (server to client) persistent HTTP connection where the server pushes events.

```java
@RestController
@RequestMapping("/api/notifications")
public class NotificationController {

    @Autowired private NotificationService notificationService;

    // SSE endpoint — Spring automatically sets Content-Type: text/event-stream
    @GetMapping(value = "/{userId}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe(@PathVariable Long userId) {
        SseEmitter emitter = new SseEmitter(300_000L);  // 5 min timeout

        notificationService.register(userId, emitter);

        emitter.onCompletion(() -> notificationService.unregister(userId));
        emitter.onTimeout(() -> notificationService.unregister(userId));
        emitter.onError(e -> notificationService.unregister(userId));

        return emitter;
    }
}

@Service
public class NotificationService {

    private final Map<Long, SseEmitter> emitters = new ConcurrentHashMap<>();

    public void sendNotification(Long userId, Notification notification) {
        SseEmitter emitter = emitters.get(userId);
        if (emitter != null) {
            try {
                emitter.send(SseEmitter.event()
                    .id(notification.getId().toString())
                    .name("notification")
                    .data(notification, MediaType.APPLICATION_JSON));
            } catch (IOException e) {
                emitters.remove(userId);
            }
        }
    }
}
```

---

**Q9. What is a `@RestControllerAdvice` and how does exception handling work globally?**

```java
// @RestControllerAdvice = @ControllerAdvice + @ResponseBody
// Applies to all controllers in the application

@RestControllerAdvice
public class GlobalExceptionHandler {

    // Specific exception → specific HTTP status
    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ProblemDetail handleNotFound(OrderNotFoundException ex, HttpServletRequest req) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setType(URI.create("/errors/order-not-found"));
        return pd;
    }

    // Validation exception
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                e -> e.getDefaultMessage() != null ? e.getDefaultMessage() : "Invalid value"
            ));
        return ResponseEntity.badRequest().body(errors);
    }

    // Default catch-all
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleAll(Exception ex) {
        log.error("Unexpected error", ex);
        ProblemDetail pd = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
        pd.setDetail("An unexpected error occurred");
        return ResponseEntity.status(500).body(pd);
    }
}
```

---

**Q10. What is request validation with Bean Validation?**

```java
// Request DTO with Bean Validation constraints
public record CreateOrderRequest(
    @NotNull(message = "customerId is required")
    Long customerId,

    @NotEmpty(message = "items cannot be empty")
    @Valid
    List<OrderItemRequest> items,

    @NotBlank(message = "deliveryAddress is required")
    @Size(max = 500)
    String deliveryAddress,

    @Pattern(regexp = "^[A-Z]{4}\\d{4}$", message = "promo code format: PROMO1234")
    String promoCode
) {
    public record OrderItemRequest(
        @NotBlank String productId,
        @Positive @Max(100) int quantity
    ) {}
}

// Controller — @Valid triggers validation
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(
        @RequestBody @Valid CreateOrderRequest request) { ... }
// If validation fails → MethodArgumentNotValidException → 400 with field errors

// Service-level validation (validates method parameters)
@Service
@Validated  // Enables method-level validation
public class OrderService {
    public Order getOrder(@NotNull @Positive Long orderId) { ... }
}
```

---

**Q11. How do you implement API versioning?**

```java
// Option 1: URL path versioning (most common)
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 { ... }

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 { ... }

// Option 2: Accept header versioning
@GetMapping(value = "/orders/{id}",
            produces = "application/vnd.example.v1+json")
public OrderV1Response getOrderV1(...) { ... }

@GetMapping(value = "/orders/{id}",
            produces = "application/vnd.example.v2+json")
public OrderV2Response getOrderV2(...) { ... }

// Option 3: Request parameter
@GetMapping(value = "/orders/{id}", params = "version=1")
public OrderV1Response getOrderV1(...) { ... }

// Strategy comparison:
// URL: Most visible, easy to test/cache, breaks REST (resource URL changes with version)
// Header: Cleaner URLs, hard to test in browser, good for internal APIs
// Param: Simple, but pollutes URL space
```

---

**Q12. What is HATEOAS and how do you implement it?**

HATEOAS (Hypermedia As The Engine Of Application State) — responses include links to related actions.

```java
// Spring HATEOAS
@RestController
public class OrderController {

    @GetMapping("/orders/{id}")
    public EntityModel<Order> getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id);

        return EntityModel.of(order,
            linkTo(methodOn(OrderController.class).getOrder(id)).withSelfRel(),
            linkTo(methodOn(OrderController.class).cancelOrder(id, null)).withRel("cancel"),
            linkTo(methodOn(CustomerController.class).getCustomer(order.getCustomerId()))
                .withRel("customer")
        );
    }
}

// Response:
// {
//   "id": "ORD-123",
//   "status": "PENDING",
//   "_links": {
//     "self": { "href": "/api/orders/ORD-123" },
//     "cancel": { "href": "/api/orders/ORD-123/cancel" },
//     "customer": { "href": "/api/customers/1" }
//   }
// }
```

---

**Q13. What are HTTP cookies and sessions in Spring?**

```java
// Session-based auth (stateful — avoid in microservices)
@PostMapping("/login")
public ResponseEntity<?> login(@RequestBody LoginRequest req, HttpSession session) {
    User user = authService.authenticate(req.username(), req.password());
    session.setAttribute("userId", user.getId());
    session.setMaxInactiveInterval(3600);  // 1 hour
    return ResponseEntity.ok().build();
}

// Cookie management
@GetMapping("/set-cookie")
public ResponseEntity<?> setCookie(HttpServletResponse response) {
    Cookie cookie = new Cookie("preferences", "theme=dark");
    cookie.setHttpOnly(true);    // Not accessible via JavaScript
    cookie.setSecure(true);      // HTTPS only
    cookie.setMaxAge(86400);     // 24 hours
    cookie.setPath("/");
    cookie.setSameSite("Strict");  // CSRF protection
    response.addCookie(cookie);
    return ResponseEntity.ok().build();
}

// Spring Security remembers cookies are session cookies by default
// @EnableWebSecurity sets SameSite=Lax on JSESSIONID by default
```

---

**Q14. What is request context and how do you store per-request data?**

```java
// ThreadLocal for per-request context (web requests are thread-per-request in traditional Servlet model)
public class RequestContext {
    private static final ThreadLocal<RequestData> context = new ThreadLocal<>();

    public static void set(RequestData data) { context.set(data); }
    public static RequestData get() { return context.get(); }
    public static void clear() { context.remove(); }  // CRITICAL: prevent leaks
}

// Filter sets context at start of request
@Component
public class RequestContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpReq = (HttpServletRequest) req;
        RequestData data = new RequestData(
            httpReq.getHeader("X-Correlation-ID"),
            httpReq.getRemoteAddr(),
            httpReq.getHeader("User-Agent")
        );
        RequestContext.set(data);
        MDC.put("correlationId", data.correlationId());
        try {
            chain.doFilter(req, res);
        } finally {
            RequestContext.clear();
            MDC.clear();
        }
    }
}

// Spring's RequestContextHolder — similar but Spring-managed
RequestAttributes attributes = RequestContextHolder.getRequestAttributes();
HttpServletRequest request = ((ServletRequestAttributes) attributes).getRequest();
```

---

**Q15. What is the difference between `@Controller` and `@RestController`?**

```java
// @Controller — returns view names or uses @ResponseBody explicitly
@Controller
public class OrderController {
    @GetMapping("/orders/{id}")
    public String getOrderPage(@PathVariable Long id, Model model) {
        model.addAttribute("order", orderService.findById(id));
        return "orders/detail";  // Returns view name (Thymeleaf template)
    }

    @GetMapping("/api/orders/{id}")
    @ResponseBody  // Serialize return value to JSON
    public Order getOrderJson(@PathVariable Long id) {
        return orderService.findById(id);
    }
}

// @RestController = @Controller + @ResponseBody on all methods
@RestController
public class OrderRestController {
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable Long id) {  // Auto-serialized to JSON
        return orderService.findById(id);
    }
}
```

---

**Q16. What is the `WebMvcConfigurer` interface used for?**

```java
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    // CORS configuration
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("Authorization", "Content-Type")
            .allowCredentials(true)
            .maxAge(3600);
    }

    // Interceptors
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new RateLimitInterceptor())
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/health");
    }

    // Message converters (add XML support, customize Jackson)
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new MappingJackson2HttpMessageConverter(customObjectMapper()));
    }

    // Exception resolvers
    @Override
    public void configureHandlerExceptionResolvers(
            List<HandlerExceptionResolver> resolvers) {
        // Customize exception handling order
    }

    // Resource handling (static files)
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/static/**")
                .addResourceLocations("classpath:/static/")
                .setCacheControl(CacheControl.maxAge(7, TimeUnit.DAYS));
    }
}
```

---

**Q17. How does Spring handle async request processing?**

```java
// Callable — moves processing off Servlet thread
@GetMapping("/orders/expensive")
public Callable<Order> getExpensiveOrder(@RequestParam String id) {
    return () -> orderService.computeExpensiveOrder(id);
    // Returns immediately from Servlet thread
    // Callable executed by Spring's async executor thread
}

// DeferredResult — for external async completion
@GetMapping("/orders/async")
public DeferredResult<Order> getOrderAsync(@RequestParam Long id) {
    DeferredResult<Order> result = new DeferredResult<>(5000L);  // 5s timeout

    CompletableFuture.supplyAsync(() -> orderService.findById(id))
        .thenAccept(result::setResult)
        .exceptionally(ex -> { result.setErrorResult(ex); return null; });

    return result;
}

// WebFlux (reactive) — non-blocking throughout
@GetMapping("/orders/reactive")
public Mono<Order> getOrderReactive(@RequestParam Long id) {
    return orderService.findByIdReactive(id);
}
```

---

**Q18. What is Spring's `RestTemplate` vs `WebClient`?**

```java
// RestTemplate — synchronous, blocking (Spring MVC)
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    return builder
        .setConnectTimeout(Duration.ofSeconds(3))
        .setReadTimeout(Duration.ofSeconds(10))
        .errorHandler(new DefaultResponseErrorHandler())
        .build();
}

// Usage
Order order = restTemplate.getForObject(
    "https://external-api/orders/{id}", Order.class, orderId);

ResponseEntity<Order> response = restTemplate.exchange(
    "https://external-api/orders/{id}",
    HttpMethod.GET,
    HttpEntity.EMPTY,
    Order.class, orderId);

// WebClient — non-blocking, reactive (Spring WebFlux and Spring MVC)
@Bean
public WebClient webClient(WebClient.Builder builder) {
    return builder
        .baseUrl("https://external-api")
        .defaultHeader("Accept", "application/json")
        .filter(ExchangeFilterFunction.ofRequestProcessor(request -> {
            log.debug("Request: {}", request.url());
            return Mono.just(request);
        }))
        .build();
}

// Sync usage from WebClient (non-reactive context)
Order order = webClient.get()
    .uri("/orders/{id}", orderId)
    .retrieve()
    .bodyToMono(Order.class)
    .block();  // Block for synchronous usage

// Async usage
Mono<Order> orderMono = webClient.get()
    .uri("/orders/{id}", orderId)
    .retrieve()
    .onStatus(HttpStatus::is4xxClientError, response ->
        Mono.error(new OrderNotFoundException(orderId.toString())))
    .bodyToMono(Order.class);
```

**RestTemplate is deprecated** in favor of WebClient since Spring 5.0. Use WebClient for new code.
