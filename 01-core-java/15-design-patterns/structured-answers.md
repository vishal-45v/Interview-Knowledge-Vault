# Design Patterns — Structured Answers

---

## Q1: Implement All 4 Thread-Safe Singleton Variants in Java

**Approach 1: Eager Initialization**
```java
public class EagerSingleton {
    // Created at class-load time — always thread-safe
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    private EagerSingleton() {}
    public static EagerSingleton getInstance() { return INSTANCE; }
}
```
Drawback: created even if never used.

**Approach 2: Synchronized Lazy Initialization**
```java
public class SynchronizedSingleton {
    private static SynchronizedSingleton instance;
    private SynchronizedSingleton() {}

    public static synchronized SynchronizedSingleton getInstance() {
        if (instance == null) {
            instance = new SynchronizedSingleton();
        }
        return instance;
    }
}
```
Drawback: `synchronized` on every call — bottleneck under high concurrency.

**Approach 3: Double-Checked Locking (DCL)**
```java
public class DCLSingleton {
    // volatile prevents partially-constructed object being visible to other threads
    private static volatile DCLSingleton instance;

    private DCLSingleton() {
        if (instance != null) {
            throw new RuntimeException("Use getInstance() — reflection blocked");
        }
    }

    public static DCLSingleton getInstance() {
        if (instance == null) {
            synchronized (DCLSingleton.class) {
                if (instance == null) {
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}
```
Best performance for lazy init without enum. `volatile` is mandatory — without it, the JVM may reorder object construction and assignment, allowing another thread to see a non-null but uninitialized instance.

**Approach 4: Static Holder (Bill Pugh — preferred)**
```java
public class HolderSingleton {
    private HolderSingleton() {}

    // Inner class is only loaded when getInstance() is first called
    private static class Holder {
        private static final HolderSingleton INSTANCE = new HolderSingleton();
    }

    public static HolderSingleton getInstance() {
        return Holder.INSTANCE;
    }
}
```
Leverages JVM class-loading contract: a class is initialized exactly once, under JVM lock. No explicit synchronization. Lazy. Simple.

