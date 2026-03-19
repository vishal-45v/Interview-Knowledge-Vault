# Chapter 08: Advanced Java — Structured Answers

Detailed, interview-ready answers with code examples.

---

## Q1: What is Type Erasure and What Problems Does it Cause?

**One-line answer:** Type erasure is the process by which the Java compiler removes all generic type information from compiled bytecode, replacing type parameters with their bounds (or `Object`), so the JVM never sees `List<String>` — only `List`.

**Why it exists:** Java generics were added in Java 5 as a backwards-compatible feature. The JVM did not change, so the compiler takes on all type checking and then strips the types to produce `.class` files compatible with pre-generics JVMs.

**How it works:**
- `List<String>` becomes `List` (raw type)
- `T` becomes `Object` (if unbounded)
- `T extends Comparable<T>` becomes `Comparable`
- The compiler inserts checkcast instructions where types need to be verified

**Problems caused:**

1. **Cannot use `instanceof` with generic types:**
```java
// COMPILE ERROR
if (obj instanceof List<String>) { }

// Must use raw type or unbounded wildcard
if (obj instanceof List<?>) { }
```

2. **Cannot create generic arrays:**
```java
// COMPILE ERROR
T[] arr = new T[10];

// Workaround
T[] arr = (T[]) new Object[10]; // unchecked cast warning
```

3. **Cannot use generics with `new` for type parameters:**
```java
// COMPILE ERROR — T is erased, JVM doesn't know what to construct
T obj = new T();

// Workaround — pass Class<T> token
public <T> T create(Class<T> clazz) throws Exception {
    return clazz.getDeclaredConstructor().newInstance();
}
```

4. **Heap pollution:**
```java
List<String> strings = new ArrayList<>();
List raw = strings;
raw.add(42);                     // no compile error with raw type
String s = strings.get(0);      // ClassCastException at runtime
```

5. **Overloading conflicts:**
```java
// COMPILE ERROR — both erase to the same signature: void process(List)
public void process(List<String> list) {}
public void process(List<Integer> list) {}
```

6. **Cannot retrieve type parameter at runtime:**
```java
public class Repo<T> {
    // Cannot do: Class<T> type = T.class;  — T is erased
    // Must pass it in:
    private final Class<T> type;
    public Repo(Class<T> type) { this.type = type; }
}
```

**Summary table:**

| What you write | What the JVM sees |
|---|---|
| `List<String>` | `List` |
| `Map<K, V>` | `Map` |
| `T` (unbounded) | `Object` |
| `T extends Number` | `Number` |
| `Pair<Integer, String>` | `Pair` |

---

## Q2: Explain the PECS Rule for Generic Wildcards with Examples

**One-line answer:** PECS stands for "Producer Extends, Consumer Super" — use `? extends T` when a structure produces (you read from it), and `? super T` when a structure consumes (you write to it).

**The core insight:** When you have a `List<? extends Number>`, the list might actually be a `List<Integer>` or `List<Double>`. You can safely read anything as `Number`, but you cannot write — you do not know the exact type. Conversely, `List<? super Integer>` might be `List<Integer>`, `List<Number>`, or `List<Object>` — you can safely put an `Integer` in any of these, but reading gives you only an `Object`.

**Producer Extends — use when reading:**
```java
// Works with List<Integer>, List<Double>, List<Number>
public double sum(List<? extends Number> numbers) {
    double total = 0;
    for (Number n : numbers) {   // Reading is safe — always at least a Number
        total += n.doubleValue();
    }
    return total;
    // numbers.add(3.14); // COMPILE ERROR — might be List<Integer>, can't add Double
}

// Call with any Number subtype list
sum(List.of(1, 2, 3));     // List<Integer> — works
sum(List.of(1.1, 2.2));    // List<Double> — works
sum(List.of(1, 2.0, 3L));  // List<Number> — works
```

**Consumer Super — use when writing:**
```java
// Works with List<Integer>, List<Number>, List<Object>
public void fillWithDefaults(List<? super Integer> list, int count) {
    for (int i = 0; i < count; i++) {
        list.add(0);  // Writing Integer is safe — all supertypes accept Integer
    }
    // Integer x = list.get(0); // COMPILE ERROR — might be List<Object>
    Object o = list.get(0);     // Only safe as Object
}

List<Integer> ints = new ArrayList<>();
List<Number> nums = new ArrayList<>();
List<Object> objs = new ArrayList<>();

fillWithDefaults(ints, 3); // works
fillWithDefaults(nums, 3); // works
fillWithDefaults(objs, 3); // works
```

