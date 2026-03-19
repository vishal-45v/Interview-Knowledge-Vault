# Web Development — Scenario Questions

## How to Use This File
Real-world scenarios asked at Senior/Staff level interviews. Each scenario tests your ability to design, debug, and reason about production web applications. Provide concrete code, not just concepts.

---

## Scenario 1: Custom Filter for Request Logging and Correlation IDs

**Setup:** You need to add a correlation ID to every request for distributed tracing. If the incoming request has an `X-Correlation-ID` header, use it; otherwise generate a UUID. The ID must be available in all log statements via MDC, propagated to downstream HTTP calls, and included in all responses.

**Question:** Implement the complete filter chain solution.

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CorrelationIdFilter extends OncePerRequestFilter {

    private static final String HEADER = "X-Correlation-ID";
    private static final String MDC_KEY = "correlationId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain)
            throws ServletException, IOException {

        String correlationId = Optional
            .ofNullable(request.getHeader(HEADER))
            .filter(s -> !s.isBlank())
            .orElse(UUID.randomUUID().toString());

        MDC.put(MDC_KEY, correlationId);
        response.setHeader(HEADER, correlationId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);  // CRITICAL: prevent thread pool leak
        }
    }
}

// Propagate to downstream RestTemplate calls
@Bean
public RestTemplate restTemplate() {
    RestTemplate rt = new RestTemplate();
    rt.getInterceptors().add((request, body, execution) -> {
        String id = MDC.get("correlationId");
        if (id != null) {
            request.getHeaders().set("X-Correlation-ID", id);
        }
        return execution.execute(request, body);
    });
    return rt;
}

// For WebClient
@Bean
public WebClient webClient() {
    return WebClient.builder()
        .filter((request, next) -> {
            String id = MDC.get("correlationId");
            ClientRequest updated = ClientRequest.from(request)
                .header("X-Correlation-ID", id != null ? id : "unknown")
                .build();
            return next.exchange(updated);
        })
        .build();
}
```

**Trap:** Forgetting `MDC.remove()` in the `finally` block causes correlation IDs from previous requests to bleed into new requests on reused threads.

---

## Scenario 2: File Upload with Validation and Async Processing

**Setup:** Build an endpoint that accepts CSV file uploads (max 10MB), validates content type and file size, then processes the file asynchronously and returns a job ID.

```java
@RestController
@RequestMapping("/api/v1/imports")
public class ImportController {

    private final ImportService importService;

    @PostMapping(consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<ImportJobResponse> upload(
            @RequestParam("file") MultipartFile file,
            @RequestParam(defaultValue = "UTF-8") String charset) {

        validateFile(file);

        String jobId = importService.submitAsync(file, charset);

        return ResponseEntity
            .accepted()  // 202
            .header("Location", "/api/v1/imports/" + jobId)
            .body(new ImportJobResponse(jobId, "PENDING"));
    }

    @GetMapping("/{jobId}")
    public ResponseEntity<ImportJobResponse> status(@PathVariable String jobId) {
        return importService.findJob(jobId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    private void validateFile(MultipartFile file) {
        if (file.isEmpty()) {
            throw new ValidationException("File is empty");
        }
        if (file.getSize() > 10 * 1024 * 1024) {
            throw new ValidationException("File exceeds 10MB limit");
        }
        String contentType = file.getContentType();
        if (!"text/csv".equals(contentType) && !"application/csv".equals(contentType)) {
            // Also check magic bytes — content-type header can be spoofed
            throw new ValidationException("Only CSV files are accepted");
        }
    }
}

@Service
public class ImportService {

    private final Map<String, ImportJob> jobs = new ConcurrentHashMap<>();
    private final Executor executor = Executors.newVirtualThreadPerTaskExecutor(); // Java 21

    public String submitAsync(MultipartFile file, String charset) {
        String jobId = UUID.randomUUID().toString();
        // Copy bytes eagerly — MultipartFile temp file may be deleted after request ends
        byte[] content;
        try {
            content = file.getBytes();
        } catch (IOException e) {
            throw new RuntimeException("Failed to read uploaded file", e);
        }

        jobs.put(jobId, new ImportJob(jobId, "PENDING", 0, null));

        executor.execute(() -> processFile(jobId, content, charset));
        return jobId;
    }

    private void processFile(String jobId, byte[] content, String charset) {
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(new ByteArrayInputStream(content), charset))) {
            // process CSV...
            jobs.put(jobId, new ImportJob(jobId, "COMPLETED", 100, null));
        } catch (Exception e) {
            jobs.put(jobId, new ImportJob(jobId, "FAILED", 0, e.getMessage()));
        }
    }
}
```

**Key insight:** Copy `MultipartFile` bytes before async handoff — the temp file backing the multipart is deleted when the request scope ends.

---

## Scenario 3: SSE (Server-Sent Events) for Real-Time Notifications

**Setup:** Implement a notification endpoint that streams real-time events to the browser using SSE. Handle client disconnections gracefully and implement heartbeats.

```java
@RestController
@RequestMapping("/api/v1/notifications")
public class NotificationController {

