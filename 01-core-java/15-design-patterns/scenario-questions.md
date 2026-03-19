# Design Patterns — Scenario Questions

## How to Use This File
Each scenario presents a realistic engineering problem. Identify the applicable pattern(s), implement the solution, and explain the trade-offs.

---

## Scenario 1: Payment Processor Strategy with Runtime Selection

**Setup:** Your e-commerce platform needs to support Stripe, PayPal, and Braintree. The processor is chosen at runtime based on region, currency, and customer preference.

```java
public interface PaymentProcessor {
    PaymentResult charge(PaymentRequest request);
    RefundResult refund(String transactionId, BigDecimal amount);
    boolean supports(Currency currency, Region region);
}

@Component("stripe")
public class StripeProcessor implements PaymentProcessor {
    private final StripeClient stripe;

    @Override
    public PaymentResult charge(PaymentRequest request) {
        PaymentIntent intent = stripe.paymentIntents().create(
            PaymentIntentCreateParams.builder()
                .setAmount(request.getAmountCents())
                .setCurrency(request.getCurrency().getCode())
                .build()
        );
        return PaymentResult.success(intent.getId());
    }

    @Override
    public boolean supports(Currency currency, Region region) {
        return region != Region.CHINA;  // Stripe not available in China
    }
}

// Strategy selector — Factory + Strategy combined
@Service
public class PaymentService {

    private final List<PaymentProcessor> processors;  // All processors injected

    // Spring injects all PaymentProcessor beans as a list
    public PaymentService(List<PaymentProcessor> processors) {
        this.processors = processors;
    }

    public PaymentResult processPayment(PaymentRequest request) {
        PaymentProcessor processor = processors.stream()
            .filter(p -> p.supports(request.getCurrency(), request.getRegion()))
            .filter(p -> p.getClass().getSimpleName().toLowerCase()
                .contains(request.getPreferredProvider().toLowerCase()))
            .findFirst()
            .orElseThrow(() -> new NoSupportedProcessorException(
                "No processor for " + request.getCurrency() + " in " + request.getRegion()));

        return processor.charge(request);
    }
}
```

**Interview discussion points:**
- Adding a new processor (e.g., Adyen) = new `@Component` class, zero changes to `PaymentService` → OCP satisfied
- Testing: inject `List.of(mockProcessor)` — each processor unit-tested independently
- Chain variant: if the first processor fails, try the next (Chain of Responsibility + Strategy)

---

## Scenario 2: Decorator Chain for HTTP Client

**Setup:** You need an HTTP client with optional capabilities: retry, caching, metrics, circuit breaking. Each capability should be independently composable, not inherited.

```java
public interface HttpClient {
    Response execute(Request request);
}

// Base implementation
public class SimpleHttpClient implements HttpClient {
    @Override
    public Response execute(Request request) {
        // actual HTTP call using java.net.http.HttpClient
        return doExecute(request);
    }
}

// Decorators
public class RetryingHttpClient implements HttpClient {
    private final HttpClient delegate;
    private final int maxAttempts;

    @Override
    public Response execute(Request request) {
        int attempts = 0;
        while (true) {
            try {
                return delegate.execute(request);
            } catch (TransientException e) {
                if (++attempts >= maxAttempts) throw e;
                backoff(attempts);
            }
        }
    }
}

public class CachingHttpClient implements HttpClient {
    private final HttpClient delegate;
    private final Cache<String, Response> cache;

    @Override
    public Response execute(Request request) {
        if (!"GET".equals(request.getMethod())) {
            return delegate.execute(request);  // only cache GETs
        }
        return cache.get(request.getCacheKey(), key -> delegate.execute(request));
    }
}

public class MetricsHttpClient implements HttpClient {
    private final HttpClient delegate;
    private final MeterRegistry registry;

    @Override
    public Response execute(Request request) {
        Timer.Sample sample = Timer.start(registry);
        try {
            Response response = delegate.execute(request);
            sample.stop(registry.timer("http.client",
                "status", String.valueOf(response.getStatus()),
                "method", request.getMethod()));
            return response;
        } catch (Exception e) {
            sample.stop(registry.timer("http.client", "status", "error"));
            throw e;
        }
    }
}

// Assembly via Spring @Bean
@Configuration
public class HttpClientConfig {

    @Bean
    public HttpClient inventoryClient(MeterRegistry registry) {
        return new MetricsHttpClient(
            new RetryingHttpClient(
                new CachingHttpClient(
                    new SimpleHttpClient(),
                    buildCache()
                ),
                3
            ),
            registry
        );
    }
}
```

