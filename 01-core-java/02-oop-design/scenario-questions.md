# OOP Design — Scenario Questions

> 25 scenario-based questions about applying OOP design principles in real Java applications.

---

## Scenario 1: Violating Open/Closed Principle

**Context:** A payment processor handles multiple payment types:
```java
class PaymentProcessor {
    void process(Payment payment) {
        if (payment.type == CREDIT_CARD) { processCreditCard(payment); }
        else if (payment.type == PAYPAL) { processPayPal(payment); }
        else if (payment.type == CRYPTO) { processCrypto(payment); }
    }
}
```
**Question:** Redesign to follow OCP.

**Answer:**
```java
interface PaymentHandler {
    boolean canHandle(PaymentType type);
    PaymentResult process(Payment payment);
}

@Component
class CreditCardHandler implements PaymentHandler { ... }
@Component
class PayPalHandler implements PaymentHandler { ... }
@Component
class CryptoHandler implements PaymentHandler { ... }

@Service
class PaymentProcessor {
    private final List<PaymentHandler> handlers;

    PaymentProcessor(List<PaymentHandler> handlers) {
        this.handlers = handlers;  // all handlers injected by Spring
    }

    PaymentResult process(Payment payment) {
        return handlers.stream()
            .filter(h -> h.canHandle(payment.type()))
            .findFirst()
            .map(h -> h.process(payment))
            .orElseThrow(() -> new UnsupportedPaymentTypeException(payment.type()));
    }
    // Adding new payment type = just add a new class. No modification here.
}
```

---

## Scenario 2: LSP Violation in REST Responses

**Context:** A base `ApiResponse<T>` class is extended for specific cases, but some usages break:

```java
class ApiResponse<T> {
    T data;
    String message;
    int status;
    ApiResponse(T data, int status, String message) { ... }
}

class PaginatedResponse<T> extends ApiResponse<List<T>> {
    int totalCount;
    int page;
    // getters/setters
}

// Usage:
ApiResponse<List<User>> response = getUsers();
// Code that calls response.getData() ALWAYS expects a List<User>
// PaginatedResponse satisfies this — LSP maintained
```

This case respects LSP. Show a violation:
```java
// LSP violation:
class ErrorResponse extends ApiResponse<Object> {
    @Override
    Object getData() { throw new UnsupportedOperationException("Error responses have no data"); }
}
// Caller using ApiResponse<Object> and calling getData() crashes with ErrorResponse!
// Fix: Use composition or separate error/success response types
```

---

## Scenario 3: Choosing Interface vs Abstract Class

**Context:** Designing a notification system that supports Email, SMS, and Push notifications.

**Question:** Should you use an interface or abstract class?

**Answer:**
```java
// Use interface for the contract (what):
interface NotificationSender {
    void send(Notification notification);
    boolean supports(NotificationType type);
}

// Use abstract class for shared behavior (how) — template method:
abstract class AbstractNotificationSender implements NotificationSender {
    private final NotificationLogger logger;
    private final MetricsService metrics;

    protected AbstractNotificationSender(NotificationLogger logger, MetricsService metrics) {
        this.logger = logger;
        this.metrics = metrics;
    }

    @Override
    public void send(Notification notification) {
        logger.log("Sending: " + notification);
        try {
            long start = System.currentTimeMillis();
            doSend(notification);  // abstract — subclass implements
            metrics.recordSuccess(getClass().getSimpleName(),
                System.currentTimeMillis() - start);
        } catch (Exception e) {
            metrics.recordFailure(getClass().getSimpleName());
            throw e;
        }
    }

    protected abstract void doSend(Notification notification);
}

class EmailNotificationSender extends AbstractNotificationSender {
    @Override protected void doSend(Notification n) { emailClient.send(n); }
    @Override public boolean supports(NotificationType t) { return t == EMAIL; }
}
```

---

## Scenario 4: Fixing Encapsulation Violation