    private final Map<String, SseEmitter> emitters = new ConcurrentHashMap<>();
    private final ScheduledExecutorService scheduler =
        Executors.newSingleThreadScheduledExecutor();

    @PostConstruct
    public void startHeartbeat() {
        scheduler.scheduleAtFixedRate(this::sendHeartbeats, 15, 15, TimeUnit.SECONDS);
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter subscribe(@RequestParam String userId) {
        SseEmitter emitter = new SseEmitter(Long.MAX_VALUE);

        emitters.put(userId, emitter);

        emitter.onCompletion(() -> emitters.remove(userId));
        emitter.onTimeout(() -> emitters.remove(userId));
        emitter.onError(e -> emitters.remove(userId));

        // Send initial connection event
        try {
            emitter.send(SseEmitter.event()
                .name("connected")
                .data("{\"status\":\"connected\",\"userId\":\"" + userId + "\"}")
                .id("0"));
        } catch (IOException e) {
            emitter.completeWithError(e);
        }

        return emitter;
    }

    public void sendNotification(String userId, NotificationEvent event) {
        SseEmitter emitter = emitters.get(userId);
        if (emitter == null) return;

        try {
            emitter.send(SseEmitter.event()
                .name("notification")
                .data(event)
                .id(event.getId())
                .reconnectTime(3000));
        } catch (IOException e) {
            emitters.remove(userId);
            emitter.completeWithError(e);
        }
    }

    private void sendHeartbeats() {
        List<String> dead = new ArrayList<>();
        emitters.forEach((userId, emitter) -> {
            try {
                emitter.send(SseEmitter.event()
                    .comment("heartbeat")
                    .data(""));
            } catch (IOException e) {
                dead.add(userId);
            }
        });
        dead.forEach(emitters::remove);
    }
}
```

**Production considerations:**
- SSE connections are persistent HTTP connections — need nginx timeout config (`proxy_read_timeout`)
- In clustered environments, use Redis pub/sub to fan out notifications across nodes
- Browser reconnects automatically with `Last-Event-ID` header — implement event replay

---

## Scenario 4: Global Exception Handler with RFC 7807 Problem Details

**Setup:** Design a comprehensive exception handler that returns RFC 7807 Problem Details for all error types, including validation errors, business exceptions, and unexpected errors.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // Spring Boot 3.x built-in ProblemDetail support
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex,
                                          HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.UNPROCESSABLE_ENTITY,
            "Request validation failed"
        );
        problem.setTitle("Validation Error");
        problem.setType(URI.create("https://api.example.com/errors/validation"));
        problem.setInstance(URI.create(request.getRequestURI()));

        Map<String, List<String>> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.groupingBy(
                FieldError::getField,
                Collectors.mapping(FieldError::getDefaultMessage, Collectors.toList())
            ));
        problem.setProperty("fieldErrors", fieldErrors);
        problem.setProperty("timestamp", Instant.now().toString());
        return problem;
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex,
                                        HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, ex.getMessage()
        );
        problem.setTitle("Resource Not Found");
        problem.setType(URI.create("https://api.example.com/errors/not-found"));
        problem.setInstance(URI.create(request.getRequestURI()));
        return problem;
    }

    @ExceptionHandler(BusinessException.class)
    public ProblemDetail handleBusiness(BusinessException ex, HttpServletRequest request) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            ex.getStatus(), ex.getMessage()
        );
        problem.setTitle(ex.getErrorCode());
        problem.setType(URI.create("https://api.example.com/errors/" + ex.getErrorCode()));
        problem.setProperty("correlationId", MDC.get("correlationId"));
        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex, HttpServletRequest request) {
        // Log with full stack trace, but don't expose internals to client
        String correlationId = MDC.get("correlationId");
        log.error("Unhandled exception [correlationId={}]", correlationId, ex);

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred. Reference: " + correlationId
        );
        problem.setTitle("Internal Server Error");
        return problem;
    }
}
```

**Critical:** Never expose stack traces or internal error messages to clients. Use correlation ID so clients can report issues that you can trace in logs.

---

## Scenario 5: API Versioning Strategy Implementation

**Setup:** Your team needs to support v1 and v2 of the same API simultaneously. v2 changes the response shape of the `GET /orders/{id}` endpoint. Implement versioning without duplicating all controller code.

```java
// Strategy: URL path versioning with shared service layer

// v1 controller
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {

    private final OrderService orderService;
    private final OrderMapperV1 mapperV1;

    @GetMapping("/{id}")
    public OrderResponseV1 getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", id));
        return mapperV1.toResponse(order);
    }
}

// v2 controller — different response shape, same service
@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {

    private final OrderService orderService;
    private final OrderMapperV2 mapperV2;

    @GetMapping("/{id}")
    public OrderResponseV2 getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", id));
        return mapperV2.toResponse(order);  // Different shape: nested customer object
    }
}

// Alternative: Header-based versioning using HandlerMapping
@Configuration
public class ApiVersioningConfig implements WebMvcConfigurer {

    @Bean
    public RequestMappingHandlerMapping versionedHandlerMapping() {
        RequestMappingHandlerMapping mapping = new RequestMappingHandlerMapping() {
            @Override
            protected RequestCondition<?> getCustomTypeCondition(Class<?> handlerType) {
                ApiVersion apiVersion = AnnotationUtils.findAnnotation(handlerType, ApiVersion.class);
                return apiVersion != null ? new ApiVersionCondition(apiVersion.value()) : null;
            }
        };
        mapping.setOrder(0);
        return mapping;
    }
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ApiVersion {
    String value();
}

// Usage:
@RestController
@RequestMapping("/orders")
@ApiVersion("2")
public class OrderControllerV2Header { ... }
```

**Trade-offs:**

| Strategy | Pros | Cons |
|---|---|---|
| URL Path `/v1/` | Cache-friendly, explicit | URL pollution, client must change |
| Header `API-Version: 2` | Clean URLs | Not cacheable by default, harder to test |
| Query param `?version=2` | Easy to test in browser | Not RESTful |
| Content negotiation | Most RESTful | Complex, poor browser support |

---

## Scenario 6: Custom HandlerInterceptor for Rate Limiting

**Setup:** Implement a rate limiting interceptor that limits API calls per user per minute using a sliding window in Redis.

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    private final RedisTemplate<String, String> redis;
    private final int maxRequests;
    private final Duration window;

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) throws Exception {

        if (!(handler instanceof HandlerMethod)) return true;

        HandlerMethod method = (HandlerMethod) handler;
        RateLimit rateLimit = method.getMethodAnnotation(RateLimit.class);
        if (rateLimit == null) return true;

        String userId = extractUserId(request);
        String key = "rate:" + userId + ":" + method.getMethod().getName();

        long count = incrementAndExpire(key, rateLimit.windowSeconds());

        int limit = rateLimit.maxRequests();
        response.setHeader("X-RateLimit-Limit", String.valueOf(limit));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(Math.max(0, limit - count)));