**Key insight:** Order matters. `Metrics → Retry → Cache → HTTP` means metrics measure total time including retries. If you want metrics per attempt, put Metrics inside Retry.

---

## Scenario 3: Domain Event System with Observer Pattern

**Setup:** Design an event system for domain events that supports synchronous and asynchronous handlers, transactional boundaries, and dead letter queue for failed handlers.

```java
// Domain event base
public abstract class DomainEvent {
    private final UUID eventId = UUID.randomUUID();
    private final Instant occurredAt = Instant.now();
    // getters...
}

// Events
public class OrderShippedEvent extends DomainEvent {
    private final Long orderId;
    private final String trackingNumber;
}

// Transactional outbox pattern — events stored in DB, published after commit
@Entity
public class OutboxEvent {
    @Id @GeneratedValue private Long id;
    private String eventType;
    @Column(columnDefinition = "TEXT")
    private String payload;
    private Instant createdAt;
    private boolean published = false;
}

@Service
@Transactional
public class OrderService {
    private final OutboxEventRepository outbox;
    private final ObjectMapper mapper;

    public Order shipOrder(Long orderId, String trackingNumber) {
        Order order = findAndValidate(orderId);
        order.ship(trackingNumber);

        // Store event in same transaction — guaranteed consistency
        OrderShippedEvent event = new OrderShippedEvent(orderId, trackingNumber);
        outbox.save(new OutboxEvent(
            event.getClass().getSimpleName(),
            mapper.writeValueAsString(event),
            Instant.now()
        ));

        return order;
    }
}

// Poller publishes outbox events after commit
@Component
public class OutboxPoller {

    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void publishPending() {
        List<OutboxEvent> pending = outbox.findTop100ByPublishedFalseOrderByCreatedAtAsc();
        pending.forEach(event -> {
            try {
                kafkaTemplate.send(event.getEventType(), event.getPayload());
                event.setPublished(true);
            } catch (Exception e) {
                log.error("Failed to publish event {}", event.getId(), e);
                // Will retry on next poll
            }
        });
    }
}
```

**vs Spring's `@EventListener`:** Spring events are in-memory and synchronous within the transaction by default. For cross-service events or guaranteed delivery, use the Outbox pattern.

---

## Scenario 4: Command Pattern for Undo/Redo in a Document Editor API

**Setup:** A collaborative document editing API needs full undo/redo support. Implement using the Command pattern.