**Context:**
```java
public class Order {
    public List<OrderItem> items = new ArrayList<>();  // public mutable field!
    public double total = 0.0;
}
// External code: order.items.clear(); order.total = -999.99;
```
**Question:** Redesign with proper encapsulation.

**Answer:**
```java
public class Order {
    private final List<OrderItem> items = new ArrayList<>();
    private double total = 0.0;
    private OrderStatus status = OrderStatus.PENDING;

    public void addItem(OrderItem item) {
        if (status != OrderStatus.PENDING) throw new IllegalStateException("Cannot modify non-pending order");
        items.add(item);
        total += item.getPrice() * item.getQuantity();
    }

    public void removeItem(OrderItem item) {
        if (items.remove(item)) total -= item.getPrice() * item.getQuantity();
    }

    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);  // defensive: return view, not the list
    }

    public double getTotal() { return total; }
    public OrderStatus getStatus() { return status; }

    // Only through this method — enforces business rules
    void confirm() {
        if (items.isEmpty()) throw new IllegalStateException("Cannot confirm empty order");
        this.status = OrderStatus.CONFIRMED;
    }
}
```

---

## Scenario 5: Composition for Flexible Logging

**Context:** Multiple services need audit logging, but not all need all logging behaviors (some need file+DB, some need only DB).

**Question:** Design using composition.

**Answer:**
```java
interface AuditLogger {
    void log(AuditEvent event);
}

class FileAuditLogger implements AuditLogger { ... }
class DatabaseAuditLogger implements AuditLogger { ... }
class KafkaAuditLogger implements AuditLogger { ... }

// Composite logger — composition!
class CompositeAuditLogger implements AuditLogger {
    private final List<AuditLogger> loggers;

    CompositeAuditLogger(AuditLogger... loggers) {
        this.loggers = List.of(loggers);
    }

    @Override
    public void log(AuditEvent event) {
        loggers.forEach(logger -> {
            try { logger.log(event); }
            catch (Exception e) { log.warn("Logger {} failed", logger.getClass().getName(), e); }
        });
    }
}

// Configuration — no modification to services:
@Bean
AuditLogger paymentAuditLogger() {
    return new CompositeAuditLogger(new DatabaseAuditLogger(), new KafkaAuditLogger());
}
@Bean
AuditLogger userAuditLogger() {
    return new CompositeAuditLogger(new FileAuditLogger(), new DatabaseAuditLogger());
}
```

---

## Scenario 6: Interface Segregation in Repository Layer

**Context:** A reporting service only reads data but is coupled to a full CRUD repository.

**Question:** Apply ISP to fix.

**Answer:**
```java
// Segregated interfaces:
interface OrderReader {
    Optional<Order> findById(Long id);
    List<Order> findByStatus(OrderStatus status);
    Page<Order> findAll(Pageable pageable);
}

interface OrderWriter {
    Order save(Order order);
    void deleteById(Long id);
}

interface OrderRepository extends OrderReader, OrderWriter {}

// Reporting service — only depends on what it needs:
@Service
class OrderReportService {
    private final OrderReader orderReader;  // NOT OrderRepository!

    OrderReportService(OrderReader orderReader) {
        this.orderReader = orderReader;
    }
    // Can only read, cannot accidentally call save() or delete()
}
```

---

## Scenario 7: Template Method Pattern Implementation

**Context:** Processing different document types (PDF, Word, CSV) with same overall steps: load → parse → validate → transform → export.

**Answer:**
```java
abstract class DocumentProcessor {
    // Template method — defines the algorithm skeleton:
    public final ProcessingResult process(Path file) {
        Document doc = load(file);
        ParsedContent content = parse(doc);
        validate(content);  // may throw ValidationException
        TransformedContent transformed = transform(content);
        return export(transformed);
    }

    protected abstract Document load(Path file);
    protected abstract ParsedContent parse(Document doc);
    protected abstract TransformedContent transform(ParsedContent content);
    protected abstract ProcessingResult export(TransformedContent content);

    // Hook method — optional override:
    protected void validate(ParsedContent content) {
        // Default: no validation
    }
}

class PdfProcessor extends DocumentProcessor {
    @Override protected Document load(Path file) { return PdfDocument.open(file); }
    @Override protected ParsedContent parse(Document doc) { return PdfParser.parse(doc); }
    @Override protected void validate(ParsedContent c) { validatePdfContent(c); }  // override hook
    @Override protected TransformedContent transform(ParsedContent c) { ... }
    @Override protected ProcessingResult export(TransformedContent c) { ... }
}
```