        if (count > limit) {
            response.setStatus(429);
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write(
                "{\"error\":\"Too Many Requests\",\"retryAfter\":" + rateLimit.windowSeconds() + "}"
            );
            return false;
        }
        return true;
    }

    private long incrementAndExpire(String key, int windowSeconds) {
        // Atomic increment + expire using Lua script
        String luaScript = """
            local count = redis.call('INCR', KEYS[1])
            if count == 1 then
                redis.call('EXPIRE', KEYS[1], ARGV[1])
            end
            return count
            """;
        Long count = redis.execute(
            RedisScript.of(luaScript, Long.class),
            List.of(key),
            String.valueOf(windowSeconds)
        );
        return count != null ? count : 0L;
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RateLimit {
    int maxRequests() default 100;
    int windowSeconds() default 60;
}
```

**Interceptor vs Filter for rate limiting:**
- Use Interceptor if you need access to the `HandlerMethod` (to check annotations)
- Use Filter if you need to rate-limit before Spring MVC processes the request (e.g., by IP at the Servlet level)

---

## Scenario 7: Content Negotiation for Multiple Response Formats

**Setup:** An API endpoint must return the same data as JSON, XML, or CSV depending on the `Accept` header. Implement custom content negotiation.

```java
@RestController
@RequestMapping("/api/v1/reports")
public class ReportController {

    private final ReportService reportService;

    @GetMapping(
        value = "/sales",
        produces = {
            MediaType.APPLICATION_JSON_VALUE,
            MediaType.APPLICATION_XML_VALUE,
            "text/csv"
        }
    )
    public ResponseEntity<?> getSalesReport(
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
            @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to,
            HttpServletRequest request) {

        List<SalesRecord> records = reportService.getSales(from, to);

        String acceptHeader = request.getHeader(HttpHeaders.ACCEPT);

        if ("text/csv".equals(acceptHeader)) {
            String csv = toCsv(records);
            return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType("text/csv"))
                .header(HttpHeaders.CONTENT_DISPOSITION,
                    "attachment; filename=\"sales-" + from + ".csv\"")
                .body(csv);
        }

        // JSON and XML handled by message converters
        return ResponseEntity.ok(new SalesReport(records));
    }
}

