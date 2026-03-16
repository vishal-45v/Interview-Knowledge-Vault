# Advanced Java — Scenario Questions

> 16 real-world scenarios on generics, records, sealed classes, enums, annotations, and modern Java features.

---

## Scenario 1: Type-Safe Heterogeneous Container

**Situation:** You need a container that can store values of different types but retrieve them in a type-safe way without unchecked casts.

```java
// Standard generic container: all elements must be same type
Map<String, Object> map = new HashMap<>();  // Not type-safe — requires casting

// Type-safe heterogeneous container (Bloch, Effective Java Item 33)
public class TypeSafeRegistry {

    private final Map<Class<?>, Object> registry = new ConcurrentHashMap<>();

    public <T> void register(Class<T> type, T instance) {
        registry.put(Objects.requireNonNull(type), type.cast(instance));
    }

    public <T> Optional<T> lookup(Class<T> type) {
        return Optional.ofNullable(type.cast(registry.get(type)));
    }

    public <T> T lookupOrThrow(Class<T> type) {
        return lookup(type).orElseThrow(
            () -> new NoSuchElementException("No instance registered for " + type.getName()));
    }
}

// Usage — type-safe without unchecked casts
TypeSafeRegistry registry = new TypeSafeRegistry();
registry.register(UserService.class, new UserService());
registry.register(OrderService.class, new OrderService());

UserService userService = registry.lookupOrThrow(UserService.class);   // Correct type!
OrderService orderService = registry.lookupOrThrow(OrderService.class); // Correct type!

// Generics with generic types — using ParameterizedTypeReference
// TypeToken pattern for generic types like List<String>
public abstract class TypeToken<T> {
    private final Type type;

    protected TypeToken() {
        // Capture the generic type T via superclass
        this.type = ((ParameterizedType) getClass().getGenericSuperclass())
                        .getActualTypeArguments()[0];
    }

    public Type getType() { return type; }
}

// Usage: new TypeToken<List<String>>() {}  — creates anonymous subclass, captures the type
```

---

## Scenario 2: Generic Repository with Bounded Type Parameters

**Situation:** Create a generic base repository that works with any entity that has an ID.

```java
// Entity marker interface
public interface Entity<ID> {
    ID getId();
}

// Generic repository
public abstract class GenericRepository<T extends Entity<ID>, ID> {

    protected final Class<T> entityClass;

    @SuppressWarnings("unchecked")
    protected GenericRepository() {
        // Capture T's class via reflection (Type erasure workaround)
        this.entityClass = (Class<T>)
            ((ParameterizedType) getClass().getGenericSuperclass())
                .getActualTypeArguments()[0];
    }

    public abstract Optional<T> findById(ID id);
    public abstract T save(T entity);
    public abstract void deleteById(ID id);
    public abstract List<T> findAll();

    // Template method — find or throw
    public T findByIdOrThrow(ID id) {
        return findById(id).orElseThrow(() ->
            new EntityNotFoundException(entityClass.getSimpleName() + " not found: " + id));
    }

    // Batch save with validation
    public List<T> saveAll(Collection<T> entities) {
        return entities.stream()
            .peek(this::validate)
            .map(this::save)
            .collect(Collectors.toList());
    }

    protected void validate(T entity) {
        if (entity.getId() == null) {
            throw new IllegalArgumentException(
                entityClass.getSimpleName() + " ID cannot be null");
        }
    }
}

// Concrete repository
public class OrderRepository extends GenericRepository<Order, Long> {

    @Override
    public Optional<Order> findById(Long id) {
        return Optional.ofNullable(jdbcTemplate.queryForObject(
            "SELECT * FROM orders WHERE id = ?",
            orderRowMapper, id));
    }
    // ... other implementations
}
```

---

## Scenario 3: Enum-Based State Machine

**Situation:** Model an order's state machine using enums with allowed transitions.

