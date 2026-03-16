# Exception Handling — Scenario Questions

> 18 real-world scenarios covering exception handling patterns, Spring Boot error responses, and production debugging.

---

## Scenario 1: Designing a Global Exception Handler for REST API

**Situation:** Your Spring Boot REST API returns raw stack traces to clients on errors. Standardize error responses.

```java
// Standard error response body
public record ErrorResponse(
    String type,           // Machine-readable error type
    String title,          // Human-readable summary
    int status,            // HTTP status code
    String detail,         // Specific detail about this occurrence
    String instance,       // URI identifying this occurrence
    Instant timestamp,
    String correlationId
) {}

// Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            OrderNotFoundException ex, HttpServletRequest request) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(
                "/errors/order-not-found",
                "Order Not Found",
                404,
                ex.getMessage(),
                request.getRequestURI(),
                Instant.now(),
                MDC.get("correlationId")));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            fieldErrors.put(error.getField(), error.getDefaultMessage()));

        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ValidationErrorResponse(
                "/errors/validation-failed",
                "Validation Failed",
                400,
                "One or more fields have invalid values",
                request.getRequestURI(),
                fieldErrors,
                Instant.now(),
                MDC.get("correlationId")));
    }

    @ExceptionHandler(Exception.class)  // Catch-all — last resort
    public ResponseEntity<ErrorResponse> handleUnexpected(
            Exception ex, HttpServletRequest request) {
        log.error("Unexpected error processing {}", request.getRequestURI(), ex);
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse(
                "/errors/internal-error",
                "Internal Server Error",
                500,
                "An unexpected error occurred",  // Don't expose internals!
                request.getRequestURI(),
                Instant.now(),
                MDC.get("correlationId")));
    }
}
```

---

## Scenario 2: Checked Exception in Stream — Real Production Pattern

**Situation:** You have a list of file paths and need to read their contents in a stream. `Files.readString()` throws `IOException` (checked).

```java
// Problem: compile error because IOException is checked
List<String> contents = paths.stream()
    .map(path -> Files.readString(path))  // COMPILE ERROR
    .collect(Collectors.toList());

// Solution 1: Inline try-catch with UncheckedIOException wrapper
List<String> contents = paths.stream()
    .map(path -> {
        try {
            return Files.readString(path);
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to read: " + path, e);
        }
    })
    .collect(Collectors.toList());

// Solution 2: Utility method — cleaner, reusable
public static <T, R> Function<T, R> wrapChecked(ThrowingFunction<T, R> f) {
    return t -> {
        try {
            return f.apply(t);
        } catch (Exception e) {
            throw e instanceof RuntimeException ? (RuntimeException) e
                                                : new RuntimeException(e);
        }
    };
}

List<String> contents = paths.stream()
    .map(wrapChecked(Files::readString))  // Clean!
    .collect(Collectors.toList());

// Solution 3: Collect errors instead of failing fast
record Result<T>(T value, Exception error) {
    static <T> Result<T> of(T value) { return new Result<>(value, null); }
    static <T> Result<T> failed(Exception e) { return new Result<>(null, e); }
    boolean isSuccess() { return error == null; }
}

List<Result<String>> results = paths.stream()
    .map(path -> {
        try { return Result.of(Files.readString(path)); }
        catch (IOException e) { return Result.<String>failed(e); }
    })
    .collect(Collectors.toList());

List<String> successful = results.stream()
    .filter(Result::isSuccess)
    .map(Result::value)
    .collect(Collectors.toList());

results.stream()
    .filter(r -> !r.isSuccess())
    .forEach(r -> log.error("Failed to read file", r.error()));
```

---

## Scenario 3: Exception in @Transactional Method — Rollback Behavior

**Situation:** Your `createOrder` method throws a custom exception. The order was saved to the DB but the inventory wasn't updated. The transaction wasn't rolled back. Why?