---

## Scenario 8: Strategy Pattern for Sorting

**Context:** An e-commerce search needs multiple sort strategies (price, relevance, rating, newest).

**Answer:**
```java
interface SortStrategy {
    List<Product> sort(List<Product> products);
}

class PriceAscStrategy implements SortStrategy {
    @Override
    public List<Product> sort(List<Product> products) {
        return products.stream()
            .sorted(Comparator.comparingDouble(Product::getPrice))
            .collect(Collectors.toList());
    }
}

class RelevanceStrategy implements SortStrategy {
    private final SearchScorer scorer;
    @Override
    public List<Product> sort(List<Product> products) {
        return products.stream()
            .sorted(Comparator.comparingDouble(p -> -scorer.score(p)))
            .collect(Collectors.toList());
    }
}

@Service
class ProductSearchService {
    private final Map<SortOrder, SortStrategy> strategies = new EnumMap<>(SortOrder.class);

    ProductSearchService(List<SortStrategy> allStrategies) {
        // Spring injects all SortStrategy beans
        // Map them to their enum values
    }

    List<Product> search(String query, SortOrder sortOrder) {
        List<Product> results = performSearch(query);
        return strategies.getOrDefault(sortOrder, new RelevanceStrategy())
                         .sort(results);
    }
}
```

---

## Scenario 9: Builder Pattern for Complex Object Construction

**Context:** A `Report` object has many optional parameters. Constructors are becoming unmanageable.

**Answer:**
```java
public class Report {
    private final String title;
    private final LocalDate startDate;
    private final LocalDate endDate;
    private final String format;
    private final boolean includeCharts;
    private final List<String> sections;
    private final String author;

    private Report(Builder builder) {
        this.title = builder.title;
        this.startDate = builder.startDate;
        this.endDate = builder.endDate;
        this.format = builder.format;
        this.includeCharts = builder.includeCharts;
        this.sections = List.copyOf(builder.sections);
        this.author = builder.author;
    }

    public static class Builder {
        private final String title;  // required
        private LocalDate startDate = LocalDate.now().minusMonths(1);
        private LocalDate endDate = LocalDate.now();
        private String format = "PDF";
        private boolean includeCharts = true;
        private List<String> sections = new ArrayList<>();
        private String author = "System";

        public Builder(String title) {
            this.title = Objects.requireNonNull(title, "title required");
        }

        public Builder dateRange(LocalDate start, LocalDate end) {
            this.startDate = start; this.endDate = end; return this;
        }
        public Builder format(String format) { this.format = format; return this; }
        public Builder withoutCharts() { this.includeCharts = false; return this; }
        public Builder sections(String... sections) {
            this.sections = Arrays.asList(sections); return this;
        }
        public Builder author(String author) { this.author = author; return this; }

        public Report build() {
            if (startDate.isAfter(endDate)) throw new IllegalArgumentException("Start after end");
            return new Report(this);
        }
    }
}

// Usage:
Report report = new Report.Builder("Q3 Sales Report")
    .dateRange(LocalDate.of(2024, 7, 1), LocalDate.of(2024, 9, 30))
    .format("HTML")
    .author("Analytics Team")
    .sections("Summary", "By Region", "By Product")
    .build();
```

---

## Scenario 10: Observer Pattern for Event System

**Context:** Multiple components need to react to user registration events (send welcome email, create audit log, initialize user preferences).

