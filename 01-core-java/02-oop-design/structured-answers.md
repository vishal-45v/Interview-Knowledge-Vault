# OOP Design — Structured Answers

---

## Q1: SOLID Principles with Java Examples

**S — Single Responsibility Principle:**
A class should have only one reason to change.

```java
// Violation: UserService does too many things
class UserService {
    void createUser(User user) { ... }
    void sendWelcomeEmail(User user) { ... }  // Email concern
    void generateUserReport(User user) { ... } // Reporting concern
    void saveToDatabase(User user) { ... }     // Persistence concern
}

// Correct: Each class has one responsibility
class UserService { void createUser(User user) { ... } }
class UserEmailService { void sendWelcomeEmail(User user) { ... } }
class UserReportService { Report generate(User user) { ... } }
class UserRepository { void save(User user) { ... } }
```

**O — Open/Closed Principle:**
Open for extension, closed for modification.

```java
// Violation: Adding discount types requires modifying PriceCalculator
class PriceCalculator {
    double calculate(Order order) {
        if (order.getType() == REGULAR) return order.getTotal();
        else if (order.getType() == VIP) return order.getTotal() * 0.9;
        // Adding PREMIUM requires modifying this class!
    }
}

// Correct: Strategy pattern — extend by adding new strategies
interface DiscountStrategy { double apply(double price); }
class RegularDiscount implements DiscountStrategy { ... }
class VipDiscount implements DiscountStrategy { double apply(double p) { return p * 0.9; } }
class PriceCalculator {
    double calculate(Order order, DiscountStrategy discount) {
        return discount.apply(order.getTotal());
    }
}
```

**L — Liskov Substitution Principle:**
Objects of a superclass should be replaceable with objects of subclasses.

```java
// Violation: Square breaks Rectangle contract
class Rectangle {
    void setWidth(int w) { this.width = w; }
    void setHeight(int h) { this.height = h; }
    int area() { return width * height; }
}
class Square extends Rectangle {
    void setWidth(int w) { this.width = w; this.height = w; }  // LSP violation!
    // Rectangle r = new Square(); r.setWidth(5); r.setHeight(3);
    // Expected area: 15, actual: 9 (Square overwrites height)
}
```

**I — Interface Segregation Principle:**
Clients should not depend on interfaces they don't use.

```java
// Violation: Fat interface
interface Worker {
    void work();
    void eat();    // Robots don't eat!
    void sleep();  // Robots don't sleep!
}

// Correct: Segregated interfaces
interface Workable { void work(); }
interface Feedable  { void eat(); }
interface Restable  { void sleep(); }
class Human implements Workable, Feedable, Restable { ... }
class Robot implements Workable { ... }
```

**D — Dependency Inversion Principle:**
Depend on abstractions, not concretions.

```java
// Violation
class OrderService {
    private MySqlOrderRepository repository = new MySqlOrderRepository();
    // Depends on concrete class — hard to test, hard to swap
}

// Correct
class OrderService {
    private final OrderRepository repository;  // Depend on abstraction
    public OrderService(OrderRepository repository) {
        this.repository = repository;  // Inject concrete via DI
    }
}
```

---

## Q2: Composition vs Inheritance — When to Choose Each

**Inheritance:**
- "IS-A" relationship
- Car IS-A Vehicle
- Square IS-A Shape (only if Liskov Substitution is maintained)
- Limited to class hierarchy — Java has single inheritance

**Composition:**
- "HAS-A" or "USES-A" relationship
- Car HAS-A Engine, HAS-A Transmission
- Preferred over inheritance for code reuse
- More flexible — can swap components at runtime

```java
// Inheritance for true IS-A:
abstract class Animal {
    abstract void makeSound();
}
class Dog extends Animal {
    void makeSound() { System.out.println("Woof"); }
}

// Composition for HAS-A:
class Car {
    private final Engine engine;        // HAS-A
    private final Transmission gearbox; // HAS-A
    private final GPS gps;              // HAS-A

    Car(Engine engine, Transmission gearbox, GPS gps) {
        this.engine = engine;
        this.gearbox = gearbox;
        this.gps = gps;
    }
    // Can swap different Engine implementations at construction time
}
```

---

## Q3: Abstract Class vs Interface — Decision Guide

**Use Abstract Class when:**
- Sharing code between related classes
- Non-public methods needed
- Some methods have default implementation
- Constructors needed in base
- Evolving base class (add non-abstract methods without breaking)

**Use Interface when:**
- Defining a contract that unrelated classes implement (Serializable, Comparable)
- Multiple "inheritance" needed (implement multiple interfaces)
- Defining callbacks or functional interfaces
- API surface only, no shared implementation

```java
// Abstract class: Template Method pattern
abstract class DataProcessor {
    // Template method — defines algorithm skeleton
    public final void process() {
        readData();      // Abstract — subclass implements
        transformData(); // Abstract
        writeData();     // Concrete — shared implementation
    }
    protected abstract void readData();
    protected abstract void transformData();
    private void writeData() { /* shared implementation */ }
}

// Interface: Contract
interface Sortable {
    int compareTo(Object other);
    default boolean lessThan(Object other) { return compareTo(other) < 0; }
}
```