```java
// PROBLEM: Custom checked exception does NOT trigger rollback by default
@Transactional
public Order createOrder(OrderRequest request) throws InsufficientInventoryException {
    Order order = orderRepository.save(new Order(request));   // Saved
    inventoryService.reserve(request.getProductId(), request.getQuantity());  // Throws checked exception
    // Transaction commits because InsufficientInventoryException is CHECKED
    return order;  // Data inconsistency!
}

// Solution 1: Make the exception unchecked (RuntimeException)
public class InsufficientInventoryException extends RuntimeException { ... }
// Spring rolls back on RuntimeException by default

// Solution 2: Explicitly configure rollback for checked exceptions
@Transactional(rollbackFor = InsufficientInventoryException.class)
public Order createOrder(OrderRequest request) throws InsufficientInventoryException {
    // Now rolls back even for checked exception
}

// Solution 3: noRollbackFor for expected unchecked exceptions
@Transactional(noRollbackFor = OptimisticLockException.class)
public void updateOrder(Long id) {
    // OptimisticLockException won't trigger rollback — handle it explicitly
}
```

**Spring's default rollback rules:**
- Roll back on: `RuntimeException` and `Error`
- Do NOT roll back on: checked `Exception`

---

## Scenario 4: Exception Handling in CompletableFuture Chain

**Situation:** Build an order processing pipeline with error handling at each stage.

```java
@Service
public class OrderProcessingPipeline {

    public CompletableFuture<OrderResult> processOrder(OrderRequest request) {
        return CompletableFuture
            // Stage 1: Validate
            .supplyAsync(() -> validateOrder(request), executor)

            // Stage 2: Reserve inventory (may fail)
            .thenApplyAsync(validated -> {
                try {
                    return inventoryService.reserve(validated);
                } catch (InsufficientInventoryException e) {
                    // Transform to domain result instead of exception
                    return OrderResult.insufficientInventory(validated.getProductId());
                }
            }, executor)

            // Stage 3: Process payment (if inventory reserved)
            .thenComposeAsync(reserved -> {
                if (!reserved.isSuccess()) {
                    return CompletableFuture.completedFuture(reserved);
                }
                return paymentService.charge(reserved.getPaymentDetails());
            }, executor)

            // Handle ALL exceptions from the entire chain
            .exceptionally(throwable -> {
                Throwable cause = throwable instanceof CompletionException
                    ? throwable.getCause() : throwable;

                log.error("Order processing failed for request {}", request.getId(), cause);

                if (cause instanceof PaymentDeclinedException) {
                    return OrderResult.paymentDeclined();
                }
                if (cause instanceof ValidationException) {
                    return OrderResult.validationFailed(cause.getMessage());
                }
                return OrderResult.failed("Unexpected error: " + cause.getMessage());
            })

            // Ensure cleanup regardless
            .whenComplete((result, error) -> {
                if (result != null) {
                    auditService.logOrderAttempt(request.getId(), result.getStatus());
                }
            });
    }
}
```

---

## Scenario 5: Retry Logic with Exception Classification