**Approach 5: Enum (Bloch's recommendation)**
```java
public enum EnumSingleton {
    INSTANCE;

    private int counter = 0;  // state is fine in enum

    public int nextId() { return ++counter; }
    public void reset()  { counter = 0; }
}

// Usage
EnumSingleton.INSTANCE.nextId();
```
Thread-safe, serialization-safe, reflection-proof. The only limitation: cannot extend another class.

---

## Q2: What Is the Difference Between Factory Method and Abstract Factory?

**Factory Method** defines an interface for creating **one product**, but lets subclasses decide which class to instantiate.

```java
// Factory Method
public abstract class LoggerFactory {
    public void log(String msg) {
        Logger logger = createLogger();   // factory method
        logger.write(msg);
    }
    protected abstract Logger createLogger();
}

public class FileLoggerFactory extends LoggerFactory {
    @Override protected Logger createLogger() { return new FileLogger(); }
}

public class ConsoleLoggerFactory extends LoggerFactory {
    @Override protected Logger createLogger() { return new ConsoleLogger(); }
}
```

**Abstract Factory** provides an interface for creating **families of related products** without specifying their concrete classes.

```java
// Abstract Factory — creates a whole family of consistent UI components
public interface UIComponentFactory {
    Button createButton();
    TextField createTextField();
    Dialog createDialog();
}

public class WindowsUIFactory implements UIComponentFactory {
    @Override public Button    createButton()    { return new WindowsButton(); }
    @Override public TextField createTextField() { return new WindowsTextField(); }
    @Override public Dialog    createDialog()    { return new WindowsDialog(); }
}

public class MacUIFactory implements UIComponentFactory {
    @Override public Button    createButton()    { return new MacButton(); }
    @Override public TextField createTextField() { return new MacTextField(); }
    @Override public Dialog    createDialog()    { return new MacDialog(); }
}

// Client uses the factory without knowing which OS it's on
public class Application {
    private final UIComponentFactory factory;

    public Application(UIComponentFactory factory) {
        this.factory = factory;
    }

    public void render() {
        Button btn = factory.createButton();    // guaranteed consistent theme
        Dialog dlg = factory.createDialog();
        btn.render();
        dlg.render();
    }
}
```

**Key difference:** Factory Method uses inheritance (override a method to change the product). Abstract Factory uses composition (inject a factory object to change the whole product family).

---

## Q3: How Does the Builder Pattern Prevent the Telescoping Constructor Anti-Pattern?

The **telescoping constructor** anti-pattern occurs when you need to support many combinations of optional parameters:

```java
// Anti-pattern — telescoping constructors
public class Pizza {
    public Pizza(String size) { ... }
    public Pizza(String size, boolean cheese) { ... }
    public Pizza(String size, boolean cheese, boolean pepperoni) { ... }
    public Pizza(String size, boolean cheese, boolean pepperoni, boolean mushrooms) { ... }
    // Grows to n! combinations with n toppings
}

// Confusing call site — what does 'true, false, true' mean?
Pizza p = new Pizza("large", true, false, true);
```

**Builder solution:**
```java
public class Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }

    private final Size size;          // required
    private final boolean cheese;
    private final boolean pepperoni;
    private final boolean mushrooms;
    private final boolean extraSauce;

    private Pizza(Builder b) {
        this.size       = b.size;
        this.cheese     = b.cheese;
        this.pepperoni  = b.pepperoni;
        this.mushrooms  = b.mushrooms;
        this.extraSauce = b.extraSauce;
    }

    public static class Builder {
        private final Size size;
        private boolean cheese     = false;
        private boolean pepperoni  = false;
        private boolean mushrooms  = false;
        private boolean extraSauce = false;

        public Builder(Size size) {
            Objects.requireNonNull(size, "size is required");
            this.size = size;
        }

        public Builder cheese()     { this.cheese     = true; return this; }
        public Builder pepperoni()  { this.pepperoni  = true; return this; }
        public Builder mushrooms()  { this.mushrooms  = true; return this; }
        public Builder extraSauce() { this.extraSauce = true; return this; }

        public Pizza build() { return new Pizza(this); }
    }
}

// Clear, self-documenting call site
Pizza p = new Pizza.Builder(Pizza.Size.LARGE)
    .cheese()
    .pepperoni()
    .extraSauce()
    .build();
```

Additional benefit: `Pizza` is **immutable** — no setters, all fields final. The object cannot be in an invalid intermediate state.

---

## Q4: How Does Spring Use the Proxy Pattern?

Spring AOP (Aspect-Oriented Programming) creates proxy objects around your beans to apply cross-cutting concerns (transactions, security, logging, caching) without modifying your business code.

```
Your Code         Spring Creates
-----------       --------------------------
@Service          --> TransactionProxy wrapping OrderService
OrderService           |-- begins transaction
  placeOrder()         |-- calls real placeOrder()
                       |-- commits or rolls back
```

**JDK Dynamic Proxy (when bean implements interface):**
```java
// Spring creates this at runtime:
Object proxy = Proxy.newProxyInstance(
    OrderService.class.getClassLoader(),
    new Class<?>[]{ IOrderService.class },
    new InvocationHandler() {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            TransactionStatus tx = txManager.getTransaction(txDef);
            try {
                Object result = method.invoke(realOrderService, args);
                txManager.commit(tx);
                return result;
            } catch (Exception e) {
                txManager.rollback(tx);
                throw e;
            }
        }
    }
);
```

**CGLIB Proxy (when bean has no interface):**
```java
@Service
public class ProductService {           // no interface
    @Transactional
    public void saveProduct(Product p) { ... }
}
// Spring uses CGLIB to subclass ProductService
// and override saveProduct() to add transaction management
```

**Important consequence:** because Spring injects the proxy, not `this`, calling a `@Transactional` method from within the same class bypasses the proxy:

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder() {
        validateOrder();  // TRAP: self-invocation — no transaction for validateOrder
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateOrder() {
        // This @Transactional is IGNORED when called from placeOrder() above
        // because it's called on 'this', not the proxy
    }
}
```

---

## Q5: Implement a Real-World Observer Pattern Example in Java

```java
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

// Observer interface
public interface OrderStatusObserver {
    void onStatusChange(String orderId, OrderStatus oldStatus, OrderStatus newStatus);
}

public enum OrderStatus { PLACED, PAID, SHIPPED, DELIVERED, CANCELLED }

// Subject
public class Order {
    private final String orderId;
    private OrderStatus status;
    private final List<OrderStatusObserver> observers = new CopyOnWriteArrayList<>();

    public Order(String orderId) {
        this.orderId = orderId;
        this.status  = OrderStatus.PLACED;
    }

    public void addObserver(OrderStatusObserver o)    { observers.add(o); }
    public void removeObserver(OrderStatusObserver o) { observers.remove(o); }

    public void transitionTo(OrderStatus newStatus) {
        OrderStatus old = this.status;
        this.status = newStatus;
        observers.forEach(o -> o.onStatusChange(orderId, old, newStatus));
    }
}

// Concrete observers
public class EmailNotificationObserver implements OrderStatusObserver {
    @Override
    public void onStatusChange(String orderId, OrderStatus old, OrderStatus newStatus) {
        System.out.printf("EMAIL: Order %s moved from %s to %s%n", orderId, old, newStatus);
    }
}

public class InventoryObserver implements OrderStatusObserver {
    @Override
    public void onStatusChange(String orderId, OrderStatus old, OrderStatus newStatus) {
        if (newStatus == OrderStatus.SHIPPED) {
            System.out.printf("INVENTORY: Decrement stock for order %s%n", orderId);
        }
    }
}

public class AuditLogObserver implements OrderStatusObserver {
    @Override
    public void onStatusChange(String orderId, OrderStatus old, OrderStatus newStatus) {
        System.out.printf("AUDIT: [%s] %s: %s -> %s%n",
            Instant.now(), orderId, old, newStatus);
    }
}

// Usage
public class Main {
    public static void main(String[] args) {
        Order order = new Order("ORD-001");
        order.addObserver(new EmailNotificationObserver());
        order.addObserver(new InventoryObserver());
        order.addObserver(new AuditLogObserver());

        order.transitionTo(OrderStatus.PAID);
        order.transitionTo(OrderStatus.SHIPPED);
        order.transitionTo(OrderStatus.DELIVERED);
    }
}
```

Note: `CopyOnWriteArrayList` is used to avoid `ConcurrentModificationException` if an observer removes itself during notification.

---

## Q6: What Is the Difference Between Strategy and State Pattern?

Both delegate behavior to an interface, but they differ in who drives the change and whether the states know about each other.

**Strategy:** client chooses the algorithm; it doesn't change automatically.
```java
// PaymentStrategy — client picks the strategy
public interface PaymentStrategy {
    void pay(double amount);
}

public class CreditCardStrategy implements PaymentStrategy {
    private final String cardNumber;
    CreditCardStrategy(String cardNumber) { this.cardNumber = cardNumber; }

    @Override public void pay(double amount) {
        System.out.printf("Charged %.2f to card %s%n", amount, cardNumber);
    }
}

public class PayPalStrategy implements PaymentStrategy {
    private final String email;
    PayPalStrategy(String email) { this.email = email; }

    @Override public void pay(double amount) {
        System.out.printf("Sent %.2f via PayPal to %s%n", amount, email);
    }
}

public class ShoppingCart {
    private PaymentStrategy strategy;
    public void setPaymentStrategy(PaymentStrategy s) { this.strategy = s; }
    public void checkout(double total) { strategy.pay(total); }
}
```

**State:** the object transitions between states automatically based on events.
```java
// TrafficLight — transitions happen inside state objects
public interface TrafficLightState {
    void handle(TrafficLight light);
    String getColor();
}

public class GreenState implements TrafficLightState {
    @Override public void handle(TrafficLight light) {
        System.out.println("Green -> switching to Yellow");
        light.setState(new YellowState());   // transition happens inside state
    }
    @Override public String getColor() { return "GREEN"; }
}

public class YellowState implements TrafficLightState {
    @Override public void handle(TrafficLight light) {
        System.out.println("Yellow -> switching to Red");
        light.setState(new RedState());
    }
    @Override public String getColor() { return "YELLOW"; }
}

public class RedState implements TrafficLightState {
    @Override public void handle(TrafficLight light) {
        System.out.println("Red -> switching to Green");
        light.setState(new GreenState());
    }
    @Override public String getColor() { return "RED"; }
}

public class TrafficLight {
    private TrafficLightState state = new GreenState();
    public void setState(TrafficLightState s) { this.state = s; }
    public void tick() { state.handle(this); }
}
```

**Summary:** Strategy = algorithm selection by client. State = automatic self-managed transitions.

---

## Q7: How Does the Decorator Pattern Differ From Inheritance?

**Inheritance:**
- Compile-time decision
- Creates a rigid hierarchy — modifying a base class affects all subclasses
- Each combination requires a new class

**Decorator:**
- Runtime composition
- Follows Open/Closed Principle — add behavior without modifying existing classes
- Combinations happen at instantiation

```java
// Real-world: Java I/O streams are Decorators
InputStream raw         = new FileInputStream("file.txt");
InputStream buffered    = new BufferedInputStream(raw);       // adds buffering
InputStream gzipped     = new GZIPInputStream(buffered);      // adds decompression
InputStreamReader reader = new InputStreamReader(gzipped, StandardCharsets.UTF_8); // adapts bytes to chars
BufferedReader br        = new BufferedReader(reader);         // adds readLine()

// Stack is built at runtime based on what you need.
// With inheritance, you'd need:
// BufferedGZIPFileInputStream, UnbufferedGZIPFileInputStream, etc.
```

**Custom Decorator example:**
```java
public interface DataStore {
    void save(String key, String value);
    String load(String key);
}

public class InMemoryStore implements DataStore {
    private Map<String, String> map = new HashMap<>();
    @Override public void save(String k, String v) { map.put(k, v); }
    @Override public String load(String k) { return map.get(k); }
}

public class LoggingDecorator implements DataStore {
    private final DataStore delegate;
    public LoggingDecorator(DataStore d) { this.delegate = d; }

    @Override public void save(String k, String v) {
        System.out.println("Saving: " + k);
        delegate.save(k, v);
    }
    @Override public String load(String k) {
        System.out.println("Loading: " + k);
        return delegate.load(k);
    }
}

public class CachingDecorator implements DataStore {
    private final DataStore delegate;
    private final Map<String, String> cache = new HashMap<>();
    public CachingDecorator(DataStore d) { this.delegate = d; }

    @Override public void save(String k, String v) {
        cache.put(k, v);
        delegate.save(k, v);
    }
    @Override public String load(String k) {
        return cache.computeIfAbsent(k, delegate::load);
    }
}

// Stack at runtime: cache on top of logging on top of memory store
DataStore store = new CachingDecorator(new LoggingDecorator(new InMemoryStore()));
```

---

## Q8: When Would You Use Chain of Responsibility?

Use it when:
- Multiple objects may handle a request, and the handler isn't known at compile time
- You want to issue a request to one of several handlers without specifying the receiver explicitly
- You want to add or remove handlers without changing the client

**Real-world use cases:**
- Servlet filter chains (`javax.servlet.Filter`)
- Spring Security filter chain
- Log level hierarchy (`DEBUG → INFO → WARN → ERROR`)
- Middleware in Express/Node-style frameworks
- Support ticket escalation

```java
public abstract class AuthorizationHandler {
    protected AuthorizationHandler successor;

    public final void handle(SecurityContext ctx, String resource) {
        if (canHandle(ctx, resource)) {
            doHandle(ctx, resource);
        } else if (successor != null) {
            successor.handle(ctx, resource);
        } else {
            throw new AccessDeniedException("Access denied to: " + resource);
        }
    }

    protected abstract boolean canHandle(SecurityContext ctx, String resource);
    protected abstract void doHandle(SecurityContext ctx, String resource);
    public void setSuccessor(AuthorizationHandler h) { this.successor = h; }
}

public class OwnerAuthHandler extends AuthorizationHandler {
    @Override protected boolean canHandle(SecurityContext ctx, String resource) {
        return ctx.getUserId().equals(getOwnerId(resource));
    }
    @Override protected void doHandle(SecurityContext ctx, String resource) {
        System.out.println("Owner access granted");
    }
}

public class AdminAuthHandler extends AuthorizationHandler {
    @Override protected boolean canHandle(SecurityContext ctx, String resource) {
        return ctx.getRoles().contains("ADMIN");
    }
    @Override protected void doHandle(SecurityContext ctx, String resource) {
        System.out.println("Admin access granted");
    }
}
```

---

## Q9: What Is the Template Method Pattern and Where Does Spring Use It?

Template Method defines the **skeleton of an algorithm** in a base class, deferring some steps to subclasses. Subclasses can override specific steps without changing the algorithm's structure.

```java
// Template
public abstract class DataImporter {
    // Template method — final to prevent structural changes
    public final void importData(String source) {
        List<String> raw       = readData(source);         // step 1
        List<Object> parsed    = parseData(raw);           // step 2 — abstract
        List<Object> validated = validateData(parsed);     // step 3
        saveData(validated);                               // step 4 — abstract
        postProcess();                                     // step 5 — hook (optional)
    }

    protected abstract List<Object> parseData(List<String> raw);
    protected abstract void saveData(List<Object> data);

    protected List<String> readData(String source) {
        return Files.readAllLines(Path.of(source));        // default impl
    }

    protected List<Object> validateData(List<Object> data) {
        return data.stream().filter(Objects::nonNull).toList();  // default validation
    }

    protected void postProcess() {}   // hook — empty by default, override if needed
}

public class CsvOrderImporter extends DataImporter {
    @Override protected List<Object> parseData(List<String> raw) {
        return raw.stream().map(CsvParser::parseOrder).collect(Collectors.toList());
    }
    @Override protected void saveData(List<Object> data) {
        orderRepository.saveAll((List<Order>) data);
    }
}
```

**Spring uses Template Method extensively:**

1. `JdbcTemplate` — handles connection, statement prep, exception mapping; you provide the SQL and `RowMapper`.
2. `HibernateTemplate` — manages Session lifecycle; you provide the HibernateCallback.
3. `AbstractSecurityInterceptor` in Spring Security — the security check algorithm is fixed; subclasses (MethodSecurityInterceptor, FilterSecurityInterceptor) provide specifics.
4. `AbstractController` (Spring MVC) — handles HTTP method dispatch; you override `handleRequestInternal()`.

```java
// JdbcTemplate usage — Template Method in action
List<User> users = jdbcTemplate.query(
    "SELECT * FROM users WHERE active = ?",
    (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")),  // your step
    true
);
// JdbcTemplate handles: getting connection, creating PreparedStatement,
// executing query, iterating ResultSet, closing resources, translating exceptions
```

---

## Q10: How Does the Command Pattern Enable Undo/Redo Functionality?

Each Command encapsulates both `execute()` and `undo()`. A history stack tracks executed commands.

```java
public interface Command {
    void execute();
    void undo();
    String getDescription();
}

// Concrete commands for a text editor
public class InsertTextCommand implements Command {
    private final TextBuffer buffer;
    private final int position;
    private final String text;

    public InsertTextCommand(TextBuffer buffer, int position, String text) {
        this.buffer   = buffer;
        this.position = position;
        this.text     = text;
    }

    @Override public void execute() { buffer.insert(position, text); }
    @Override public void undo()    { buffer.delete(position, position + text.length()); }
    @Override public String getDescription() { return "Insert '" + text + "' at " + position; }
}

public class DeleteTextCommand implements Command {
    private final TextBuffer buffer;
    private final int start;
    private final int end;
    private String deletedText;  // saved for undo

    @Override public void execute() {
        deletedText = buffer.getText(start, end);   // save before delete
        buffer.delete(start, end);
    }
    @Override public void undo() { buffer.insert(start, deletedText); }
    @Override public String getDescription() { return "Delete text from " + start + " to " + end; }
}

// Invoker with undo/redo stacks
public class TextEditorHistory {
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();

    public void execute(Command cmd) {
        cmd.execute();
        undoStack.push(cmd);
        redoStack.clear();   // redo history cleared after new command
    }

    public void undo() {
        if (undoStack.isEmpty()) return;
        Command cmd = undoStack.pop();
        cmd.undo();
        redoStack.push(cmd);
    }

    public void redo() {
        if (redoStack.isEmpty()) return;
        Command cmd = redoStack.pop();
        cmd.execute();
        undoStack.push(cmd);
    }

    public List<String> getHistory() {
        return undoStack.stream()
            .map(Command::getDescription)
            .collect(Collectors.toList());
    }
}

// Usage
TextBuffer buffer   = new TextBuffer();
TextEditorHistory h = new TextEditorHistory();

h.execute(new InsertTextCommand(buffer, 0, "Hello"));
h.execute(new InsertTextCommand(buffer, 5, " World"));
// buffer: "Hello World"

h.undo();
// buffer: "Hello"

h.redo();
// buffer: "Hello World"
```

**Additional Command Pattern benefits:**
- **Macro commands:** group multiple commands into one composite command
- **Logging:** serialize commands to disk for crash recovery
- **Queue:** commands can be placed in a work queue and executed asynchronously
- **Transaction script:** a database transaction can be modeled as a Command with `execute()` and `rollback()`