// Register custom CSV message converter
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorPathExtension(false)  // deprecated, don't use .json suffix
            .favorParameter(true)
            .parameterName("format")
            .ignoreAcceptHeader(false)
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML)
            .mediaType("csv", MediaType.parseMediaType("text/csv"));
    }
}
```

---

## Scenario 8: Streaming Large Responses with StreamingResponseBody

**Setup:** An endpoint needs to return a 500MB report file. Loading it into memory would cause OOM. Implement streaming response.

```java
@GetMapping("/reports/{reportId}/download")
public ResponseEntity<StreamingResponseBody> downloadReport(@PathVariable String reportId) {

    ReportMetadata metadata = reportService.getMetadata(reportId);

    StreamingResponseBody stream = outputStream -> {
        try (InputStream in = reportService.openStream(reportId);
             BufferedOutputStream out = new BufferedOutputStream(outputStream, 8192)) {

            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = in.read(buffer)) != -1) {
                out.write(buffer, 0, bytesRead);
            }
            out.flush();
        }
        // Any exception here will be propagated as an error to the client
    };

    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .header(HttpHeaders.CONTENT_DISPOSITION,
            "attachment; filename=\"" + metadata.getFilename() + "\"")
        .header(HttpHeaders.CONTENT_LENGTH, String.valueOf(metadata.getSizeBytes()))
        .body(stream);
}

// For CSV generation on-the-fly with Jackson's streaming API
@GetMapping(value = "/export", produces = "text/csv")
public StreamingResponseBody exportCsv() {
    return outputStream -> {
        PrintWriter writer = new PrintWriter(
            new OutputStreamWriter(outputStream, StandardCharsets.UTF_8));
        writer.println("id,name,amount,date");

        reportService.streamRecords(record -> {
            writer.printf("%d,%s,%.2f,%s%n",
                record.getId(),
                escapeCsv(record.getName()),
                record.getAmount(),
                record.getDate()
            );
            // Don't buffer — flush frequently for very large sets
            if (record.getId() % 1000 == 0) writer.flush();
        });
        writer.flush();
    };
}
```

**Critical:** `StreamingResponseBody` executes in a separate thread by default. Ensure the task executor is configured for long-running tasks:
```java
@Override
public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
    configurer.setDefaultTimeout(300_000); // 5 minutes
    configurer.setTaskExecutor(new ConcurrentTaskExecutor(
        Executors.newVirtualThreadPerTaskExecutor()
    ));
}
```

---

## Scenario 9: Request Context Propagation in Async Methods

**Setup:** A `@Async` service method needs access to the current user's context (from `SecurityContextHolder`) and MDC correlation ID, but async methods run on different threads.

```java
// Problem: SecurityContextHolder and MDC are ThreadLocal — lost in @Async
@Service
public class AsyncOrderService {

