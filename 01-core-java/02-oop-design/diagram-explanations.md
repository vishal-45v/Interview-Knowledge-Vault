# OOP Design — Diagram Explanations

---

## 1. Class Hierarchy (Inheritance)

```
          ┌─────────────┐
          │   Animal    │  ← Parent / Base class
          │─────────────│
          │ + name      │
          │ + eat()     │
          │ + breathe() │
          └──────┬──────┘
                 │  extends
        ┌────────┴────────┐
        │                 │
┌───────▼──────┐   ┌──────▼───────┐
│     Dog      │   │     Cat      │
│──────────────│   │──────────────│
│ + breed      │   │ + indoor     │
│ + bark()     │   │ + meow()     │
└──────────────┘   └──────────────┘
```

---

## 2. Interface Implementation

```
         ┌──────────────────┐
         │   <<interface>>  │
         │     Flyable      │
         │──────────────────│
         │ + fly(): void    │
         │ + land(): void   │
         └──────────────────┘
                  ▲
         implements
        ┌──────────┴──────────┐
        │                     │
┌───────┴──────┐     ┌────────┴─────┐
│    Bird      │     │  Airplane    │
│──────────────│     │──────────────│
│ + fly()      │     │ + fly()      │
│ + land()     │     │ + land()     │
└──────────────┘     └──────────────┘
```

---

## 3. Abstract Class vs Concrete Class

```
    ┌───────────────────────────┐
    │  <<abstract>>             │
    │       Shape               │
    │───────────────────────────│
    │ + color: String           │
    │ + getColor(): String      │
    │ + draw(): void  [abstract]│  ← No implementation
    └──────────────┬────────────┘
                   │
          ┌────────┴────────┐
          │                 │
  ┌───────▼──────┐   ┌──────▼───────┐
  │   Circle     │   │  Rectangle   │
  │──────────────│   │──────────────│
  │ + radius     │   │ + width      │
  │ + draw() ✓   │   │ + height     │
  └──────────────┘   │ + draw() ✓   │
                     └──────────────┘
```

---

## 4. Polymorphism at Runtime

```
  Shape shape;

  ┌──────────────────────────┐
  │  shape = new Circle();   │ ──→  circle.draw() called
  │  shape.draw();           │
  └──────────────────────────┘

  ┌──────────────────────────┐
  │  shape = new Rectangle();│ ──→  rectangle.draw() called
  │  shape.draw();           │
  └──────────────────────────┘

  Same variable type (Shape)
  Different runtime behavior ✓
```

---

## 5. Composition vs Inheritance

```
  ❌ INHERITANCE (wrong approach):
  ┌─────────────┐
  │     Car     │
  └──────┬──────┘
         │ extends
  ┌──────▼──────┐
  │   Engine    │  ← Car IS-AN Engine? ❌ Wrong!
  └─────────────┘

  ✅ COMPOSITION (correct approach):
  ┌────────────────────────────────────────┐
  │               Car                      │
  │  ┌──────────┐  ┌──────────┐            │
  │  │  Engine  │  │  Wheels  │  ← Has-A  │
  │  └──────────┘  └──────────┘            │
  │  ┌──────────────────────────┐           │
  │  │      SteeringWheel       │           │
  │  └──────────────────────────┘           │
  └────────────────────────────────────────┘
```

---

## 6. SOLID — Dependency Inversion Principle

```
  ❌ WITHOUT DIP (tightly coupled):
  ┌─────────────────┐        ┌──────────────────┐
  │  OrderService   │───────▶│  MySQLDatabase   │
  └─────────────────┘        └──────────────────┘
  (OrderService depends on concrete class)

  ✅ WITH DIP (loosely coupled):
  ┌─────────────────┐        ┌──────────────────┐
  │  OrderService   │───────▶│  <<interface>>   │
  └─────────────────┘        │   IDatabase      │
                             └────────┬─────────┘
                                      │ implements
                           ┌──────────┴──────────┐
                           │                     │
                  ┌────────▼────┐      ┌──────────▼──┐
                  │  MySQLDB    │      │  MongoDBDB  │
                  └─────────────┘      └─────────────┘
```

---

## 7. Method Resolution Order (Polymorphism)

```
  Compile time:
  ┌────────────────────────────────────┐
  │  Animal a = new Dog();             │
  │  a.speak();  ← checked on Animal   │
  └────────────────────────────────────┘

  Runtime:
  ┌────────────────────────────────────┐
  │  JVM sees: actual object is Dog    │
  │  Calls: Dog.speak()  ← overridden  │
  └────────────────────────────────────┘

  ┌──────────────────────────────────────────────┐
  │  METHOD TABLE (vtable)                       │
  │                                              │
  │  Animal.speak() ──────────▶ "Generic sound"  │
  │  Dog.speak()    ──────────▶ "Woof!"         │
  │  Cat.speak()    ──────────▶ "Meow!"         │
  └──────────────────────────────────────────────┘
```

---

## 8. Encapsulation — Access Control

```
  ┌──────────────────────────────────────────┐
  │              BankAccount                 │
  │──────────────────────────────────────────│
  │ - balance: double        ← private       │
  │ - accountNumber: String  ← private       │
  │──────────────────────────────────────────│
  │ + getBalance(): double   ← public        │
  │ + deposit(amount)        ← public        │
  │ + withdraw(amount)       ← public        │
  │ - validateAmount(amount) ← private       │
  └──────────────────────────────────────────┘

  External code:
  account.balance = -1000;  ❌ Compile error (private)
  account.deposit(100);     ✅ OK (public method)
```

---

## 9. Constructor Chaining

```
  class Person {

      ┌──────────────────────────────────────┐
      │  Person(name, age, email)            │
      │  ← Most complete constructor         │
      └────────────────┬─────────────────────┘
                       ▲ this(name, age, "none")
      ┌────────────────┴─────────────────────┐
      │  Person(name, age)                   │
      │  ← Delegates to full constructor     │
      └────────────────┬─────────────────────┘
                       ▲ this(name, 0)
      ┌────────────────┴─────────────────────┐
      │  Person(name)                        │
      │  ← Delegates with defaults           │
      └──────────────────────────────────────┘
```

---

## 10. Interface Default Methods (Java 8+)

```
  ┌────────────────────────────────────┐
  │  interface Vehicle                 │
  │────────────────────────────────────│
  │ + start(): void  [abstract]        │
  │ + stop(): void   [abstract]        │
  │ + describe(): String [default] ─── │──→ "I am a vehicle"
  └───────────────┬────────────────────┘
                  │
         ┌────────┴────────┐
         │                 │
  ┌──────▼──────┐   ┌──────▼──────┐
  │    Car      │   │    Bike     │
  │  + start()  │   │  + start()  │
  │  + stop()   │   │  + stop()   │
  │             │   │  + describe()│──→ Override default
  └─────────────┘   └─────────────┘
```
