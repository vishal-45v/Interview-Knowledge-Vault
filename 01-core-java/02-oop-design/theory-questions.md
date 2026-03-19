# OOP Design — Theory Questions

> 28 theory questions covering object-oriented design principles, patterns, and Java-specific OOP features.

---

## Q1: What are the four pillars of OOP and how does Java implement them?

**Answer:**

**1. Encapsulation:** Binding data and methods together, hiding internal implementation.
```java
public class BankAccount {
    private double balance;  // hidden
    public void deposit(double amount) {
        if (amount > 0) balance += amount;  // controlled access
    }
}
```

**2. Abstraction:** Exposing essential behavior while hiding complexity.
```java
abstract class Shape {
    abstract double area();  // interface without implementation
    void print() { System.out.println("Area: " + area()); }
}
```

**3. Inheritance:** Reusing and extending behavior from parent classes.
```java
class Dog extends Animal {
    @Override void sound() { System.out.println("Woof"); }
}
```

**4. Polymorphism:** One interface, multiple implementations.
- Compile-time (method overloading)
- Runtime (method overriding via dynamic dispatch)

---

## Q2: What is the difference between an interface and an abstract class?

**Answer:**

| Feature | Interface | Abstract Class |
|---------|-----------|----------------|
| Multiple inheritance | Yes (multiple implements) | No (single extends) |
| Constructor | No | Yes |
| State (instance fields) | No (only static final) | Yes |
| Default implementation | Yes (default methods, Java 8+) | Yes |
| Access modifiers for methods | public (implicit) | Any |
| When to use | Define a contract/capability | Share code between related classes |

```java
// Interface — defines capability (HAS-A relationship)
interface Flyable { void fly(); }
interface Swimmable { void swim(); }

class Duck implements Flyable, Swimmable {
    public void fly() { ... }
    public void swim() { ... }
}

// Abstract class — shares code (IS-A relationship)
abstract class Vehicle {
    protected String brand;
    abstract void accelerate();
    void stop() { System.out.println("Brake applied"); }  // shared implementation
}
```

**Rule of thumb:** Use interface to define what a class can do. Use abstract class to share code between related classes.

---

## Q3: Explain the SOLID principles with Java examples.

**Answer:**

**S — Single Responsibility Principle:** A class should have only one reason to change.
```java
// Bad: UserService handles auth + email + persistence
class UserService {
    void saveUser(User u) { db.save(u); }
    void sendEmail(User u) { emailClient.send(u.getEmail()); }
    boolean authenticate(String password) { ... }
}

// Good: separate classes
class UserRepository { void save(User u) { db.save(u); } }
class EmailService { void sendWelcome(User u) { ... } }
class AuthenticationService { boolean authenticate(String password) { ... } }
```

**O — Open/Closed Principle:** Open for extension, closed for modification.
```java
// Extend behavior without modifying existing code:
interface DiscountStrategy { double apply(double price); }
class BlackFridayDiscount implements DiscountStrategy { ... }
class LoyaltyDiscount implements DiscountStrategy { ... }
// Add new discounts without changing Order class
```

**L — Liskov Substitution Principle:** Subtypes must be substitutable for their base types.
```java
// Violation: Square extends Rectangle but breaks area calculation
class Rectangle { void setWidth(int w); void setHeight(int h); }
class Square extends Rectangle {
    @Override void setWidth(int w) { super.setWidth(w); super.setHeight(w); } // breaks LSP!
}
```

**I — Interface Segregation Principle:** Don't force clients to depend on methods they don't use.
```java
// Bad: fat interface
interface Worker { void work(); void eat(); void sleep(); }
// Robot must implement eat() and sleep() even though it doesn't need them

// Good: segregated interfaces
interface Workable { void work(); }
interface Feedable { void eat(); }
interface Restable { void sleep(); }
```

**D — Dependency Inversion Principle:** Depend on abstractions, not concretions.
```java
// Bad: high-level depends on low-level
class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository();  // coupled!
}

// Good: depend on interface
class OrderService {
    private final OrderRepository repo;  // abstraction
    OrderService(OrderRepository repo) { this.repo = repo; }  // injected
}
```

---

## Q4: What is composition vs inheritance and when should you prefer each?

**Answer:**

