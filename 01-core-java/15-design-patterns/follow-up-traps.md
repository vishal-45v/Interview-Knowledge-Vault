# Design Patterns — Follow-Up Traps

These are the questions that separate candidates who memorized patterns from those who truly understand them.

---

## Trap 1: All 4 Thread-Safe Singleton Approaches

**Q: How many ways can you make a Singleton thread-safe in Java?**

**Approach 1: Synchronized method (simplest, slowest)**
```java
public class Singleton {
    private static Singleton instance;
    private Singleton() {}

    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```
Problem: `synchronized` is acquired on *every* call, even after initialization. High contention under load.

**Approach 2: Double-Checked Locking (DCL)**
```java
public class Singleton {
    private static volatile Singleton instance;   // volatile is MANDATORY
    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```
Only synchronizes during initialization. After that, reads are unsynchronized. `volatile` is critical to prevent reading a partially constructed object due to instruction reordering.

**Approach 3: Static Holder (Bill Pugh) — Recommended**
```java
public class Singleton {
    private Singleton() {}

    private static class Holder {
        // Loaded only when Holder is first referenced
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;
    }
}
```
Relies on the JVM class-loading guarantee: a class is initialized only once, under the protection of a class loader lock. No explicit synchronization needed. Lazy-loaded. Thread-safe by the JVM spec.

**Approach 4: Enum Singleton — Best for most cases**
```java
public enum Singleton {
    INSTANCE;

    public void doSomething() {
        System.out.println("Doing something");
    }
}

// Usage
Singleton.INSTANCE.doSomething();
```

---

## Trap 2: Why Is Enum Singleton the Best?

The enum approach is recommended by Joshua Bloch (*Effective Java*, Item 3) because:

1. **Thread-safe by JVM spec** — enum instances are initialized once during class loading, under JVM synchronization guarantees.
2. **Serialization-safe** — Java guarantees that enum values are never duplicated during deserialization. The JVM automatically handles `readResolve()`.
3. **Reflection-safe** — you cannot reflectively call the private constructor of an enum. The JVM throws `IllegalArgumentException` if you try.

```java
// This WILL throw an exception for enum:
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor(String.class, int.class);
constructor.setAccessible(true);
Singleton s = constructor.newInstance("INSTANCE", 0);
// java.lang.IllegalArgumentException: Cannot reflectively create enum objects
```

The only drawback: if your singleton must extend another class, you cannot use an enum (enums cannot extend classes).

---

## Trap 3: Singleton Breaks With Reflection — How to Prevent?

```java
// Attack on a regular singleton:
Constructor<Singleton> c = Singleton.class.getDeclaredConstructor();
c.setAccessible(true);
Singleton s1 = Singleton.getInstance();
Singleton s2 = c.newInstance();  // creates a SECOND instance!
System.out.println(s1 == s2);    // false — singleton broken
```

**Prevention:** Guard in the private constructor:
```java
private Singleton() {
    if (instance != null) {
        throw new RuntimeException(
            "Use Singleton.getInstance() — reflection attack prevented"
        );
    }
}
```

But this only works if `instance` is set before the reflective call (e.g., via eager initialization). For the static holder pattern, the constructor guard is unreliable because `instance` is in an inner class.

The only bullet-proof solution is the **enum** approach.

---

## Trap 4: Singleton Breaks With Serialization — How to Fix?

```java
// Serialize then deserialize
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("singleton.ser"));
oos.writeObject(Singleton.getInstance());
oos.close();

ObjectInputStream ois = new ObjectInputStream(new FileInputStream("singleton.ser"));
Singleton deserialized = (Singleton) ois.readObject();
System.out.println(Singleton.getInstance() == deserialized);  // false!
```

**Fix: Implement `readResolve()`:**
```java
public class Singleton implements Serializable {
    private static final long serialVersionUID = 1L;
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() {}
    public static Singleton getInstance() { return INSTANCE; }

    // Called by deserialization — returns existing instance instead of new one
    protected Object readResolve() {
        return INSTANCE;
    }
}
```

Again, the enum automatically handles this — no `readResolve()` needed.

---

## Trap 5: Factory Method vs Abstract Factory vs Builder — When to Use Which?

| Aspect | Factory Method | Abstract Factory | Builder |
|---|---|---|---|
| Creates | One product type | A family of related products | One complex product |
| How | Subclasses decide the type | An abstract factory interface | Step-by-step via a builder |
| Variation axis | Product type | Product family/platform | Construction steps |
| Example | `Calendar.getInstance()` | UI toolkit (Windows vs Mac buttons, checkboxes) | `StringBuilder`, Lombok `@Builder` |

**Factory Method** — when you want to defer the decision of which class to instantiate to a subclass.

**Abstract Factory** — when your system must be independent of how its products are created, composed, and represented, and you need to ensure consistency among products in a family (e.g., a dark-theme button must pair with a dark-theme dialog).

**Builder** — when a constructor would need many parameters (more than 4-5), especially optional ones. Builder makes construction readable and prevents invalid intermediate states.

---

## Trap 6: Observer Pattern Memory Leaks

**Q: What is the most common bug in Observer pattern implementations?**

If an Observer registers with a Subject but **never unregisters**, the Subject holds a reference to the Observer indefinitely. This prevents the Observer from being garbage-collected, even if the rest of the application no longer needs it.

```java
// Common leak pattern in Android/Swing
public class EventBus {
    private List<EventListener> listeners = new ArrayList<>();
    public void register(EventListener l)   { listeners.add(l); }
    public void unregister(EventListener l) { listeners.remove(l); }
}

// If you forget to call unregister(), the listener leaks
```