```java
public interface DocumentCommand {
    void execute(Document document);
    void undo(Document document);
    String describe();  // for audit log
}

public class InsertTextCommand implements DocumentCommand {
    private final int position;
    private final String text;

    @Override
    public void execute(Document doc) {
        doc.insertAt(position, text);
    }

    @Override
    public void undo(Document doc) {
        doc.deleteRange(position, position + text.length());
    }
}

public class DeleteTextCommand implements DocumentCommand {
    private final int start;
    private final int end;
    private String deletedText;  // captured during execute for undo

    @Override
    public void execute(Document doc) {
        deletedText = doc.getRange(start, end);  // snapshot for undo
        doc.deleteRange(start, end);
    }

    @Override
    public void undo(Document doc) {
        doc.insertAt(start, deletedText);
    }
}

// Command history manager
@Component
@Scope("prototype")  // one per editing session
public class DocumentEditor {
    private final Deque<DocumentCommand> undoStack = new ArrayDeque<>();
    private final Deque<DocumentCommand> redoStack = new ArrayDeque<>();
    private final int maxHistorySize = 100;

    public void execute(DocumentCommand command, Document document) {
        command.execute(document);
        undoStack.push(command);
        redoStack.clear();  // new action clears redo stack

        if (undoStack.size() > maxHistorySize) {
            // Remove oldest entries (bottom of deque)
            ((ArrayDeque<DocumentCommand>) undoStack).removeLast();
        }
    }

    public void undo(Document document) {
        if (undoStack.isEmpty()) return;
        DocumentCommand command = undoStack.pop();
        command.undo(document);
        redoStack.push(command);
    }

    public void redo(Document document) {
        if (redoStack.isEmpty()) return;
        DocumentCommand command = redoStack.pop();
        command.execute(document);
        undoStack.push(command);
    }

    // Macro command — composite command
    public DocumentCommand macro(List<DocumentCommand> commands) {
        return new MacroCommand(commands);
    }
}

// Composite Command
public class MacroCommand implements DocumentCommand {
    private final List<DocumentCommand> commands;

    @Override
    public void execute(Document doc) {
        commands.forEach(c -> c.execute(doc));
    }

    @Override
    public void undo(Document doc) {
        // Undo in reverse order
        ListIterator<DocumentCommand> it = commands.listIterator(commands.size());
        while (it.hasPrevious()) {
            it.previous().undo(doc);
        }
    }
}
```

---

## Scenario 5: Template Method for Multi-Format Report Generation

**Setup:** Reports need to be generated in PDF, Excel, and CSV formats. The process is always: gather data → validate → format → write output. Only the formatting step differs.

```java
public abstract class ReportGenerator<T> {

    // Template method — sealed, subclasses cannot override
    public final byte[] generate(ReportRequest request) {
        log.info("Generating {} report for request {}", getFormat(), request.getId());

        List<T> data = gatherData(request);        // subclasses can override for custom queries
        validate(data);                             // fixed business rule
        byte[] content = format(data, request);    // abstract — must implement
        audit(request, content.length);            // fixed cross-cutting concern
        return content;
    }

    // Subclass must implement
    protected abstract byte[] format(List<T> data, ReportRequest request);
    protected abstract String getFormat();

    // Subclass may override (hook methods)
    protected List<T> gatherData(ReportRequest request) {
        return reportRepository.findByDateRange(request.getFrom(), request.getTo());
    }

    // Fixed steps — not overridable
    private void validate(List<T> data) {
        if (data.isEmpty()) throw new EmptyReportException("No data for report period");
    }

    private void audit(ReportRequest request, int bytes) {
        auditLog.record("REPORT_GENERATED", Map.of(
            "format", getFormat(),
            "requestId", request.getId(),
            "bytes", bytes
        ));
    }
}

@Component
public class PdfReportGenerator extends ReportGenerator<SalesRecord> {
    private final PdfRenderer pdfRenderer;

    @Override
    protected byte[] format(List<SalesRecord> data, ReportRequest request) {
        return pdfRenderer.render(buildPdfDocument(data, request));
    }

    @Override
    protected String getFormat() { return "PDF"; }
}

@Component
public class ExcelReportGenerator extends ReportGenerator<SalesRecord> {

    @Override
    protected byte[] format(List<SalesRecord> data, ReportRequest request) {
        try (Workbook wb = new XSSFWorkbook()) {
            Sheet sheet = wb.createSheet("Sales");
            // populate sheet...
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            wb.write(out);
            return out.toByteArray();
        }
    }

    @Override
    protected String getFormat() { return "XLSX"; }
}

// Factory to select generator based on requested format
@Service
public class ReportService {
    private final Map<String, ReportGenerator<?>> generators;

    public byte[] generateReport(ReportRequest request) {
        ReportGenerator<?> generator = generators.get(request.getFormat().toUpperCase());
        if (generator == null) throw new UnsupportedFormatException(request.getFormat());
        return generator.generate(request);
    }
}
```

