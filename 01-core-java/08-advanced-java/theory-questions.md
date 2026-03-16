# Advanced Java — Theory Questions

> 22 theory questions on generics, enums, inner classes, annotations, records, sealed classes, and modern Java features for Senior Java Backend Engineers.

---

## Generics

**Q1. What is type erasure in Java generics? What are its implications?**

Type erasure means generic type information is removed at compile time and not available at runtime. The JVM sees raw types.

```java
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();

strings.getClass() == ints.getClass();  // true! Both are ArrayList.class at runtime

// Cannot do at runtime (compiler error):
if (list instanceof List<String>) { ... }  // COMPILE ERROR

// Only raw type check is allowed:
if (list instanceof List<?>) { ... }   // OK
if (list instanceof List) { ... }      // OK (raw type)
```

**Implications:**
1. No `new T()` — can't instantiate generic types
2. No `new T[]` — can't create generic arrays
3. No `instanceof T` — can't check generic type at runtime
4. Static members can't use class type parameter
5. Cannot catch or throw generic exceptions

**Why erasure was chosen:** backward compatibility with pre-generics Java code.

---

**Q2. What are bounded type parameters? Explain upper and lower bounds.**

```java
// Upper bound — T must be Number or a subtype
public <T extends Number> double sum(List<T> list) {
    return list.stream().mapToDouble(Number::doubleValue).sum();
}
// Can call: sum(List<Integer>), sum(List<Double>), sum(List<BigDecimal>)
// Cannot call: sum(List<String>)

// Lower bound — T must be Number or a supertype
public <T> void addNumbers(List<? super Number> list) {
    list.add(42);      // OK — Integer is a Number
    list.add(3.14);    // OK — Double is a Number
}
// List<Number> works, List<Object> works
// List<Integer> does NOT work (? super Number means Number or above)

// Multiple bounds
public <T extends Comparable<T> & Serializable> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}
// T must implement both Comparable and Serializable
// Class bound must come first if there is one
```

**PECS rule (Producer Extends, Consumer Super):**
- Use `<? extends T>` when you read (produce) from a collection
- Use `<? super T>` when you write (consume) into a collection

---

**Q3. What is a wildcard `<?>` and when do you use it?**

```java
// Unbounded wildcard — any type
public void printList(List<?> list) {
    for (Object elem : list) {  // Can only read as Object
        System.out.println(elem);
    }
    // list.add("something");  // COMPILE ERROR — can't add anything except null
}

// Upper bounded wildcard
public double sumNumbers(List<? extends Number> numbers) {
    return numbers.stream().mapToDouble(Number::doubleValue).sum();
}
// Read as Number — OK
// Write — COMPILE ERROR (might be List<Integer> or List<Double>)

// Lower bounded wildcard
public void addIntegers(List<? super Integer> list) {
    list.add(1);    // OK — Integer is accepted
    list.add(100);  // OK
    // Integer elem = list.get(0);  // Can only read as Object
}

// Wildcard capture
public void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);  // Need helper to capture the type
}
private <T> void swapHelper(List<T> list, int i, int j) {
    T temp = list.get(i);
    list.set(i, list.get(j));
    list.set(j, temp);
}
```

---

**Q4. What is the difference between `List<Object>` and `List<?>`?**

```java
List<String> strings = new ArrayList<>();

// List<Object> is NOT a supertype of List<String>
// (even though Object is supertype of String)
List<Object> objects = strings;  // COMPILE ERROR!

// Why: If allowed, you could add Integer to List<Object> → ClassCastException when reading String

// List<?> IS a supertype of List<String>
List<?> wildcard = strings;   // OK
// But you can't add anything (except null) — type-safe!
wildcard.add("hello");  // COMPILE ERROR

// Practical difference:
void m1(List<Object> list) { list.add("hello"); }  // Can add
void m2(List<?> list) { /* read only */ }           // Cannot add
```

---

**Q5. What are generic methods and when do you use them over generic classes?**