    @Async
    public CompletableFuture<Void> processOrderAsync(Long orderId) {
        // SecurityContextHolder.getContext().getAuthentication() is NULL here!
        // MDC.get("correlationId") is NULL here!
    }
}

// Solution 1: DelegatingSecurityContextAsyncTaskExecutor
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-");
        executor.initialize();
        // Wraps executor to propagate SecurityContext
        return new DelegatingSecurityContextAsyncTaskExecutor(executor);
    }
}

// Solution 2: Manual propagation with MDC
@Service
public class AsyncOrderService {

    @Async
    public CompletableFuture<Void> processOrderAsync(Long orderId,
            SecurityContext securityContext,
            String correlationId) {

        SecurityContextHolder.setContext(securityContext);
        MDC.put("correlationId", correlationId);
        try {
            // Process with full context
            return CompletableFuture.completedFuture(null);
        } finally {
            SecurityContextHolder.clearContext();
            MDC.remove("correlationId");
        }
    }
}

// Caller
public void handleRequest(Long orderId) {
    SecurityContext ctx = SecurityContextHolder.getContext();
    String corrId = MDC.get("correlationId");
    asyncOrderService.processOrderAsync(orderId, ctx, corrId);
}
```

---

## Scenario 10: WebClient with Retry, Timeout, and Circuit Breaker

**Setup:** Implement a resilient HTTP client using WebClient that handles timeouts, retries transient failures, and uses a circuit breaker for downstream protection.

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient inventoryWebClient(CircuitBreakerFactory cbFactory) {
        HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
            .responseTimeout(Duration.ofSeconds(10))
            .doOnConnected(conn ->
                conn.addHandlerLast(new ReadTimeoutHandler(10))
                    .addHandlerLast(new WriteTimeoutHandler(3)));

        return WebClient.builder()
            .baseUrl("https://inventory-service.internal")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(retryFilter())
            .filter(loggingFilter())
            .build();
    }

    private ExchangeFilterFunction retryFilter() {
        return (request, next) -> next.exchange(request)
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                .maxBackoff(Duration.ofSeconds(2))
                .filter(e -> isTransient(e))
                .onRetryExhaustedThrow((spec, signal) ->
                    new ServiceUnavailableException("Inventory service exhausted retries")));
    }

    private boolean isTransient(Throwable e) {
        return e instanceof ConnectException
            || e instanceof TimeoutException
            || (e instanceof WebClientResponseException wex
                && wex.getStatusCode().is5xxServerError());
    }
}

@Service
public class InventoryService {

    private final WebClient client;
    private final CircuitBreaker cb;

    public Mono<StockLevel> getStock(String sku) {
        return Mono.defer(() ->
            client.get()
                .uri("/stock/{sku}", sku)
                .retrieve()
                .onStatus(HttpStatus::is4xxClientError,
                    response -> response.bodyToMono(String.class)
                        .flatMap(body -> Mono.error(new ClientException(body))))
                .bodyToMono(StockLevel.class)
        )
        .transform(CircuitBreakerOperator.of(cb))
        .onErrorReturn(CallNotPermittedException.class,
            StockLevel.unknown(sku));  // fallback when CB is open
    }
}
```

---

## Scenario 11: HATEOAS-Driven REST API