**Situation:** Your service calls an external API that fails with different exceptions. Some are transient (retry), some are permanent (don't retry).

```java
@Service
public class ExternalApiClient {

    private static final int MAX_RETRIES = 3;

    public ApiResponse callWithRetry(String endpoint, Object payload) {
        int attempt = 0;
        Exception lastException = null;

        while (attempt < MAX_RETRIES) {
            try {
                return httpClient.post(endpoint, payload);
            } catch (HttpClientErrorException e) {
                // 4xx — client error, NEVER retry (our fault)
                if (e.getStatusCode().is4xxClientError()) {
                    throw new NonRetryableException(
                        "Client error " + e.getStatusCode(), e);
                }
                lastException = e;
            } catch (HttpServerErrorException e) {
                // 5xx — server error, MAY retry
                if (e.getStatusCode() == HttpStatus.SERVICE_UNAVAILABLE
                        || e.getStatusCode() == HttpStatus.GATEWAY_TIMEOUT) {
                    lastException = e;
                } else {
                    throw new NonRetryableException("Server error: " + e.getStatusCode(), e);
                }
            } catch (ResourceAccessException e) {
                // Connection refused, timeout — retry
                lastException = e;
            }

            attempt++;
            if (attempt < MAX_RETRIES) {
                long delay = (long) Math.pow(2, attempt) * 200L;  // Exponential backoff
                try {
                    Thread.sleep(delay);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new RuntimeException("Interrupted during retry", ie);
                }
            }
        }

        throw new ExternalApiException(
            "Failed after " + MAX_RETRIES + " attempts", lastException);
    }
}

// Using Spring Retry annotation (cleaner)
@Retryable(
    value = {ResourceAccessException.class, HttpServerErrorException.ServiceUnavailable.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 200, multiplier = 2, maxDelay = 5000)
)
public ApiResponse callExternalApi(String endpoint, Object payload) {
    return httpClient.post(endpoint, payload);
}

@Recover
public ApiResponse fallback(Exception e, String endpoint, Object payload) {
    log.error("All retries exhausted for {}", endpoint, e);
    return ApiResponse.unavailable();
}
```

---

## Scenario 6: Exception Handling with Resource Cleanup Pattern

**Situation:** You open database and file resources in a method. If the DB write succeeds but the file write fails, you need to clean up both.

```java
// WITHOUT try-with-resources — brittle
public void exportOrdersToFile(String filePath) throws IOException {
    Connection conn = null;
    PreparedStatement stmt = null;
    FileWriter writer = null;

    try {
        conn = dataSource.getConnection();
        stmt = conn.prepareStatement("SELECT * FROM orders WHERE exported = false");
        ResultSet rs = stmt.executeQuery();
        writer = new FileWriter(filePath);

        while (rs.next()) {
            writer.write(rs.getString("data") + "\n");
        }

        // Update exported flag
        conn.prepareStatement("UPDATE orders SET exported = true").executeUpdate();
        conn.commit();

    } catch (SQLException e) {
        if (conn != null) {
            try { conn.rollback(); } catch (SQLException ex) { log.warn("Rollback failed", ex); }
        }
        throw new RuntimeException("DB error during export", e);
    } finally {
        if (writer != null) try { writer.close(); } catch (IOException e) { log.warn("Close failed", e); }
        if (stmt != null) try { stmt.close(); } catch (SQLException e) { log.warn("Close failed", e); }
        if (conn != null) try { conn.close(); } catch (SQLException e) { log.warn("Close failed", e); }
    }
}

// WITH try-with-resources — clean
public void exportOrdersToFile(String filePath) throws IOException, SQLException {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement("SELECT * FROM orders WHERE exported = false");
         ResultSet rs = stmt.executeQuery();
         BufferedWriter writer = Files.newBufferedWriter(Paths.get(filePath))) {

        conn.setAutoCommit(false);
        while (rs.next()) {
            writer.write(rs.getString("data") + "\n");
        }
        conn.commit();
        // All resources closed in reverse order automatically
    }
    // If any exception occurs, all resources are closed before exception propagates
}
```

---

## Scenario 7: Avoiding Exception Swallowing in Async Code

**Situation:** Tasks submitted to an `ExecutorService` silently fail — no error is logged, no alert fires. It's like they disappear.

```java
// PROBLEM: execute() swallows exceptions in runnable
// (UncaughtExceptionHandler IS called, but most setups don't configure it)
executor.execute(() -> {
    processOrder(orderId);  // If this throws RuntimeException — silently swallowed!
});

// PROBLEM: submit() captures exception in Future, but if Future.get() not called — lost
Future<?> future = executor.submit(() -> processOrder(orderId));
// Future never retrieved → exception silently discarded

// SOLUTION 1: Wrap runnable to log exceptions
executor.execute(() -> {
    try {
        processOrder(orderId);
    } catch (Exception e) {
        log.error("Failed to process order {}", orderId, e);
        metrics.increment("order.processing.failure");
        // Optionally: dead letter queue, retry, alert
    }
});

// SOLUTION 2: Thread factory with UncaughtExceptionHandler
ThreadFactory factory = runnable -> {
    Thread t = new Thread(runnable);
    t.setUncaughtExceptionHandler((thread, e) -> {
        log.error("Uncaught exception in thread pool", e);
        alertingService.sendAlert("Thread pool failure: " + e.getMessage());
    });
    return t;
};
ExecutorService executor = Executors.newFixedThreadPool(4, factory);

// SOLUTION 3: Reactive approach — CompletableFuture with exception handling
CompletableFuture
    .runAsync(() -> processOrder(orderId), executor)
    .exceptionally(e -> {
        log.error("Order processing failed", e);
        return null;
    });
```

---

## Scenario 8: Distinguishing Exception Types in Catch Blocks

**Situation:** You're catching `DataAccessException` but need to respond differently to constraint violations vs connection failures.

```java
@Service
public class UserService {

    public void createUser(UserRequest request) {
        try {
            userRepository.save(new User(request));
        } catch (DataIntegrityViolationException e) {
            // Specific subtype: constraint violation (duplicate email, FK violation)
            if (e.getCause() instanceof ConstraintViolationException cve) {
                String constraint = cve.getConstraintName();
                if (constraint != null && constraint.contains("email")) {
                    throw new DuplicateEmailException("Email already exists: " + request.getEmail());
                }
            }
            throw new UserCreationException("Data integrity violation", e);
        } catch (DataAccessResourceFailureException e) {
            // Database connection failed — propagate for circuit breaker
            throw new ServiceUnavailableException("Database unavailable", e);
        } catch (QueryTimeoutException e) {
            // Query took too long
            log.warn("Slow query on user creation", e);
            throw new ServiceTimeoutException("User creation timed out", e);
        }
        // Let other DataAccessException subtypes propagate naturally
    }
}
```

---

## Scenario 9: Exception Handling in Spring Batch

**Situation:** A Spring Batch job processes 10,000 records. Some records are malformed. Configure it to skip bad records, retry transient failures, and abort on critical errors.

```java
@Configuration
public class BatchJobConfig {

    @Bean
    public Step processOrdersStep() {
        return stepBuilderFactory.get("processOrdersStep")
            .<OrderRecord, ProcessedOrder>chunk(100)
            .reader(orderReader())
            .processor(orderProcessor())
            .writer(orderWriter())

            // Skip up to 500 bad records (skip list)
            .faultTolerant()
            .skipLimit(500)
            .skip(MalformedOrderException.class)       // Skip and continue
            .skip(DataParseException.class)

            // Retry transient errors 3 times
            .retryLimit(3)
            .retry(TransientDataAccessException.class) // DB connection blip

            // Never skip critical errors — abort job
            .noSkip(CriticalOrderException.class)
            .noRetry(ValidationException.class)

            .listener(new SkipListener<OrderRecord, ProcessedOrder>() {
                @Override
                public void onSkipInRead(Throwable t) {
                    log.warn("Skipped malformed record during read", t);
                    metrics.increment("batch.records.skipped");
                }
                @Override
                public void onSkipInProcess(OrderRecord item, Throwable t) {
                    log.warn("Skipped record {} during processing: {}", item.getId(), t.getMessage());
                    deadLetterQueue.send(item);  // Save for manual review
                }
                @Override
                public void onSkipInWrite(ProcessedOrder item, Throwable t) {
                    log.error("Skipped record {} during write", item.getId(), t);
                }
            })
            .build();
    }
}
```

---

## Scenario 10: Defensive Exception Handling at System Boundaries

**Situation:** Your service integrates with 3 third-party APIs. Each has its own exception model. Create a unified exception handling layer.

```java
// Anti-corruption layer — translate third-party exceptions to your domain
@Service
public class PaymentGatewayAdapter {

    private final StripePaymentClient stripeClient;

    public PaymentResult charge(PaymentDetails details) {
        try {
            StripeCharge charge = stripeClient.create(details.toStripeRequest());
            return PaymentResult.success(charge.getId());

        } catch (StripeCardException e) {
            // Card declined — expected, map to domain exception
            return PaymentResult.declined(e.getDeclineCode(), e.getMessage());

        } catch (StripeInvalidRequestException e) {
            // Bad request — our fault
            throw new PaymentValidationException(
                "Invalid payment request: " + e.getParam(), e);

        } catch (StripeAuthenticationException e) {
            // API key invalid — configuration problem
            log.error("Stripe authentication failed — check API key configuration", e);
            throw new PaymentConfigurationException("Payment service misconfigured", e);

        } catch (StripeRateLimitException e) {
            // Rate limited — retry
            throw new PaymentRateLimitException("Payment service rate limited", e);

        } catch (StripeException e) {
            // Unknown Stripe error — log full details internally, expose generic externally
            log.error("Unexpected Stripe error: {} code={}", e.getMessage(), e.getCode(), e);
            throw new PaymentServiceException("Payment processing failed", e);
        }
    }
}
```

---

## Scenario 11: Exception Propagation in Microservices (HTTP)

**Situation:** Service A calls Service B via REST. Service B returns 422 (validation error). How does Service A handle this and map it to its own error model?

```java
// Service A's HTTP client error handler
@Component
public class ServiceBClient {

    private final RestTemplate restTemplate;

    public ServiceBResponse callServiceB(ServiceBRequest request) {
        try {
            return restTemplate.postForObject(
                "http://service-b/api/process", request, ServiceBResponse.class);

        } catch (HttpClientErrorException.UnprocessableEntity e) {
            // 422 from Service B — map to our domain exception
            ErrorResponse errorBody = objectMapper.readValue(
                e.getResponseBodyAsString(), ErrorResponse.class);
            throw new UpstreamValidationException(
                "Service B rejected request: " + errorBody.getDetail());

        } catch (HttpClientErrorException.NotFound e) {
            throw new ResourceNotFoundException("Service B resource not found");

        } catch (HttpServerErrorException e) {
            // 5xx — Service B has internal error
            log.error("Service B internal error: status={}", e.getStatusCode());
            throw new DependencyUnavailableException("Service B unavailable", e);

        } catch (ResourceAccessException e) {
            // Network error (connection refused, timeout)
            throw new DependencyUnavailableException("Service B unreachable", e);
        }
    }
}
```

---

## Scenario 12: Exception Handling in Event-Driven Architecture

**Situation:** A Kafka consumer throws an exception on a message. What happens to the message? How do you handle poison pills (always-failing messages)?

```java
@Component
public class OrderEventConsumer {

    @KafkaListener(topics = "order-events")
    public void handleOrderEvent(ConsumerRecord<String, String> record) {
        try {
            OrderEvent event = objectMapper.readValue(record.value(), OrderEvent.class);
            orderService.process(event);

        } catch (JsonProcessingException e) {
            // Deserialization failure — message is malformed (poison pill)
            // NEVER retry this — it will always fail
            log.error("Malformed event at offset {}: {}", record.offset(), record.value());
            deadLetterTopicService.send("order-events.DLT", record);
            // Do NOT rethrow — acknowledge and move on

        } catch (OrderAlreadyProcessedException e) {
            // Idempotency — already processed, ignore
            log.info("Duplicate event ignored: {}", record.key());

        } catch (TransientException e) {
            // Transient failure — let Kafka retry by throwing
            log.warn("Transient error processing event, will retry", e);
            throw new RuntimeException("Transient failure — retry", e);
            // Kafka will redeliver based on consumer group config
        }
    }
}

// Better: Use Spring Kafka's built-in error handling
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<?, ?> kafkaTemplate) {
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate,
        (record, ex) -> new TopicPartition(record.topic() + ".DLT", record.partition()));

    // Retry up to 3 times with exponential backoff
    ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(3);
    backOff.setInitialInterval(500);
    backOff.setMultiplier(2.0);

    DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);
    // Don't retry deserialization errors (poison pills)
    handler.addNotRetryableExceptions(JsonProcessingException.class);
    return handler;
}
```

---

## Scenario 13: Circuit Breaker Pattern for Exception Handling

**Situation:** Your service calls an external weather API. When the API is down, implement a circuit breaker that returns cached data instead of throwing exceptions.

```java
@Service
public class WeatherService {

    @Autowired private WeatherApiClient weatherApiClient;
    @Autowired private WeatherCache weatherCache;

    @CircuitBreaker(name = "weatherApi", fallbackMethod = "getWeatherFromCache")
    @Retry(name = "weatherApi")
    @TimeLimiter(name = "weatherApi")
    public CompletableFuture<WeatherData> getWeather(String city) {
        return CompletableFuture.supplyAsync(
            () -> weatherApiClient.fetchWeather(city));
    }

    // Called when circuit is open or all retries exhausted
    public CompletableFuture<WeatherData> getWeatherFromCache(
            String city, Exception e) {
        log.warn("Weather API unavailable for {}, using cached data. Error: {}",
                 city, e.getMessage());

        return CompletableFuture.supplyAsync(() -> {
            WeatherData cached = weatherCache.get(city);
            if (cached != null) {
                return cached.withStale(true);  // Mark as potentially outdated
            }
            // If no cache, return default (better than failing completely)
            return WeatherData.unavailable(city);
        });
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      weatherApi:
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 3s
        wait-duration-in-open-state: 30s
        sliding-window-size: 20
  retry:
    instances:
      weatherApi:
        max-attempts: 2
        wait-duration: 500ms
  timelimiter:
    instances:
      weatherApi:
        timeout-duration: 2s
```

---

## Scenario 14: Testing Exception Scenarios

**Situation:** Write tests that verify correct exception handling behavior.

```java
@SpringBootTest
class OrderServiceExceptionTest {

    @Autowired private OrderService orderService;
    @MockBean private OrderRepository orderRepository;
    @MockBean private InventoryService inventoryService;

    @Test
    void whenProductNotFound_shouldThrowOrderNotFoundException() {
        when(orderRepository.findById(99L))
            .thenThrow(new EmptyResultDataAccessException(1));

        assertThatThrownBy(() -> orderService.getOrder(99L))
            .isInstanceOf(OrderNotFoundException.class)
            .hasMessageContaining("99")
            .hasNoCause();  // Or: .hasCauseInstanceOf(EmptyResultDataAccessException.class)
    }

    @Test
    void whenDatabaseDown_shouldThrowServiceUnavailableException() {
        when(orderRepository.save(any()))
            .thenThrow(new DataAccessResourceFailureException("DB down",
                new SQLException("Connection refused")));

        assertThatThrownBy(() -> orderService.createOrder(validRequest()))
            .isInstanceOf(ServiceUnavailableException.class)
            .hasCauseInstanceOf(DataAccessResourceFailureException.class);
    }

    @Test
    void whenDuplicateEmail_shouldThrowDuplicateEmailException() {
        when(userRepository.save(any()))
            .thenThrow(new DataIntegrityViolationException("constraint violation",
                new ConstraintViolationException(null, null, "users_email_unique")));

        assertThatThrownBy(() -> userService.createUser(requestWithExistingEmail()))
            .isInstanceOf(DuplicateEmailException.class)
            .hasMessageContaining("already exists");
    }

    @Test
    void createOrder_shouldRollbackOnInventoryFailure() {
        when(inventoryService.reserve(any(), anyInt()))
            .thenThrow(new InsufficientInventoryException("product-1", 5, 2));

        assertThatThrownBy(() -> orderService.createOrder(validRequest()))
            .isInstanceOf(InsufficientInventoryException.class);

        // Verify order was NOT saved (transaction rolled back)
        verify(orderRepository, never()).save(any());
        // Or check with @Rollback and database verification
    }
}
```

---

## Scenario 15: Exception Handling in Spring Scheduler

**Situation:** A scheduled job that processes overnight reports throws an exception. The next scheduled run should still execute.

```java
// PROBLEM: Unhandled exception in @Scheduled method can stop future executions
// (Depends on scheduler configuration, but the task context may be marked failed)
@Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
public void generateNightlyReport() {
    reportService.generate();  // If this throws, is next 2 AM run affected?
}

// CORRECT: Catch all exceptions within scheduled methods
@Scheduled(cron = "0 0 2 * * *")
public void generateNightlyReport() {
    try {
        log.info("Starting nightly report generation");
        reportService.generate();
        log.info("Nightly report generation completed successfully");
    } catch (Exception e) {
        // Log the error but don't let it propagate
        // Spring will still schedule the next execution
        log.error("Nightly report generation failed", e);
        alertingService.sendAlert("Report generation failed: " + e.getMessage());
        metrics.increment("reports.generation.failure");
    }
}

// Configure global scheduled exception handler
@Configuration
public class SchedulerConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setErrorHandler(t -> {
            log.error("Scheduled task failed", t);
            alertingService.sendAlert("Scheduled task error: " + t.getMessage());
        });
        scheduler.initialize();
        registrar.setScheduler(scheduler);
    }
}
```

---

## Scenario 16: Logging Exceptions Without Stack Trace Noise

**Situation:** Your logs are flooded with stack traces for expected exceptions (e.g., validation errors, not-found). How do you control what gets logged?

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Expected exceptions — log at WARN level, no stack trace
    @ExceptionHandler({
        ResourceNotFoundException.class,
        ValidationException.class,
        DuplicateEmailException.class
    })
    public ResponseEntity<ErrorResponse> handleExpectedExceptions(RuntimeException ex,
                                                                    HttpServletRequest request) {
        // No stack trace — these are expected, stack trace is noise
        log.warn("Expected exception on {}: {} - {}", request.getRequestURI(),
                 ex.getClass().getSimpleName(), ex.getMessage());
        // Return appropriate HTTP response
        return buildErrorResponse(ex, request);
    }

    // Unexpected exceptions — log at ERROR with full stack trace
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex,
                                                           HttpServletRequest request) {
        log.error("Unexpected error on {} {}: {}",
                  request.getMethod(), request.getRequestURI(), ex.getMessage(), ex);
        // Don't expose internal details
        return ResponseEntity.status(500).body(
            ErrorResponse.internalError(MDC.get("correlationId")));
    }
}
```

**Logback configuration to suppress specific exceptions:**
```xml
<turboFilter class="ch.qos.logback.classic.turbo.MarkerFilter">
    <!-- Can suppress specific log markers -->