```java
// Generic class — type parameter on the class
public class Box<T> {
    private T value;
    public T get() { return value; }
}

// Generic method — type parameter on the method
public class Utils {
    // Type parameter <T> is scoped to this method only
    public static <T> List<T> listOf(T... elements) {
        return new ArrayList<>(Arrays.asList(elements));
    }

    // Generic return type inferred from usage
    public static <T> Optional<T> firstOrEmpty(List<T> list) {
        return list.isEmpty() ? Optional.empty() : Optional.of(list.get(0));
    }
}

// When to use generic methods vs generic classes:
// - Generic class: when the whole class needs to be type-safe (Box<T>, Repository<T>)
// - Generic method: when only one method needs a type parameter (utility methods)
// - Generic methods are more flexible (type inferred at call site)
```

---

## Enums

**Q6. What are the advanced features of Java enums?**

```java
// Enum with abstract method — each constant implements differently
public enum Operation {
    ADD("+") {
        @Override
        public BigDecimal apply(BigDecimal x, BigDecimal y) { return x.add(y); }
    },
    SUBTRACT("-") {
        @Override
        public BigDecimal apply(BigDecimal x, BigDecimal y) { return x.subtract(y); }
    },
    MULTIPLY("*") {
        @Override
        public BigDecimal apply(BigDecimal x, BigDecimal y) { return x.multiply(y); }
    };

    private final String symbol;

    Operation(String symbol) { this.symbol = symbol; }

    public abstract BigDecimal apply(BigDecimal x, BigDecimal y);

    public String getSymbol() { return symbol; }
}

// Enum implementing interface
public enum Status implements Describable {
    ACTIVE {
        @Override
        public String describe() { return "Currently active and processing"; }
    },
    INACTIVE {
        @Override
        public String describe() { return "Not currently active"; }
    };
}

// EnumMap and EnumSet — highly efficient implementations
EnumMap<Status, List<Order>> ordersByStatus = new EnumMap<>(Status.class);
EnumSet<Permission> adminPermissions = EnumSet.of(Permission.READ, Permission.WRITE, Permission.DELETE);
```

---

**Q7. How does enum work internally in Java?**

Enums are compiled to classes that extend `java.lang.Enum<E>`:

```java
// This:
public enum Color { RED, GREEN, BLUE }

// Compiles to approximately:
public final class Color extends Enum<Color> {
    public static final Color RED = new Color("RED", 0);
    public static final Color GREEN = new Color("GREEN", 1);
    public static final Color BLUE = new Color("BLUE", 2);

    private static final Color[] $VALUES = {RED, GREEN, BLUE};

    private Color(String name, int ordinal) { super(name, ordinal); }

    public static Color[] values() { return $VALUES.clone(); }
    public static Color valueOf(String name) { /* ... */ }
}
```