**Answer:**
```java
// Spring's ApplicationEvent (preferred in Spring apps):
class UserRegisteredEvent extends ApplicationEvent {
    private final User user;
    UserRegisteredEvent(Object source, User user) { super(source); this.user = user; }
    User getUser() { return user; }
}

@Service
class UserRegistrationService {
    private final ApplicationEventPublisher eventPublisher;

    void register(RegistrationRequest request) {
        User user = createUser(request);
        userRepository.save(user);
        eventPublisher.publishEvent(new UserRegisteredEvent(this, user));
        // No knowledge of what happens next — decoupled!
    }
}

// Each listener is independent:
@Component
class WelcomeEmailListener {
    @EventListener
    @Async  // run in separate thread
    void onUserRegistered(UserRegisteredEvent event) {
        emailService.sendWelcome(event.getUser());
    }
}

@Component
class AuditLogListener {
    @EventListener
    void onUserRegistered(UserRegisteredEvent event) {
        auditService.log("USER_REGISTERED", event.getUser().getId());
    }
}
```

---

## Scenario 11: Decorator Pattern

**Context:** A `DataReader` needs optional behaviors: caching, compression, encryption, logging.

**Answer:**
```java
interface DataReader {
    byte[] read(String key);
}

class S3DataReader implements DataReader {
    @Override public byte[] read(String key) { return s3Client.getObject(key); }
}

// Decorator base:
abstract class DataReaderDecorator implements DataReader {
    protected final DataReader delegate;
    DataReaderDecorator(DataReader delegate) { this.delegate = delegate; }
}

class CachingDataReader extends DataReaderDecorator {
    private final Cache cache;
    CachingDataReader(DataReader delegate, Cache cache) {
        super(delegate); this.cache = cache;
    }
    @Override
    public byte[] read(String key) {
        return cache.computeIfAbsent(key, k -> delegate.read(k));
    }
}

class EncryptingDataReader extends DataReaderDecorator {
    private final EncryptionService enc;
    EncryptingDataReader(DataReader delegate, EncryptionService enc) {
        super(delegate); this.enc = enc;
    }
    @Override
    public byte[] read(String key) {
        return enc.decrypt(delegate.read(key));
    }
}

// Usage — compose behaviors:
DataReader reader = new CachingDataReader(
    new EncryptingDataReader(new S3DataReader(), encService),
    cache
);
```

---

## Scenario 12: Object Cloning Pitfalls

**Context:**
```java
class Order implements Cloneable {
    List<OrderItem> items;
    @Override public Object clone() throws CloneNotSupportedException {
        return super.clone();  // shallow copy!
    }
}

Order original = new Order();
original.items.add(new OrderItem("Widget", 10.0));
Order copy = (Order) original.clone();
copy.items.add(new OrderItem("Extra", 5.0));
System.out.println(original.items.size());  // 2! Shared list!
```

**Question:** Fix the clone to be a deep copy.

**Answer:**
```java
@Override
public Order clone() throws CloneNotSupportedException {
    Order clone = (Order) super.clone();
    clone.items = new ArrayList<>(this.items);  // new list with same items
    // If OrderItem is mutable, need deep copy of items too:
    clone.items = this.items.stream()
        .map(item -> item.clone())  // deep copy each item
        .collect(Collectors.toList());
    return clone;
}

// Better alternative — copy constructor (more explicit than Cloneable):
public Order(Order other) {
    this.items = other.items.stream().map(OrderItem::new).collect(Collectors.toList());
    this.status = other.status;
    this.total = other.total;
}
```

---

## Scenario 13: Abstract Class for Framework Extension Points

**Context:** Building a batch processing framework where users can plug in custom logic.