```java
public enum OrderStatus {
    CREATED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return EnumSet.of(CONFIRMED, CANCELLED);
        }
    },
    CONFIRMED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return EnumSet.of(PROCESSING, CANCELLED);
        }
    },
    PROCESSING {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return EnumSet.of(SHIPPED, CANCELLED);
        }
    },
    SHIPPED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return EnumSet.of(DELIVERED);
        }
    },
    DELIVERED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return EnumSet.of(RETURNED);
        }
    },
    RETURNED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return EnumSet.noneOf(OrderStatus.class);
        }
    },
    CANCELLED {
        @Override
        public Set<OrderStatus> allowedTransitions() {
            return EnumSet.noneOf(OrderStatus.class);
        }
    };

    public abstract Set<OrderStatus> allowedTransitions();

    public boolean canTransitionTo(OrderStatus newStatus) {
        return allowedTransitions().contains(newStatus);
    }

    public OrderStatus transition(OrderStatus newStatus) {
        if (!canTransitionTo(newStatus)) {
            throw new IllegalStateException(
                String.format("Cannot transition from %s to %s", this, newStatus));
        }
        return newStatus;
    }
}

// Service using the state machine
@Service
public class OrderStateService {

    @Transactional
    public Order transitionOrder(Long orderId, OrderStatus newStatus) {
        Order order = orderRepository.findByIdOrThrow(orderId);
        OrderStatus nextStatus = order.getStatus().transition(newStatus);
        order.setStatus(nextStatus);
        order.addStatusHistory(new StatusHistoryEntry(newStatus, Instant.now()));
        return orderRepository.save(order);
    }
}
```

---

## Scenario 4: Records for Immutable DTOs

**Situation:** Design a REST API request/response model using records.

```java
// Request — immutable, validated at construction
public record CreateOrderRequest(
    @NotNull Long customerId,
    @NotEmpty List<OrderItemRequest> items,
    @NotNull String deliveryAddress,
    @Nullable String promoCode
) {
    // Compact constructor for validation/normalization
    public CreateOrderRequest {
        Objects.requireNonNull(customerId, "customerId");
        Objects.requireNonNull(items, "items");
        if (items.isEmpty()) throw new IllegalArgumentException("items cannot be empty");
        Objects.requireNonNull(deliveryAddress, "deliveryAddress");
        items = List.copyOf(items);  // Defensive copy — ensure immutability
        promoCode = promoCode != null ? promoCode.strip().toUpperCase() : null;
    }

    public record OrderItemRequest(
        @NotNull String productId,
        @Positive int quantity,
        @Nullable BigDecimal negotiatedPrice
    ) {}
}

// Response — immutable, builder-style construction
public record OrderResponse(
    String orderId,
    String status,
    BigDecimal total,
    List<OrderItemResponse> items,
    Instant createdAt
) {
    public record OrderItemResponse(
        String productId,
        String productName,
        int quantity,
        BigDecimal unitPrice,
        BigDecimal lineTotal
    ) {}

    // Factory method for convenient construction
    public static OrderResponse from(Order order) {
        return new OrderResponse(
            order.getId().toString(),
            order.getStatus().name(),
            order.getTotal(),
            order.getItems().stream()
                 .map(item -> new OrderItemResponse(
                     item.getProductId(),
                     item.getProductName(),
                     item.getQuantity(),
                     item.getUnitPrice(),
                     item.getLineTotal()))
                 .collect(Collectors.toList()),
            order.getCreatedAt()
        );
    }
}

// Controller
@PostMapping("/orders")
public ResponseEntity<OrderResponse> createOrder(
        @RequestBody @Valid CreateOrderRequest request) {
    Order order = orderService.createOrder(request);
    return ResponseEntity
        .status(HttpStatus.CREATED)
        .body(OrderResponse.from(order));
}
```

---

## Scenario 5: Sealed Classes for Result Types

**Situation:** Implement a Result type that can be either Success or Failure, using sealed classes for exhaustive pattern matching.