**Inheritance (IS-A):** Dog IS-A Animal. Tight coupling between parent and child.
**Composition (HAS-A):** Car HAS-A Engine. Loosely coupled.

```java
// Inheritance — tight coupling:
class Stack extends Vector<Integer> {  // inherits all Vector methods including insertAt!
    // Stack users can call vector.insertAt(0, element) breaking stack invariants!
}

// Composition — loose coupling:
class Stack<T> {
    private final Deque<T> storage = new ArrayDeque<>();  // HAS-A deque
    void push(T item) { storage.push(item); }
    T pop() { return storage.pop(); }
    // Only exposes stack operations
}
```

**Prefer composition when:**
1. The "IS-A" relationship doesn't truly hold
2. You want to restrict the interface (only expose subset of behavior)
3. You need to change the implementation at runtime
4. You want to combine behaviors from multiple sources

**Inheritance is appropriate when:**
1. The subclass truly IS-A superclass (every Dog truly IS an Animal)
2. You want to benefit from polymorphism
3. You need to override and customize behavior for genuine specialization

---

## Q5: What is method overloading vs method overriding?

**Answer:**

**Overloading (compile-time polymorphism):**
- Same class, same method name, different parameters
- Resolved at compile time based on declared type
- Return type can differ (but not as sole distinguisher)

```java
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }
    int add(int a, int b, int c) { return a + b + c; }
    // int add(int a, int b) { return a + b; }  // ERROR — same signature
}
```