**Answer:**
```java
public abstract class BatchProcessor<T, R> {
    private final int batchSize;
    private final MetricsService metrics;

    protected BatchProcessor(int batchSize, MetricsService metrics) {
        this.batchSize = batchSize;
        this.metrics = metrics;
    }

    public final ProcessingReport process(Iterable<T> items) {
        ProcessingReport report = new ProcessingReport();
        List<T> batch = new ArrayList<>(batchSize);

        for (T item : items) {
            batch.add(item);
            if (batch.size() == batchSize) {
                processBatch(batch, report);
                batch.clear();
            }
        }
        if (!batch.isEmpty()) processBatch(batch, report);

        return report;
    }

    private void processBatch(List<T> batch, ProcessingReport report) {
        try {
            List<R> results = transform(batch);  // user implements
            persist(results);                     // user implements
            report.recordSuccess(batch.size());
        } catch (Exception e) {
            handleError(batch, e, report);        // hook — user may override
        }
    }

    protected abstract List<R> transform(List<T> input);
    protected abstract void persist(List<R> results);

    // Hook method with default — user can override if needed:
    protected void handleError(List<T> batch, Exception e, ProcessingReport report) {
        log.error("Batch failed", e);
        report.recordFailure(batch.size());
    }
}
```

---

## Scenario 14: Choosing the Right Polymorphism

**Context:** A shape area calculator. Should you use instanceof checking, method overriding, or visitor pattern?

**Answer:**
```java
// Option 1 — instanceof (bad): requires modifying when new shapes added
double area = shape instanceof Circle c ? PI * c.r * c.r : 0;

// Option 2 — Method overriding (good for stable hierarchies):
interface Shape { double area(); }
class Circle implements Shape { @Override double area() { return PI * r * r; } }
// Adding Circle.perimeter() later is easy

// Option 3 — Visitor (good when operations change frequently):
interface ShapeVisitor {
    double visitCircle(Circle c);
    double visitRectangle(Rectangle r);
}

interface Shape { double accept(ShapeVisitor visitor); }
class Circle implements Shape {
    @Override public double accept(ShapeVisitor v) { return v.visitCircle(this); }
}

class AreaVisitor implements ShapeVisitor {
    @Override public double visitCircle(Circle c) { return PI * c.r * c.r; }
    @Override public double visitRectangle(Rectangle r) { return r.w * r.h; }
}
class PerimeterVisitor implements ShapeVisitor { ... }
// Adding a new operation = new Visitor class, no shape modification
```

---

## Scenario 15: Preventing Subclassing with final

**Context:** You're building a security-critical class that should not be extended.

**Answer:**
```java
public final class SecureToken {  // final prevents subclassing
    private final String value;
    private final Instant expiresAt;
    private final String userId;

    private SecureToken(String value, Instant expiresAt, String userId) {  // private constructor
        this.value = value;
        this.expiresAt = expiresAt;
        this.userId = userId;
    }

    // Factory method — validates and creates
    public static SecureToken generate(String userId, Duration validity) {
        String tokenValue = generateSecureRandom();
        return new SecureToken(tokenValue, Instant.now().plus(validity), userId);
    }

    public boolean isExpired() { return Instant.now().isAfter(expiresAt); }
    public boolean isValidFor(String userId) {
        return !isExpired() && this.userId.equals(userId);
    }
    // No setters — fully immutable
}
// final class + private constructor + no setters = cannot be subclassed or mutated
// This is security-critical: cannot extend and override isExpired() to always return false!
```

---

## Scenario 16: Encapsulation in Domain Model

**Context:** Design a bank account that enforces business rules.

**Answer:**
```java
public class BankAccount {
    private final String accountNumber;
    private BigDecimal balance;
    private AccountStatus status;
    private final List<Transaction> history = new ArrayList<>();

    private BankAccount(String accountNumber, BigDecimal initialBalance) {
        this.accountNumber = accountNumber;
        this.balance = initialBalance;
        this.status = AccountStatus.ACTIVE;
    }

    public static BankAccount open(BigDecimal initialDeposit) {
        if (initialDeposit.compareTo(BigDecimal.valueOf(100)) < 0) {
            throw new IllegalArgumentException("Minimum opening balance is $100");
        }
        return new BankAccount(generateAccountNumber(), initialDeposit);
    }

    public void deposit(BigDecimal amount) {
        requireActive();
        if (amount.compareTo(BigDecimal.ZERO) <= 0) throw new IllegalArgumentException("Amount must be positive");
        balance = balance.add(amount);
        history.add(new Transaction(DEPOSIT, amount, Instant.now()));
    }

    public void withdraw(BigDecimal amount) {
        requireActive();
        if (amount.compareTo(balance) > 0) throw new InsufficientFundsException();
        balance = balance.subtract(amount);
        history.add(new Transaction(WITHDRAWAL, amount, Instant.now()));
    }

    private void requireActive() {
        if (status != AccountStatus.ACTIVE) throw new IllegalStateException("Account is " + status);
    }

    public BigDecimal getBalance() { return balance; }
    public List<Transaction> getHistory() { return Collections.unmodifiableList(history); }
}
```