**Setup:** Implement a REST endpoint that returns hypermedia links following HATEOAS principles. The links should change based on the resource state (e.g., an order that is PENDING has a "cancel" link; a SHIPPED order does not).

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderHateoasController {

    @GetMapping("/{id}")
    public EntityModel<OrderResponse> getOrder(@PathVariable Long id) {
        Order order = orderService.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", id));

        OrderResponse response = mapper.toResponse(order);
        EntityModel<OrderResponse> model = EntityModel.of(response);

        // Self link — always present
        model.add(linkTo(methodOn(OrderHateoasController.class).getOrder(id))
            .withSelfRel());

        // State-dependent links
        switch (order.getStatus()) {
            case PENDING -> {
                model.add(linkTo(methodOn(OrderHateoasController.class)
                    .cancelOrder(id)).withRel("cancel"));
                model.add(linkTo(methodOn(OrderHateoasController.class)
                    .confirmOrder(id)).withRel("confirm"));
            }
            case CONFIRMED -> model.add(linkTo(methodOn(OrderHateoasController.class)
                .cancelOrder(id)).withRel("cancel"));
            case SHIPPED -> model.add(linkTo(methodOn(OrderHateoasController.class)
                .trackShipment(id)).withRel("tracking"));
        }

        // Collection link
        model.add(linkTo(methodOn(OrderHateoasController.class)
            .listOrders(null, null)).withRel("orders"));

        return model;
    }

    @GetMapping
    public CollectionModel<EntityModel<OrderResponse>> listOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {

        Page<Order> orders = orderService.findAll(PageRequest.of(page, size));

        List<EntityModel<OrderResponse>> items = orders.stream()
            .map(o -> getOrder(o.getId()))
            .collect(Collectors.toList());

        CollectionModel<EntityModel<OrderResponse>> collection =
            CollectionModel.of(items);
        collection.add(linkTo(methodOn(OrderHateoasController.class)
            .listOrders(page, size)).withSelfRel());

        if (orders.hasNext()) {
            collection.add(linkTo(methodOn(OrderHateoasController.class)
                .listOrders(page + 1, size)).withRel("next"));
        }
        if (orders.hasPrevious()) {
            collection.add(linkTo(methodOn(OrderHateoasController.class)
                .listOrders(page - 1, size)).withRel("prev"));
        }

        return collection;
    }
}
```

---

## Scenario 12: Async Request Processing with DeferredResult

**Setup:** An endpoint triggers a long-running computation. Instead of blocking the servlet thread, use `DeferredResult` to release it immediately and complete the response when the computation finishes.

```java
@RestController
@RequestMapping("/api/v1/analytics")
public class AnalyticsController {

    private final AnalyticsEngine engine;
    private final Map<String, DeferredResult<AnalyticsResult>> pending =
        new ConcurrentHashMap<>();

    // Client polls this endpoint or subscribes via SSE
    @PostMapping("/compute")
    public DeferredResult<ResponseEntity<AnalyticsResult>> compute(
            @RequestBody AnalyticsRequest request) {

        DeferredResult<ResponseEntity<AnalyticsResult>> deferredResult =
            new DeferredResult<>(30_000L,  // 30s timeout
                ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
                    .body(AnalyticsResult.timeout()));

        deferredResult.onTimeout(() ->
            log.warn("Analytics request timed out: {}", request.getId()));

        engine.submitAsync(request, result -> {
            deferredResult.setResult(ResponseEntity.ok(result));
        });

        return deferredResult;
    }
}

// Webhook-style completion
@RestController
public class LongRunningController {

    private final Map<String, DeferredResult<String>> resultHolder = new ConcurrentHashMap<>();

    @PostMapping("/start-job")
    public DeferredResult<String> startJob(@RequestBody JobRequest req) {
        DeferredResult<String> result = new DeferredResult<>(60_000L, "TIMEOUT");
        resultHolder.put(req.getJobId(), result);
        jobService.submit(req);
        return result;
    }

    // Called by job service when done (or via message queue consumer)
    public void onJobComplete(String jobId, String output) {
        DeferredResult<String> result = resultHolder.remove(jobId);
        if (result != null) {
            result.setResult(output);
        }
    }
}
```

---

## Scenario 13: Custom Argument Resolver

**Setup:** Multiple controllers need to access the current authenticated user as a domain object (`CurrentUser` POJO). Implement a `HandlerMethodArgumentResolver` to inject it automatically.

```java
// Annotation to mark the parameter
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface AuthenticatedUser {}

// The resolver
@Component
public class CurrentUserArgumentResolver implements HandlerMethodArgumentResolver {