```java
// Sealed Result type
public sealed interface Result<T> permits Result.Success, Result.Failure {

    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String errorCode, String message, Throwable cause) implements Result<T> {
        public Failure(String errorCode, String message) {
            this(errorCode, message, null);
        }
    }

    // Static factories
    static <T> Result<T> success(T value) { return new Success<>(value); }
    static <T> Result<T> failure(String code, String message) {
        return new Failure<>(code, message);
    }

    // Helper methods via default/static interface methods
    default boolean isSuccess() { return this instanceof Success; }
    default boolean isFailure() { return this instanceof Failure; }

    @SuppressWarnings("unchecked")
    default T getValue() {
        return switch (this) {
            case Success<T> s -> s.value();
            case Failure<T> f -> throw new NoSuchElementException("Result is a failure: " + f.message());
        };
    }

    default <U> Result<U> map(Function<T, U> mapper) {
        return switch (this) {
            case Success<T> s -> Result.success(mapper.apply(s.value()));
            case Failure<T> f -> Result.failure(f.errorCode(), f.message());
        };
    }

    default <U> Result<U> flatMap(Function<T, Result<U>> mapper) {
        return switch (this) {
            case Success<T> s -> mapper.apply(s.value());
            case Failure<T> f -> Result.failure(f.errorCode(), f.message());
        };
    }

    default void ifSuccess(Consumer<T> consumer) {
        if (this instanceof Success<T> s) consumer.accept(s.value());
    }
}

// Service returning Result
@Service
public class OrderValidationService {

    public Result<ValidatedOrder> validate(CreateOrderRequest request) {
        if (!customerExists(request.customerId())) {
            return Result.failure("CUSTOMER_NOT_FOUND", "Customer " + request.customerId() + " not found");
        }
        for (var item : request.items()) {
            if (!productExists(item.productId())) {
                return Result.failure("PRODUCT_NOT_FOUND", "Product " + item.productId() + " not found");
            }
        }
        return Result.success(new ValidatedOrder(request));
    }
}

// Usage with pattern matching
Result<Order> result = validationService.validate(request)
    .flatMap(validated -> orderService.create(validated));

switch (result) {
    case Result.Success<Order> s -> log.info("Created order: {}", s.value().getId());
    case Result.Failure<Order> f -> log.error("Failed: {} - {}", f.errorCode(), f.message());
}
```

---

## Scenario 6: Custom Annotation for Rate Limiting

**Situation:** Implement a `@RateLimit` annotation that can be applied to controller methods.

```java
// Annotation definition
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface RateLimit {
    int requests() default 100;           // Max requests
    int windowSeconds() default 60;       // Per time window
    String key() default "";              // Key expression (SpEL)
    RateLimitStrategy strategy() default RateLimitStrategy.PER_IP;

    enum RateLimitStrategy { PER_IP, PER_USER, PER_API_KEY, GLOBAL }
}

// AOP-based processor
@Aspect
@Component
public class RateLimitAspect {

    @Autowired private RateLimitService rateLimitService;
    @Autowired private ExpressionParser spelParser;

    @Around("@annotation(rateLimit)")
    public Object enforceRateLimit(ProceedingJoinPoint pjp, RateLimit rateLimit)
            throws Throwable {

        String key = buildKey(pjp, rateLimit);

        if (!rateLimitService.tryAcquire(key, rateLimit.requests(), rateLimit.windowSeconds())) {
            throw new RateLimitExceededException(
                "Rate limit exceeded: " + rateLimit.requests() +
                " requests per " + rateLimit.windowSeconds() + "s");
        }

        return pjp.proceed();
    }

    private String buildKey(ProceedingJoinPoint pjp, RateLimit rateLimit) {
        String baseKey = pjp.getSignature().toShortString();

        if (!rateLimit.key().isEmpty()) {
            // Evaluate SpEL expression for custom keys
            MethodSignature sig = (MethodSignature) pjp.getSignature();
            EvaluationContext context = new StandardEvaluationContext();
            String[] paramNames = sig.getParameterNames();
            Object[] args = pjp.getArgs();
            for (int i = 0; i < paramNames.length; i++) {
                context.setVariable(paramNames[i], args[i]);
            }
            return baseKey + ":" + spelParser.parseExpression(rateLimit.key()).getValue(context);
        }

        return switch (rateLimit.strategy()) {
            case PER_IP -> baseKey + ":ip:" + getCurrentIp();
            case PER_USER -> baseKey + ":user:" + getCurrentUserId();
            case PER_API_KEY -> baseKey + ":key:" + getCurrentApiKey();
            case GLOBAL -> baseKey;
        };
    }
}

// Usage
@RestController
public class SearchController {

    @GetMapping("/search")
    @RateLimit(requests = 10, windowSeconds = 1, strategy = RateLimit.RateLimitStrategy.PER_IP)
    public SearchResults search(@RequestParam String query) { ... }

    @GetMapping("/reports/{reportId}")
    @RateLimit(requests = 5, windowSeconds = 60, key = "#reportId",
               strategy = RateLimit.RateLimitStrategy.PER_USER)
    public Report getReport(@PathVariable String reportId) { ... }
}
```

