# Design Patterns — Diagram Explanations

---

## 1. Singleton — Class Diagram + Thread-Safe Implementations

```
+---------------------------+
|        Singleton          |
+---------------------------+
| - instance: Singleton     |
+---------------------------+
| - Singleton()             |  <-- private constructor
| + getInstance(): Singleton|
+---------------------------+
```

**Thread-Safe Double-Checked Locking (DCL):**

```java
public class Singleton {
    // volatile prevents instruction reordering; crucial for DCL correctness
    private static volatile Singleton instance;

    private Singleton() {
        // prevent reflection attacks
        if (instance != null) {
            throw new RuntimeException("Use getInstance()");
        }
    }

    public static Singleton getInstance() {
        if (instance == null) {                    // first check (no locking)
            synchronized (Singleton.class) {
                if (instance == null) {            // second check (with locking)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

Without `volatile`, the JVM can reorder `instance = new Singleton()` into:
1. Allocate memory
2. Assign reference to `instance` (now non-null!)
3. Call constructor

Another thread seeing non-null `instance` could get an incompletely constructed object.

---

## 2. Factory Method — Creator Hierarchy

```
        <<abstract>>
          Creator
        +-----------+
        | + create()|  <-- calls factoryMethod()
        | # factoryMethod(): Product  <<abstract>>
        +-----------+
              |
     ---------+---------
     |                 |
ConcreteCreatorA   ConcreteCreatorB
  + factoryMethod()  + factoryMethod()
  returns ProductA   returns ProductB

        <<interface>>
           Product
        +-----------+
        | + use()   |
        +-----------+
              |
     ---------+---------
     |                 |
  ProductA          ProductB
  + use()           + use()
```

```java
// Abstract creator
public abstract class NotificationFactory {
    public void sendNotification(String message) {
        Notification n = createNotification();   // factory method
        n.send(message);
    }
    protected abstract Notification createNotification();
}

// Concrete creators
public class EmailNotificationFactory extends NotificationFactory {
    @Override
    protected Notification createNotification() {
        return new EmailNotification();
    }
}

public class SMSNotificationFactory extends NotificationFactory {
    @Override
    protected Notification createNotification() {
        return new SMSNotification();
    }
}
```

---

## 3. Builder Pattern — Director, Builder, Product

```
  Director                Builder (interface)
+----------+           +--------------------+
| construct|---------->| + buildPartA()     |
|          |           | + buildPartB()     |
+----------+           | + getResult()      |
                       +--------------------+
                                |
                    ------------+------------
                    |                       |
           ConcreteBuilderX        ConcreteBuilderY
           + buildPartA()          + buildPartA()
           + buildPartB()          + buildPartB()
           + getResult(): Product  + getResult(): Product
```

**Fluent Builder (common in Java — no Director needed):**

```java
public class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final String body;
    private final int timeoutMs;

    private HttpRequest(Builder builder) {
        this.url       = builder.url;
        this.method    = builder.method;
        this.headers   = Collections.unmodifiableMap(builder.headers);
        this.body      = builder.body;
        this.timeoutMs = builder.timeoutMs;
    }

    public static class Builder {
        private final String url;          // required
        private String method = "GET";     // optional with default
        private Map<String, String> headers = new HashMap<>();
        private String body;
        private int timeoutMs = 5000;

        public Builder(String url) { this.url = url; }

        public Builder method(String method)   { this.method = method; return this; }
        public Builder header(String k, String v) { headers.put(k, v); return this; }
        public Builder body(String body)       { this.body = body; return this; }
        public Builder timeout(int ms)         { this.timeoutMs = ms; return this; }

        public HttpRequest build() {
            if (url == null || url.isBlank()) throw new IllegalStateException("URL required");
            return new HttpRequest(this);
        }
    }
}

