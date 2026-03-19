# Chapter 08: Advanced Java — Follow-Up Traps

These are the tricky questions interviewers ask after you give a good basic answer. Each one has a clear trap and the correct explanation.

---

## Trap 1: Type Erasure and instanceof with Generics

**Interviewer:** "Can you use `instanceof` to check if something is a `List<String>`?"

**The Trap:** Most candidates say yes without thinking.

**Answer:** No. Due to type erasure, `List<String>` does not exist at runtime — only `List` exists. The compiler will reject `instanceof List<String>` as an error.

```java
List<String> list = new ArrayList<>();

// COMPILE ERROR — illegal generic type for instanceof
// if (list instanceof List<String>) { }

// Correct — check raw type only
if (list instanceof List<?>) {
    System.out.println("It's a List");  // this works
}

// Java 16+ pattern matching — still erased
if (list instanceof List<?> l) {
    // Can cast elements, but type argument is unknown at runtime
}

// What about checking element type? You'd have to check the elements:
boolean isStringList = list.stream().allMatch(e -> e instanceof String);
```

---

## Trap 2: Cannot Create a Generic Array

**Interviewer:** "Why can't you do `new T[10]` in a generic class?"

**The Trap:** Candidates often say "generics don't work with arrays" but can't explain why.

**Answer:** Array creation requires a concrete type at runtime (for `ArrayStoreException` safety). Because `T` is erased to `Object` at runtime, `new T[10]` would create `Object[]`, which would be unsound — the compiler prevents this.

```java
public class Box<T> {
    // T[] items = new T[10]; // COMPILE ERROR — generic array creation

    // Workaround 1: use Object[] and cast
    @SuppressWarnings("unchecked")
    private T[] items = (T[]) new Object[10]; // works, but heap pollution warning

    // Workaround 2: require a Class<T> token
    private T[] items2;
    public Box(Class<T> clazz, int size) {
        items2 = (T[]) java.lang.reflect.Array.newInstance(clazz, size);
    }

    // Why arrays care: arrays are covariant and check at runtime
    Object[] strings = new String[3];
    strings[0] = 42; // ArrayStoreException at runtime — arrays defend themselves
    // Generics are invariant — List<String> is NOT a List<Object>
}
```

---

## Trap 3: Wildcard PECS Rule — Which Way Does it Go?

**Interviewer:** "If I have a method that both reads and writes to a list, which wildcard should I use?"

**The Trap:** Candidates try to use a wildcard when they should not.

**Answer:** If you need both read and write with a specific type, use no wildcard — use the concrete type parameter `<T>`. Wildcards are for one-direction operations.

```java
// WRONG — trying to use wildcard for both read and write
public <T> void swap(List<? extends T> list, int i, int j) {
    T temp = list.get(i);
    list.set(j, temp); // COMPILE ERROR — can't write to ? extends T
}

// CORRECT — use T directly for read+write
public <T> void swap(List<T> list, int i, int j) {
    T temp = list.get(i);
    list.set(j, list.get(i)); // fine
    list.set(i, temp);        // fine
}

// Memory trick: PECS
// Collections.copy(List<? super T> dst, List<? extends T> src)
// dst is Consumer (super), src is Producer (extends)
```

---

## Trap 4: Annotation Retention Policies — What is the Default?

**Interviewer:** "If you define a custom annotation and don't specify `@Retention`, can you read it at runtime?"

**The Trap:** Many assume the default is RUNTIME.

**Answer:** The default retention is `CLASS` — the annotation is stored in the `.class` file but NOT available via reflection at runtime. To read it at runtime, you must explicitly declare `@Retention(RetentionPolicy.RUNTIME)`.

```java
// Default retention — CLASS, NOT readable at runtime
public @interface MyAnnotation {
    String value();
}

@MyAnnotation("test")
public class MyClass {}

// At runtime:
MyAnnotation ann = MyClass.class.getAnnotation(MyAnnotation.class);
System.out.println(ann); // null! Annotation not available at runtime

// Fix: explicitly declare RUNTIME retention
@Retention(RetentionPolicy.RUNTIME)
public @interface MyRuntimeAnnotation {
    String value();
}
```

---

## Trap 5: Reflection Breaking Encapsulation — Is setAccessible Always Allowed?

**Interviewer:** "Can you always call `setAccessible(true)` to access private fields?"

**The Trap:** In pre-Java-9 code, yes — but from Java 9+ with modules, this can fail.

**Answer:** In Java 9+, if the class belongs to a module that does not `open` its package, `setAccessible(true)` throws `InaccessibleObjectException`. The module system enforces encapsulation even against reflection.