**Overriding (runtime polymorphism):**
- Subclass redefines a method from the parent
- Resolved at runtime based on actual object type
- Must have same signature (same name, same parameters)
- Return type can be covariant (subtype of parent's return type)
- Access modifier can be same or wider
- Checked exceptions can be same or narrower

```java
class Animal {
    Animal create() { return new Animal(); }  // covariant return allows:
    protected void makeSound() throws IOException { ... }
}
class Dog extends Animal {
    @Override
    Dog create() { return new Dog(); }  // covariant return — Dog IS-A Animal
    @Override
    public void makeSound() { }  // wider access, no checked exception (narrower)
}
```

---

## Q6: What is the diamond problem and how does Java handle it?

**Answer:**

The diamond problem occurs when a class inherits the same method from two paths:

```
        Interface A
        default hello()
       /               \
Interface B           Interface C
  default hello()       default hello()
       \               /
          Class D
          (which hello()?)
```

Java resolves this with explicit override or using `InterfaceName.super.method()`:

```java
interface A { default String hello() { return "A"; } }
interface B extends A { default String hello() { return "B"; } }
interface C extends A { default String hello() { return "C"; } }

class D implements B, C {
    @Override
    public String hello() {
        return B.super.hello();  // explicitly choose B's implementation
        // or C.super.hello() for C's
    }
}
```

Rules:
1. Class always wins over interface
2. More specific interface wins (B extends A, so B's default preferred over A's)
3. If still ambiguous, must explicitly override

---

## Q7: What are covariant return types?

**Answer:**

An overriding method can return a subtype of the parent method's return type.

```java
class Animal {
    Animal clone() { return new Animal(); }
}

class Dog extends Animal {
    @Override
    Dog clone() { return new Dog(); }  // covariant — Dog is-a Animal
    // Before Java 5, this would require Animal return type
}

// Useful in builder patterns:
class Builder {
    Builder withName(String name) { this.name = name; return this; }
}
class AdvancedBuilder extends Builder {
    @Override
    AdvancedBuilder withName(String name) { // covariant return
        super.withName(name); return this; // allows method chaining without cast
    }
    AdvancedBuilder withExtra(String extra) { ... return this; }
}
```

---

## Q8: What is constructor chaining in Java?

**Answer:**

Constructor chaining calls one constructor from another. `this()` calls another constructor in the same class, `super()` calls the parent constructor.

```java
class Person {
    String name;
    int age;
    String email;

    Person(String name) {
        this(name, 0);  // chains to Person(String, int)
    }

    Person(String name, int age) {
        this(name, age, null);  // chains to Person(String, int, String)
    }

    Person(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
        // Common initialization in one place
    }
}

class Employee extends Person {
    String department;

    Employee(String name, String department) {
        super(name);  // must be first statement
        this.department = department;
    }
}
```

**Rules:**
- `this()` or `super()` must be the FIRST statement in constructor
- Cannot have both `this()` and `super()` in same constructor
- Default constructor calls `super()` implicitly

---

## Q9: What is the difference between early (static) and late (dynamic) binding?

**Answer:**

**Static binding (early):** Method resolved at compile time. Applies to: static methods, private methods, final methods, constructors, overloaded methods.

**Dynamic binding (late):** Method resolved at runtime based on actual object type. Applies to: overridden instance methods.

```java
class Parent {
    static void staticMethod() { System.out.println("Parent static"); }
    void instanceMethod() { System.out.println("Parent instance"); }
}

class Child extends Parent {
    static void staticMethod() { System.out.println("Child static"); }
    @Override void instanceMethod() { System.out.println("Child instance"); }
}

Parent obj = new Child();
obj.staticMethod();   // "Parent static" — static binding, declared type wins
obj.instanceMethod(); // "Child instance" — dynamic binding, actual type wins
```

---

## Q10: What is encapsulation and why is it important?

**Answer:**

Encapsulation bundles data (fields) and behavior (methods) together, and restricts direct access to the data.

```java
public class Temperature {
    private double celsius;  // hidden state

    public Temperature(double celsius) {
        setCelsius(celsius);  // validation through setter
    }

    public double getCelsius() { return celsius; }
    public double getFahrenheit() { return celsius * 9.0 / 5.0 + 32; }

    public void setCelsius(double celsius) {
        if (celsius < -273.15) {
            throw new IllegalArgumentException("Below absolute zero");
        }
        this.celsius = celsius;
    }
}
```

**Benefits:**
1. **Data validation:** Enforce invariants in setters
2. **Change internal representation:** Change from celsius to kelvin internally without breaking callers
3. **Thread safety:** Synchronize on setters/getters
4. **Debugging:** Single point of access — easy to add logging
5. **Reduced coupling:** External code depends on the API, not the internal structure

---

## Q11: What are marker interfaces and when are they used?

**Answer:**

A marker interface has no methods — it "marks" a class as having a certain property.

```java
// Built-in marker interfaces:
public interface Serializable {}   // marks class as serializable
public interface Cloneable {}      // marks class as cloneable via Object.clone()
public interface RandomAccess {}   // marks List as supporting fast random access

// How they work — runtime check:
if (object instanceof Serializable) {
    serialize(object);  // safe to serialize
}

// Modern alternative — annotations (preferred in new code):
@Retention(RetentionPolicy.RUNTIME)
@interface Auditable { }

@Auditable
class UserEntity { ... }

// Runtime check via annotation:
if (entity.getClass().isAnnotationPresent(Auditable.class)) {
    auditLog(entity);
}
```

---

## Q12: Explain the concept of abstraction with a real-world example.

**Answer:**

Abstraction means hiding implementation details and showing only the essential features.

```java
// Real-world: JDBC abstraction
// You call the same methods regardless of the underlying database:
Connection conn = DriverManager.getConnection(url);  // MySQL, PostgreSQL, Oracle...
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
stmt.setInt(1, userId);
ResultSet rs = stmt.executeQuery();

// You don't need to know how MySQL or PostgreSQL execute the query internally.
// The abstraction (Connection, PreparedStatement, ResultSet interfaces)
// hides the complexity.

// Another example — Java Collections:
List<String> list = new ArrayList<>();  // or LinkedList, CopyOnWriteArrayList
list.add("item");      // same code regardless of List implementation
list.get(0);
list.remove(0);
```

---

## Q13: What is the Open/Closed Principle in practice?

**Answer:**

```java
// Before OCP — adding a new shape requires modifying AreaCalculator:
class AreaCalculator {
    double calculate(Object shape) {
        if (shape instanceof Circle c) return Math.PI * c.radius * c.radius;
        if (shape instanceof Rectangle r) return r.width * r.height;
        if (shape instanceof Triangle t) return 0.5 * t.base * t.height;
        // Every new shape requires modifying this method! VIOLATION
        throw new UnsupportedOperationException();
    }
}

// After OCP — closed for modification, open for extension:
interface Shape {
    double area();  // each shape knows its own area calculation
}
record Circle(double radius) implements Shape {
    public double area() { return Math.PI * radius * radius; }
}
record Rectangle(double width, double height) implements Shape {
    public double area() { return width * height; }
}
// Adding Triangle just requires a new class — no modifications to existing code!

class AreaCalculator {
    double calculate(Shape shape) { return shape.area(); }  // never needs modification
}
```

---

## Q14: What is the Liskov Substitution Principle and what happens when it's violated?

**Answer:**

LSP: If S is a subtype of T, then objects of type T may be replaced with objects of type S without breaking program correctness.

**Classic violation — Square extends Rectangle:**
```java
class Rectangle {
    int width, height;
    void setWidth(int w) { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}

class Square extends Rectangle {
    @Override void setWidth(int side) { this.width = this.height = side; }   // LSP violation!
    @Override void setHeight(int side) { this.width = this.height = side; }
}

// Client code that works with Rectangle breaks with Square:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(3);
System.out.println(r.area());  // Expected: 15, Got: 9 (Square made both 3)
```

**Fix:** Use separate classes without inheritance, or use composition.

---

## Q15: What are default methods in interfaces and what problem do they solve?

**Answer:**

Default methods (Java 8+) allow interfaces to have concrete implementations. This solves backward compatibility — you can add new methods to existing interfaces without breaking all implementing classes.

```java
interface Collection<E> {
    // Adding forEach() in Java 8 without default method would break
    // every class implementing Collection!
    default void forEach(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        for (E e : this) {
            action.accept(e);
        }
    }
}

// Custom interface with default method:
interface Sortable<T extends Comparable<T>> {
    List<T> getItems();

    default List<T> getSorted() {  // default implementation
        return getItems().stream().sorted().collect(Collectors.toList());
    }

    default T getMin() {
        return getItems().stream().min(Comparator.naturalOrder()).orElseThrow();
    }
}
```

---

## Q16: What is the difference between abstraction and encapsulation?

**Answer:**

These are related but distinct concepts:

**Abstraction** is about DESIGN — what to expose. "Show only what's necessary."
- Happens at the interface/abstract class level
- Example: `List` interface hides whether it's an ArrayList or LinkedList

**Encapsulation** is about IMPLEMENTATION — how to hide. "Bundle and protect data."
- Happens at the class level with access modifiers
- Example: private fields with controlled getter/setter access

```
Abstraction: "Push the button" — you don't need to know what happens inside
Encapsulation: The machine keeps its internal state hidden from you
```

---

## Q17: Can a class implement multiple interfaces? What are the rules?

**Answer:**

Yes, a class can implement any number of interfaces.

```java
class PrintablePdfDocument implements Printable, Saveable, Serializable, Closeable {
    @Override public void print() { ... }
    @Override public void save(Path path) { ... }
    // Serializable is a marker interface — no methods
    @Override public void close() { ... }
}
```

**Rules:**
1. Must implement ALL abstract methods from all interfaces
2. If two interfaces have the same default method, must override it
3. Cannot implement two interfaces with same method signature but different checked exceptions
4. Cannot implement two interfaces if same constant name conflicts

---

## Q18: What is the interface segregation principle with a real-world Spring example?

**Answer:**

ISP: Clients should not be forced to depend on interfaces they do not use.

```java
// Violation — fat repository interface:
interface UserRepository {
    User findById(Long id);
    List<User> findAll();
    void save(User user);
    void delete(Long id);
    List<User> runNativeQuery(String sql);  // admin only!
    void truncate();  // admin only!
}

// Now every class that needs just findById must have access to truncate()

// Better — segregated interfaces:
interface UserReader {
    User findById(Long id);
    List<User> findAll();
    List<User> findByEmail(String email);
}

interface UserWriter {
    User save(User user);
    void delete(Long id);
}

interface UserAdminOperations extends UserReader, UserWriter {
    void truncate();
    List<User> runNativeQuery(String sql);
}

// Spring example:
interface UserReadService {
    UserDTO getById(Long id);  // read-only service for API endpoints
}

@Service
class UserReadServiceImpl implements UserReadService {
    UserReadServiceImpl(UserRepository repo) { ... }
    // Only depends on read operations
}
```

---

## Q19: What is the difference between `super` keyword uses?

**Answer:**

```java
class Animal {
    String name;
    Animal(String name) { this.name = name; }
    void sound() { System.out.println("..."); }
}

class Dog extends Animal {
    String breed;

    Dog(String name, String breed) {
        super(name);  // 1. Call parent constructor — MUST be first statement
        this.breed = breed;
    }

    @Override
    void sound() {
        super.sound();  // 2. Call parent method — can be anywhere
        System.out.println("Woof!");
    }

    String display() {
        return super.name + " (" + breed + ")";  // 3. Access parent field
        // (but private fields cannot be accessed this way)
    }
}
```

---

## Q20: What is a functional interface?

**Answer:**

A functional interface has exactly ONE abstract method. It can have any number of default or static methods. Enables lambda expressions.

```java
@FunctionalInterface  // optional but recommended — compiler checks
interface Transformer<T, R> {
    R transform(T input);  // exactly one abstract method
    default Transformer<T, R> andThen(Consumer<R> after) { ... }  // default OK
    static <T> Transformer<T, T> identity() { return t -> t; }   // static OK
}

// Lambda usage:
Transformer<String, Integer> length = s -> s.length();
Transformer<String, String> upper = String::toUpperCase;  // method reference

// Built-in functional interfaces (java.util.function):
// Function<T,R>     — takes T, returns R
// Consumer<T>       — takes T, returns void
// Supplier<T>       — takes nothing, returns T
// Predicate<T>      — takes T, returns boolean
// BiFunction<T,U,R> — takes T and U, returns R
// UnaryOperator<T>  — takes T, returns T
// BinaryOperator<T> — takes T and T, returns T
```

---

## Q21: What is the `instanceof` pattern matching and how does it relate to OOP?

**Answer:**

Java 16+ pattern matching for instanceof reduces boilerplate in polymorphic code:

```java
// Before — verbose:
void describe(Shape shape) {
    if (shape instanceof Circle) {
        Circle c = (Circle) shape;
        System.out.println("Circle with radius " + c.radius());
    } else if (shape instanceof Rectangle) {
        Rectangle r = (Rectangle) shape;
        System.out.println("Rectangle " + r.width() + "x" + r.height());
    }
}

// After — pattern matching:
void describe(Shape shape) {
    if (shape instanceof Circle c) {
        System.out.println("Circle with radius " + c.radius());
    } else if (shape instanceof Rectangle r) {
        System.out.println("Rectangle " + r.width() + "x" + r.height());
    }
}

// Even better — switch expression (Java 21+):
String description = switch (shape) {
    case Circle c    -> "Circle with radius " + c.radius();
    case Rectangle r -> "Rectangle " + r.width() + "x" + r.height();
    case Triangle t  -> "Triangle";
};
```

---

## Q22: What are the rules for abstract classes?

**Answer:**

```java
abstract class AbstractValidator<T> {
    // CAN have:
    private String name;                    // instance fields
    protected abstract void validate(T t);  // abstract methods (no body)
    public void log(String msg) { ... }     // concrete methods
    static void staticHelper() { ... }      // static methods
    AbstractValidator(String name) { ... }  // constructors

    // CANNOT:
    // abstract static void method(); — static methods can't be abstract
    // new AbstractValidator(); — cannot instantiate directly
}

// Rules:
// 1. Cannot instantiate abstract class directly
// 2. Concrete subclass MUST implement all abstract methods
//    (or also be abstract)
// 3. Can have any mix of abstract and concrete methods
// 4. Can have constructors (called via super() from subclass)
// 5. Can have state (fields)
```

---

## Q23: What is the difference between `this` and `super` references?

**Answer:**

- `this` refers to the current object instance
- `super` refers to the parent class

```java
class Vehicle {
    String type = "vehicle";
    void describe() { System.out.println("I am a vehicle"); }
}

class Car extends Vehicle {
    String type = "car";  // shadows parent field

    void showType() {
        System.out.println(this.type);  // "car" — current object's field
        System.out.println(super.type); // "vehicle" — parent's field
    }

    @Override
    void describe() {
        super.describe();  // call parent's describe()
        System.out.println("Specifically, I am a car");
    }

    Car() {
        super();  // call parent constructor
        // this() would call another Car constructor
    }
}
```

---

## Q24: What is a nested class vs inner class vs static nested class?

**Answer:**

```java
class Outer {
    private int x = 10;

    // Static nested class — doesn't need Outer instance, can't access Outer's instance members
    static class StaticNested {
        void doSomething() { /* no access to Outer.x */ }
    }

    // Inner class (non-static) — needs Outer instance, can access all Outer members
    class Inner {
        void doSomething() { System.out.println(x); }  // accesses Outer.x
    }

    void method() {
        // Local class — defined inside a method
        class Local { void run() { System.out.println(x); } }
        new Local().run();

        // Anonymous class — ad-hoc implementation
        Runnable r = new Runnable() {
            @Override public void run() { System.out.println(x); }
        };
    }
}

// Instantiation:
Outer.StaticNested sn = new Outer.StaticNested();  // no Outer needed
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();  // needs Outer instance
```

---

## Q25: What is the Dependency Inversion Principle in Spring Boot?

**Answer:**

DIP says: high-level modules should not depend on low-level modules — both should depend on abstractions.

Spring Boot's IoC container embodies DIP:

```java
// High-level module depends on abstraction:
@Service
public class OrderService {
    private final OrderRepository repository;  // interface, not implementation
    private final PaymentGateway paymentGateway;  // interface

    public OrderService(OrderRepository repository, PaymentGateway paymentGateway) {
        this.repository = repository;
        this.paymentGateway = paymentGateway;
    }
}

// Low-level modules implement the abstractions:
@Repository
public class JpaOrderRepository implements OrderRepository { ... }

@Component
public class StripePaymentGateway implements PaymentGateway { ... }

// Spring wires them together — OrderService doesn't know about JPA or Stripe
```

This enables easy testing:
```java
@Test
void testPlaceOrder() {
    OrderRepository mockRepo = mock(OrderRepository.class);
    PaymentGateway mockPayment = mock(PaymentGateway.class);
    OrderService service = new OrderService(mockRepo, mockPayment);
    // No Spring context, no database, no payment network needed
}
```

---

## Q26: What is method hiding in Java?

**Answer:**

Static methods are hidden, not overridden. If a subclass defines a static method with the same signature as a parent's static method, the subclass method hides the parent's.

```java
class Parent {
    static void hello() { System.out.println("Parent static"); }
    void world() { System.out.println("Parent instance"); }
}

class Child extends Parent {
    static void hello() { System.out.println("Child static"); }  // hiding
    @Override void world() { System.out.println("Child instance"); }  // overriding
}

Parent p = new Child();
p.hello();  // "Parent static" — reference type decides (hiding is static binding)
p.world();  // "Child instance" — object type decides (overriding is dynamic binding)

Child.hello();  // "Child static"
Parent.hello(); // "Parent static"
```

---

## Q27: What is the role of the `Object` class?

**Answer:**

`Object` is the root of the Java class hierarchy. Every class implicitly extends `Object`.

Key methods from Object:
```java
// Equality and identity:
boolean equals(Object obj)       // value equality — must override with hashCode
int hashCode()                   // integer hash — must override with equals
String toString()                // string representation

// Cloning:
Object clone()                   // protected — override in your class

// Threading:
void wait()                      // release lock and wait for notify
void wait(long timeout)
void notify()                    // wake one waiting thread
void notifyAll()                 // wake all waiting threads

// Other:
Class<?> getClass()              // get runtime class
void finalize()                  // deprecated — don't use

// Usage:
@Override
public String toString() {
    return "Person{name='" + name + "', age=" + age + "}";
}
```

---

## Q28: What is the difference between strong, weak, soft, and phantom references?

**Answer:**

Java has four types of object references that interact with GC differently:

```java
// Strong reference — GC never collects while strongly reachable:
String strong = "hello";  // standard reference

// Soft reference — GC collects ONLY when JVM needs memory (pre-OOM):
SoftReference<byte[]> soft = new SoftReference<>(new byte[1024 * 1024]);
byte[] data = soft.get();  // may be null if GC'd
// Use: memory-sensitive caches

// Weak reference — GC collects whenever it runs (no strong reference):
WeakReference<User> weak = new WeakReference<>(user);
User u = weak.get();  // may be null
// Use: WeakHashMap, canonicalization

// Phantom reference — GC collects, but get() always returns null:
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantom = new PhantomReference<>(obj, queue);
// obj is GC'd → phantom is enqueued in queue
// Use: post-GC cleanup actions (replacing finalizers)
```

**Strength order:** Strong > Soft > Weak > Phantom

An object is collected when it is only reachable through references of a given type or weaker.