---

## Scenario 17: Anonymous Classes vs Lambdas

**Context:** When should you still use anonymous classes instead of lambdas?

**Answer:**
```java
// Lambda works for functional interfaces (single abstract method):
Runnable r = () -> doWork();
Comparator<String> c = (a, b) -> a.compareTo(b);

// Anonymous class needed when:

// 1. Need to maintain state:
Iterator<Integer> counter = new Iterator<Integer>() {
    int i = 0;
    @Override public boolean hasNext() { return i < 10; }
    @Override public Integer next() { return i++; }
    @Override public void remove() { throw new UnsupportedOperationException(); }
};

// 2. Implement multiple methods:
MouseListener ml = new MouseAdapter() {
    @Override public void mouseClicked(MouseEvent e) { ... }
    @Override public void mouseEntered(MouseEvent e) { ... }
};

// 3. Need to extend a class (not just implement an interface):
Thread t = new Thread("worker") {
    @Override public void run() { doWork(); }
    @Override public String getName() { return "custom-" + super.getName(); }
};

// 4. Referencing 'this' meaning the anonymous class itself:
Runnable self = new Runnable() {
    @Override public void run() {
        executor.schedule(this, 1, TimeUnit.SECONDS);  // 'this' = this Runnable
    }
};
// In lambda, 'this' refers to the enclosing class
```

---

## Scenario 18: Polymorphism Pitfall with Collections

**Context:**
```java
List<Dog> dogs = new ArrayList<>();
List<Animal> animals = dogs;  // COMPILE ERROR — why?
```

**Question:** Explain and how do you pass a List<Dog> where List<Animal> is expected.

**Answer:**
`List<Dog>` is NOT a `List<Animal>` even though `Dog extends Animal`. This is Java's type invariance for generics. If allowed:
```java
List<Animal> animals = dogs;  // hypothetically
animals.add(new Cat());  // adds Cat to what is actually a List<Dog>!
Dog d = dogs.get(0);  // ClassCastException — it's actually a Cat
```

**Solutions:**

```java
// Use bounded wildcard — read-only view:
void processAnimals(List<? extends Animal> animals) {
    for (Animal a : animals) { a.makeSound(); }  // can read
    // animals.add(new Dog());  // COMPILE ERROR — cannot add (except null)
}
processAnimals(dogs);  // OK — List<Dog> accepted

// Use unbounded wildcard — for truly generic operations:
void printAll(List<?> items) {
    items.forEach(System.out::println);
}

// Producer Extends Consumer Super (PECS):
// List<? extends Animal> — read from it (producer)
// List<? super Dog> — write to it (consumer)
void addDogs(List<? super Dog> animals, int count) {
    for (int i = 0; i < count; i++) animals.add(new Dog());
}
```

---

## Scenario 19: Mixing OOP and Functional Patterns

**Context:** A service needs to apply multiple validators to an order before processing.