    private final UserRepository userRepository;

    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return parameter.hasParameterAnnotation(AuthenticatedUser.class)
            && parameter.getParameterType().equals(UserProfile.class);
    }

    @Override
    public Object resolveArgument(MethodParameter parameter,
                                   ModelAndViewContainer mavContainer,
                                   NativeWebRequest webRequest,
                                   WebDataBinderFactory binderFactory) {

        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            throw new UnauthorizedException("No authenticated user in context");
        }

        String userId = (String) auth.getPrincipal();
        return userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User", userId));
    }
}

// Registration
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
        resolvers.add(new CurrentUserArgumentResolver(userRepository));
    }
}

// Usage in controller — clean and testable
@GetMapping("/profile")
public UserProfileResponse getProfile(@AuthenticatedUser UserProfile user) {
    return mapper.toResponse(user);
}

@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(
        @AuthenticatedUser UserProfile user,
        @Valid @RequestBody CreateOrderRequest request) {
    return ResponseEntity.ok(orderService.create(user, request));
}
```

---

## Scenario 14: Multi-Part Form with JSON and File

**Setup:** An endpoint receives a mixed request: a JSON metadata part and a binary file part in the same multipart request.

```java
@PostMapping(value = "/products", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
public ResponseEntity<ProductResponse> createProduct(
        @RequestPart("metadata") @Valid ProductMetadata metadata,
        @RequestPart("image") MultipartFile image,
        @RequestPart(value = "spec", required = false) MultipartFile specSheet) {

    // Spring automatically deserializes "metadata" part as JSON if content-type is application/json
    // Use @RequestPart (not @RequestParam) for JSON parts

    validateImage(image);
    String imageUrl = storageService.upload(image, "products/images");

    String specUrl = null;
    if (specSheet != null && !specSheet.isEmpty()) {
        specUrl = storageService.upload(specSheet, "products/specs");
    }

    Product product = productService.create(metadata, imageUrl, specUrl);
    return ResponseEntity.status(HttpStatus.CREATED).body(mapper.toResponse(product));
}

private void validateImage(MultipartFile image) {
    String ct = image.getContentType();
    if (!List.of("image/jpeg", "image/png", "image/webp").contains(ct)) {
        throw new ValidationException("Only JPEG, PNG, WebP images are supported");
    }
    if (image.getSize() > 5 * 1024 * 1024) {
        throw new ValidationException("Image must be smaller than 5MB");
    }
    // Check magic bytes (file signature), not just content-type
    try {
        byte[] header = Arrays.copyOf(image.getBytes(), 4);
        if (!isSupportedImageMagicBytes(header)) {
            throw new ValidationException("File content does not match declared image type");
        }
    } catch (IOException e) {
        throw new ValidationException("Cannot read file content");
    }
}
```

**Client request format (curl):**
```bash
curl -X POST https://api.example.com/products \
  -F 'metadata={"name":"Widget","price":9.99};type=application/json' \
  -F 'image=@/path/to/product.jpg;type=image/jpeg'
```

---

## Scenario 15: Idempotency Key Pattern for POST Requests

**Setup:** Payment endpoints must be idempotent — if a client sends the same request twice (network retry), the payment must only be processed once. Implement using an idempotency key in headers.

```java
@Component
public class IdempotencyFilter extends OncePerRequestFilter {

    private final RedisTemplate<String, String> redis;
    private final ObjectMapper objectMapper;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {

        String idempotencyKey = request.getHeader("Idempotency-Key");
        if (idempotencyKey == null || !"POST".equals(request.getMethod())) {
            chain.doFilter(request, response);
            return;
        }

        String cacheKey = "idempotency:" + idempotencyKey;
        String cached = redis.opsForValue().get(cacheKey);

        if (cached != null) {
            // Return cached response
            CachedResponse cachedResponse = objectMapper.readValue(cached, CachedResponse.class);
            response.setStatus(cachedResponse.getStatus());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setHeader("X-Idempotency-Replayed", "true");
            response.getWriter().write(cachedResponse.getBody());
            return;
        }

        // Capture response
        ContentCachingResponseWrapper wrappedResponse =
            new ContentCachingResponseWrapper(response);

        chain.doFilter(request, wrappedResponse);

        // Cache the response for 24 hours
        CachedResponse toCache = new CachedResponse(
            wrappedResponse.getStatus(),
            new String(wrappedResponse.getContentAsByteArray(), StandardCharsets.UTF_8)
        );
        redis.opsForValue().set(cacheKey,
            objectMapper.writeValueAsString(toCache),
            Duration.ofHours(24));

        wrappedResponse.copyBodyToResponse();
    }
}

// Controller stays clean — idempotency is cross-cutting
@PostMapping("/payments")
public ResponseEntity<PaymentResult> processPayment(@Valid @RequestBody PaymentRequest request) {
    PaymentResult result = paymentService.process(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(result);
}
```

**Edge case:** Handle the race condition where two concurrent requests arrive with the same key simultaneously — use Redis `SET NX` (set if not exists) with a "processing" sentinel value before executing, then update with the real response.

---

## Scenario 16: Custom HTTP Message Converter

**Setup:** An internal API needs to support a proprietary binary protocol alongside JSON. Implement a custom `HttpMessageConverter`.

```java
public class ProtobufHttpMessageConverter extends AbstractHttpMessageConverter<Message> {

    public ProtobufHttpMessageConverter() {
        super(MediaType.parseMediaType("application/x-protobuf"),
              MediaType.APPLICATION_JSON);  // Also support JSON via protobuf-java-util
    }

    @Override
    protected boolean supports(Class<?> clazz) {
        return Message.class.isAssignableFrom(clazz);
    }

    @Override
    protected Message readInternal(Class<? extends Message> clazz,
                                    HttpInputMessage inputMessage)
            throws IOException, HttpMessageNotReadableException {

        MediaType contentType = inputMessage.getHeaders().getContentType();
        byte[] body = inputMessage.getBody().readAllBytes();

        try {
            Method parseFrom = clazz.getMethod("parseFrom", byte[].class);
            return (Message) parseFrom.invoke(null, (Object) body);
        } catch (ReflectiveOperationException e) {
            throw new HttpMessageNotReadableException(
                "Could not parse protobuf message", inputMessage);
        }
    }

    @Override
    protected void writeInternal(Message message,
                                  HttpOutputMessage outputMessage) throws IOException {
        MediaType contentType = outputMessage.getHeaders().getContentType();
        if (MediaType.APPLICATION_JSON.isCompatibleWith(contentType)) {
            // Use protobuf's JSON printer
            String json = JsonFormat.printer()
                .includingDefaultValueFields()
                .print(message);
            outputMessage.getBody().write(json.getBytes(StandardCharsets.UTF_8));
        } else {
            message.writeTo(outputMessage.getBody());
        }
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(0, new ProtobufHttpMessageConverter());
    }
}
```

---

## Follow-Up Questions

1. **Filter vs Interceptor:** When would you choose a Servlet Filter over a Spring HandlerInterceptor for request logging?
   - *Filter* runs before Spring MVC — use for security, encoding, correlation IDs, rate limiting at the raw HTTP level. *Interceptor* has access to handler method info — use for authorization checks based on annotations, audit logging with user context.

2. **SSE vs WebSocket:** When would you use SSE instead of WebSocket?
   - SSE for server-to-client only (notifications, live feeds, progress updates). SSE works over HTTP/1.1, simpler infrastructure, auto-reconnect built in. WebSocket for bidirectional real-time communication (chat, collaborative editing, gaming).

3. **StreamingResponseBody thread model:** What executor does `StreamingResponseBody` use by default?
   - Spring uses `SimpleAsyncTaskExecutor` by default which creates a new thread per request. In production, configure a dedicated `ThreadPoolTaskExecutor` with bounded queue to prevent resource exhaustion.

4. **Idempotency race condition:** How do you handle two simultaneous requests with the same idempotency key?
   - Use Redis `SET NX EX` to acquire a lock. First request sets a "PROCESSING" sentinel. Second request gets "PROCESSING" — return `409 Conflict` or wait with polling. First request updates with real result.

5. **HATEOAS adoption:** Why is HATEOAS rarely used in practice despite being a REST constraint?
   - Client complexity: clients must parse and follow links rather than hardcoding URLs. Framework support is inconsistent. JSON:API and GraphQL address the same discoverability problem differently. Most APIs stop at Richardson Maturity Level 2 (resources + HTTP verbs).