**Classic example — `Collections.copy`:**
```java
// From JDK source — the canonical PECS example
public static <T> void copy(List<? super T> dest,   // Consumer — writes T
                             List<? extends T> src)  // Producer — reads T
{
    for (T item : src) {   // Read from producer
        dest.add(item);    // Write to consumer
    }
}
```

**Unbounded wildcard (`?`):**
```java
// When you only use Object methods — neither producer nor consumer
public void printAll(List<?> list) {
    for (Object o : list) {
        System.out.println(o);  // toString() is on Object — fine
    }
    // list.add("x"); // COMPILE ERROR — can't add anything except null
}
```

---

## Q3: How Do Annotations Work at Runtime? Show a Custom Annotation Example

**One-line answer:** Annotations with `@Retention(RetentionPolicy.RUNTIME)` are stored in the `.class` file and accessible via the Reflection API (`getAnnotation()`, `getDeclaredAnnotations()`). Frameworks use this to drive behavior without modifying the annotated code.

**Step 1 — Define the annotation:**
```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)  // Must be RUNTIME to read at runtime
@Target(ElementType.METHOD)           // Can only be placed on methods
@Documented                           // Will appear in Javadoc
public @interface Timed {
    String label() default "";           // Optional label, defaults to method name
    int warningThresholdMs() default 500; // Log warning if method exceeds this
}
```

**Step 2 — Apply the annotation:**
```java
public class ReportService {

    @Timed(label = "generateMonthlyReport", warningThresholdMs = 2000)
    public Report generateMonthlyReport(Month month) {
        // ... expensive computation ...
        return new Report();
    }

    @Timed  // uses defaults
    public List<Report> listReports() {
        return List.of();
    }
}
```

**Step 3 — Process it at runtime (manual example, no framework):**
```java
public class TimingInterceptor {

    public Object invoke(Object target, String methodName, Object... args) throws Exception {
        Method method = target.getClass().getMethod(methodName,
            Arrays.stream(args).map(Object::getClass).toArray(Class[]::new));

        Timed timed = method.getAnnotation(Timed.class);

        if (timed == null) {
            return method.invoke(target, args); // no annotation — call directly
        }

        String label = timed.label().isEmpty() ? method.getName() : timed.label();
        long start = System.currentTimeMillis();

        Object result = method.invoke(target, args);

        long elapsed = System.currentTimeMillis() - start;
        System.out.printf("[%s] took %dms%n", label, elapsed);

        if (elapsed > timed.warningThresholdMs()) {
            System.out.printf("WARNING: [%s] exceeded threshold (%dms > %dms)%n",
                label, elapsed, timed.warningThresholdMs());
        }

        return result;
    }
}
```

**Step 4 — Scanning all annotated methods in a class:**
```java
// Find all @Timed methods in a class
Class<?> clazz = ReportService.class;
for (Method method : clazz.getDeclaredMethods()) {
    Timed timed = method.getAnnotation(Timed.class);
    if (timed != null) {
        System.out.println("Found @Timed on: " + method.getName()
            + " threshold=" + timed.warningThresholdMs() + "ms");
    }
}
```

---

## Q4: What is a Dynamic Proxy and How Does Spring Use It?

**One-line answer:** A dynamic proxy is a class generated at runtime that implements one or more interfaces and delegates all method calls to an `InvocationHandler`, allowing you to intercept every call without modifying the original object.

**Manual dynamic proxy:**
```java
public interface UserService {
    User findById(long id);
    void save(User user);
}

public class UserServiceImpl implements UserService {
    public User findById(long id) { /* database call */ return new User(id); }
    public void save(User user)   { /* database call */ }
}

// InvocationHandler — intercepts every method call
public class TransactionHandler implements InvocationHandler {
    private final Object target;

    public TransactionHandler(Object target) { this.target = target; }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("BEGIN TRANSACTION");
        try {
            Object result = method.invoke(target, args);
            System.out.println("COMMIT");
            return result;
        } catch (InvocationTargetException e) {
            System.out.println("ROLLBACK");
            throw e.getCause();
        }
    }
}

// Creating the proxy at runtime
UserService service = new UserServiceImpl();
UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class[]{UserService.class},
    new TransactionHandler(service)
);

proxy.save(new User()); // prints: BEGIN TRANSACTION ... COMMIT
```