**Answer:**
```java
@FunctionalInterface
interface OrderValidator {
    ValidationResult validate(Order order);

    default OrderValidator andThen(OrderValidator other) {
        return order -> {
            ValidationResult first = this.validate(order);
            if (!first.isValid()) return first;  // short-circuit
            return other.validate(order);
        };
    }
}

// Validators as lambdas or method references:
OrderValidator notEmpty = order -> order.getItems().isEmpty()
    ? ValidationResult.error("Order has no items")
    : ValidationResult.ok();

OrderValidator validCustomer = order -> customerService.exists(order.getCustomerId())
    ? ValidationResult.ok()
    : ValidationResult.error("Unknown customer");

OrderValidator stockAvailable = order -> inventoryService.allAvailable(order.getItems())
    ? ValidationResult.ok()
    : ValidationResult.error("Insufficient stock");

// Compose validators:
OrderValidator compositeValidator = notEmpty
    .andThen(validCustomer)
    .andThen(stockAvailable);

ValidationResult result = compositeValidator.validate(order);
```

---

## Scenario 20: SRP in Service Layer

**Context:** A 500-line UserService handles registration, authentication, profile updates, password reset, and email verification.

**Question:** How would you apply SRP?

**Answer:**
Split by responsibility:
```java
@Service class UserRegistrationService {
    void register(RegistrationRequest request) { ... }
    void verifyEmail(String token) { ... }
}

@Service class AuthenticationService {
    AuthToken login(LoginRequest request) { ... }
    void logout(String token) { ... }
    boolean validateToken(String token) { ... }
}

@Service class UserProfileService {
    UserProfile getProfile(Long userId) { ... }
    void updateProfile(Long userId, ProfileUpdateRequest request) { ... }
    void uploadAvatar(Long userId, MultipartFile file) { ... }
}

@Service class PasswordService {
    void changePassword(Long userId, ChangePasswordRequest request) { ... }
    void requestReset(String email) { ... }
    void resetPassword(String token, String newPassword) { ... }
}
// Each service has one clear reason to change
// Team members can work on each independently
```

---

## Scenario 21: Abstract Factory Pattern

**Context:** Your application needs to create different UI components for web and mobile.

**Answer:**
```java
interface Button { void render(); }
interface InputField { void render(); }
interface Dialog { void show(); }

interface UIFactory {
    Button createButton(String label);
    InputField createInputField(String placeholder);
    Dialog createDialog(String message);
}

class WebUIFactory implements UIFactory {
    @Override public Button createButton(String label) { return new HtmlButton(label); }
    @Override public InputField createInputField(String ph) { return new HtmlInputField(ph); }
    @Override public Dialog createDialog(String msg) { return new HtmlDialog(msg); }
}

class MobileUIFactory implements UIFactory {
    @Override public Button createButton(String label) { return new NativeButton(label); }
    @Override public InputField createInputField(String ph) { return new NativeInputField(ph); }
    @Override public Dialog createDialog(String msg) { return new NativeDialog(msg); }
}

@Service
class RegistrationForm {
    private final UIFactory factory;

    RegistrationForm(UIFactory factory) { this.factory = factory; }  // inject platform-specific factory

    void render() {
        Button submit = factory.createButton("Register");
        InputField email = factory.createInputField("Enter email");
        Dialog confirmDialog = factory.createDialog("Registration successful!");
        // same code works for both web and mobile!
    }
}
```

---

## Scenario 22: Avoiding "God Object" Anti-Pattern

**Context:** An `ApplicationManager` class has grown to 3000 lines with 50+ methods.

**Question:** What's wrong and how to refactor?

**Answer:**
A "God Object" violates SRP, DIP, and makes testing nearly impossible. Signs: many instance fields, many unrelated methods, difficult to name the class without "Manager" or "Handler."

**Refactoring approach:**
1. **Identify responsibilities:** Group methods by what they're doing
2. **Extract classes:** One class per responsibility
3. **Use dependency injection:** New classes injected where needed
4. **Introduce domain objects:** Rich domain objects carry their own behavior

```java
// Before:
class ApplicationManager {
    // 50 methods: user ops, order ops, payment ops, reporting, notifications...
}

// After:
@Service class UserDomainService { ... }
@Service class OrderDomainService { ... }
@Service class PaymentDomainService { ... }
@Service class ReportingService { ... }
@Service class NotificationService { ... }

// High-level orchestration in a thin facade if needed:
@Service class OrderFulfillmentOrchestrator {
    OrderFulfillmentOrchestrator(OrderDomainService orders,
                                  PaymentDomainService payments,
                                  NotificationService notifications) { ... }
    FulfillmentResult fulfill(Order order) { ... }
}
```