---

## Scenario 7: Generic Event Bus

**Situation:** Build a type-safe in-process event bus where handlers receive only their specific event type.

```java
// Event base (or use a marker interface)
public abstract class Event {
    private final Instant occurredAt = Instant.now();
    public Instant getOccurredAt() { return occurredAt; }
}

// Event handler functional interface
@FunctionalInterface
public interface EventHandler<T extends Event> {
    void handle(T event);
}

// Type-safe event bus
@Component
public class EventBus {

    // Map from event class to list of handlers
    private final Map<Class<?>, List<EventHandler<? extends Event>>> handlers
        = new ConcurrentHashMap<>();

    @SuppressWarnings("unchecked")
    public <T extends Event> void subscribe(Class<T> eventType, EventHandler<T> handler) {
        handlers.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                .add(handler);
    }

    @SuppressWarnings("unchecked")
    public <T extends Event> void publish(T event) {
        List<EventHandler<? extends Event>> eventHandlers =
            handlers.getOrDefault(event.getClass(), List.of());

        for (EventHandler handler : eventHandlers) {  // Raw type for invocation
            try {
                handler.handle(event);
            } catch (Exception e) {
                log.error("Error in event handler for {}", event.getClass().getSimpleName(), e);
            }
        }
    }
}

// Usage
@Component
public class OrderEventHandlers {

    @Autowired private EventBus eventBus;

    @PostConstruct
    public void registerHandlers() {
        eventBus.subscribe(OrderCreatedEvent.class, event -> {
            notificationService.sendOrderConfirmation(event.getOrderId());
        });

        eventBus.subscribe(OrderShippedEvent.class, event -> {
            trackingService.startTracking(event.getOrderId(), event.getTrackingNumber());
        });
    }
}
```

---

## Scenario 8: Builder Pattern with Generic Type

**Situation:** Create a paginated query builder that is type-safe and reusable across entity types.

```java
public class PagedQuery<T> {

    private final Class<T> type;
    private final String sortField;
    private final boolean ascending;
    private final int page;
    private final int size;
    private final Map<String, Object> filters;

    private PagedQuery(Builder<T> builder) {
        this.type = builder.type;
        this.sortField = builder.sortField;
        this.ascending = builder.ascending;
        this.page = builder.page;
        this.size = builder.size;
        this.filters = Map.copyOf(builder.filters);
    }

    public static <T> Builder<T> of(Class<T> type) {
        return new Builder<>(type);
    }

    public static class Builder<T> {
        private final Class<T> type;
        private String sortField = "id";
        private boolean ascending = true;
        private int page = 0;
        private int size = 20;
        private final Map<String, Object> filters = new LinkedHashMap<>();

        private Builder(Class<T> type) {
            this.type = Objects.requireNonNull(type);
        }

        public Builder<T> sortBy(String field, boolean ascending) {
            this.sortField = field;
            this.ascending = ascending;
            return this;
        }

        public Builder<T> page(int page) {
            if (page < 0) throw new IllegalArgumentException("Page must be >= 0");
            this.page = page;
            return this;
        }

        public Builder<T> size(int size) {
            if (size < 1 || size > 1000) throw new IllegalArgumentException("Size must be 1-1000");
            this.size = size;
            return this;
        }

        public Builder<T> filter(String field, Object value) {
            filters.put(field, value);
            return this;
        }

        public PagedQuery<T> build() {
            return new PagedQuery<>(this);
        }
    }
}

// Usage
PagedQuery<Order> query = PagedQuery.of(Order.class)
    .sortBy("createdAt", false)
    .page(0).size(25)
    .filter("status", "PENDING")
    .filter("customerId", customerId)
    .build();

Page<Order> orders = orderRepository.findAll(query);
```

---

## Scenario 9: Annotation Processor for Code Validation

**Situation:** Create a custom `@ValidOrderAmount` annotation that validates BigDecimal fields during bean validation.