```java
// Java 8 and earlier — setAccessible(true) always worked
Field f = String.class.getDeclaredField("value");
f.setAccessible(true); // worked fine in Java 8

// Java 9+ without --add-opens — throws InaccessibleObjectException
// java.lang is in java.base module, not opened to unnamed module

// Fix: command line flag (not recommended for production)
// --add-opens java.base/java.lang=ALL-UNNAMED

// Fix: open your own packages in module-info.java
// opens com.myapp.internal to testing.framework;

// Checking before accessing
try {
    f.setAccessible(true);
} catch (InaccessibleObjectException e) {
    System.out.println("Module won't allow access: " + e.getMessage());
}
```

---

## Trap 6: ClassCastException with Generics at Runtime

**Interviewer:** "Can you get a ClassCastException from a generic collection if you never cast anything?"

**The Trap:** Candidates who understand type erasure know casts disappear — but they forget about heap pollution.

**Answer:** Yes — if you use raw types or unchecked casts to bypass the generic system ("heap pollution"), a ClassCastException can appear at a line with no visible cast. The compiler-inserted checkcast at the point of use will fail.

```java
// Heap pollution — inserting wrong type via raw type
List<String> strings = new ArrayList<>();
List raw = strings;     // raw type — unchecked warning
raw.add(42);            // bypasses generic check — Integer goes into List<String>

// ClassCastException happens HERE, not where you added the Integer
String s = strings.get(0); // ClassCastException: Integer cannot be cast to String
// The cast is compiler-inserted — there's no explicit (String) in your source

// @SafeVarargs helps document heap-pollution-free varargs
@SafeVarargs
public static <T> List<T> listOf(T... elements) {
    return Arrays.asList(elements);
}
```

---

## Trap 7: Dynamic Proxy Limitations — What Can't It Proxy?

**Interviewer:** "A class does not implement any interface. Can you create a JDK dynamic proxy for it?"

**The Trap:** Candidates say "yes, use `Proxy.newProxyInstance`" without knowing the interface constraint.

**Answer:** No. `java.lang.reflect.Proxy` can only create proxies for interfaces. If you need to proxy a concrete class, you need CGLIB (which subclasses the class) or ByteBuddy. Spring AOP uses CGLIB automatically when no interface is available.

```java
// JDK Proxy — ONLY works with interfaces
interface Service { void doWork(); }
class ServiceImpl implements Service { public void doWork() { System.out.println("work"); } }

Service proxy = (Service) Proxy.newProxyInstance(
    Service.class.getClassLoader(),
    new Class[]{Service.class},    // ← must be interfaces
    handler
);

// CGLIB — works with concrete classes (used by Spring when no interface)
// Enhancer enhancer = new Enhancer();
// enhancer.setSuperclass(ServiceImpl.class);   // subclasses it
// enhancer.setCallback(new MethodInterceptor() { ... });
// ServiceImpl proxy = (ServiceImpl) enhancer.create();

// TRAP: final classes and final methods CANNOT be proxied by CGLIB
// because CGLIB subclasses, and you can't override final methods
public final class CannotProxy { }  // No proxy possible with either approach
```

---

## Trap 8: Module System and Reflection — The opens Keyword

**Interviewer:** "Your Spring app breaks with `InaccessibleObjectException` after upgrading to Java 17. What's happening?"

**The Trap:** Candidates blame Spring or the JVM version without understanding the module boundary.

**Answer:** Spring uses reflection extensively for dependency injection. In Java 9+, modules are encapsulated by default. Your class's package must be `open`ed to the framework module (or `ALL-UNNAMED` for classpath apps) for Spring to inject into private fields.

```java
// module-info.java in a modular Spring app
module com.myapp {
    requires spring.core;
    requires spring.context;
    requires spring.beans;

    // Must open packages so Spring can inject via reflection
    opens com.myapp.service to spring.beans;
    opens com.myapp.controller to spring.beans;

    // For non-modular Spring (classpath) — open to all unnamed modules
    opens com.myapp.model;  // opens to ALL-UNNAMED implicitly for classpath
}

// Without opens, Spring's field injection fails:
// InaccessibleObjectException: Unable to make field private String
//   com.myapp.service.UserService.userRepository accessible:
//   module com.myapp does not 'opens com.myapp.service' to module spring.beans
```

---

## Trap 9: Record Limitations — Cannot Extend, and the "Mutable" Trick

**Interviewer:** "Records are immutable, right? Can you make a mutable record?"

**The Trap:** Candidates say "records are always immutable" — but that is only half true.

**Answer:** Record components are final — you cannot reassign them. But if a component is a mutable object (like a `List`), you can mutate that object's contents. Records cannot extend other classes (they implicitly extend `java.lang.Record`), and no class can extend a record.