---

## Scenario 23: Type Safety with Generics and Polymorphism

**Context:** A generic repository interface and multiple implementations.

**Answer:**
```java
interface Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    T save(T entity);
    void delete(ID id);
}

@Repository
class JpaUserRepository implements Repository<User, Long> {
    @Override public Optional<User> findById(Long id) { return userJpaRepo.findById(id); }
    // ...
}

// Generic service:
abstract class CrudService<T, ID> {
    protected final Repository<T, ID> repository;
    CrudService(Repository<T, ID> repository) { this.repository = repository; }

    public T getById(ID id) {
        return repository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Not found: " + id));
    }
    public T save(T entity) { return repository.save(entity); }
}

@Service
class UserService extends CrudService<User, Long> {
    UserService(Repository<User, Long> repository) { super(repository); }
    // Specific user business logic here
}
```

---

## Scenario 24: Immutable Value Objects in Domain Model

**Context:** Design a `Price` value object that represents a monetary amount with currency.

**Answer:**
```java
public final class Price implements Comparable<Price> {
    private final BigDecimal amount;
    private final Currency currency;

    public Price(BigDecimal amount, Currency currency) {
        Objects.requireNonNull(amount, "amount");
        Objects.requireNonNull(currency, "currency");
        if (amount.signum() < 0) throw new IllegalArgumentException("Price cannot be negative");
        this.amount = amount.setScale(currency.getDefaultFractionDigits(), RoundingMode.HALF_UP);
        this.currency = currency;
    }

    public static Price of(double amount, String currencyCode) {
        return new Price(BigDecimal.valueOf(amount), Currency.getInstance(currencyCode));
    }

    public Price add(Price other) {
        requireSameCurrency(other);
        return new Price(amount.add(other.amount), currency);
    }

    public Price multiply(int quantity) {
        return new Price(amount.multiply(BigDecimal.valueOf(quantity)), currency);
    }

    public boolean isGreaterThan(Price other) {
        requireSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }

    private void requireSameCurrency(Price other) {
        if (!currency.equals(other.currency))
            throw new CurrencyMismatchException(currency, other.currency);
    }

    @Override public int compareTo(Price other) {
        requireSameCurrency(other);
        return amount.compareTo(other.amount);
    }

    @Override public boolean equals(Object o) {
        if (!(o instanceof Price p)) return false;
        return amount.compareTo(p.amount) == 0 && currency.equals(p.currency);
    }

    @Override public int hashCode() { return Objects.hash(amount.stripTrailingZeros(), currency); }
    @Override public String toString() { return currency.getSymbol() + amount.toPlainString(); }
}
```

---

## Scenario 25: Polymorphic Dispatch with Sealed Classes

**Context:** Design a payment result that can be Success, Failure, or Pending with exhaustive handling.

**Answer:**
```java
public sealed interface PaymentResult permits PaymentResult.Success, PaymentResult.Failure, PaymentResult.Pending {

    record Success(String transactionId, BigDecimal amount, Instant processedAt) implements PaymentResult {}

    record Failure(String errorCode, String message, boolean retryable) implements PaymentResult {}

    record Pending(String referenceId, Instant estimatedCompletionTime) implements PaymentResult {}
}

// Exhaustive handling — compiler enforces all cases:
String handleResult(PaymentResult result) {
    return switch (result) {
        case PaymentResult.Success s -> "Payment " + s.transactionId() + " completed";
        case PaymentResult.Failure f -> f.retryable()
            ? "Payment failed, retrying: " + f.message()
            : "Payment permanently failed: " + f.message();
        case PaymentResult.Pending p -> "Payment pending, ETA: " + p.estimatedCompletionTime();
        // No default needed — compiler knows all subtypes
    };
}
```