</turboFilter>
```

---

## Scenario 17: Safe Exception Handling with Optional

**Situation:** Rewrite exception-heavy code using `Optional` for cleaner null handling.

```java
// Exception-based approach — verbose and slow
public String getUserEmailDomain(Long userId) {
    try {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        String email = user.getEmail();
        if (email == null) throw new IllegalStateException("User has no email");
        return email.substring(email.indexOf('@') + 1);
    } catch (IndexOutOfBoundsException e) {
        throw new IllegalStateException("Invalid email format: " + user.getEmail());
    }
}

// Optional-based approach — clean, no exceptions for expected absence
public Optional<String> getUserEmailDomain(Long userId) {
    return userRepository.findById(userId)    // Optional<User>
        .map(User::getEmail)                   // Optional<String>
        .filter(email -> email.contains("@")) // Optional<String> (empty if no @)
        .map(email -> email.substring(email.indexOf('@') + 1));
}

// Caller:
String domain = getUserEmailDomain(userId)
    .orElse("unknown.com");

// Or throw only when explicitly needed:
String domain = getUserEmailDomain(userId)
    .orElseThrow(() -> new UserEmailRequiredException(userId));
```

---

## Scenario 18: Exception Safety in Builder Pattern

**Situation:** Ensure that partially built objects are not created when validation fails.

```java
// Builder with validation at build() time — all-or-nothing
public class Order {