```java
// Custom constraint annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = OrderAmountValidator.class)
@Documented
public @interface ValidOrderAmount {
    String message() default "Order amount must be positive and within allowed range";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    BigDecimal min() default "0.01";
    BigDecimal max() default "100000.00";
}

// Validator implementation
public class OrderAmountValidator implements ConstraintValidator<ValidOrderAmount, BigDecimal> {

    private BigDecimal min;
    private BigDecimal max;

    @Override
    public void initialize(ValidOrderAmount annotation) {
        this.min = new BigDecimal(annotation.min());
        this.max = new BigDecimal(annotation.max());
    }

    @Override
    public boolean isValid(BigDecimal value, ConstraintValidatorContext context) {
        if (value == null) return true;  // Use @NotNull separately

        if (value.compareTo(min) < 0) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(
                "Amount must be at least " + min)
                .addConstraintViolation();
            return false;
        }

        if (value.compareTo(max) > 0) {
            context.disableDefaultConstraintViolation();
            context.buildConstraintViolationWithTemplate(
                "Amount must not exceed " + max)
                .addConstraintViolation();
            return false;
        }

        return true;
    }
}

// Usage
public record CreateOrderRequest(
    @NotNull Long customerId,
    @NotNull @ValidOrderAmount BigDecimal amount
) {}
```

---

## Scenario 10: Fluent API with Generics

**Situation:** Build a fluent HTTP request builder that returns different types based on the response class.

```java
public class FluentHttpClient {

    private final RestTemplate restTemplate;

    public RequestBuilder<Void> post(String url) {
        return new RequestBuilder<>(HttpMethod.POST, url);
    }

    public RequestBuilder<Void> get(String url) {
        return new RequestBuilder<>(HttpMethod.GET, url);
    }

    public class RequestBuilder<B> {
        private final HttpMethod method;
        private final String url;
        private final HttpHeaders headers = new HttpHeaders();
        private Object body;

        private RequestBuilder(HttpMethod method, String url) {
            this.method = method;
            this.url = url;
            headers.setContentType(MediaType.APPLICATION_JSON);
        }

        public RequestBuilder<B> header(String name, String value) {
            headers.set(name, value);
            return this;
        }

        public RequestBuilder<B> bearer(String token) {
            headers.setBearerAuth(token);
            return this;
        }

        public RequestBuilder<B> body(Object body) {
            this.body = body;
            return this;
        }

        // Terminal method — specify return type
        public <T> T as(Class<T> responseType) {
            HttpEntity<Object> entity = new HttpEntity<>(body, headers);
            return restTemplate.exchange(url, method, entity, responseType).getBody();
        }

        public <T> T as(ParameterizedTypeReference<T> typeRef) {
            HttpEntity<Object> entity = new HttpEntity<>(body, headers);
            return restTemplate.exchange(url, method, entity, typeRef).getBody();
        }

        public void execute() {
            as(Void.class);
        }
    }
}

// Usage
User user = httpClient.get("https://api.example.com/users/123")
    .bearer(accessToken)
    .as(User.class);

List<Order> orders = httpClient.get("https://api.example.com/orders")
    .header("X-Page", "1")
    .as(new ParameterizedTypeReference<List<Order>>() {});

httpClient.post("https://api.example.com/orders")
    .bearer(token)
    .body(new CreateOrderRequest(customerId, items))
    .execute();
```

---

## Scenario 11: Pattern Matching with Records

**Situation:** Process different types of payment events using pattern matching.