**How Spring uses dynamic proxies:**

Spring AOP creates proxies for beans annotated with `@Transactional`, `@Cacheable`, `@Async`, etc. When you wire a `@Transactional` service, Spring actually injects a proxy, not your real bean.

```java
@Service
public class OrderService {

    @Transactional           // Spring wraps this in a proxy
    public void placeOrder(Order order) {
        // ... business logic ...
    }
}

// In your controller, what you get is NOT OrderService — it's a proxy
@Autowired
private OrderService orderService; // Actually a $Proxy42 or OrderService$$EnhancerByCGLIB

// Spring's rule:
// - If class implements interface(s) → JDK dynamic proxy
// - If class has no interface (or spring.aop.proxy-target-class=true) → CGLIB proxy

// CGLIB creates a subclass:
// class OrderService$$EnhancerByCGLIB extends OrderService {
//     @Override
//     public void placeOrder(Order order) {
//         // start transaction
//         super.placeOrder(order);
//         // commit/rollback
//     }
// }
```

**Critical trap — self-invocation:**
```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        validateOrder(order);  // calls validateOrder through 'this', bypassing proxy!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void validateOrder(Order order) {
        // This @Transactional is IGNORED when called from placeOrder above
        // because it's called on 'this', not through the proxy
    }
}
```

---

## Q5: How Does the ClassLoader Delegation Model Work?

**One-line answer:** ClassLoaders form a parent-child hierarchy. Before loading a class, a ClassLoader always delegates to its parent first. Only if the parent cannot find the class does the child attempt to load it.

**The delegation chain:**
```
Bootstrap CL (null) → Platform CL → Application CL → Custom CL
```

**Step-by-step loading of `com.myapp.Service`:**
1. Application ClassLoader receives request for `com.myapp.Service`
2. It delegates UP to Platform ClassLoader
3. Platform CL delegates UP to Bootstrap ClassLoader
4. Bootstrap CL checks the JDK rt.jar / boot modules — not found
5. Platform CL checks platform modules — not found
6. Application CL checks the classpath — FOUND, loads it

**Why this matters:**
```java
// This is why you cannot replace java.lang.String with your own version
// Even if you put a custom String.class on the classpath, Bootstrap always loads
// the real String first — your version is never reached

// Demonstrating the chain
ClassLoader loader = Thread.currentThread().getContextClassLoader();
while (loader != null) {
    System.out.println(loader);
    loader = loader.getParent();
}
// Output:
// jdk.internal.loader.ClassLoaders$AppClassLoader@...
// jdk.internal.loader.ClassLoaders$PlatformClassLoader@...
// (null for Bootstrap)
```

**Custom ClassLoader — loading from a non-standard location:**
```java
public class PluginClassLoader extends URLClassLoader {

    public PluginClassLoader(Path pluginJar) throws MalformedURLException {
        super(new URL[]{pluginJar.toUri().toURL()},
              PluginClassLoader.class.getClassLoader()); // parent = app CL
    }

    // Optional: override to load plugin classes first (parent-last / child-first)
    @Override
    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        // Application server pattern — check own JAR before delegating
        // Used by Tomcat, OSGi for isolation
        synchronized (getClassLoadingLock(name)) {
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    c = findClass(name); // try own JAR first
                } catch (ClassNotFoundException e) {
                    c = super.loadClass(name, resolve); // fall back to parent
                }
            }
            return c;
        }
    }
}
```

**Class identity problem:**
```java
// Same class, two ClassLoaders = two distinct Class objects
ClassLoader cl1 = new PluginClassLoader(path1);
ClassLoader cl2 = new PluginClassLoader(path2);

Class<?> class1 = cl1.loadClass("com.plugin.Service");
Class<?> class2 = cl2.loadClass("com.plugin.Service");

System.out.println(class1 == class2);   // false! Different ClassLoaders
System.out.println(class1.equals(class2)); // false
// Objects from class1 cannot be cast to class2 even though same source code
```

---

## Q6: What Are Records in Java and When Should You Use Them?

**One-line answer:** Records (Java 16+) are a concise way to declare immutable data carrier classes — the compiler auto-generates constructor, accessors, `equals()`, `hashCode()`, and `toString()` based on the declared components.