**Key properties:**
- `name()` — the constant name as defined in source
- `ordinal()` — zero-based position (fragile — don't use for persistence!)
- `values()` — all constants in declaration order
- `valueOf(String)` — look up by name
- Enums are singletons — `==` comparison is safe (and faster than `.equals()`)
- Enum serialization is automatic and guaranteed to maintain singleton property

---

## Records

**Q8. What are Java records (Java 16+) and how do they work?**

Records are immutable data carriers — transparent holders for a fixed set of values.

```java
// Record declaration
public record Point(double x, double y) {}

// Compiler generates:
// - private final fields: x, y
// - Canonical constructor Point(double x, double y)
// - Accessor methods: x(), y() (NOT getX() — just x())
// - equals(), hashCode() based on all components
// - toString() like "Point[x=1.0, y=2.0]"

// Compact canonical constructor — validate/normalize
public record Email(String address) {
    public Email {  // Compact constructor — no parens after constructor name
        Objects.requireNonNull(address, "address");
        address = address.strip().toLowerCase();  // Normalize in compact constructor
        if (!address.contains("@")) {
            throw new IllegalArgumentException("Invalid email: " + address);
        }
    }
}

// Custom methods in records
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException("Currency mismatch");
        return new Money(amount.add(other.amount), currency);
    }

    public boolean isPositive() { return amount.compareTo(BigDecimal.ZERO) > 0; }
}

// Records implement interfaces
public record Name(String first, String last) implements Comparable<Name> {
    @Override
    public int compareTo(Name other) {
        int lastCmp = last.compareTo(other.last);
        return lastCmp != 0 ? lastCmp : first.compareTo(other.first);
    }
}
```

**Records can NOT:**
- Extend other classes (implicitly extend Record)
- Be subclassed
- Have non-static instance fields outside component list
- Declare mutable fields

---

**Q9. What are sealed classes (Java 17+)?**

Sealed classes restrict which classes can extend or implement them.

```java
// Sealed class — only listed classes can extend it
public sealed class Shape permits Circle, Rectangle, Triangle {}

public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
}

public non-sealed class Rectangle extends Shape {
    // non-sealed: can be extended by any class
    private final double width, height;
    public Rectangle(double width, double height) {
        this.width = width; this.height = height;
    }
}

public final class Triangle extends Shape {
    private final double base, height;
}

// Sealed interfaces
public sealed interface Expr permits Num, Add, Mul {}
public record Num(int value) implements Expr {}
public record Add(Expr left, Expr right) implements Expr {}
public record Mul(Expr left, Expr right) implements Expr {}
```

**Why sealed classes?** Enable exhaustive pattern matching — the compiler can verify you've handled all cases:

```java
// Pattern matching switch (Java 21) — exhaustive check
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
    // No default needed! Compiler knows all subclasses
};
```

---

## Annotations

**Q10. How do you create a custom annotation in Java?**

```java
// Custom annotation definition
@Target({ElementType.METHOD, ElementType.TYPE})  // Where can it be applied
@Retention(RetentionPolicy.RUNTIME)               // Available at runtime via reflection
@Documented                                        // Included in Javadoc
@Inherited                                         // Subclasses inherit from annotated class
public @interface Audit {
    String action() default "";
    boolean logParameters() default true;
    AuditLevel level() default AuditLevel.INFO;
}

public enum AuditLevel { DEBUG, INFO, WARN, ERROR }

// Usage
@Audit(action = "CREATE_ORDER", level = AuditLevel.INFO)
public Order createOrder(OrderRequest request) { ... }

// Processing at runtime via reflection
Method method = OrderService.class.getMethod("createOrder", OrderRequest.class);
Audit audit = method.getAnnotation(Audit.class);
if (audit != null) {
    log.info("Action: {}, Level: {}", audit.action(), audit.level());
}
```

**Retention policies:**
- `SOURCE` — only in source code, stripped by compiler (e.g., `@Override`)
- `CLASS` — in bytecode, not at runtime (default, rare use)
- `RUNTIME` — available via reflection at runtime (most framework annotations)

---

**Q11. What is the difference between `@Target(ElementType.TYPE)` and other targets?**

```
ElementType.TYPE          — class, interface, enum, record, annotation
ElementType.METHOD        — methods
ElementType.FIELD         — fields (including enum constants)
ElementType.PARAMETER     — method/constructor parameters
ElementType.CONSTRUCTOR   — constructors
ElementType.LOCAL_VARIABLE — local variables (only SOURCE retention)
ElementType.ANNOTATION_TYPE — other annotations (meta-annotation)
ElementType.PACKAGE       — package declarations
ElementType.TYPE_PARAMETER — generic type parameters (Java 8+)
ElementType.TYPE_USE      — any use of a type (Java 8+)
ElementType.RECORD_COMPONENT — record components (Java 16+)
```

`TYPE_USE` is for nullability annotations like `@NonNull String`, `@Nullable Integer`.

---

## Inner Classes

**Q12. What are the four types of nested classes and when do you use each?**

```java
// 1. Static nested class — no reference to outer class
class Outer {
    private static int count = 0;

    static class StaticNested {
        void show() {
            System.out.println(count);  // Can access static members
            // Cannot access outer instance members
        }
    }
}
// Use when: logically grouped with outer class but doesn't need outer instance
// Common: Builder pattern, helper classes, entry classes
// Instantiate: new Outer.StaticNested()

// 2. Inner class (non-static) — has implicit reference to outer instance
class Outer {
    private String data = "hello";

    class Inner {
        void show() {
            System.out.println(data);  // Accesses outer instance
            System.out.println(Outer.this.data);  // Explicit outer reference
        }
    }
}
// Use when: needs access to outer instance (Iterator implementations, callbacks)
// Pitfall: outer instance prevents GC — potential memory leak
// Instantiate: outer.new Inner()

// 3. Local class — defined inside a method
void method() {
    class Local implements Runnable {
        @Override
        public void run() { /* can access effectively final local vars */ }
    }
    new Thread(new Local()).start();
}
// Rarely used — prefer lambda or anonymous class

// 4. Anonymous class — unnamed, instantiated inline
Runnable r = new Runnable() {
    @Override
    public void run() { System.out.println("running"); }
};
// Use when: one-time use implementation of interface/abstract class
// Largely replaced by lambdas in modern Java
```

---

**Q13. What memory implications do inner (non-static) classes have?**

Non-static inner classes hold an implicit reference to the outer class instance:

```java
// Memory leak scenario
class ActivityManager {
    private List<Runnable> callbacks = new ArrayList<>();

    public void registerCallback() {
        callbacks.add(new Runnable() {
            @Override
            public void run() {
                // This anonymous Runnable holds a reference to ActivityManager!
                doWork();
            }
        });
    }
}

// If callbacks list is stored somewhere else,
// it keeps ActivityManager alive via the implicit outer reference.
```

**Fix: use static nested class or lambda with no outer reference:**
```java
// Lambda — only captures what it explicitly references
callbacks.add(() -> ActivityManager.doStaticWork());  // No outer reference if static

// Static nested class — no implicit outer reference
static class StaticCallback implements Runnable {
    @Override
    public void run() { /* no outer reference */ }
}
```

---

## Pattern Matching and Switch Expressions

**Q14. What is pattern matching for instanceof (Java 16+)?**

```java
// Old way — verbose
if (obj instanceof String) {
    String s = (String) obj;  // Explicit cast
    System.out.println(s.toUpperCase());
}

// Pattern matching instanceof (Java 16+)
if (obj instanceof String s) {
    System.out.println(s.toUpperCase());  // s is in scope, already cast
}

// With conditions (guards)
if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s);
}

// In complex logic — s is in scope only within the true branch
if (!(obj instanceof String s)) {
    return;  // Early return
}
// s is in scope here after the guard
processString(s);
```

---

**Q15. What are switch expressions (Java 14+) and how do they differ from switch statements?**

```java
// Old switch statement — fall-through bugs, verbose
String result;
switch (day) {
    case MONDAY:
    case TUESDAY:
        result = "Early week";
        break;
    case FRIDAY:
        result = "TGIF!";
        break;
    default:
        result = "Mid week";
}

// New switch expression — exhaustive, no fall-through, yields value
String result = switch (day) {
    case MONDAY, TUESDAY -> "Early week";
    case FRIDAY -> "TGIF!";
    default -> "Mid week";
};

// With blocks — use yield
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY -> 7;
    case THURSDAY, SATURDAY -> 8;
    case WEDNESDAY -> {
        System.out.println("Longest day name!");
        yield 9;  // yield returns value from block
    }
};

// Pattern matching switch (Java 21)
String formatted = switch (obj) {
    case Integer i -> "int " + i;
    case Long l    -> "long " + l;
    case String s  -> "String: " + s;
    case null      -> "null value";
    default        -> "unknown: " + obj;
};
```

---

## Optional

**Q16. What is `Optional` and when should/shouldn't you use it?**

`Optional<T>` is a container that may or may not contain a non-null value. Primarily for return types.

```java
// Good uses — return types where absence is meaningful
public Optional<User> findUserById(Long id) {
    return userRepository.findById(id);  // Returns Optional<User>
}

// Optional API
Optional<User> optUser = findUserById(42L);

optUser.isPresent();  // Check
optUser.isEmpty();    // Java 11+

optUser.get();        // Get value (throws NoSuchElementException if empty!)
optUser.orElse(User.anonymous());  // Default value
optUser.orElseGet(() -> createAnonymousUser());  // Lazy default
optUser.orElseThrow(() -> new UserNotFoundException(42L));

optUser.map(User::getName)         // Optional<String>
       .map(String::toUpperCase)   // Optional<String>
       .orElse("ANONYMOUS");

optUser.filter(u -> u.isActive())  // Optional<User> (empty if predicate fails)
       .ifPresent(u -> process(u));

// ifPresentOrElse (Java 9)
optUser.ifPresentOrElse(
    u -> sendWelcomeEmail(u),
    () -> log.warn("User not found")
);
```

**When NOT to use Optional:**
```java
// WRONG — Optional as parameter
public void process(Optional<String> name) { ... }
// Better: use @Nullable or method overloading

// WRONG — Optional as field
private Optional<String> middleName;  // Not serializable, overhead
// Better: @Nullable String middleName

// WRONG — Optional wrapping collections
Optional<List<Order>> orders;  // If no orders, return empty list, not Optional.empty()
// Better: List<Order> orders, return List.of() if none

// WRONG — Optional in streams (use flatMap instead)
list.stream().map(this::findOptional).filter(Optional::isPresent).map(Optional::get)
// Better: list.stream().map(this::findOptional).flatMap(Optional::stream)
```

---

## Functional Interfaces

**Q17. What are the core functional interfaces in java.util.function?**

```java
// Function<T, R> — takes T, returns R
Function<String, Integer> length = String::length;
Function<Integer, String> toStr = Object::toString;
Function<String, String> combined = length.andThen(toStr); // compose

// Predicate<T> — takes T, returns boolean
Predicate<String> nonEmpty = s -> !s.isEmpty();
Predicate<String> isEmail = s -> s.contains("@");
Predicate<String> validEmail = nonEmpty.and(isEmail);
Predicate<String> invalidEmail = validEmail.negate();

// Consumer<T> — takes T, returns void
Consumer<String> print = System.out::println;
Consumer<String> log = s -> logger.info(s);
Consumer<String> both = print.andThen(log);

// Supplier<T> — takes nothing, returns T
Supplier<List<String>> listFactory = ArrayList::new;
Supplier<UUID> idGen = UUID::randomUUID;

// Specialized for primitives (avoid boxing):
IntFunction<String> intToStr = Integer::toString;
ToIntFunction<String> strToInt = Integer::parseInt;
IntUnaryOperator doubleIt = x -> x * 2;
IntBinaryOperator add = Integer::sum;
IntPredicate isEven = x -> x % 2 == 0;
IntSupplier counter = new AtomicInteger()::getAndIncrement;
IntConsumer printer = System.out::println;

// BiFunction<T, U, R> — takes T and U, returns R
BiFunction<String, Integer, String> repeat = (s, n) -> s.repeat(n);
```

---

## var (Local Variable Type Inference)

**Q18. What is `var` in Java (Java 10+)?**

`var` allows the compiler to infer the type of local variables.

```java
// var — type is inferred at compile time, not runtime
var list = new ArrayList<String>();  // ArrayList<String>
var map = new HashMap<String, Integer>();  // HashMap<String, Integer>
var entry = map.entrySet().iterator().next();  // Map.Entry<String, Integer>

// Useful for long generic types
var response = restTemplate.exchange(
    url, HttpMethod.GET, entity,
    new ParameterizedTypeReference<List<OrderDto>>() {}
);  // ResponseEntity<List<OrderDto>>

// Works with enhanced for loop
for (var entry : map.entrySet()) {
    // entry is Map.Entry<String, Integer>
}

// Cannot use var for:
var x;                // ERROR — no initializer
var x = null;         // ERROR — can't infer from null
var x = () -> {};     // ERROR — lambda without target type
var x = { 1, 2, 3 }; // ERROR — array initializer

// Fields, parameters, return types:
// var cannot be used for these — only local variables
```

---

**Q19. What are text blocks and how do they simplify Java code?**

Text blocks (Java 15+) are multi-line string literals with minimal escaping.

```java
// Before text blocks — ugly escaped string
String json = "{\n" +
              "  \"name\": \"" + name + "\",\n" +
              "  \"age\": " + age + "\n" +
              "}";

// With text blocks
String json = """
        {
          "name": "%s",
          "age": %d
        }
        """.formatted(name, age);

// SQL
String sql = """
        SELECT o.id, o.total, c.email
        FROM orders o
        JOIN customers c ON c.id = o.customer_id
        WHERE o.status = 'PENDING'
        ORDER BY o.created_at DESC
        """;

// HTML in tests
String html = """
        <html>
          <body>
            <h1>Hello, World!</h1>
          </body>
        </html>
        """;
```

**Indentation rules:** The closing `"""` determines baseline. Text before its column is stripped. Resulting string ends with `\n` if `"""` is on its own line.

**Special escapes:**
- `\` at end of line: suppress the newline (line continuation)
- `\s`: trailing whitespace preservation

---

**Q20. What are the differences between Comparable and Comparator?**

```java
// Comparable — natural ordering (intrinsic to the class)
public class Employee implements Comparable<Employee> {
    private String name;
    private int salary;

    @Override
    public int compareTo(Employee other) {
        return this.name.compareTo(other.name);  // Natural order: alphabetical by name
    }
}

// Sort uses natural order:
Collections.sort(employees);
employees.sort(null);  // null uses Comparable

// Comparator — external, customizable ordering
// Define multiple orderings for the same type:
Comparator<Employee> bySalary = Comparator.comparingInt(Employee::getSalary);
Comparator<Employee> byNameThenSalary = Comparator
    .comparing(Employee::getName)
    .thenComparingInt(Employee::getSalary);
Comparator<Employee> bySalaryDesc = Comparator.comparingInt(Employee::getSalary).reversed();
Comparator<Employee> nullsFirst = Comparator.nullsFirst(Comparator.comparing(Employee::getName));

employees.sort(bySalary);
employees.sort(Comparator.comparing(Employee::getDepartment)
                         .thenComparing(Employee::getName));

// TreeMap with custom ordering
TreeMap<Employee, List<Order>> ordersByEmployee = new TreeMap<>(bySalary);
```

---

**Q21. What are method references and what forms do they take?**

```java
// 1. Static method reference
Function<String, Integer> parser = Integer::parseInt;  // Integer.parseInt(s)

// 2. Instance method of particular object
String prefix = "Hello";
Predicate<String> startsWith = prefix::startsWith;    // prefix.startsWith(s)

// 3. Instance method of arbitrary object of a given type
Function<String, String> toUpper = String::toUpperCase;  // s.toUpperCase()
// (instance method on the parameter itself)

// 4. Constructor reference
Supplier<ArrayList<String>> listMaker = ArrayList::new;  // new ArrayList<>()
Function<Integer, int[]> arrayMaker = int[]::new;         // new int[n]
BiFunction<String, Integer, StringBuilder> sbMaker = StringBuilder::new;

// When to use method references vs lambdas:
// Method reference: when lambda just calls a method with matching params
list.stream().map(s -> s.toUpperCase())  // OK
list.stream().map(String::toUpperCase)  // Cleaner

// Lambda preferred when:
list.stream().map(s -> s.substring(0, 3))  // 2 params, method reference awkward
list.stream().filter(s -> s.length() > 5 && s.startsWith("A"))  // Complex predicate
```

---

**Q22. What are the key differences between `Iterable`, `Iterator`, and `Spliterator`?**

```java
// Iterable — can be iterated (has iterator())
// Used with enhanced for loop
for (String s : myIterable) { ... }  // calls myIterable.iterator()

// Iterator — one-pass traversal cursor
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("remove")) {
        it.remove();  // Safe removal during iteration
    }
}

// Spliterator (Java 8) — splittable iterator for parallel processing
// Used internally by Stream's parallel operations
Spliterator<String> sp = list.spliterator();
sp.characteristics();  // ORDERED, SIZED, DISTINCT, etc.
sp.estimateSize();      // Estimated remaining elements
sp.tryAdvance(s -> process(s));  // Process one element
sp.forEachRemaining(s -> process(s));  // Process all remaining

// Splitting for parallel processing
Spliterator<String> first = sp.trySplit();  // Split roughly in half
// Process first and sp in parallel
ForkJoinPool.commonPool().submit(() -> first.forEachRemaining(s -> process(s)));
sp.forEachRemaining(s -> process(s));

// Custom Spliterator for custom parallel source
class RangeSpliterator implements Spliterator.OfInt {
    // Used to efficiently split integer ranges for parallel stream
}
```