---

## Scenario 6: Builder Pattern for Complex Test Data

**Setup:** Integration tests need to create complex domain objects with many related entities. Implement a fluent builder that also handles persistence.

```java
// Test fixture builder — readable, maintainable test setup
public class OrderTestBuilder {
    private Long customerId = 1L;
    private List<OrderItem> items = new ArrayList<>();
    private OrderStatus status = OrderStatus.PENDING;
    private LocalDateTime createdAt = LocalDateTime.now();
    private String shippingAddress = "123 Main St, Springfield";

    private final OrderRepository orderRepository;  // for persistence

    public static OrderTestBuilder anOrder(OrderRepository repo) {
        return new OrderTestBuilder(repo);
    }

    public OrderTestBuilder forCustomer(Long customerId) {
        this.customerId = customerId;
        return this;
    }

    public OrderTestBuilder withItem(String sku, int qty, BigDecimal price) {
        this.items.add(new OrderItem(sku, qty, price));
        return this;
    }

    public OrderTestBuilder withStatus(OrderStatus status) {
        this.status = status;
        return this;
    }

    public OrderTestBuilder createdAt(LocalDateTime time) {
        this.createdAt = time;
        return this;
    }

    public Order build() {
        if (items.isEmpty()) {
            items.add(new OrderItem("DEFAULT-SKU", 1, BigDecimal.TEN));
        }
        Order order = new Order(customerId, items, status, shippingAddress);
        order.setCreatedAt(createdAt);
        return order;
    }

    public Order buildAndSave() {
        return orderRepository.save(build());
    }
}

// Usage in tests — readable and intention-revealing
@Test
void shouldApplyDiscountToHighValueOrders() {
    Order order = OrderTestBuilder.anOrder(orderRepository)
        .forCustomer(VIP_CUSTOMER_ID)
        .withItem("LAPTOP", 2, new BigDecimal("1500.00"))
        .withItem("WARRANTY", 2, new BigDecimal("200.00"))
        .withStatus(OrderStatus.CONFIRMED)
        .buildAndSave();

    BigDecimal price = pricingService.calculateFinalPrice(order);

    assertThat(price).isLessThan(order.getSubtotal());
}
```

---

## Scenario 7: Abstract Factory for Multi-Tenant Infrastructure

**Setup:** Your SaaS platform serves Enterprise and Starter tier customers. Enterprise tier gets PostgreSQL with read replicas; Starter tier gets shared PostgreSQL. All infrastructure components must be consistent per tier.

```java
// Abstract Factory — creates consistent families of infrastructure
public interface TenantInfrastructureFactory {
    DataSource getPrimaryDataSource();
    DataSource getReadDataSource();
    CacheManager getCacheManager();
    MessagePublisher getMessagePublisher();
}

@Component
public class EnterpriseInfrastructureFactory implements TenantInfrastructureFactory {
    private final TenantConfig config;

    @Override
    public DataSource getPrimaryDataSource() {
        return buildHikariPool(config.getPrimaryDbUrl(), 20);
    }

    @Override
    public DataSource getReadDataSource() {
        return buildHikariPool(config.getReadReplicaUrl(), 40);  // larger pool for reads
    }

    @Override
    public CacheManager getCacheManager() {
        return new RedisCacheManager(config.getRedisUrl());  // dedicated Redis
    }

    @Override
    public MessagePublisher getMessagePublisher() {
        return new KafkaMessagePublisher(config.getKafkaBrokers());
    }
}

@Component
public class StarterInfrastructureFactory implements TenantInfrastructureFactory {
    @Override
    public DataSource getPrimaryDataSource() {
        return sharedDataSource;  // shared pool
    }

    @Override
    public DataSource getReadDataSource() {
        return primaryDataSource;  // same as primary — no read replica
    }

    @Override
    public CacheManager getCacheManager() {
        return new CaffeineLocalCacheManager();  // local cache, no Redis
    }

    @Override
    public MessagePublisher getMessagePublisher() {
        return new SqsMessagePublisher(awsClient);
    }
}

// Factory selector
@Service
public class TenantInfrastructureRegistry {
    private final Map<TenantTier, TenantInfrastructureFactory> factories;

    public TenantInfrastructureFactory forTenant(Tenant tenant) {
        return factories.get(tenant.getTier());
    }
}
```