**Full record capabilities:**
```java
// Minimal declaration — compiler generates everything
public record Point(double x, double y) {}

// Equivalent to a class with:
// - private final double x, y
// - canonical constructor: Point(double x, double y)
// - accessors: x(), y() (NOT getX()/getY())
// - equals(), hashCode(), toString() based on all components

Point p1 = new Point(1.0, 2.0);
Point p2 = new Point(1.0, 2.0);
System.out.println(p1.x());       // 1.0
System.out.println(p1.equals(p2)); // true — value-based equality
System.out.println(p1);            // Point[x=1.0, y=2.0]
```

**Compact constructor — validation and normalization:**
```java
public record Range(int min, int max) {
    // Compact constructor — same parameters, no need to declare them
    public Range {
        if (min > max) {
            throw new IllegalArgumentException(
                "min (%d) cannot exceed max (%d)".formatted(min, max));
        }
        // Assignment to this.min and this.max happens automatically after this block
    }
}

// Defensive copy for mutable components
public record ImmutableList<T>(List<T> items) {
    public ImmutableList {
        items = List.copyOf(items); // defensive copy — reassignment allowed in compact ctor
    }
}
```

**Custom methods in records:**
```java
public record Money(BigDecimal amount, Currency currency) {

    // Static factory
    public static Money of(String amount, String currencyCode) {
        return new Money(new BigDecimal(amount), Currency.getInstance(currencyCode));
    }

    // Instance methods
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    // Override toString for custom format
    @Override
    public String toString() {
        return amount.toPlainString() + " " + currency.getCurrencyCode();
    }
}
```

**When to use records:**

1. **DTOs and value objects** — API request/response bodies, database projections
2. **Tuple-like returns** — when you need to return multiple values
3. **Configuration data** — immutable settings objects
4. **Pattern matching** — records work natively with sealed hierarchies

```java
// Perfect record use cases
record Coordinates(double lat, double lon) {}
record UserDto(long id, String name, String email) {}
record PageRequest(int page, int size, String sortBy) {}

// NOT a good fit for records:
// - Entities with lifecycle (JPA @Entity — needs no-arg constructor, mutable fields)
// - Objects with complex behavior and state changes
// - Classes that need to extend another class
```

---

## Q7: How Do Sealed Classes Improve Pattern Matching in Java 17+?

**One-line answer:** Sealed classes close the inheritance hierarchy to a known set of subtypes, enabling the compiler to verify that switch expressions cover all cases — eliminating both the need for `default` and the risk of unhandled subtypes.

**Without sealed — the open hierarchy problem:**
```java
// Anyone can extend Shape — switch can never be exhaustive
abstract class Shape {}
class Circle extends Shape { double radius; }
class Rectangle extends Shape { double w, h; }
// Someone adds: class Hexagon extends Shape {...} — old switches silently miss it

double area(Shape s) {
    if (s instanceof Circle c) return Math.PI * c.radius * c.radius;
    if (s instanceof Rectangle r) return r.w * r.h;
    throw new RuntimeException("Unknown shape: " + s.getClass()); // runtime failure
}
```

**With sealed — exhaustive and compiler-verified:**
```java
// Java 17+: sealed hierarchy — only these three subtypes exist
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}

// Exhaustive switch — no default, compiler verifies all cases covered
double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t  -> 0.5 * t.base() * t.height();
        // If you add a 4th type to permits → COMPILE ERROR here
    };
}
```

**Guard patterns (Java 21+ preview):**
```java
String describe(Shape shape) {
    return switch (shape) {
        case Circle c when c.radius() > 10 -> "Large circle (r=" + c.radius() + ")";
        case Circle c                       -> "Small circle (r=" + c.radius() + ")";
        case Rectangle r when r.width() == r.height() -> "Square (" + r.width() + ")";
        case Rectangle r                    -> "Rectangle " + r.width() + "x" + r.height();
        case Triangle t                     -> "Triangle";
    };
}
```

**Sealed + records — algebraic data types in Java:**
```java
// Modeling a Result type (like Rust's Result<T, E>)
public sealed interface Result<T> permits Result.Success, Result.Failure {
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String error, Exception cause) implements Result<T> {}

    default T getOrThrow() {
        return switch (this) {
            case Success<T> s -> s.value();
            case Failure<T> f -> throw new RuntimeException(f.error(), f.cause());
        };
    }

    default <U> Result<U> map(java.util.function.Function<T, U> fn) {
        return switch (this) {
            case Success<T> s -> new Success<>(fn.apply(s.value()));
            case Failure<T> f -> new Failure<>(f.error(), f.cause());
        };
    }
}
```

