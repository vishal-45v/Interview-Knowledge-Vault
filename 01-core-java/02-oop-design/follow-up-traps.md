# OOP Design — Follow-up Trap Questions

> These are tricky follow-up questions interviewers ask after you answer the basics. Know these cold.

---

## Trap 1: "Can an abstract class have a constructor?"

**What most people say:** "No, abstract classes can't be instantiated so they can't have constructors."

**Correct answer:** YES, abstract classes CAN have constructors. The constructor is called when a concrete subclass is instantiated via `super()`. It's used to initialize fields defined in the abstract class.

```java
abstract class Animal {
    private String name;

    // ✅ This is valid
    public Animal(String name) {
        this.name = name;
    }
}

class Dog extends Animal {
    public Dog(String name) {
        super(name); // calls Animal's constructor
    }
}
```

---

## Trap 2: "Can an interface have instance variables?"

**What most people say:** "Yes, interfaces can have variables."

**Correct answer:** Interfaces can only have `public static final` (constant) fields. They cannot have instance variables. Any field declared in an interface is implicitly `public static final`.

```java
interface Config {
    int MAX_RETRY = 3; // implicitly: public static final int MAX_RETRY = 3
}
```

---

## Trap 3: "What's the difference between method overloading and overriding in terms of polymorphism?"

**Trap:** Interviewers expect you to know the TYPE of polymorphism each represents.

**Answer:**
- **Overloading** = Compile-time (static) polymorphism. Resolved by the compiler based on method signature.
- **Overriding** = Runtime (dynamic) polymorphism. Resolved by the JVM at runtime based on actual object type.

---

## Trap 4: "Can you override a private method?"

**What most people say:** "No, private methods can't be overridden."

**Correct nuanced answer:** Private methods cannot be overridden — they are not inherited. If a subclass defines a method with the same name, it is a NEW method, not an override. The `@Override` annotation would cause a compile error.

```java
class Parent {
    private void secret() { System.out.println("Parent secret"); }
    public void callSecret() { secret(); } // calls Parent's version
}

class Child extends Parent {
    private void secret() { System.out.println("Child secret"); } // NOT an override
}

new Child().callSecret(); // prints "Parent secret" — NOT "Child secret"
```

---

## Trap 5: "Can a class implement two interfaces that have the same default method?"

**What most people say:** "It would work fine."

**Correct answer:** It causes a **compile-time error** unless the implementing class overrides the conflicting default method to resolve the ambiguity.

```java
interface A { default void greet() { System.out.println("A"); } }
interface B { default void greet() { System.out.println("B"); } }

class C implements A, B {
    @Override
    public void greet() { A.super.greet(); } // must explicitly resolve
}
```

---

## Trap 6: "Is Java purely object-oriented?"

**Expected answer:** NO. Java is not purely OO because:
1. It has **primitive types** (`int`, `boolean`, etc.) that are not objects.
2. Static methods/fields exist outside of object instances.
3. `null` is not an object.

Purely OO languages (like Smalltalk) treat everything as objects.

---

## Trap 7: "What is the diamond problem in Java and how does Java solve it?"

**Answer:** The diamond problem occurs when a class inherits from two classes that both inherit from a common base class — creating ambiguity about which method to use. Java avoids this by **not allowing multiple class inheritance**. However, with default methods in interfaces (Java 8), a similar problem can occur and is resolved by forcing the implementing class to override the conflicting method.

---

## Trap 8: "Can a final class be extended? Can a final method be overridden?"

**Answer:**
- A `final` class **cannot** be extended (e.g., `String`, `Integer`).
- A `final` method **cannot** be overridden by subclasses.
- A `final` variable **cannot** be reassigned after initialization.

---

## Trap 9: "What is the difference between IS-A and HAS-A relationships?"

**Answer:**
- **IS-A (Inheritance):** `Dog IS-A Animal`. Use when subclass is a specialized version of the parent.
- **HAS-A (Composition):** `Car HAS-AN Engine`. Use when a class contains another class as a component.

**Follow-up trap:** "When would you choose composition over inheritance?"
- When behavior changes at runtime
- When you want to avoid tight coupling
- When the IS-A relationship doesn't truly hold (e.g., a `Stack` should NOT extend `Vector`)

---

## Trap 10: "What is object slicing? Does it happen in Java?"

**Answer:** Object slicing occurs in C++ when a derived class object is assigned to a base class variable — the derived class's extra data is "sliced off." In Java, this does NOT happen because:
- Java uses references, not value copies.
- The object itself is not copied; only the reference type changes.
- The actual object retains all its data (but you can only access base class methods through a base reference).

---

## Trap 11: "Can constructors be inherited?"

**Answer:** No. Constructors are NOT inherited in Java. Each class must define its own constructors. However, a subclass constructor can call the parent constructor using `super()`. If `super()` is not explicitly called, Java implicitly calls the no-arg parent constructor.

---

## Trap 12: "What is covariant return type?"

**Answer:** A method override can return a subtype of the return type declared in the parent method.

```java
class Animal {
    Animal create() { return new Animal(); }
}

class Dog extends Animal {
    @Override
    Dog create() { return new Dog(); } // ✅ Covariant — Dog is subtype of Animal
}
```

This was introduced in Java 5 and allows more specific return types without casting.

---

## Trap 13: "Explain the difference between aggregation and composition."

**Answer:**
Both are HAS-A relationships, but differ in lifecycle dependency:

- **Composition (strong):** The contained object cannot exist without the container. If `House` is destroyed, its `Room` objects are also destroyed. Rooms don't exist independently.
- **Aggregation (weak):** The contained object can exist independently. A `University` HAS `Students`, but students can exist without the university.

---

## Trap 14: "What problem does the Liskov Substitution Principle solve?"

**Answer:** LSP ensures that inheritance is used correctly — that a subclass truly IS-A parent class in behavior, not just in type. Violating LSP means subtypes change expected behavior, breaking code that uses the parent type.

**Classic violation example:**
```java
class Rectangle { setWidth(w), setHeight(h) }
class Square extends Rectangle { // violates LSP
    // A square must keep width == height, but that breaks Rectangle's contract
}
```