```java
// Component is final — cannot reassign
public record Point(int x, int y) {}
Point p = new Point(1, 2);
// p.x = 5; // COMPILE ERROR — x is final

// But mutable components — the "immutable" label is misleading
public record Container(List<String> items) {}
Container c = new Container(new ArrayList<>(List.of("a", "b")));
c.items().add("c"); // Works! The reference is final, not the list contents
System.out.println(c.items()); // [a, b, c]

// To truly protect it:
public record SafeContainer(List<String> items) {
    public SafeContainer {
        items = List.copyOf(items); // defensive copy in compact constructor
    }
}

// Cannot extend:
// class Special extends Point {} // COMPILE ERROR
// class NotAllowed extends Record {} // COMPILE ERROR

// Records CAN implement interfaces
public record Timestamped(String data, long timestamp) implements Comparable<Timestamped> {
    @Override
    public int compareTo(Timestamped other) {
        return Long.compare(this.timestamp, other.timestamp);
    }
}
```

---

## Trap 10: Sealed Class Exhaustiveness in Switch

**Interviewer:** "Does adding a new permitted subclass to a sealed hierarchy break existing switch expressions?"

**The Trap:** Candidates say "no, switch has a default" — not understanding the compile-time guarantee and the migration cost.

**Answer:** If the switch expression has no `default`, adding a new permitted subclass causes a compile error in all switch expressions over that type — which is actually the intended benefit (find all sites that need updating). If you have `default`, the exhaustiveness guarantee is lost.

```java
sealed interface Notification permits Email, SMS, Push {}
record Email(String to, String body) implements Notification {}
record SMS(String phone, String message) implements Notification {}
record Push(String deviceId, String title) implements Notification {}

// Exhaustive switch — NO default needed, compiler verifies completeness
String send(Notification n) {
    return switch (n) {
        case Email e -> "Sending email to " + e.to();
        case SMS s   -> "Sending SMS to " + s.phone();
        case Push p  -> "Pushing to " + p.deviceId();
        // Compiler verifies all permits are covered
    };
}

// Now add: record InAppMessage(...) implements Notification {}
// The above switch NOW FAILS TO COMPILE — compiler reports missing case
// This is INTENTIONAL — sealed classes make additions a compile-time event

// With default — exhaustiveness guarantee is gone
String sendWithDefault(Notification n) {
    return switch (n) {
        case Email e -> "email";
        default      -> "other";  // Adding InAppMessage silently hits default — bug risk
    };
}
```

---

## Trap 11: Bounded Type Parameters vs Wildcards in Methods

**Interviewer:** "What is the difference between `<T extends Number> void foo(List<T> list)` and `void foo(List<? extends Number> list)`?"

**The Trap:** Candidates say they are the same.

**Answer:** They behave similarly for reading, but the bounded type parameter version lets you reference `T` elsewhere in the signature or body — relating the type across multiple uses. Wildcards are "use once" anonymous types.

```java
// With bounded type parameter — T is a named type, usable everywhere
public <T extends Number> T getFirst(List<T> list) {
    return list.get(0); // return type T — compiler knows it's the same T
}

// With wildcard — cannot name the type, cannot use it as return type
public Number getFirstWild(List<? extends Number> list) {
    return list.get(0); // must upcast to Number — lost precision
    // return list.get(0) as "?" — impossible
}

// Multiple uses of T — wildcards cannot do this
public <T> void swap(List<T> list, int i, int j) {
    T temp = list.get(i); // T known, can declare local variable
    list.set(i, list.get(j));
    list.set(j, temp);
}
```

---

## Trap 12: Annotation on a Record Component — Which Element Does it Apply To?

**Interviewer:** "If you put `@NotNull` on a record component, does it apply to the field, the constructor parameter, or the accessor method?"

**The Trap:** Candidates assume it is obvious. It is not.

**Answer:** It depends on the annotation's `@Target`. If `@NotNull` targets `FIELD`, `PARAMETER`, and `METHOD`, then placing it on the record component applies it to all three simultaneously. If it only targets one, only that element receives it. This matters for Bean Validation and other frameworks.

```java
// Annotation applied to record component — propagates based on @Target
public record User(
    @NotNull @Size(min=2, max=50) String name,  // applies to field + parameter + accessor
    @Min(0) @Max(150) int age
) {}

// What the compiler generates for @NotNull on name:
// - private final @NotNull String name;      ← field annotation
// - public User(@NotNull String name, int age) {...} ← constructor param annotation
// - @NotNull public String name() {...}      ← accessor method annotation

// To target specifically:
public record Precise(
    @field:NotNull           // only the field
    @param:NotNull           // only the constructor parameter
    @get:NotNull             // only the accessor method
    String value
) {}
```