```java
// Sealed event hierarchy
public sealed interface PaymentEvent permits
    PaymentEvent.PaymentCreated,
    PaymentEvent.PaymentAuthorized,
    PaymentEvent.PaymentCaptured,
    PaymentEvent.PaymentFailed,
    PaymentEvent.PaymentRefunded {

    String paymentId();
    Instant occurredAt();

    record PaymentCreated(String paymentId, BigDecimal amount,
                          String currency, Instant occurredAt) implements PaymentEvent {}

    record PaymentAuthorized(String paymentId, String authCode,
                             Instant occurredAt) implements PaymentEvent {}

    record PaymentCaptured(String paymentId, BigDecimal capturedAmount,
                           Instant occurredAt) implements PaymentEvent {}

    record PaymentFailed(String paymentId, String errorCode,
                         String errorMessage, Instant occurredAt) implements PaymentEvent {}

    record PaymentRefunded(String paymentId, BigDecimal refundAmount,
                           String reason, Instant occurredAt) implements PaymentEvent {}
}

// Processor using pattern matching
@Service
public class PaymentEventProcessor {

    public void process(PaymentEvent event) {
        // Exhaustive switch — compiler error if a case is missing
        switch (event) {
            case PaymentEvent.PaymentCreated e -> {
                log.info("Payment {} created for {} {}",
                         e.paymentId(), e.amount(), e.currency());
                metrics.increment("payments.created");
                notifyCustomer(e.paymentId(), "Payment initiated");
            }
            case PaymentEvent.PaymentAuthorized e -> {
                log.info("Payment {} authorized with code {}",
                         e.paymentId(), e.authCode());
                orderService.updatePaymentStatus(e.paymentId(), "AUTHORIZED");
            }
            case PaymentEvent.PaymentCaptured e -> {
                log.info("Payment {} captured: {}", e.paymentId(), e.capturedAmount());
                orderService.confirmPayment(e.paymentId());
                notifyCustomer(e.paymentId(), "Payment confirmed");
            }
            case PaymentEvent.PaymentFailed e -> {
                log.error("Payment {} failed: {} - {}", e.paymentId(), e.errorCode(), e.errorMessage());
                orderService.markPaymentFailed(e.paymentId(), e.errorCode());
                notifyCustomer(e.paymentId(), "Payment failed: " + e.errorMessage());
                if ("INSUFFICIENT_FUNDS".equals(e.errorCode())) {
                    metrics.increment("payments.failed.insufficient_funds");
                }
            }
            case PaymentEvent.PaymentRefunded e -> {
                log.info("Payment {} refunded: {} for reason: {}",
                         e.paymentId(), e.refundAmount(), e.reason());
                orderService.processRefund(e.paymentId(), e.refundAmount());
            }
        }
    }
}
```

---

## Scenario 12: Functional Configuration with Lambdas

**Situation:** Design a Spring Boot configuration class that builds complex objects using functional composition.

```java
@Configuration
public class PipelineConfig {

    // Build a processing pipeline using function composition
    @Bean
    public Function<RawOrder, ProcessedOrder> orderProcessingPipeline(
            OrderValidator validator,
            PricingService pricingService,
            InventoryService inventoryService) {

        // Each step is a Function<T, T> that transforms the order
        Function<RawOrder, ValidatedOrder> validateStep =
            raw -> validator.validate(raw)
                            .orElseThrow(() -> new ValidationException("Invalid order"));

        Function<ValidatedOrder, PricedOrder> priceStep =
            validated -> pricingService.applyPricing(validated);

        Function<PricedOrder, ProcessedOrder> inventoryStep =
            priced -> inventoryService.checkAndReserve(priced);

        // Compose the pipeline — andThen chains functions
        return validateStep
            .andThen(priceStep)
            .andThen(inventoryStep);
    }

    // Build a validation chain using Predicate composition
    @Bean
    public Predicate<Order> orderFraudFilter(FraudDetectionService fraudService) {
        Predicate<Order> amountCheck = order ->
            order.getTotal().compareTo(new BigDecimal("50000")) < 0;

        Predicate<Order> velocityCheck = order ->
            fraudService.getOrdersLastHour(order.getCustomerId()) < 10;

        Predicate<Order> countryCheck = order ->
            !fraudService.isHighRiskCountry(order.getShippingCountry());

        // Combine all checks
        return amountCheck.and(velocityCheck).and(countryCheck);
    }
}
```

---

## Scenario 13: Using Records for Value Objects in Domain Model

**Situation:** Implement domain value objects (DDD) using Java records.