**Key property:** You never get a Starter tenant using enterprise Redis or a PostgreSQL primary with no matching read replica — the factory guarantees a consistent, compatible set.

---

## Scenario 8: Composite for Hierarchical Authorization

**Setup:** Your permission system needs to check permissions at multiple levels: individual permissions, roles (groups of permissions), and permission groups with AND/OR logic.

```java
// Component
public interface PermissionCheck {
    boolean isGranted(UserContext user);
    String describe();
}

// Leaf
public class SimplePermission implements PermissionCheck {
    private final String permission;

    @Override
    public boolean isGranted(UserContext user) {
        return user.getPermissions().contains(permission);
    }

    @Override
    public String describe() { return permission; }
}

// Composite — ALL must pass (AND logic)
public class AllOfPermission implements PermissionCheck {
    private final List<PermissionCheck> checks;

    @Override
    public boolean isGranted(UserContext user) {
        return checks.stream().allMatch(c -> c.isGranted(user));
    }

    @Override
    public String describe() {
        return checks.stream().map(PermissionCheck::describe)
            .collect(Collectors.joining(" AND ", "(", ")"));
    }
}

// Composite — ANY must pass (OR logic)
public class AnyOfPermission implements PermissionCheck {
    private final List<PermissionCheck> checks;

    @Override
    public boolean isGranted(UserContext user) {
        return checks.stream().anyMatch(c -> c.isGranted(user));
    }
}

// Building complex permission trees
public class Permissions {
    public static PermissionCheck allOf(PermissionCheck... checks) {
        return new AllOfPermission(List.of(checks));
    }
    public static PermissionCheck anyOf(PermissionCheck... checks) {
        return new AnyOfPermission(List.of(checks));
    }
    public static PermissionCheck permission(String perm) {
        return new SimplePermission(perm);
    }
}

// Usage — readable and composable
PermissionCheck canPublish = Permissions.allOf(
    Permissions.anyOf(
        Permissions.permission("EDITOR"),
        Permissions.permission("ADMIN")
    ),
    Permissions.permission("CONTENT_MANAGEMENT_ENABLED")
);

// In Spring Security
@PreAuthorize("@permissionEvaluator.check(authentication, 'EDIT_ORDER')")
public void editOrder(Long orderId, EditOrderRequest request) { ... }
```

---

## Scenario 9: Proxy for Lazy-Loading Large Objects

**Setup:** A `Customer` entity has a `purchaseHistory` field that contains potentially thousands of records. Load it only when accessed (virtual proxy).

```java
// Without lazy loading — always loads all purchase history
public class Customer {
    private final Long id;
    private final String name;
    private final List<Purchase> purchaseHistory;  // loaded eagerly — expensive!
}

// Virtual Proxy approach
public interface CustomerProfile {
    Long getId();
    String getName();
    List<Purchase> getPurchaseHistory();  // potentially expensive
}

public class LazyCustomerProfile implements CustomerProfile {
    private final Long id;
    private final String name;
    private final PurchaseRepository purchaseRepository;
    private List<Purchase> purchaseHistory;  // null until accessed

    @Override
    public List<Purchase> getPurchaseHistory() {
        if (purchaseHistory == null) {
            purchaseHistory = purchaseRepository.findByCustomerId(id);
        }
        return Collections.unmodifiableList(purchaseHistory);
    }
}

// JPA equivalent — @OneToMany(fetch = FetchType.LAZY)
@Entity
public class Customer {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    private List<Purchase> purchaseHistory;  // proxy until first access
}

// Pitfall: LazyInitializationException outside transaction
// Solution: load within transaction or use JOIN FETCH
@Transactional(readOnly = true)
public CustomerDTO getCustomerWithHistory(Long id) {
    Customer customer = customerRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Customer", id));
    // purchaseHistory.size() forces load while still in transaction
    customer.getPurchaseHistory().size();
    return mapper.toDto(customer);
}
```

