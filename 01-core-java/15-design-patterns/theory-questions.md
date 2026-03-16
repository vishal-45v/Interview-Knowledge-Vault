# Design Patterns — Theory Questions

## How to Use This File
These questions target Senior/Staff Java Engineers. Focus on *why* a pattern exists, *when* to use vs avoid it, and *real Spring/Java ecosystem examples*.

---

## Question 1: What problem does the Singleton pattern solve, and what are its dangers in Spring?

**Pattern:** Creational — ensures only one instance of a class exists in the application.

**Spring's take:** Spring beans are singletons by default (`@Scope("singleton")`). The IoC container manages the single instance — you don't implement `getInstance()`.

**Dangers:**
1. **Shared mutable state** — if a singleton holds mutable fields, concurrent access causes races.
2. **Hidden dependencies** — `getInstance()` is a static call, breaking testability (can't inject a mock).
3. **Classloader issues** — in OSGi or application servers, "singleton" per classloader ≠ JVM singleton.

```java
// BAD: classic singleton — untestable, static coupling
public class ConfigManager {
    private static ConfigManager INSTANCE;
    private Map<String, String> config = new HashMap<>(); // MUTABLE — race condition!

    private ConfigManager() {}

    public static ConfigManager getInstance() {
        if (INSTANCE == null) {  // NOT thread-safe
            INSTANCE = new ConfigManager();
        }
        return INSTANCE;
    }
}

// GOOD: Spring-managed singleton with immutable state
@Service
public class ConfigService {
    private final Map<String, String> config;  // immutable after construction

    public ConfigService(@Value("${app.config-file}") String path) {
        this.config = loadConfig(path);  // final, loaded once
    }
}
```

**Thread-safe singleton idiom (if you must):**
```java
public class Registry {
    private Registry() {}

    private static class Holder {  // loaded only when first accessed
        static final Registry INSTANCE = new Registry();
    }

    public static Registry getInstance() {
        return Holder.INSTANCE;  // thread-safe without synchronization
    }
}
```

---

## Question 2: Explain the Factory Method vs Abstract Factory patterns. Where does Spring use them?

**Factory Method:**
- Defines an interface for creating an object, but lets subclasses decide which class to instantiate.
- Decouples client code from concrete implementations.

**Abstract Factory:**
- Creates *families* of related objects without specifying concrete classes.
- Used when you need consistent families (e.g., create a UI toolkit that's either Windows-style or Mac-style, never mixed).

```java
// Factory Method
public interface NotificationFactory {
    Notification createNotification(String message);
}

@Component
public class EmailNotificationFactory implements NotificationFactory {
    @Override
    public Notification createNotification(String message) {
        return new EmailNotification(message, smtpClient);
    }
}

// Abstract Factory — creates consistent families
public interface UIFactory {
    Button createButton();
    TextField createTextField();
    Dialog createDialog();
}

@Component
@ConditionalOnProperty(name = "ui.theme", havingValue = "material")
public class MaterialUIFactory implements UIFactory { ... }

@Component
@ConditionalOnProperty(name = "ui.theme", havingValue = "fluent")
public class FluentUIFactory implements UIFactory { ... }
```

**Spring examples:**
- `BeanFactory` / `ApplicationContext` — creates beans (factory for all managed objects)
- `SessionFactory` in Hibernate — creates `Session` instances
- `DataSourceFactory` — creates `DataSource` instances
- `@Bean` methods are factory methods

---

## Question 3: When would you use the Builder pattern vs a constructor or static factory?

**Use Builder when:**
- More than 3-4 parameters, especially optional ones
- Several parameters of the same type (easy to mix up `firstName`, `lastName`)
- Need immutable objects with many fields
- Want readable, self-documenting construction code

```java
// Telescoping constructor — BAD, hard to read
Order order = new Order(customerId, item, quantity, address, discount, priority, notes);

// Builder — readable, immutable, no invalid intermediate state
Order order = Order.builder()
    .customerId("C123")
    .item("Widget", 5)
    .shippingAddress(address)
    .priority(Priority.STANDARD)
    .build();  // validate in build() — fail fast

public class Order {
    // All fields final — immutable
    private final String customerId;
    private final String itemSku;
    private final int quantity;
    ...

    private Order(Builder b) {
        this.customerId = Objects.requireNonNull(b.customerId, "customerId required");
        this.itemSku = Objects.requireNonNull(b.itemSku, "itemSku required");
        this.quantity = b.quantity > 0 ? b.quantity
            : throw new IllegalArgumentException("quantity must be positive");
        ...
    }

    public static Builder builder() { return new Builder(); }

    public static class Builder {
        private String customerId;
        private String itemSku;
        private int quantity;
        ...
        public Builder customerId(String v) { this.customerId = v; return this; }
        public Order build() { return new Order(this); }
    }
}
```

**Lombok shortcut:** `@Builder` generates this boilerplate. Use `@Builder.Default` for default values, `@Singular` for collections.

**Java Records (Java 16+):** For simple immutable data holders with few fields, records are simpler:
```java
record Point(double x, double y) {}
```

---

## Question 4: Explain the Strategy pattern. How does it relate to the Open/Closed Principle?

**Strategy:** Define a family of algorithms, encapsulate each one, and make them interchangeable. The client delegates to the strategy without knowing the implementation.

**OCP Connection:** You can add new strategies without modifying existing code — open for extension, closed for modification.

```java
// Strategy interface
public interface PricingStrategy {
    BigDecimal calculatePrice(Order order);
}

// Concrete strategies
@Component("standardPricing")
public class StandardPricingStrategy implements PricingStrategy {
    public BigDecimal calculatePrice(Order order) {
        return order.getBasePrice();
    }
}

@Component("vipPricing")
public class VipPricingStrategy implements PricingStrategy {
    public BigDecimal calculatePrice(Order order) {
        return order.getBasePrice().multiply(BigDecimal.valueOf(0.80)); // 20% discount
    }
}

// Context — delegates to strategy
@Service
public class OrderService {
    private final Map<String, PricingStrategy> strategies;

    public OrderService(Map<String, PricingStrategy> strategies) {
        this.strategies = strategies;  // Spring injects all PricingStrategy beans
    }

    public BigDecimal getPrice(Order order, CustomerTier tier) {
        String strategyName = tier == CustomerTier.VIP ? "vipPricing" : "standardPricing";
        return strategies.get(strategyName).calculatePrice(order);
    }
}
```

**Real examples in Java:**
- `Comparator<T>` — strategy for comparison
- `ExecutorService` implementations — strategy for task execution
- Spring's `TransactionManager` — strategy for transaction management

---

## Question 5: What is the Observer pattern? How does Spring's event system implement it?

**Observer (Publish-Subscribe):** One-to-many dependency. When the subject changes state, all observers are notified automatically.

```java
// Spring ApplicationEvent — built-in Observer
public class OrderPlacedEvent extends ApplicationEvent {
    private final Order order;

    public OrderPlacedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
}

// Publisher
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public Order placeOrder(CreateOrderRequest request) {
        Order order = createOrder(request);
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderPlacedEvent(this, order));
        return order;
    }
}

// Observers — decoupled from publisher
@Component
public class NotificationObserver {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        notificationService.sendOrderConfirmation(event.getOrder());
    }
}

@Component
public class InventoryObserver {
    @EventListener
    @Async  // non-blocking observer
    public void onOrderPlaced(OrderPlacedEvent event) {
        inventoryService.reserveItems(event.getOrder());
    }
}

@Component
public class AuditObserver {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Only fires if the transaction committing the order succeeds
        auditService.log("ORDER_PLACED", event.getOrder().getId());
    }
}
```

**`@TransactionalEventListener`** is crucial — `@EventListener` fires during the transaction (before commit), so if the transaction rolls back, you've already sent the email.

---

## Question 6: Explain the Decorator pattern. How does Java I/O use it?

**Decorator:** Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

```
Component interface
    ↑
ConcreteComponent   Decorator (wraps a Component)
                        ↑
                    ConcreteDecoratorA  ConcreteDecoratorB
```

**Java I/O is the canonical example:**
```java
// Layered decorators
InputStream raw = new FileInputStream("data.gz");
InputStream decompressed = new GZIPInputStream(raw);     // Decorator 1: decompression
InputStream buffered = new BufferedInputStream(decompressed); // Decorator 2: buffering
Reader reader = new InputStreamReader(buffered, UTF_8);  // Decorator 3: charset

// Chain can be built dynamically based on content type
InputStream in = new FileInputStream(file);
if (filename.endsWith(".gz")) {
    in = new GZIPInputStream(in);
}
if (needsDecryption) {
    in = new CipherInputStream(in, cipher);
}
in = new BufferedInputStream(in);
```

**Application example:**
```java
// Decorator for adding metrics to a repository
public class MeteredOrderRepository implements OrderRepository {
    private final OrderRepository delegate;
    private final MeterRegistry registry;

    @Override
    public Order save(Order order) {
        Timer.Sample sample = Timer.start(registry);
        try {
            Order saved = delegate.save(order);
            sample.stop(registry.timer("order.save", "status", "success"));
            return saved;
        } catch (Exception e) {
            sample.stop(registry.timer("order.save", "status", "error"));
            throw e;
        }
    }
}
```

**vs Inheritance:** Decorator composes at runtime vs compile time. You can stack multiple decorators in any order. With inheritance, you'd need `GZIPBufferedFileInputStream` for every combination.

---

## Question 7: What is the Proxy pattern? How does Spring AOP use it?

**Proxy:** Provide a surrogate or placeholder for another object to control access to it.

**Types:**
- **Virtual proxy:** Lazy initialization (load expensive resource only when needed)
- **Protection proxy:** Access control
- **Remote proxy:** Local representative for remote object (RMI, gRPC stub)
- **Logging/monitoring proxy:** Add cross-cutting concerns

```java
// JDK Dynamic Proxy (interface required)
public interface OrderService { void placeOrder(Order o); }

OrderService proxy = (OrderService) Proxy.newProxyInstance(
    OrderService.class.getClassLoader(),
    new Class[]{OrderService.class},
    (proxyObj, method, args) -> {
        log.info("Before: {}", method.getName());
        Object result = method.invoke(realService, args);
        log.info("After: {}", method.getName());
        return result;
    }
);

// Spring AOP — automatic proxying
@Aspect
@Component
public class LoggingAspect {

    @Around("@annotation(Loggable)")
    public Object log(ProceedingJoinPoint pjp) throws Throwable {
        log.info("Calling {}", pjp.getSignature().getName());
        long start = System.nanoTime();
        try {
            Object result = pjp.proceed();
            log.info("Completed in {}ms", (System.nanoTime() - start) / 1_000_000);
            return result;
        } catch (Exception e) {
            log.error("Failed: {}", e.getMessage());
            throw e;
        }
    }
}
```

**Spring proxy mechanics:**
- If bean implements an interface → JDK dynamic proxy
- If bean is a class (no interface) → CGLIB proxy (subclass)
- `proxyTargetClass=true` forces CGLIB even for interfaces
- **Self-invocation bypass:** `orderService.methodA()` calling `this.methodB()` bypasses the proxy — `@Transactional` on `methodB` won't work!

---

## Question 8: Explain the Template Method pattern. Where does Spring use it?

**Template Method:** Define the skeleton of an algorithm in a base class, deferring some steps to subclasses. Subclasses override specific steps without changing the algorithm's structure.

```java
// Abstract template
public abstract class DataImporter {

    // Template method — defines the algorithm skeleton
    public final void importData(InputStream source) {
        List<String[]> rawData = readData(source);    // step 1 — fixed
        validate(rawData);                             // step 2 — fixed
        List<Record> records = transform(rawData);    // step 3 — abstract
        persist(records);                              // step 4 — fixed
        postProcess(records);                          // step 5 — hook (optional)
    }

    protected abstract List<Record> transform(List<String[]> rawData);

    // Hook — subclasses may override or not
    protected void postProcess(List<Record> records) {}

    private List<String[]> readData(InputStream in) { ... }
    private void validate(List<String[]> data) { ... }
    private void persist(List<Record> records) { ... }
}

// Concrete implementations
public class CsvImporter extends DataImporter {
    @Override
    protected List<Record> transform(List<String[]> rawData) {
        return rawData.stream().map(CsvImporter::parseCsvRow).toList();
    }
}
```

**Spring uses Template Method extensively:**
- `JdbcTemplate.execute()` — handles connection lifecycle, delegates SQL execution to callback
- `AbstractPlatformTransactionManager` — defines transaction lifecycle, delegates to `doBegin()`, `doCommit()`, `doRollback()`
- `WebMvcConfigurer` — hook methods for configuring Spring MVC
- `AbstractHealthIndicator` — base for custom health checks

**Modern alternative:** Functional composition with lambdas replaces some Template Method uses:
```java
// Instead of abstract class hierarchy, pass a Function
public void importData(InputStream source, Function<List<String[]>, List<Record>> transformer) {
    List<String[]> raw = readData(source);
    List<Record> records = transformer.apply(raw);
    persist(records);
}
```

---

## Question 9: What is the Command pattern? How does it enable undo/redo?

**Command:** Encapsulate a request as an object, allowing parameterization, queuing, logging, and undo operations.

```java
public interface Command {
    void execute();
    void undo();
}

public class TransferMoneyCommand implements Command {
    private final Account source;
    private final Account target;
    private final BigDecimal amount;
    private boolean executed = false;

    @Override
    public void execute() {
        source.debit(amount);
        target.credit(amount);
        executed = true;
    }

    @Override
    public void undo() {
        if (!executed) throw new IllegalStateException("Not executed");
        target.debit(amount);
        source.credit(amount);
        executed = false;
    }
}

// Command processor with undo stack
public class CommandProcessor {
    private final Deque<Command> history = new ArrayDeque<>();

    public void execute(Command command) {
        command.execute();
        history.push(command);
    }

    public void undo() {
        if (history.isEmpty()) return;
        history.pop().undo();
    }
}
```

**Real-world uses:**
- Database transaction logs (WAL) — commands that can be replayed or rolled back
- Message queues — command messages (decouple sender from receiver)
- `Runnable` / `Callable` in Java — encapsulate computation as an object
- `CompletableFuture.thenApply()` chains — functional commands

---

## Question 10: SOLID Principles — explain each with a Java violation and fix

**S — Single Responsibility Principle (SRP)**
```java
// VIOLATION: UserService does too much
class UserService {
    void createUser(User user) { /* DB insert */ }
    void sendWelcomeEmail(User user) { /* SMTP */ }
    void logUserCreation(User user) { /* file write */ }
}

// FIX: separate classes
class UserService { void createUser(User user) { ... } }
class EmailService { void sendWelcomeEmail(User user) { ... } }
class AuditService { void log(String event, Object data) { ... } }
```

**O — Open/Closed Principle (OCP)**
```java
// VIOLATION: adding a new shape requires modifying existing class
class AreaCalculator {
    double area(Object shape) {
        if (shape instanceof Circle c) return Math.PI * c.radius * c.radius;
        if (shape instanceof Rectangle r) return r.width * r.height;
        // Adding Triangle requires modifying this method!
    }
}

// FIX: extend via new classes, not modification
interface Shape { double area(); }
record Circle(double radius) implements Shape { public double area() { return Math.PI * radius * radius; } }
record Rectangle(double w, double h) implements Shape { public double area() { return w * h; } }
// Adding Triangle? Add a new class — no existing code changes.
```

**L — Liskov Substitution Principle (LSP)**
```java
// VIOLATION: Square extends Rectangle but breaks postconditions
class Rectangle { int width, height; void setWidth(int w) { this.width = w; } }
class Square extends Rectangle {
    @Override void setWidth(int w) { this.width = this.height = w; } // breaks invariant!
}
// Code expecting Rectangle breaks when given Square

// FIX: use composition or separate hierarchy
interface Shape { int area(); }
record Rectangle(int width, int height) implements Shape { public int area() { return width * height; } }
record Square(int side) implements Shape { public int area() { return side * side; } }
```

**I — Interface Segregation Principle (ISP)**
```java
// VIOLATION: fat interface forces empty implementations
interface Worker { void work(); void eat(); void sleep(); }
class Robot implements Worker {
    public void work() { ... }
    public void eat() { throw new UnsupportedOperationException(); } // forced!
    public void sleep() { throw new UnsupportedOperationException(); }
}

// FIX: segregate interfaces
interface Workable { void work(); }
interface HumanNeedable { void eat(); void sleep(); }
class Robot implements Workable { public void work() { ... } }
class Human implements Workable, HumanNeedable { ... }
```

**D — Dependency Inversion Principle (DIP)**
```java
// VIOLATION: high-level module depends on low-level detail
class OrderService {
    private MySqlOrderRepository repository = new MySqlOrderRepository(); // direct instantiation!
}

// FIX: depend on abstraction
class OrderService {
    private final OrderRepository repository;  // interface
    public OrderService(OrderRepository repository) { this.repository = repository; }
}
// Spring DI injects the concrete implementation — OrderService doesn't know the DB type
```

---

## Question 11: What is the Adapter pattern? Give a real Spring example.

**Adapter:** Convert the interface of a class into another interface that clients expect. Enables incompatible interfaces to work together.

```java
// Scenario: Our code uses OrderPort, but external library uses LegacyOrderClient
public interface OrderPort {
    OrderResult processOrder(Order order);
}

// Legacy API — different method signature and types
public class LegacyOrderClient {
    public LegacyResponse submitOrder(String orderId, int qty, String sku) { ... }
}

// Adapter
@Component
public class LegacyOrderAdapter implements OrderPort {
    private final LegacyOrderClient legacyClient;

    @Override
    public OrderResult processOrder(Order order) {
        LegacyResponse response = legacyClient.submitOrder(
            order.getId().toString(),
            order.getQuantity(),
            order.getSku()
        );
        return OrderResult.of(response.getConfirmationNumber(), response.isSuccess());
    }
}
```

**Spring examples:**
- `HandlerAdapter` — adapts various handler types (`@Controller`, `HttpRequestHandler`, `Servlet`) to Spring MVC's handler invocation
- `PlatformTransactionManager` adapts different transaction APIs (JPA, JDBC, JMS)
- Spring Data's `Repository` adapts JPA/MongoDB/Redis to a unified interface

---

## Question 12: Explain the Composite pattern. Where is it used in practice?

**Composite:** Compose objects into tree structures to represent part-whole hierarchies. Clients treat individual objects and compositions uniformly.

```java
// Component
public interface MenuComponent {
    String getName();
    double getPrice();
    void print();
    default double getCost() { return getPrice(); }  // default for leaf
}

// Leaf
public class MenuItem implements MenuComponent {
    private final String name;
    private final double price;
    public void print() { System.out.println("  " + name + ": $" + price); }
}

// Composite
public class MenuCategory implements MenuComponent {
    private final String name;
    private final List<MenuComponent> children = new ArrayList<>();

    public void add(MenuComponent c) { children.add(c); }

    @Override
    public double getPrice() {
        return children.stream().mapToDouble(MenuComponent::getCost).sum();
    }

    @Override
    public void print() {
        System.out.println(name + ":");
        children.forEach(MenuComponent::print);
    }
}
```

**Real examples:**
- **Spring Security's `FilterChainProxy`** — composite of filter chains
- **`CompositeHealthIndicator`** — aggregates multiple `HealthIndicator`s
- **File system** — `File` represents both files and directories
- **HTML DOM** — elements contain other elements
- **JUnit `@Suite`** — suite contains tests and nested suites

---

## Question 13: What is the Chain of Responsibility pattern? Servlet Filter chain is an example — explain how.

**Chain of Responsibility:** Pass a request along a chain of handlers. Each handler decides to process it and/or pass it to the next.

```java
// Servlet Filter chain
@Component
@Order(1)
public class AuthenticationFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        // Handle: validate JWT
        if (!isValidToken(req)) {
            ((HttpServletResponse)resp).sendError(401);
            return;  // STOP the chain
        }
        chain.doFilter(req, resp);  // PASS to next handler
    }
}

@Component
@Order(2)
public class RateLimitFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        if (isRateLimited(req)) {
            ((HttpServletResponse)resp).sendError(429);
            return;
        }
        chain.doFilter(req, resp);
    }
}
```

**Spring AOP interceptor chain** is also Chain of Responsibility:
- `@Transactional` interceptor wraps `@Cacheable` interceptor wraps `@Secured` interceptor wraps actual method

**Key characteristic:** Handlers can *short-circuit* the chain (return early without calling `chain.doFilter()`) or pass through.

---

## Question 14: What is the Flyweight pattern? When would you use it in Java?

**Flyweight:** Use sharing to efficiently support a large number of fine-grained objects by storing common state externally and sharing it.

**Java examples:**
- `Integer.valueOf(-128 to 127)` — cached instances (why `Integer.valueOf(100) == Integer.valueOf(100)` is `true`)
- `String.intern()` — string pool
- `Boolean.TRUE` / `Boolean.FALSE` — cached instances

```java
// Application example: formatting rules for a document editor
// Don't create a new Font object for every character!

public class FontFactory {
    private static final Map<String, Font> cache = new ConcurrentHashMap<>();

    public static Font getFont(String family, int size, boolean bold) {
        String key = family + "-" + size + (bold ? "-bold" : "");
        return cache.computeIfAbsent(key, k -> new Font(family, size, bold));
    }
}

// Each Character holds a reference to shared Font (intrinsic state)
// Position in document is extrinsic state — NOT stored in Character
public class Character {
    private final char value;       // intrinsic
    private final Font font;        // intrinsic — shared!
    // position is passed as parameter, not stored
}
```

**When to use:** Millions of similar objects where most state can be shared. Identify intrinsic (shareable) vs extrinsic (context-dependent) state.

---

## Question 15: What patterns does Spring's `@Transactional` use internally?

`@Transactional` is a showcase of multiple patterns working together:

1. **Proxy Pattern** — Spring wraps the bean in a proxy (JDK or CGLIB). The proxy intercepts method calls.

2. **Template Method** — `AbstractPlatformTransactionManager` defines the transaction lifecycle skeleton: `getTransaction()` → `processRollback()` / `processCommit()`. Concrete managers (JDBC, JPA, JMS) implement the abstract steps.

3. **Strategy Pattern** — `PlatformTransactionManager` is a strategy. You swap `DataSourceTransactionManager` for `JpaTransactionManager` without changing `@Transactional` usage.

4. **Chain of Responsibility** — `TransactionInterceptor` is one link in the AOP interceptor chain.

5. **ThreadLocal Pattern** — Transaction synchronization uses `TransactionSynchronizationManager` with `ThreadLocal` to bind the current transaction to the executing thread, allowing JDBC connection reuse within the same transaction.

```java
// What happens when you call a @Transactional method:
// 1. PROXY intercepts the call
// 2. TransactionInterceptor.invoke() runs
// 3. TransactionManager.getTransaction() — checks ThreadLocal for existing tx
// 4. If none: begin new transaction (STRATEGY decides JPA/JDBC/JMS)
// 5. Bind connection to ThreadLocal
// 6. PROCEED to real method
// 7. On success: commit; on RuntimeException: rollback
// 8. Remove from ThreadLocal
```

---

## Question 16: What is the difference between GoF Facade and Mediator patterns?

**Facade:** Provides a simplified interface to a complex subsystem. The subsystem components don't know about the facade.

```java
// Facade — simplifies the checkout subsystem
@Service
public class CheckoutFacade {
    private final InventoryService inventory;
    private final PaymentService payment;
    private final ShippingService shipping;
    private final NotificationService notification;

    public OrderConfirmation checkout(Cart cart, PaymentMethod pm) {
        inventory.reserve(cart.getItems());
        PaymentResult payment = this.payment.charge(cart.getTotal(), pm);
        ShipmentLabel label = shipping.createLabel(cart.getShippingAddress());
        notification.sendConfirmation(cart.getCustomerId());
        return new OrderConfirmation(payment.getTransactionId(), label.getTrackingNumber());
    }
}
```

**Mediator:** Defines an object that encapsulates how a set of objects interact. Promotes loose coupling by preventing objects from referring to each other explicitly. Components *know about* the mediator.

```java
// Mediator — components communicate through it, not directly
public interface EventBus {
    void publish(Event event);
    <T extends Event> void subscribe(Class<T> type, Consumer<T> handler);
}

// Components communicate via mediator (EventBus), not directly
orderService.publish(new OrderPlacedEvent(...));
// inventoryService is subscribed to OrderPlacedEvent
// shippingService is subscribed to OrderPlacedEvent
// They don't know about each other
```

**Difference:**
- Facade: one-directional simplification — client → facade → subsystem
- Mediator: many-to-many coordination — each component ↔ mediator ↔ other components

---

## Question 17: What is the Repository pattern and how does Spring Data implement it?

**Repository:** Mediates between the domain and data mapping layers. Acts like an in-memory collection of domain objects.

```java
// Domain-driven Repository — collection-like interface
public interface OrderRepository {
    Order findById(OrderId id);
    List<Order> findByCustomer(CustomerId customerId);
    void save(Order order);
    void remove(OrderId id);
}

// Spring Data JPA implementation — almost zero boilerplate
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerIdAndStatus(Long customerId, OrderStatus status);

    @Query("SELECT o FROM Order o WHERE o.createdAt BETWEEN :from AND :to")
    List<Order> findInDateRange(@Param("from") LocalDateTime from,
                                @Param("to") LocalDateTime to);

    // Custom implementation for complex queries
    List<Order> findByComplexCriteria(OrderSearchCriteria criteria);
}

// Custom implementation
public class OrderRepositoryImpl implements OrderRepositoryCustom {
    @Override
    public List<Order> findByComplexCriteria(OrderSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Order> cq = cb.createQuery(Order.class);
        Root<Order> root = cq.from(Order.class);
        List<Predicate> predicates = buildPredicates(cb, root, criteria);
        cq.where(predicates.toArray(new Predicate[0]));
        return entityManager.createQuery(cq).getResultList();
    }
}
```

**Pattern benefits:**
- Domain objects don't contain persistence logic
- Easy to mock in unit tests
- Can switch from JPA to MongoDB without changing service layer

---

## Question 18: Explain the Specification pattern for complex query building

**Specification:** Encapsulate a business rule or query condition as a reusable, combinable object.

```java
public interface Specification<T> {
    boolean isSatisfiedBy(T entity);  // in-memory filtering
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb); // JPA
}

// Specifications
public class ActiveOrderSpec implements Specification<Order> {
    public Predicate toPredicate(Root<Order> root, CriteriaQuery<?> q, CriteriaBuilder cb) {
        return cb.equal(root.get("status"), OrderStatus.ACTIVE);
    }
}

public class OrderByCustomerSpec implements Specification<Order> {
    private final Long customerId;
    public Predicate toPredicate(Root<Order> root, CriteriaQuery<?> q, CriteriaBuilder cb) {
        return cb.equal(root.get("customerId"), customerId);
    }
}

// Spring Data JPA's built-in Specification support
public interface OrderRepository extends JpaRepository<Order, Long>,
        JpaSpecificationExecutor<Order> {}

// Dynamic query building
public List<Order> findOrders(Long customerId, boolean activeOnly, LocalDate from) {
    Specification<Order> spec = Specification.where(null);

    if (customerId != null) {
        spec = spec.and(new OrderByCustomerSpec(customerId));
    }
    if (activeOnly) {
        spec = spec.and(new ActiveOrderSpec());
    }
    if (from != null) {
        spec = spec.and((root, q, cb) ->
            cb.greaterThanOrEqualTo(root.get("createdAt"), from.atStartOfDay()));
    }

    return orderRepository.findAll(spec);
}
```

---

## Question 19: What is the Null Object pattern? How does Java's Optional relate?

**Null Object:** Provide a do-nothing default implementation for an interface instead of using `null`. Avoids null checks scattered throughout code.

```java
// Without Null Object — null checks everywhere
User user = userRepository.findById(id);
if (user != null && user.getPermissions() != null) {
    user.getPermissions().canEdit(); // NPE risk
}

// Null Object — safe default behavior
public class GuestUser implements User {
    @Override public String getName() { return "Guest"; }
    @Override public Permissions getPermissions() { return Permissions.READ_ONLY; }
    @Override public boolean isAuthenticated() { return false; }
}

// Usage — no null checks needed
User user = userRepository.findById(id).orElse(new GuestUser());
user.getPermissions().canEdit();  // safe — GuestUser always returns READ_ONLY

// Java Optional — monad-style null handling
Optional<User> user = userRepository.findById(id);
boolean canEdit = user
    .map(User::getPermissions)
    .map(Permissions::canEdit)
    .orElse(false);
```

**Optional anti-patterns:**
- `optional.get()` without `isPresent()` — defeats the purpose
- Using `Optional` for method parameters (use overloading instead)
- `Optional.of(possiblyNull)` — use `Optional.ofNullable()`
- Serializing `Optional` fields in domain objects

---

## Question 20: What is the Event Sourcing pattern? How does it differ from CQRS?

**Event Sourcing:** Store state as an immutable sequence of domain events rather than the current state. The current state is derived by replaying events.

```java
// Events as source of truth
public sealed interface OrderEvent permits
    OrderCreated, OrderItemAdded, OrderConfirmed, OrderShipped, OrderCancelled {}

public record OrderCreated(UUID orderId, UUID customerId, Instant occurredAt) implements OrderEvent {}
public record OrderItemAdded(UUID orderId, String sku, int qty, BigDecimal price) implements OrderEvent {}

// Aggregate rebuilds state from events
public class Order {
    private UUID id;
    private OrderStatus status;
    private List<OrderItem> items = new ArrayList<>();

    public static Order reconstitute(List<OrderEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }

    private void apply(OrderEvent event) {
        switch (event) {
            case OrderCreated e -> { this.id = e.orderId(); this.status = CREATED; }
            case OrderItemAdded e -> items.add(new OrderItem(e.sku(), e.qty(), e.price()));
            case OrderConfirmed e -> this.status = CONFIRMED;
            case OrderShipped e -> this.status = SHIPPED;
            case OrderCancelled e -> this.status = CANCELLED;
        }
    }
}
```

**CQRS (Command Query Responsibility Segregation):** Separate the write model (commands that change state) from the read model (queries that return data). Often paired with Event Sourcing but independent.

```
CQRS:
Command → CommandHandler → Write Model (normalized DB)
                         ↓ events
Query ← ReadModel ← Projections (denormalized, optimized for reads)
```

**Event Sourcing benefits:**
- Complete audit trail — replay to any point in time
- Temporal queries — "what was the state on Jan 1?"
- Event-driven integration — publish events to other services

**Costs:**
- Eventual consistency for read models
- Schema evolution is complex (old events must still replay)
- Storage grows indefinitely (mitigated by snapshots)

---

## Quick Reference Table

| Pattern | Category | Problem Solved | Spring Example |
|---|---|---|---|
| Singleton | Creational | One instance | `@Bean` (default scope) |
| Factory Method | Creational | Decouple creation | `BeanFactory`, `@Bean` methods |
| Builder | Creational | Complex object construction | `UriComponentsBuilder`, `MockMvcRequestBuilders` |
| Adapter | Structural | Interface incompatibility | `HandlerAdapter`, Spring Data |
| Decorator | Structural | Dynamic responsibility | Java I/O, `BeanPostProcessor` |
| Composite | Structural | Tree structures | `FilterChainProxy`, `CompositeHealthIndicator` |
| Facade | Structural | Simplify complex subsystem | Service layer |
| Proxy | Structural | Controlled access | Spring AOP, `@Transactional` |
| Flyweight | Structural | Memory: shared objects | `Integer` cache, String pool |
| Chain of Responsibility | Behavioral | Pass request along chain | Servlet Filter chain, AOP interceptors |
| Command | Behavioral | Encapsulate request | `Runnable`, Spring Batch step |
| Observer | Behavioral | Notify on state change | `ApplicationEvent`, `@EventListener` |
| Strategy | Behavioral | Interchangeable algorithms | `Comparator`, `PricingStrategy` |
| Template Method | Behavioral | Algorithm skeleton | `JdbcTemplate`, `AbstractTransactionManager` |
| Mediator | Behavioral | Decouple many-to-many | `ApplicationEventPublisher` |
| Specification | Behavioral | Combinable query rules | Spring Data `Specification` |