```java
// Money value object — immutable, value semantics
public record Money(BigDecimal amount, Currency currency) {

    // Compact constructor — validation and normalization
    public Money {
        Objects.requireNonNull(amount, "amount");
        Objects.requireNonNull(currency, "currency");
        if (amount.scale() > currency.getDefaultFractionDigits()) {
            throw new IllegalArgumentException(
                "Amount precision exceeds currency scale: " + currency);
        }
        amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_UP);
    }

    // Factory methods
    public static Money of(String amount, String currencyCode) {
        return new Money(new BigDecimal(amount), Currency.getInstance(currencyCode));
    }

    public static Money zero(Currency currency) {
        return new Money(BigDecimal.ZERO, currency);
    }

    // Business methods
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(amount.add(other.amount), currency);
    }

    public Money multiply(int factor) {
        return new Money(amount.multiply(BigDecimal.valueOf(factor)), currency);
    }

    public Money multiply(BigDecimal factor) {
        return new Money(amount.multiply(factor), currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }

    private void assertSameCurrency(Money other) {
        if (!currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "Currency mismatch: " + currency + " vs " + other.currency);
        }
    }

    @Override
    public String toString() {
        return NumberFormat.getCurrencyInstance().format(amount) + " " + currency;
    }
}

// Address value object
public record Address(
    String street,
    String city,
    String state,
    String postalCode,
    String countryCode
) {
    public Address {
        Objects.requireNonNull(street, "street");
        Objects.requireNonNull(city, "city");
        Objects.requireNonNull(postalCode, "postalCode");
        countryCode = countryCode != null ? countryCode.toUpperCase() : "US";
    }

    public String formatted() {
        return String.format("%s\n%s, %s %s\n%s", street, city, state, postalCode, countryCode);
    }
}
```

---

## Scenario 14: Annotation-Driven Caching Configuration

**Situation:** Create a custom `@Cached` annotation with configurable TTL and cache name.

```java
// Custom cache annotation
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Cached {
    String cacheName();
    long ttlSeconds() default 300;         // 5 minutes default
    String keyExpression() default "";     // SpEL expression for cache key
    boolean refreshAsync() default false;  // Refresh in background when near expiry
}

// AOP implementation
@Aspect
@Component
public class CachedAspect {

    @Autowired private CacheManager cacheManager;
    @Autowired private ExpressionParser spelParser;

    @Around("@annotation(cached)")
    public Object cachedInvocation(ProceedingJoinPoint pjp, Cached cached) throws Throwable {
        String key = buildCacheKey(pjp, cached);
        Cache cache = cacheManager.getCache(cached.cacheName());

        if (cache != null) {
            Cache.ValueWrapper wrapper = cache.get(key);
            if (wrapper != null) {
                return wrapper.get();
            }
        }

        Object result = pjp.proceed();

        if (cache != null && result != null) {
            cache.put(key, result);
            // Configure TTL if using Redis-backed cache
            if (cache instanceof RedisCacheManager) {
                // Set expiry via RedisTemplate
            }
        }

        return result;
    }

    private String buildCacheKey(ProceedingJoinPoint pjp, Cached cached) {
        if (cached.keyExpression().isEmpty()) {
            return pjp.getSignature().toShortString() + ":" +
                   Arrays.toString(pjp.getArgs());
        }
        MethodSignature sig = (MethodSignature) pjp.getSignature();
        EvaluationContext ctx = new StandardEvaluationContext();
        String[] params = sig.getParameterNames();
        Object[] args = pjp.getArgs();
        for (int i = 0; i < params.length; i++) ctx.setVariable(params[i], args[i]);
        return String.valueOf(spelParser.parseExpression(cached.keyExpression()).getValue(ctx));
    }
}

// Usage
@Service
public class ProductService {

    @Cached(cacheName = "products", ttlSeconds = 600, keyExpression = "#productId")
    public Product getProduct(String productId) {
        return productRepository.findById(productId).orElseThrow();
    }

    @Cached(cacheName = "product-search", ttlSeconds = 60,
            keyExpression = "#category + ':' + #page + ':' + #size")
    public Page<Product> searchByCategory(String category, int page, int size) {
        return productRepository.findByCategory(category, PageRequest.of(page, size));
    }
}
```

---

## Scenario 15: Implementing Visitor Pattern with Sealed Classes

**Situation:** Process a heterogeneous tree of pricing rules without instanceof checks.