---

## Scenario 10: Specification Pattern for Dynamic Search API

**Setup:** Build a generic search endpoint that supports arbitrary combinations of filters without adding query methods for every possible combination.

```java
// Specification factory
@Component
public class ProductSpecifications {

    public Specification<Product> byCategory(String category) {
        return (root, query, cb) ->
            category == null ? null : cb.equal(root.get("category"), category);
    }

    public Specification<Product> priceRange(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            if (min != null) predicates.add(cb.greaterThanOrEqualTo(root.get("price"), min));
            if (max != null) predicates.add(cb.lessThanOrEqualTo(root.get("price"), max));
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }

    public Specification<Product> inStock() {
        return (root, query, cb) -> cb.greaterThan(root.get("stockQuantity"), 0);
    }

    public Specification<Product> nameContains(String keyword) {
        return (root, query, cb) ->
            keyword == null ? null :
                cb.like(cb.lower(root.get("name")), "%" + keyword.toLowerCase() + "%");
    }
}

// Controller builds spec from request parameters
@GetMapping("/products/search")
public Page<ProductResponse> search(
        @RequestParam(required = false) String category,
        @RequestParam(required = false) BigDecimal minPrice,
        @RequestParam(required = false) BigDecimal maxPrice,
        @RequestParam(required = false) String keyword,
        @RequestParam(defaultValue = "false") boolean inStockOnly,
        Pageable pageable) {

    Specification<Product> spec = Specification
        .where(specs.byCategory(category))
        .and(specs.priceRange(minPrice, maxPrice))
        .and(specs.nameContains(keyword))
        .and(inStockOnly ? specs.inStock() : null);

    return productRepository.findAll(spec, pageable)
        .map(mapper::toResponse);
}
```

**No-argument null specs:** `Specification.where(null)` is safe — returns all. `spec.and(null)` ignores the null. So optional filters simply pass `null` when not requested.

---

## Scenario 11: Flyweight for Internationalization Message Loading

**Setup:** A high-throughput API serves 50 locales. Message bundles are expensive to load. Implement efficient sharing using the Flyweight pattern.

```java
@Component
public class MessageBundleFactory {

    // Flyweight cache — locale is the key (shared intrinsic state)
    private final Map<Locale, MessageBundle> cache = new ConcurrentHashMap<>();

    public MessageBundle forLocale(Locale locale) {
        return cache.computeIfAbsent(locale, this::loadBundle);
    }

    private MessageBundle loadBundle(Locale locale) {
        log.info("Loading message bundle for locale {}", locale);
        // Expensive: reads from classpath, parses properties files
        ResourceBundle rb = ResourceBundle.getBundle("messages", locale,
            ResourceBundle.Control.getNoFallbackControl(ResourceBundle.Control.FORMAT_PROPERTIES));
        return new MessageBundle(rb);
    }
}

// Usage — one bundle instance shared across all requests for same locale
public String getMessage(String key, Locale locale, Object... args) {
    MessageBundle bundle = bundleFactory.forLocale(locale);
    return MessageFormat.format(bundle.getString(key), args);
}
```

**Intrinsic vs extrinsic state:**
- Intrinsic (shared): message templates, locale metadata, format patterns — stored in bundle
- Extrinsic (per-call): the actual substitution arguments (`args`), user name, count — passed per call

---

## Scenario 12: Chain of Responsibility for Request Validation Pipeline

**Setup:** API requests go through multiple validation stages: syntax validation → business rule validation → fraud detection → rate limit check. Each stage can reject or pass.