---

## Q8: What is the Difference Between @Retention(RUNTIME) and @Retention(CLASS)?

**One-line answer:** `RUNTIME` annotations are stored in `.class` files AND available via reflection at runtime. `CLASS` annotations (the default) are stored in `.class` files but stripped from the runtime representation — invisible to `getAnnotation()`.

```java
// Three retention levels
@Retention(RetentionPolicy.SOURCE)
@interface SourceOnly {}   // Exists only in .java — discarded by compiler
                            // Examples: @Override, @SuppressWarnings

@Retention(RetentionPolicy.CLASS)
@interface ClassOnly {}    // Stored in .class — NOT available at runtime (default)
                            // Useful for: bytecode tools (ASM, ByteBuddy), annotation processors

@Retention(RetentionPolicy.RUNTIME)
@interface RuntimeVisible {}  // Stored in .class AND available via reflection at runtime
                               // Examples: @Transactional, @Test, @RequestMapping
```

**Demonstrating the difference:**
```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.METHOD)
@interface ClassRetained { String value(); }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface RuntimeRetained { String value(); }

class Demo {
    @ClassRetained("class-level")
    @RuntimeRetained("runtime-level")
    public void annotatedMethod() {}
}

Method m = Demo.class.getMethod("annotatedMethod");

ClassRetained classAnn = m.getAnnotation(ClassRetained.class);
RuntimeRetained runtimeAnn = m.getAnnotation(RuntimeRetained.class);

System.out.println(classAnn);    // null — not available at runtime!
System.out.println(runtimeAnn);  // @RuntimeRetained(value="runtime-level")
```

**When to use CLASS retention:**
- Bytecode manipulation tools that read `.class` files directly (do not use JVM reflection)
- Annotation processors that run at compile time (`AbstractProcessor`) — SOURCE or CLASS both work
- Marking information for static analysis tools (e.g., `@NonNull` in IDEs)

**When to use RUNTIME retention:**
- Any framework that uses reflection to drive behavior (`@Transactional`, `@Autowired`)
- Your own annotation processing done at application startup
- Anything you need to call `getAnnotation()` on

---

## Q9: How Do You Create a Custom Annotation Processor?

**One-line answer:** Implement `AbstractProcessor`, declare `@SupportedAnnotationTypes` and `@SupportedSourceVersion`, override `process()`, and register it via `META-INF/services/javax.annotation.processing.Processor` or the `--processor-path` javac option.

```java
// Step 1: Define the annotation (with SOURCE or CLASS retention for APT)
@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
public @interface GenerateBuilder {
    // Marks a class for builder generation
}

// Step 2: Implement the processor
@SupportedAnnotationTypes("com.example.GenerateBuilder")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        for (TypeElement annotation : annotations) {
            Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(annotation);

            for (Element element : elements) {
                if (element.getKind() == ElementKind.CLASS) {
                    generateBuilder((TypeElement) element);
                }
            }
        }
        return true; // claim the annotation — no other processor should handle it
    }

    private void generateBuilder(TypeElement classElement) {
        String className = classElement.getSimpleName().toString();
        String packageName = processingEnv.getElementUtils()
            .getPackageOf(classElement).getQualifiedName().toString();

        // Get all fields
        List<VariableElement> fields = classElement.getEnclosedElements().stream()
            .filter(e -> e.getKind() == ElementKind.FIELD)
            .map(e -> (VariableElement) e)
            .collect(Collectors.toList());

        // Generate source code for the builder
        StringBuilder sb = new StringBuilder();
        sb.append("package ").append(packageName).append(";\n\n");
        sb.append("public class ").append(className).append("Builder {\n");

        for (VariableElement field : fields) {
            String fieldName = field.getSimpleName().toString();
            String fieldType = field.asType().toString();
            sb.append("    private ").append(fieldType).append(" ").append(fieldName).append(";\n");
        }

        for (VariableElement field : fields) {
            String fieldName = field.getSimpleName().toString();
            String fieldType = field.asType().toString();
            sb.append("\n    public ").append(className).append("Builder ")
              .append(fieldName).append("(").append(fieldType).append(" val) {\n")
              .append("        this.").append(fieldName).append(" = val;\n")
              .append("        return this;\n")
              .append("    }\n");
        }

        sb.append("\n    public ").append(className).append(" build() {\n")
          .append("        return new ").append(className).append("(");
        // constructor args...
        sb.append(");\n    }\n}\n");

        // Write the generated file
        try {
            JavaFileObject file = processingEnv.getFiler()
                .createSourceFile(packageName + "." + className + "Builder");
            try (Writer writer = file.openWriter()) {
                writer.write(sb.toString());
            }
        } catch (IOException e) {
            processingEnv.getMessager().printMessage(
                Diagnostic.Kind.ERROR, "Failed to generate builder: " + e.getMessage());
        }
    }
}
```