```java
// Sealed pricing rule hierarchy
public sealed interface PricingRule permits
    FixedDiscountRule, PercentageDiscountRule, BuyXGetYRule, VolumeDiscountRule {

    record FixedDiscountRule(Money discountAmount, String description) implements PricingRule {}
    record PercentageDiscountRule(double percentage, double maxDiscountAmount) implements PricingRule {}
    record BuyXGetYRule(int buyQuantity, int freeQuantity, String productId) implements PricingRule {}
    record VolumeDiscountRule(Map<Integer, Double> quantityToPercentage) implements PricingRule {}
}

// Calculator using pattern matching — exhaustive, type-safe
@Service
public class PricingCalculator {

    public Money calculateDiscount(PricingRule rule, OrderItem item) {
        return switch (rule) {
            case FixedDiscountRule r -> {
                // Fixed amount off, capped at item total
                Money itemTotal = item.getUnitPrice().multiply(item.getQuantity());
                yield r.discountAmount().isGreaterThan(itemTotal) ? itemTotal : r.discountAmount();
            }

            case PercentageDiscountRule r -> {
                Money itemTotal = item.getUnitPrice().multiply(item.getQuantity());
                BigDecimal discount = itemTotal.amount()
                    .multiply(BigDecimal.valueOf(r.percentage() / 100));
                BigDecimal maxDiscount = BigDecimal.valueOf(r.maxDiscountAmount());
                BigDecimal finalDiscount = discount.min(maxDiscount);
                yield new Money(finalDiscount, itemTotal.currency());
            }

            case BuyXGetYRule r -> {
                if (!r.productId().equals(item.getProductId())) yield Money.zero(item.getCurrency());
                int freeItems = (item.getQuantity() / r.buyQuantity()) * r.freeQuantity();
                yield item.getUnitPrice().multiply(Math.min(freeItems, item.getQuantity()));
            }

            case VolumeDiscountRule r -> {
                int qty = item.getQuantity();
                double pct = r.quantityToPercentage().entrySet().stream()
                    .filter(e -> qty >= e.getKey())
                    .mapToDouble(Map.Entry::getValue)
                    .max().orElse(0.0);
                Money itemTotal = item.getUnitPrice().multiply(qty);
                yield new Money(
                    itemTotal.amount().multiply(BigDecimal.valueOf(pct / 100)),
                    itemTotal.currency());
            }
        };
    }
}
```

---

## Scenario 16: Generic Specification Pattern for Queries

**Situation:** Implement a type-safe specification/criteria building pattern for dynamic queries.

```java
// Specification interface
@FunctionalInterface
public interface Specification<T> {
    boolean isSatisfiedBy(T candidate);

    default Specification<T> and(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) && other.isSatisfiedBy(candidate);
    }

    default Specification<T> or(Specification<T> other) {
        return candidate -> this.isSatisfiedBy(candidate) || other.isSatisfiedBy(candidate);
    }

    default Specification<T> negate() {
        return candidate -> !this.isSatisfiedBy(candidate);
    }

    // Filter a collection
    default List<T> filter(List<T> candidates) {
        return candidates.stream().filter(this::isSatisfiedBy).collect(Collectors.toList());
    }
}

// Order specifications
public class OrderSpecifications {

    public static Specification<Order> isPending() {
        return order -> order.getStatus() == OrderStatus.PENDING;
    }

    public static Specification<Order> isForCustomer(Long customerId) {
        return order -> customerId.equals(order.getCustomerId());
    }

    public static Specification<Order> hasMinimumTotal(Money minimumAmount) {
        return order -> !order.getTotal().isGreaterThan(minimumAmount);
    }

    public static Specification<Order> createdAfter(Instant timestamp) {
        return order -> order.getCreatedAt().isAfter(timestamp);
    }

    public static Specification<Order> isEligibleForPromotion(String promoCode) {
        return isPending()
            .and(hasMinimumTotal(Money.of("50.00", "USD")))
            .and(createdAfter(Instant.now().minus(30, ChronoUnit.DAYS)));
    }
}

// Usage — compose specifications
Specification<Order> criteria = OrderSpecifications.isForCustomer(customerId)
    .and(OrderSpecifications.isPending())
    .and(OrderSpecifications.createdAfter(sevenDaysAgo));

List<Order> matchingOrders = criteria.filter(allOrders);

// Bridge to Spring Data JPA Specification
public static org.springframework.data.jpa.domain.Specification<Order>
        toJpaSpec(Specification<Order> spec) {
    return (root, query, cb) -> {
        // Would need to translate to JPA Predicate
        // In practice, use Spring Data's Specification directly
        throw new UnsupportedOperationException("Use JPA Specification instead");
    };
}
```