```java
public abstract class ValidationHandler {
    private ValidationHandler next;

    public ValidationHandler setNext(ValidationHandler next) {
        this.next = next;
        return next;
    }

    public final ValidationResult validate(PaymentRequest request) {
        ValidationResult result = doValidate(request);
        if (!result.isValid()) {
            return result;  // short-circuit
        }
        if (next != null) {
            return next.validate(request);
        }
        return ValidationResult.success();
    }

    protected abstract ValidationResult doValidate(PaymentRequest request);
}

@Component
@Order(1)
public class SyntaxValidator extends ValidationHandler {
    @Override
    protected ValidationResult doValidate(PaymentRequest request) {
        if (request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            return ValidationResult.failure("INVALID_AMOUNT", "Amount must be positive");
        }
        if (request.getCurrency() == null) {
            return ValidationResult.failure("MISSING_CURRENCY", "Currency is required");
        }
        return ValidationResult.success();
    }
}

@Component
@Order(2)
public class BusinessRuleValidator extends ValidationHandler {
    @Override
    protected ValidationResult doValidate(PaymentRequest request) {
        if (request.getAmount().compareTo(new BigDecimal("50000")) > 0) {
            return ValidationResult.failure("EXCEEDS_LIMIT",
                "Single transaction cannot exceed $50,000");
        }
        return ValidationResult.success();
    }
}

@Component
@Order(3)
public class FraudDetectionValidator extends ValidationHandler {
    private final FraudScoringService fraudService;

    @Override
    protected ValidationResult doValidate(PaymentRequest request) {
        int score = fraudService.score(request);
        if (score > 80) {
            return ValidationResult.failure("FRAUD_DETECTED",
                "Transaction flagged by fraud detection");
        }
        return ValidationResult.success();
    }
}

// Auto-wire chain via @Order
@Configuration
public class ValidationChainConfig {

    @Bean
    public ValidationHandler validationChain(List<ValidationHandler> handlers) {
        // handlers sorted by @Order
        for (int i = 0; i < handlers.size() - 1; i++) {
            handlers.get(i).setNext(handlers.get(i + 1));
        }
        return handlers.get(0);
    }
}
```

---

## Scenario 13: State Machine with State Pattern

**Setup:** An `Order` has states: `PENDING → CONFIRMED → PROCESSING → SHIPPED → DELIVERED`. Not all transitions are valid. Implement using the State pattern.

```java
public interface OrderState {
    Order confirm(Order order);
    Order startProcessing(Order order);
    Order ship(Order order, String trackingNumber);
    Order deliver(Order order);
    Order cancel(Order order);
    OrderStatus getStatus();
}

// Concrete states — each knows which transitions are valid
public class PendingState implements OrderState {
    @Override
    public Order confirm(Order order) {
        order.setState(new ConfirmedState());
        order.addEvent(new OrderConfirmedEvent(order.getId()));
        return order;
    }

    @Override
    public Order cancel(Order order) {
        order.setState(new CancelledState());
        return order;
    }

    @Override
    public Order ship(Order order, String trackingNumber) {
        throw new InvalidStateTransitionException("Cannot ship a PENDING order");
    }

    @Override
    public OrderStatus getStatus() { return OrderStatus.PENDING; }
    // ... other methods throw InvalidStateTransitionException
}

public class ShippedState implements OrderState {
    @Override
    public Order deliver(Order order) {
        order.setState(new DeliveredState());
        return order;
    }

    @Override
    public Order cancel(Order order) {
        throw new InvalidStateTransitionException("Cannot cancel a SHIPPED order — initiate return");
    }

    @Override
    public OrderStatus getStatus() { return OrderStatus.SHIPPED; }
    // ... other transition methods throw
}

// Context
public class Order {
    @Transient  // Don't persist state object, derive from status field
    private OrderState state;

    @Column
    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @PostLoad
    private void restoreState() {
        this.state = switch (status) {
            case PENDING -> new PendingState();
            case CONFIRMED -> new ConfirmedState();
            case PROCESSING -> new ProcessingState();
            case SHIPPED -> new ShippedState();
            case DELIVERED -> new DeliveredState();
            case CANCELLED -> new CancelledState();
        };
    }

    public void confirm() { this.state.confirm(this); }
    public void ship(String tracking) { this.state.ship(this, tracking); }

    void setState(OrderState newState) {
        this.state = newState;
        this.status = newState.getStatus();
    }
}
```