**Step 3: Register the processor:**
```
# File: src/main/resources/META-INF/services/javax.annotation.processing.Processor
com.example.BuilderProcessor
```

**Step 4: Use it:**
```java
@GenerateBuilder
public class User {
    private String name;
    private String email;
    private int age;
}

// After compilation, UserBuilder is available:
User user = new UserBuilder()
    .name("Alice")
    .email("alice@example.com")
    .age(30)
    .build();
```

---

## Q10: What Are the Performance Implications of Reflection?

**One-line answer:** Reflection is significantly slower than direct method calls due to security checks, dynamic dispatch, and boxing/unboxing — but the overhead is mostly in the setup (Class/Method lookup), not in subsequent invocations if you cache the Method objects.

**What makes reflection slow:**

1. **Security checks** — `setAccessible`, access control checks on every invoke
2. **No JIT inlining** — JIT compiler cannot inline reflective calls the way it inlines direct calls
3. **Boxing/unboxing** — primitive arguments must be boxed into `Object[]`
4. **Dynamic dispatch** — no compile-time binding
5. **Class/Method lookup** — `getMethod()` is expensive (scans hierarchy, allocates)

**Measuring the overhead:**
```java
// Benchmark setup (simplified JMH-style)
public class ReflectionBenchmark {

    interface Calculator { int add(int a, int b); }
    static class CalculatorImpl implements Calculator {
        public int add(int a, int b) { return a + b; }
    }

    static final CalculatorImpl INSTANCE = new CalculatorImpl();
    static final Method ADD_METHOD;
    static {
        try {
            ADD_METHOD = CalculatorImpl.class.getMethod("add", int.class, int.class);
        } catch (Exception e) { throw new RuntimeException(e); }
    }

    // Direct call: ~1 ns
    int directCall() { return INSTANCE.add(1, 2); }

    // Reflection with cached Method: ~5-10 ns (5-10x slower)
    int cachedReflection() throws Exception {
        return (int) ADD_METHOD.invoke(INSTANCE, 1, 2);
    }

    // Reflection with uncached Method: ~500-1000 ns (500x+ slower)
    int uncachedReflection() throws Exception {
        Method m = CalculatorImpl.class.getMethod("add", int.class, int.class);
        return (int) m.invoke(INSTANCE, 1, 2);
    }
}
```

**How to minimize the cost:**

```java
// 1. Cache Method/Field/Constructor objects — NEVER look them up in hot paths
public class CachedReflector {
    private static final Map<String, Method> methodCache = new ConcurrentHashMap<>();

    public static Method getMethod(Class<?> cls, String name, Class<?>... params) {
        String key = cls.getName() + "#" + name;
        return methodCache.computeIfAbsent(key, k -> {
            try { return cls.getMethod(name, params); }
            catch (NoSuchMethodException e) { throw new RuntimeException(e); }
        });
    }
}

// 2. Use MethodHandles (Java 7+) — JIT-friendly, faster than Method.invoke
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle handle = lookup.findVirtual(
    CalculatorImpl.class, "add",
    MethodType.methodType(int.class, int.class, int.class));

int result = (int) handle.invoke(instance, 1, 2); // ~2-3x faster than Method.invoke

// 3. Generate code dynamically for hot paths (ByteBuddy, ASM, LambdaMetafactory)
// LambdaMetafactory creates a lambda that calls the method — near-native speed
MethodHandles.Lookup lookup2 = MethodHandles.lookup();
MethodHandle target = lookup2.findVirtual(CalculatorImpl.class, "add",
    MethodType.methodType(int.class, int.class, int.class));
// Convert to a functional interface — the JIT can now inline it
```

**General guideline:** Reflection in initialization/startup code = fine. Reflection in tight loops processing millions of items = profile and replace with `MethodHandle` or code generation.