    private final String orderId;
    private final Long customerId;
    private final List<OrderItem> items;
    private final BigDecimal total;

    private Order(Builder builder) {
        this.orderId = builder.orderId;
        this.customerId = builder.customerId;
        this.items = List.copyOf(builder.items);
        this.total = builder.total;
    }

    public static class Builder {
        private String orderId;
        private Long customerId;
        private List<OrderItem> items = new ArrayList<>();
        private BigDecimal total;

        public Builder orderId(String orderId) {
            this.orderId = Objects.requireNonNull(orderId, "orderId");
            return this;
        }

        public Builder customerId(Long customerId) {
            this.customerId = Objects.requireNonNull(customerId, "customerId");
            return this;
        }

        public Builder item(OrderItem item) {
            this.items.add(Objects.requireNonNull(item, "item"));
            return this;
        }

        // Validate all constraints at build time — Order is never partially constructed
        public Order build() {
            List<String> errors = new ArrayList<>();

            if (orderId == null || orderId.isBlank()) errors.add("orderId is required");
            if (customerId == null) errors.add("customerId is required");
            if (items.isEmpty()) errors.add("order must have at least one item");

            this.total = items.stream()
                .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
                .reduce(BigDecimal.ZERO, BigDecimal::add);

            if (total.compareTo(BigDecimal.ZERO) <= 0) errors.add("total must be positive");

            if (!errors.isEmpty()) {
                throw new IllegalStateException("Cannot build Order: " + errors);
            }

            return new Order(this);
        }
    }
}
```