---

## Scenario 14: Mediator Pattern for Microservice Communication

**Setup:** Five microservices need to exchange events. Direct service-to-service coupling creates a spaghetti architecture. Implement a mediator (event bus) to decouple them.

```java
// Each service only knows the EventBus (mediator), not other services

// Event bus interface — the mediator
public interface EventBus {
    void publish(DomainEvent event);
    <T extends DomainEvent> void subscribe(Class<T> eventType, EventHandler<T> handler);
}

// In-memory implementation for monolith
@Component
public class InMemoryEventBus implements EventBus {
    private final Map<Class<?>, List<EventHandler>> handlers = new ConcurrentHashMap<>();
    private final Executor executor = Executors.newVirtualThreadPerTaskExecutor();

    @Override
    public void publish(DomainEvent event) {
        List<EventHandler> eventHandlers = handlers.getOrDefault(event.getClass(), List.of());
        eventHandlers.forEach(handler ->
            executor.execute(() -> {
                try {
                    handler.handle(event);
                } catch (Exception e) {
                    log.error("Handler {} failed for event {}", handler, event, e);
                    deadLetterQueue.add(new FailedEvent(event, e));
                }
            })
        );
    }
}

// Services communicate via event bus — zero direct coupling
@Service
public class InventoryService {
    private final EventBus eventBus;

    @PostConstruct
    public void registerHandlers() {
        eventBus.subscribe(OrderPlacedEvent.class, this::onOrderPlaced);
    }

    private void onOrderPlaced(OrderPlacedEvent event) {
        reserveItems(event.getOrderId(), event.getItems());
        eventBus.publish(new InventoryReservedEvent(event.getOrderId()));
    }
}

@Service
public class ShippingService {
    @PostConstruct
    public void registerHandlers() {
        eventBus.subscribe(InventoryReservedEvent.class, this::onInventoryReserved);
    }

    private void onInventoryReserved(InventoryReservedEvent event) {
        createShipment(event.getOrderId());
    }
}
// OrderService publishes → InventoryService reacts → ShippingService reacts
// None of them reference each other directly
```

---

## Follow-Up Questions

1. **Decorator vs Inheritance trade-off:** When would you prefer inheritance?
   When the subclass *is-a* specialization with truly different behavior, not just additional behavior. Also when the class hierarchy is stable and closed. Decorator when you need runtime composability or multiple combinations.

2. **Strategy vs Factory:** What's the difference between the two?
   Factory is about *creation* — it decides which object to instantiate. Strategy is about *behavior* — it decides which algorithm to run. They often work together: a Factory creates the right Strategy.

3. **Observer and memory leaks:** How can the Observer pattern cause memory leaks?
   If observers register with a subject but never unregister, the subject holds references preventing GC. Solutions: `WeakReference` observers, explicit unsubscribe in lifecycle hooks (`@PreDestroy`), event bus with auto-cleanup.

4. **Command vs Event:** What is the semantic difference?
   A Command is an intent to change state ("PlaceOrder") — typically has one handler, may be rejected. An Event is a fact that something happened ("OrderPlaced") — immutable, may have zero or many handlers, never rejected.

5. **Template Method vs Strategy:** Both parameterize behavior — when do you choose each?
   Template Method uses inheritance — fix algorithm structure in base class, override steps in subclass. Strategy uses composition — inject behavior via interface. Prefer Strategy (composition over inheritance), use Template Method when the skeleton itself is the key invariant and you're OK with class hierarchy.