// Usage
HttpRequest req = new HttpRequest.Builder("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .header("Authorization", "Bearer token123")
    .body("{\"name\":\"Alice\"}")
    .timeout(3000)
    .build();
```

---

## 4. Observer Pattern — Subject to Observer Notification Flow

```
         Subject (Observable)
       +------------------------+
       | - observers: List      |
       | + register(Observer)   |
       | + unregister(Observer) |
       | + notifyObservers()    |
       | + setState(...)        |
       +------------------------+
                  |
        notifyObservers()
                  |
    +-------------+-------------+
    |             |             |
Observer1     Observer2     Observer3
+ update()    + update()    + update()

Flow:
1. Observer registers with Subject
2. Subject state changes
3. Subject calls notifyObservers()
4. Each Observer.update() is called with new state
5. Observer pulls or receives new data
```

```java
public interface StockObserver {
    void onPriceChange(String ticker, double newPrice);
}

public class StockMarket {
    private final Map<String, Double> prices = new HashMap<>();
    private final List<StockObserver> observers = new CopyOnWriteArrayList<>();

    public void addObserver(StockObserver o)    { observers.add(o); }
    public void removeObserver(StockObserver o) { observers.remove(o); }

    public void updatePrice(String ticker, double price) {
        prices.put(ticker, price);
        observers.forEach(o -> o.onPriceChange(ticker, price));
    }
}

public class AlertService implements StockObserver {
    @Override
    public void onPriceChange(String ticker, double newPrice) {
        if (newPrice > 1000) {
            System.out.println("ALERT: " + ticker + " hit " + newPrice);
        }
    }
}
```

---

## 5. Strategy Pattern — Context + Strategy Interface + Concrete Strategies

```
        Context
      +-----------------+
      | - strategy      |---------->  <<interface>>
      | + setStrategy() |             SortStrategy
      | + execute()     |           +-------------+
      +-----------------+           | + sort(int[])|
                                    +-------------+
                                          |
                          +--------------++--------------+
                          |              |               |
                   BubbleSort      QuickSort        MergeSort
                   + sort(int[])   + sort(int[])    + sort(int[])
```

```java
@FunctionalInterface
public interface SortStrategy {
    void sort(int[] data);
}

public class Sorter {
    private SortStrategy strategy;

    public Sorter(SortStrategy strategy) { this.strategy = strategy; }
    public void setStrategy(SortStrategy strategy) { this.strategy = strategy; }

    public void sort(int[] data) { strategy.sort(data); }
}

// Usage — strategy swapped at runtime
Sorter sorter = new Sorter(Arrays::sort);   // lambda as strategy
sorter.sort(new int[]{5, 3, 1, 4, 2});

// Swap strategy
sorter.setStrategy(data -> {
    // custom bubble sort
    for (int i = 0; i < data.length - 1; i++)
        for (int j = 0; j < data.length - i - 1; j++)
            if (data[j] > data[j+1]) { int t = data[j]; data[j] = data[j+1]; data[j+1] = t; }
});
```

---

## 6. Decorator Pattern — Wrapping Layers

```
         <<interface>>
           Component
         +----------+
         | + op()   |
         +----------+
              |
    ----------+----------
    |                   |
ConcreteComponent    Decorator (abstract)
+ op()             - wrappee: Component
                   + op() { wrappee.op() }
                        |
              ----------+----------
              |                   |
       DecoratorA           DecoratorB
       + op()               + op()
       (calls super + adds  (calls super + adds
        logging)             encryption)

Stacking:
  new DecoratorB(new DecoratorA(new ConcreteComponent()))

Call chain:
  DecoratorB.op()
    -> DecoratorA.op()
      -> ConcreteComponent.op()
    <- return + DecoratorA adds logging
  <- return + DecoratorB adds encryption
```

```java
public interface TextProcessor {
    String process(String text);
}

public class PlainTextProcessor implements TextProcessor {
    @Override public String process(String text) { return text; }
}

public abstract class TextDecorator implements TextProcessor {
    protected final TextProcessor wrapped;
    public TextDecorator(TextProcessor wrapped) { this.wrapped = wrapped; }
}

public class TrimDecorator extends TextDecorator {
    public TrimDecorator(TextProcessor wrapped) { super(wrapped); }
    @Override public String process(String text) { return wrapped.process(text).trim(); }
}

public class UpperCaseDecorator extends TextDecorator {
    public UpperCaseDecorator(TextProcessor wrapped) { super(wrapped); }
    @Override public String process(String text) { return wrapped.process(text).toUpperCase(); }
}

// Usage
TextProcessor processor = new UpperCaseDecorator(new TrimDecorator(new PlainTextProcessor()));
System.out.println(processor.process("  hello world  "));  // "HELLO WORLD"
```

---

## 7. Proxy Pattern — Client → Proxy → RealSubject

```
   Client
     |
     | calls interface methods
     v
   Proxy
   +---------------------+
   | - realSubject       |
   | + request()         |------> RealSubject
   |   (access control,  |        + request()
   |    caching,         |        (actual work)
   |    lazy init)       |
   +---------------------+

Types of Proxy:
- Virtual Proxy:    lazy initialization (create RealSubject only when needed)
- Protection Proxy: access control
- Caching Proxy:    cache results
- Remote Proxy:     RealSubject is in another JVM (RMI)
- Logging Proxy:    log all calls (Spring AOP does this)
```

```java
public interface ImageLoader {
    void display(String filename);
}

public class DiskImageLoader implements ImageLoader {
    @Override
    public void display(String filename) {
        System.out.println("Loading from disk: " + filename);
        // expensive disk I/O
    }
}

// Virtual proxy with lazy init + caching
public class CachingImageProxy implements ImageLoader {
    private final Map<String, ImageLoader> cache = new HashMap<>();

    @Override
    public void display(String filename) {
        cache.computeIfAbsent(filename, f -> {
            System.out.println("Cache miss, loading: " + f);
            return new DiskImageLoader();
        }).display(filename);
    }
}
```

---

## 8. Chain of Responsibility — Handler Chain

```
Client ---> Handler1 ---> Handler2 ---> Handler3 ---> null
              |               |               |
           handles?        handles?        handles?
           Yes: done       No: pass        Yes: done
           No: pass        to next

Example: HTTP request filter chain

Request --> AuthFilter --> LoggingFilter --> RateLimitFilter --> Controller
               |                |                  |
           401 if no        logs req           429 if too
           token                               many requests
```

```java
public abstract class SupportHandler {
    protected SupportHandler next;

    public SupportHandler setNext(SupportHandler next) {
        this.next = next;
        return next;  // allows chaining: h1.setNext(h2).setNext(h3)
    }

    public abstract void handle(SupportTicket ticket);

    protected void passToNext(SupportTicket ticket) {
        if (next != null) next.handle(ticket);
        else System.out.println("No handler found for: " + ticket);
    }
}

public class L1SupportHandler extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getPriority() == Priority.LOW) {
            System.out.println("L1 handling: " + ticket);
        } else {
            passToNext(ticket);
        }
    }
}

public class L2SupportHandler extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getPriority() == Priority.MEDIUM) {
            System.out.println("L2 handling: " + ticket);
        } else {
            passToNext(ticket);
        }
    }
}

// Setup
SupportHandler l1 = new L1SupportHandler();
SupportHandler l2 = new L2SupportHandler();
SupportHandler l3 = new L3SupportHandler();
l1.setNext(l2).setNext(l3);

l1.handle(new SupportTicket(Priority.HIGH, "Server down"));
// L1 passes -> L2 passes -> L3 handles
```