**Solutions:**
1. Always unregister in cleanup methods (`onDestroy()`, `@PreDestroy`, `close()`).
2. Use `WeakReference<Observer>` in the Subject's list — the Subject won't prevent GC.
3. Use event buses that support automatic cleanup (e.g., Guava EventBus with `@Subscribe`).

```java
// WeakReference approach
private List<WeakReference<StockObserver>> observers = new ArrayList<>();

public void notifyObservers(double price) {
    Iterator<WeakReference<StockObserver>> it = observers.iterator();
    while (it.hasNext()) {
        StockObserver o = it.next().get();
        if (o == null) { it.remove(); }  // collected, clean up
        else { o.onPriceChange(price); }
    }
}
```

---

## Trap 7: Decorator vs Inheritance — When to Prefer Which?

**Inheritance problems:**
- Rigid — decided at compile time
- Causes class explosion for every combination: `LoggingReadableStream`, `BufferedLoggingReadableStream`, `EncryptedBufferedLoggingReadableStream`...
- Violates Open/Closed Principle when you modify base class behavior

**Decorator advantages:**
- Flexible — composed at runtime
- Each decorator has a single responsibility
- Combinations happen at instantiation time, not class-definition time

**Prefer Inheritance when:**
- The "is-a" relationship is truly fundamental and stable
- You want to override behavior, not wrap it
- You need access to protected members of the base class

**Prefer Decorator when:**
- You want to add responsibilities to individual objects without affecting others
- Extension by subclassing would produce an unreasonable number of subclasses
- You want to add/remove behavior dynamically at runtime

---

## Trap 8: Strategy vs State Pattern — What Is the Difference?

Both patterns use an interface and swap implementations, but their intents are different:

| Aspect | Strategy | State |
|---|---|---|
| What changes | The algorithm/behavior | The object's internal state |
| Who changes it | Client sets the strategy | The context or state itself transitions |
| States know each other? | No | Often yes — states trigger transitions |
| Example | Sort algorithm | Order status (PENDING → PAID → SHIPPED → DELIVERED) |

```java
// Strategy: client decides which algorithm to use
sorter.setStrategy(new QuickSortStrategy());

// State: object itself transitions based on events
order.pay();      // transitions PENDING -> PAID internally
order.ship();     // transitions PAID -> SHIPPED internally
// Order knows its current state; state knows how to transition
```

The key tell: in State pattern, transitions often happen inside the state objects themselves. In Strategy, the context or client swaps strategies from outside.

---

## Trap 9: Is Spring @Transactional a Command Pattern?

**Short answer:** Not exactly, but it shares characteristics.

`@Transactional` is implemented via **AOP proxy** (Proxy pattern), not Command. The proxy intercepts the method call, begins a transaction, calls the method, and commits or rolls back.

However, the **Command pattern** is present in Spring Batch:
```java
// Spring Batch Step is a Command object
Step step = stepBuilderFactory.get("myStep")
    .<InputType, OutputType>chunk(100)
    .reader(reader)
    .processor(processor)
    .writer(writer)
    .build();

// Job executes commands (steps) sequentially or conditionally
Job job = jobBuilderFactory.get("myJob")
    .start(step1)
    .next(step2)
    .build();
```

Each `Step` encapsulates work that can be scheduled, retried, or rolled back independently — classic Command.

---

## Trap 10: Proxy — JDK Dynamic Proxy vs CGLIB — Which Does Spring Use When?

**JDK Dynamic Proxy:**
- Works only on **interfaces** — the target bean must implement at least one interface
- Creates a proxy that implements the same interface
- Uses `java.lang.reflect.Proxy`

**CGLIB:**
- Works on **concrete classes** — subclasses the target class at runtime using bytecode generation
- Required when the bean does not implement any interface
- Cannot proxy `final` classes or `final` methods

**Spring's decision logic:**
```
Does the bean implement an interface?
  YES --> Spring uses JDK Dynamic Proxy by default
           (unless proxyTargetClass=true is forced)
  NO  --> Spring uses CGLIB
```

```java
// Force CGLIB even if interface exists:
@EnableAspectJAutoProxy(proxyTargetClass = true)

// Or in Spring Boot application.properties:
spring.aop.proxy-target-class=true
```

**Interview trap:** If you annotate a `final` method with `@Transactional` and your class doesn't implement an interface, Spring uses CGLIB but cannot override the final method — **the transaction annotation is silently ignored**.

```java
@Service
public class OrderService {  // no interface, Spring uses CGLIB

    @Transactional
    public final void placeOrder() {  // CGLIB can't override final!
        // Transaction will NOT be applied — silent bug
    }
}
```

---

## Trap 11: When Would You Use Chain of Responsibility Over Strategy?

- **Strategy**: one algorithm is selected and used; the alternatives don't get a chance.
- **Chain of Responsibility**: the request passes through multiple handlers in sequence until one (or more) handles it.

Use Chain of Responsibility for:
- HTTP middleware/filter chains (Spring `OncePerRequestFilter`)
- Support escalation (L1 → L2 → L3)
- Event processing pipelines where multiple handlers may process the same event

---

## Trap 12: Can You Use Multiple Design Patterns Together?

Yes, and real-world frameworks do this constantly:

- **Spring AOP** = Proxy + Decorator + Template Method
- **Spring MVC** = Front Controller + Strategy (HandlerMapping) + Template Method (DispatcherServlet)
- **Java I/O** = Decorator (streams) + Adapter (`InputStreamReader` adapts `InputStream` to `Reader`)
- **Hibernate** = Proxy (lazy loading) + Identity Map (a form of Flyweight) + Unit of Work (Command)
- **Guava Cache** = Builder + Strategy (eviction policy) + Proxy (stats/recording)
