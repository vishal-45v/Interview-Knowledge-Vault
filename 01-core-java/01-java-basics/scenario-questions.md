# Java Basics — Scenario Questions

> 28 scenario-based questions simulating real interview situations.

---

## Scenario 1: The String Comparison Bug

**Context:** A junior developer writes the following order processing code:

```java
String status = getOrderStatus();  // returns "PENDING" or "COMPLETED"
if (status == "PENDING") {
    processOrder();
}
```

**Problem:** In production, orders are never processed, even when status is clearly "PENDING" in logs.

**Question:** Why is this happening? How would you fix it?

**Answer:**

The bug is using `==` for String comparison. `getOrderStatus()` likely returns a new String object (e.g., from a database query, HTTP response, or deserialization). This string is a separate heap object, not the pooled `"PENDING"` literal.

`==` compares references (memory addresses). The returned String and the literal `"PENDING"` are different objects, so `==` returns `false` even when the content is identical.

**Fix:**
```java
if ("PENDING".equals(status)) {  // null-safe — put literal first
    processOrder();
}
// Or:
if (status != null && status.equals("PENDING")) {
    processOrder();
}
// Or Java 7+ Objects utility:
if (Objects.equals(status, "PENDING")) {
    processOrder();
}
// Or for enum-like constants — consider using an enum:
if (OrderStatus.PENDING.name().equals(status)) {
    processOrder();
}
```

**Key learning:** `==` on Strings only returns `true` for pooled literals or when both references literally point to the same object. Always use `.equals()` for content comparison.

---

## Scenario 2: The Integer Cache Trap

**Context:** A code review reveals this:

```java
public boolean isSpecialUser(Integer userId) {
    Integer specialId = getSpecialUserId();  // returns Integer
    return userId == specialId;
}
```

**Problem:** Works in testing (small IDs), fails in production (large user IDs).

**Question:** Explain the behavior and fix it.

**Answer:**

Java's `Integer.valueOf()` (used by autoboxing) caches Integer objects for values from -128 to 127. For values in this range, `==` returns `true` because both references point to the cached object.

For values outside this range (128+), new Integer objects are created, so `==` compares different objects → returns `false`.

```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true — cached

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false — NOT cached, different objects

// Fix:
public boolean isSpecialUser(Integer userId) {
    Integer specialId = getSpecialUserId();
    return userId.equals(specialId);  // or Objects.equals(userId, specialId)
}
```

**Follow-up:** The cache range can be extended with `-XX:AutoBoxCacheMax=<size>`, but you should never rely on this behavior.

---

## Scenario 3: OutOfMemoryError on Large File Processing

**Context:** A service reads a 2GB CSV file into a List<String> for processing:

```java
List<String> lines = Files.readAllLines(Path.of("large.csv"));
for (String line : lines) {
    process(line);
}
```

**Problem:** Gets `java.lang.OutOfMemoryError: Java heap space` in production.

**Question:** What's wrong and how would you fix it?

**Answer:**

`Files.readAllLines()` loads the entire file into memory. A 2GB file with average 100-byte lines = ~20 million String objects, easily exceeding typical heap sizes.

**Fix — use streaming/lazy reading:**
```java
// Option 1: Stream lines lazily (Java 8+)
try (Stream<String> lines = Files.lines(Path.of("large.csv"))) {
    lines.forEach(line -> process(line));
}  // stream is closed here, resources released

// Option 2: BufferedReader — reads one line at a time
try (BufferedReader reader = new BufferedReader(new FileReader("large.csv"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        process(line);
        // line reference becomes eligible for GC after next iteration
    }
}

// Option 3: For very large files, use parallel processing with bounded queues
```

**Key principle:** Prefer streaming/lazy processing over loading entire datasets into memory.

---

## Scenario 4: Static Field Shared Across Instances

**Context:** A developer writes a counter to track API calls:

```java
public class ApiClient {
    private static int callCount = 0;
    private String baseUrl;

    public ApiClient(String baseUrl) {
        this.baseUrl = baseUrl;
        callCount = 0;  // Reset for each new client
    }

    public String get(String path) {
        callCount++;
        return // ... make HTTP call
    }
}

// Usage:
ApiClient client1 = new ApiClient("https://api1.example.com");
ApiClient client2 = new ApiClient("https://api2.example.com");
client1.get("/users");
client1.get("/orders");
client2.get("/products");  // callCount reset to 0 when client2 created!
```

**Problem:** The call count is incorrect — it gets reset every time a new client is created.

**Question:** Identify the issue and propose two different fixes.

**Answer:**

**Bug:** `callCount` is `static` — shared across ALL instances. When `client2` is created, its constructor resets `callCount = 0`, wiping out `client1`'s count.

**Fix 1 — Make it instance-level (remove static):**
```java
public class ApiClient {
    private int callCount = 0;  // per-instance counter
    private final String baseUrl;

    public ApiClient(String baseUrl) {
        this.baseUrl = baseUrl;
        // Don't reset callCount here — initialized in declaration
    }

    public String get(String path) {
        callCount++;
        return // ...
    }

    public int getCallCount() { return callCount; }
}
```

**Fix 2 — Keep static for global count, use AtomicInteger for thread safety:**
```java
public class ApiClient {
    private static final AtomicInteger globalCallCount = new AtomicInteger(0);
    private final String baseUrl;

    public ApiClient(String baseUrl) {
        this.baseUrl = baseUrl;
        // Don't reset the static counter!
    }

    public String get(String path) {
        globalCallCount.incrementAndGet();
        return // ...
    }
}
```

---

## Scenario 5: Final Variable NPE

**Context:** A developer encounters a NullPointerException:

```java
public class OrderService {
    private final List<String> validators;

    public OrderService() {
        validateOrder(new Order());  // called before validators is set!
        this.validators = new ArrayList<>();
    }

    private void validateOrder(Order order) {
        validators.add("validating...");  // NPE here!
    }
}
```

**Question:** Why does this fail even though `validators` is `final`?

**Answer:**

The `validators` field is `final`, but it hasn't been assigned yet when `validateOrder()` is called in the constructor. The sequence is:
1. Object created — all fields get default values (`null` for reference types)
2. Constructor body starts executing
3. `validateOrder(new Order())` is called — `validators` is still `null`
4. `validators.add(...)` → NullPointerException!
5. `this.validators = new ArrayList<>()` is never reached

`final` only means the field can be assigned **once** — it doesn't help if you try to use it before that one assignment.

**Fix:**
```java
public OrderService() {
    this.validators = new ArrayList<>();  // initialize first
    validateOrder(new Order());           // then use
}
```

**Better design — avoid calling overridable methods from constructors:**
```java
public OrderService() {
    this.validators = new ArrayList<>();
}

public void init() {
    validateOrder(new Order());
}
```

---

## Scenario 6: Memory Leak via String Concatenation in Loop

**Context:** A batch job builds a report string:

```java
String report = "";
for (Order order : orders) {  // 10,000 orders
    report += formatOrder(order);
}
saveReport(report);
```

**Problem:** Memory usage spikes and the job is slow.

**Question:** Explain the memory behavior and provide the fix.

**Answer:**

Each `+=` creates a new `String` object. With 10,000 iterations:
- After iteration 1: `String` of length 100
- After iteration 2: `String` of length 200 (new object)
- After iteration N: `String` of length N*100

The old strings become garbage but haven't been collected yet. At peak, you might have multiple large String objects in memory simultaneously. Creating 10,000 intermediate String objects also puts significant GC pressure.

Time complexity: O(n²) — each concatenation copies the entire string so far.

**Fix:**
```java
// StringBuilder — O(n) time, much less memory pressure
StringBuilder sb = new StringBuilder();
for (Order order : orders) {
    sb.append(formatOrder(order));
}
String report = sb.toString();

// Alternative — String.join or Collectors.joining
String report = orders.stream()
    .map(this::formatOrder)
    .collect(Collectors.joining());

// With delimiter:
String report = orders.stream()
    .map(this::formatOrder)
    .collect(Collectors.joining("\n", "=== Report ===\n", "\n=== End ==="));
```

---

## Scenario 7: ClassCastException in Production

**Context:** A legacy service stores different types in a `Map<String, Object>`:

```java
Map<String, Object> cache = new HashMap<>();
cache.put("userId", 12345L);         // stored as Long
// ... in another service:
Integer userId = (Integer) cache.get("userId");  // ClassCastException!
```

**Question:** Why does this happen? How would you design this more safely?

**Answer:**

`12345L` is a `Long` literal (64-bit). Casting a `Long` to `Integer` throws `ClassCastException` because they are different classes — there is no inheritance relationship between them.

**Safe approaches:**

```java
// Option 1 — use instanceof check
Object value = cache.get("userId");
if (value instanceof Long) {
    Long userId = (Long) value;
    // use userId
}

// Option 2 — use pattern matching (Java 16+)
if (cache.get("userId") instanceof Long userId) {
    // use userId
}

// Option 3 — use Number supertype
Number userId = (Number) cache.get("userId");
int id = userId.intValue();  // safe conversion

// Option 4 — strongly typed wrapper (best design)
record UserContext(long userId, String username) {}
Map<String, UserContext> typedCache = new HashMap<>();
```

**Design principle:** Avoid `Map<String, Object>` for structured data. Use a properly typed domain object or a typed record.

---

## Scenario 8: Pass-by-Value Confusion

**Context:** A developer writes:

```java
void processUser(User user) {
    user.setName("Modified");  // intends to modify
    user = new User("NewUser"); // intends to replace original
}

User user = new User("Original");
processUser(user);
System.out.println(user.getName());  // what does this print?
```

**Question:** What does this print and why?

**Answer:**

Prints: `"Modified"`

**Explanation:**
1. `user` (in main) holds a reference (memory address) to the `User("Original")` object
2. Java copies this reference value into `processUser`'s parameter `user` (local copy)
3. Both the original and the copy point to the SAME object
4. `user.setName("Modified")` modifies the object through the copied reference → visible everywhere
5. `user = new User("NewUser")` makes the LOCAL copy point to a NEW object
6. The original `user` variable in main still points to `User("Original")` → its name is now "Modified"

```
Before processUser:         After user.setName():      After user = new User():
main.user → [Original]      main.user → [Modified]     main.user → [Modified]
                             param.user ↗               param.user → [NewUser]
```

---

## Scenario 9: Immutability Design

**Context:** You need to design a `Money` class that represents a monetary value with currency. It must be safe to share across threads.

**Question:** Design the `Money` class with proper immutability.

**Answer:**

```java
public final class Money {
    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        if (amount == null || currency == null) {
            throw new IllegalArgumentException("amount and currency must not be null");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("amount must be non-negative");
        }
        // Defensive copy of mutable BigDecimal (technically immutable, but good habit)
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency.toUpperCase();
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }

    // Return new object instead of modifying this
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int factor) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(factor)), this.currency);
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Money m)) return false;
        return amount.compareTo(m.amount) == 0 && currency.equals(m.currency);
    }

    @Override
    public int hashCode() { return Objects.hash(amount.stripTrailingZeros(), currency); }

    @Override
    public String toString() { return currency + " " + amount.toPlainString(); }
}
```

---

## Scenario 10: StackOverflowError Analysis

**Context:** An application throws `StackOverflowError` when parsing a deeply nested JSON structure.

```java
Object parseValue(JsonNode node) {
    if (node.isObject()) return parseObject(node);
    if (node.isArray()) return parseArray(node);
    return node.asText();
}

Map<String, Object> parseObject(JsonNode node) {
    Map<String, Object> result = new HashMap<>();
    node.fields().forEachRemaining(entry ->
        result.put(entry.getKey(), parseValue(entry.getValue()))  // recursive!
    );
    return result;
}
```

**Question:** Why does this happen? How would you fix it for arbitrarily deep JSON?

**Answer:**

Each recursive call pushes a new stack frame onto the thread stack. For deeply nested JSON (1000+ levels), the stack runs out of space → `StackOverflowError`.

**Fix 1 — Iterative approach with explicit stack:**
```java
Object parseIterative(JsonNode root) {
    Deque<JsonNode> stack = new ArrayDeque<>();
    Deque<Object> resultStack = new ArrayDeque<>();
    stack.push(root);

    // Iterative tree traversal using explicit stack
    // ... (complex but stack-safe)
}
```

**Fix 2 — Increase thread stack size (temporary band-aid):**
```bash
java -Xss4m MyApplication  # increase stack size to 4MB (default ~512KB-1MB)
```

**Fix 3 — Add depth limit:**
```java
Object parseValue(JsonNode node, int depth) {
    if (depth > 100) throw new IllegalArgumentException("JSON too deeply nested");
    // ...
    parseObject(node, depth + 1);
}
```

**Best approach:** Use an established JSON library (Jackson, Gson) which handles this correctly.

---

## Scenario 11: String Pool Memory Optimization

**Context:** A service loads 1 million log entries, each containing a `status` field (one of: "INFO", "WARN", "ERROR", "DEBUG"). Currently using `String status` per entry.

**Question:** How can you reduce memory usage for the status field?

**Answer:**

If all 1M entries have their status as new `String` objects (from file parsing), each is a separate heap object. 4 distinct values × 1M entries = 1M String objects but only 4 unique values.

**Fix 1 — String.intern():**
```java
LogEntry entry = new LogEntry();
entry.setStatus(parsedStatus.intern()); // deduplicates in the String pool
```

**Fix 2 — Use an enum (best practice):**
```java
enum LogLevel { INFO, WARN, ERROR, DEBUG }

// Each entry references one of 4 enum singletons — tiny memory footprint
LogEntry entry = new LogEntry();
entry.setLevel(LogLevel.valueOf(parsedLevel));
```

**Fix 3 — Manual interning with a Map:**
```java
private static final Map<String, String> STATUS_CACHE = new ConcurrentHashMap<>();

String intern(String s) {
    return STATUS_CACHE.computeIfAbsent(s, k -> k);
}
```

Enum is the clearest approach and provides type safety.

---

## Scenario 12: Variable Shadowing Bug

**Context:**

```java
public class Config {
    private int timeout = 30;

    public Config(int timeout) {
        timeout = timeout;  // Bug!
    }

    public int getTimeout() { return timeout; }
}

Config c = new Config(60);
System.out.println(c.getTimeout());  // prints 30, not 60
```

**Question:** Explain the bug and fix it.

**Answer:**

The line `timeout = timeout;` assigns the parameter `timeout` to itself — it doesn't affect the field `this.timeout`. Both sides refer to the constructor parameter (the parameter shadows the field with the same name).

**Fix:**
```java
public Config(int timeout) {
    this.timeout = timeout;  // explicitly reference the field with 'this.'
}
```

**Best practice:** Use `this.fieldName = paramName` in constructors when parameter names match field names, OR use a naming convention that avoids shadowing:
```java
public Config(int timeoutSeconds) {
    this.timeout = timeoutSeconds;
}
```

---

## Scenario 13: Autoboxing NPE in Arithmetic

**Context:**

```java
public Double calculateDiscount(User user) {
    Double baseRate = discountService.getRate(user.getTier()); // may return null
    return baseRate * 100.0;  // NullPointerException if baseRate is null
}
```

**Question:** Why does NPE occur here? How do you fix it?

**Answer:**

`Double baseRate` is a wrapper type that can be `null`. The expression `baseRate * 100.0` triggers unboxing: `baseRate.doubleValue() * 100.0`. If `baseRate` is `null`, calling `.doubleValue()` throws NPE.

**Fix options:**
```java
// Option 1 — null check
public double calculateDiscount(User user) {
    Double baseRate = discountService.getRate(user.getTier());
    if (baseRate == null) return 0.0;
    return baseRate * 100.0;
}

// Option 2 — use primitive return type where possible
public double calculateDiscount(User user) {
    double baseRate = discountService.getRate(user.getTier());  // use double, not Double
    return baseRate * 100.0;
}

// Option 3 — Optional
public OptionalDouble calculateDiscount(User user) {
    return discountService.getOptionalRate(user.getTier())
        .map(rate -> rate * 100.0);
}

// Option 4 — use Objects.requireNonNullElse
double baseRate = Objects.requireNonNullElse(
    discountService.getRate(user.getTier()), 0.0);
```

---

## Scenario 14: Static Initializer Exception Handling

**Context:**

```java
public class DatabaseConfig {
    private static final Properties DB_PROPS;

    static {
        try {
            DB_PROPS = new Properties();
            DB_PROPS.load(new FileInputStream("db.properties"));
        } catch (IOException e) {
            throw new RuntimeException("Cannot load DB config", e);
        }
    }
}
```

**Question:** What happens if `db.properties` is missing? Can the class ever be loaded again?

**Answer:**

If the static initializer throws an exception, the JVM marks the class initialization as **failed** with an `ExceptionInInitializerError`. Subsequent attempts to use `DatabaseConfig` will throw a `NoClassDefFoundError` — even if the file is now available.

The class can only be reloaded if:
1. A new ClassLoader loads it
2. The JVM restarts

```java
// What you see:
try {
    DatabaseConfig config = new DatabaseConfig();
} catch (ExceptionInInitializerError e) {
    // First attempt: ExceptionInInitializerError wrapping RuntimeException
}

try {
    DatabaseConfig config2 = new DatabaseConfig();
} catch (NoClassDefFoundError e) {
    // Subsequent attempts: NoClassDefFoundError — class is "poisoned"
}
```

**Fix — use lazy initialization or factory pattern:**
```java
public class DatabaseConfig {
    private Properties dbProps;

    public DatabaseConfig(String configPath) throws ConfigException {
        dbProps = new Properties();
        try {
            dbProps.load(new FileInputStream(configPath));
        } catch (IOException e) {
            throw new ConfigException("Cannot load DB config", e);
        }
    }
}
```

---

## Scenario 15: Type Casting in Collections

**Context:**

```java
List<Object> items = new ArrayList<>();
items.add("hello");
items.add(42);
items.add(3.14);

for (Object item : items) {
    String str = (String) item;  // ClassCastException on second element
    System.out.println(str.toUpperCase());
}
```

**Question:** Fix this code to handle all types safely.

**Answer:**

```java
for (Object item : items) {
    // Option 1 — instanceof check
    if (item instanceof String s) {
        System.out.println(s.toUpperCase());
    } else {
        System.out.println(item.toString().toUpperCase());
    }
}

// Option 2 — use toString() for all types
for (Object item : items) {
    System.out.println(String.valueOf(item).toUpperCase());
}

// Option 3 — pattern matching switch (Java 21+)
for (Object item : items) {
    String result = switch (item) {
        case String s  -> s.toUpperCase();
        case Integer i -> "INT:" + i;
        case Double d  -> String.format("%.2f", d).toUpperCase();
        default        -> item.toString();
    };
    System.out.println(result);
}
```

---

## Scenario 16: JVM Warm-Up Effect

**Context:** A developer benchmarks a Java method and gets inconsistent results — first few calls are slow, subsequent calls are fast.

**Question:** Explain this behavior.

**Answer:**

This is **JIT compilation warm-up**:

1. First calls: JVM interprets bytecode — slow (~200ns per call)
2. After ~1000–10000 invocations: JIT detects "hot" method, compiles to native code
3. Subsequent calls: Execute native code — fast (~2ns per call)

For benchmarks, you must **warm up** the JVM before measuring:
```java
// Naive benchmark — wrong
long start = System.nanoTime();
result = myMethod();
long elapsed = System.nanoTime() - start;  // measures interpreted code

// Correct approach — use JMH (Java Microbenchmark Harness)
@Benchmark
@Warmup(iterations = 5)   // JMH runs 5 warm-up iterations before measuring
@Measurement(iterations = 10)
public String myBenchmark() {
    return myMethod();
}
```

In production, this explains why Java apps often feel slow at startup (cold start) but fast once warmed up.

---

## Scenario 17: Primitive Overflow in Calculation

**Context:**

```java
int seconds = 60 * 60 * 24 * 365 * 10;  // 10 years in seconds
System.out.println(seconds);  // prints a negative number!
```

**Question:** Explain and fix.

**Answer:**

`60 * 60 * 24 * 365 * 10 = 315,360,000` which is within `int` range. But wait — let's recalculate: that's only ~315M. That's fine.

However, `60 * 60 * 24 * 365 * 100` (100 years) = 3,153,600,000 > `Integer.MAX_VALUE` (2,147,483,647). The multiplication overflows silently.

The fix is to ensure at least one operand is `long`:
```java
// Bug:
int seconds = 60 * 60 * 24 * 365 * 100;  // overflow if > max int

// Fix — use long literal to force long arithmetic:
long seconds = 60L * 60 * 24 * 365 * 100;  // 3153600000L

// Or explicitly cast:
long seconds = (long) 60 * 60 * 24 * 365 * 100;
```

**Detecting overflow at runtime:**
```java
Math.multiplyExact(60, 60 * 24 * 365 * 100);  // throws ArithmeticException on overflow
```

---

## Scenario 18: Default Values Confusion

**Context:**

```java
public class Order {
    private int quantity;       // not initialized explicitly
    private double price;       // not initialized explicitly
    private boolean paid;       // not initialized explicitly
    private String status;      // not initialized explicitly
}

Order order = new Order();
System.out.println(order.quantity);  // ?
System.out.println(order.price);     // ?
System.out.println(order.paid);      // ?
System.out.println(order.status);    // ?
```

**Question:** What are the default values?

**Answer:**

Instance fields get default values automatically:
- `int quantity` → `0`
- `double price` → `0.0`
- `boolean paid` → `false`
- `String status` → `null`

Default values by type:
- `byte`, `short`, `int`, `long` → `0`
- `float`, `double` → `0.0`
- `boolean` → `false`
- `char` → `'\u0000'` (null character)
- Object references → `null`

**Important:** Local variables do NOT get default values — the compiler will refuse to compile code that reads an uninitialized local variable.

```java
void method() {
    int x;
    System.out.println(x);  // COMPILE ERROR: variable x might not have been initialized
}
```

---

## Scenario 19: Immutable Class Design Challenge

**Context:** Design a `PersonName` class that is immutable and safe to use as a HashMap key.

**Question:** Write the complete implementation.

**Answer:**

```java
public final class PersonName {
    private final String firstName;
    private final String lastName;
    private final int hashCode;  // cached

    public PersonName(String firstName, String lastName) {
        // Validate inputs
        Objects.requireNonNull(firstName, "firstName must not be null");
        Objects.requireNonNull(lastName, "lastName must not be null");
        if (firstName.isBlank()) throw new IllegalArgumentException("firstName must not be blank");
        if (lastName.isBlank()) throw new IllegalArgumentException("lastName must not be blank");

        // Normalize — defensive copy + normalization
        this.firstName = firstName.trim();
        this.lastName = lastName.trim();
        this.hashCode = Objects.hash(this.firstName, this.lastName);  // precompute
    }

    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public String getFullName() { return firstName + " " + lastName; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof PersonName that)) return false;
        return firstName.equals(that.firstName) && lastName.equals(that.lastName);
    }

    @Override
    public int hashCode() { return hashCode; }  // return cached value

    @Override
    public String toString() { return firstName + " " + lastName; }
}
```

---

## Scenario 20: ClassLoader Isolation

**Context:** A plugin system needs to load user-provided JAR files at runtime, with each plugin isolated from others.

**Question:** How would you implement this? What problem does it solve?

**Answer:**

Each plugin needs its own `ClassLoader` instance. This ensures classes loaded by Plugin A don't conflict with same-named classes in Plugin B.

```java
public class PluginLoader {
    public Plugin load(Path jarPath) throws Exception {
        // Each plugin gets its own class loader
        URLClassLoader pluginClassLoader = new URLClassLoader(
            new URL[]{jarPath.toUri().toURL()},
            Thread.currentThread().getContextClassLoader()  // parent
        );

        // Load the plugin's main class
        Class<?> pluginClass = pluginClassLoader.loadClass("com.myplugin.Main");
        Plugin plugin = (Plugin) pluginClass.getDeclaredConstructor().newInstance();
        return plugin;
    }

    public void unload(URLClassLoader pluginClassLoader) throws IOException {
        pluginClassLoader.close();  // Java 7+ — releases JAR file handles
        // Plugin classes become eligible for GC once no more references exist
    }
}
```

**Class identity in Java:** `Class A loaded by ClassLoader X != Class A loaded by ClassLoader Y` — even if same bytecode. This is why plugin frameworks, OSGi, and application servers use separate classloaders.

---

## Scenario 21: String Format vs Concatenation Performance

**Context:** A logging system calls this 10,000 times per second:

```java
logger.debug("User " + userId + " accessed " + endpoint + " at " + timestamp);
```

**Question:** What is the performance issue and how do you fix it?

**Answer:**

Even if debug logging is disabled, the String concatenation always executes. `"User " + userId + " accessed " + endpoint + " at " + timestamp` creates intermediate String objects before calling `logger.debug()`. The logger then discards them.

**Fix 1 — Guard with level check:**
```java
if (logger.isDebugEnabled()) {
    logger.debug("User {} accessed {} at {}", userId, endpoint, timestamp);
}
```

**Fix 2 — Use parameterized logging (SLF4J / Log4j2):**
```java
// Parameterized logging — String not constructed unless log level is enabled
logger.debug("User {} accessed {} at {}", userId, endpoint, timestamp);
```

**Fix 3 — Supplier-based lazy evaluation:**
```java
logger.debug(() -> String.format("User %s accessed %s at %s", userId, endpoint, timestamp));
```

Parameterized logging (option 2) is the standard Java approach. The `{}` placeholders are resolved only if the debug log level is enabled.

---

## Scenario 22: Final Fields and Thread Safety

**Context:** A developer argues that this class is thread-safe because all fields are final:

```java
public final class Config {
    public final List<String> allowedHosts;

    public Config(List<String> hosts) {
        this.allowedHosts = hosts;  // stores reference to external list
    }
}
```

**Question:** Is this class actually thread-safe? Why or why not?

**Answer:**

**No, it's NOT thread-safe.** `final` prevents the `allowedHosts` reference from being changed, but it does NOT prevent the `List` contents from being modified.

If the caller retains a reference to `hosts` and modifies it from another thread, `allowedHosts` changes. If multiple threads read from `allowedHosts` while it's being modified, you get race conditions (`ConcurrentModificationException` or seeing stale data).

**Fix — defensive copy + immutable collection:**
```java
public final class Config {
    public final List<String> allowedHosts;

    public Config(List<String> hosts) {
        // Defensive copy + immutable wrapper
        this.allowedHosts = List.copyOf(hosts);  // Java 10+ — unmodifiable copy
    }
}

// Alternative for Java 8+:
this.allowedHosts = Collections.unmodifiableList(new ArrayList<>(hosts));
```

Now `allowedHosts` is a copy that cannot be modified → truly thread-safe.

---

## Scenario 23: var Usage in Complex Generics

**Context:** A developer writes:

```java
var map = new HashMap<>();  // What's the inferred type?
map.put("key", 123);
```

**Question:** What is the inferred type of `map`?

**Answer:**

The inferred type is `HashMap<Object, Object>` — not `HashMap<String, Integer>`. The compiler infers from `new HashMap<>()` with the diamond operator, but without explicit type parameters, it defaults to `Object`.

```java
var map = new HashMap<>();
// map is HashMap<Object, Object>
map.put("key", 123);            // OK, but both values are Object
Integer value = map.get("key"); // COMPILE ERROR — returns Object, not Integer
Integer value = (Integer) map.get("key"); // ClassCastException risk!

// Correct usage:
var map = new HashMap<String, Integer>();  // type parameters explicitly provided
map.put("key", 123);
Integer value = map.get("key");  // type-safe
```

---

## Scenario 24: Resource Leak in Exception Path

**Context:**

```java
public String readFile(String path) throws IOException {
    InputStream is = new FileInputStream(path);
    // If an exception occurs here, is is never closed!
    String content = new String(is.readAllBytes());
    is.close();
    return content;
}
```

**Question:** Fix this resource leak.

**Answer:**

If `is.readAllBytes()` throws an `IOException`, `is.close()` is never reached. The file handle leaks until GC finalizes the object (unpredictable) or the JVM exits.

**Fix — try-with-resources:**
```java
public String readFile(String path) throws IOException {
    try (InputStream is = new FileInputStream(path)) {
        return new String(is.readAllBytes(), StandardCharsets.UTF_8);
    }  // is.close() called automatically — even if exception occurs
}

// Even cleaner with Files API:
public String readFile(String path) throws IOException {
    return Files.readString(Path.of(path), StandardCharsets.UTF_8);  // Java 11+
}
```

`try-with-resources` calls `close()` on resources even when:
- Normal return
- Exception thrown
- Multiple resources — closed in reverse order of declaration

---

## Scenario 25: Integer.parseInt vs Integer.valueOf

**Context:** A developer parses thousands of small integers from a config file:

```java
List<Integer> ids = new ArrayList<>();
for (String s : configLines) {
    ids.add(Integer.parseInt(s));  // autoboxes result
}
```

**Question:** Is there a performance difference between `Integer.parseInt()` and `Integer.valueOf()`?

**Answer:**

- `Integer.parseInt(String)` returns a primitive `int` — no object creation
- `Integer.valueOf(String)` returns an `Integer` object (uses the cache for -128 to 127, else creates new object)
- `Integer.valueOf(int)` uses the cache for -128 to 127

In this code, `Integer.parseInt(s)` returns a primitive `int`, which is then autoboxed to `Integer` for the `List` — this calls `Integer.valueOf(int)` internally, which uses the cache.

So the two approaches are equivalent here. The autoboxing step always happens when storing in `List<Integer>`.

**For memory efficiency with large numeric lists:**
```java
// Use primitive list to avoid autoboxing entirely:
int[] ids = configLines.stream()
    .mapToInt(Integer::parseInt)
    .toArray();

// Or with Eclipse Collections:
IntList ids = IntLists.mutable.empty();
for (String s : configLines) {
    ids.add(Integer.parseInt(s));
}
```

---

## Scenario 26: Double-Checked Locking Without volatile

**Context:**

```java
public class Singleton {
    private static Singleton instance;  // NOT volatile

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();  // can be partially constructed!
                }
            }
        }
        return instance;
    }
}
```

**Question:** What is wrong with this implementation?

**Answer:**

Without `volatile`, the JVM and CPU can **reorder instructions**. Creating an object `new Singleton()` consists of three steps:
1. Allocate memory
2. Call constructor (initialize fields)
3. Assign reference to `instance`

Without `volatile`, steps 2 and 3 can be reordered: the reference is assigned BEFORE the constructor runs. Thread B can see a non-null `instance` that is not yet fully initialized, and use a broken object.

**Fix:**
```java
public class Singleton {
    private static volatile Singleton instance;  // volatile prevents reordering

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

// Better alternative — initialization-on-demand holder (no volatile needed):
public class Singleton {
    private Singleton() {}

    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return Holder.INSTANCE;  // class loaded lazily, guaranteed thread-safe by JVM
    }
}
```

---

## Scenario 27: Finding the Bug in a Comparison

**Context:**

```java
public boolean isBigOrder(Order order) {
    return order.getTotal().compareTo(BigDecimal.valueOf(1000)) > 1;
}
```

**Question:** Is there a bug? When would this fail?

**Answer:**

Yes — `compareTo()` returns:
- `-1` (or negative) if this < other
- `0` if equal
- `1` (or positive) if this > other

The check `> 1` is wrong. If `order.getTotal()` is exactly 1000, `compareTo` returns `0`. If > 1000, returns `1`. Both cases should be "big orders."

But the condition `> 1` is never true because `compareTo` returns either -1, 0, or 1. So orders over 1000 would return `compareTo() == 1`, which is NOT `> 1`.

**Fix:**
```java
return order.getTotal().compareTo(BigDecimal.valueOf(1000)) > 0;  // > 0, not > 1
```

Or more readable:
```java
return order.getTotal().compareTo(new BigDecimal("1000")) >= 0;  // >= 1000
// or
return order.getTotal().compareTo(BigDecimal.valueOf(999.99)) > 0;  // > 999.99
```

---

## Scenario 28: Enum as Singleton

**Context:** A developer needs a thread-safe singleton configuration holder.

**Question:** Implement a singleton using enum and explain why it's preferred.

**Answer:**

```java
public enum AppConfig {
    INSTANCE;

    private final String dbUrl;
    private final int maxConnections;

    AppConfig() {
        // Constructor called exactly once by JVM during class initialization
        Properties props = loadProperties();
        this.dbUrl = props.getProperty("db.url");
        this.maxConnections = Integer.parseInt(props.getProperty("db.maxConn", "10"));
    }

    public String getDbUrl() { return dbUrl; }
    public int getMaxConnections() { return maxConnections; }

    private Properties loadProperties() {
        // load from classpath
        Properties p = new Properties();
        try (InputStream is = getClass().getResourceAsStream("/app.properties")) {
            p.load(is);
        } catch (IOException e) {
            throw new RuntimeException("Cannot load config", e);
        }
        return p;
    }
}

// Usage:
String url = AppConfig.INSTANCE.getDbUrl();
```

**Why enum singleton is best:**
1. **Thread-safe:** JVM guarantees enum constants are initialized exactly once
2. **Serialization-safe:** Enum singletons survive serialization/deserialization (normal singletons don't without special handling)
3. **Reflection-safe:** Cannot use `AccessibleObject.setAccessible()` to create new instances
4. **Simple:** No double-checked locking, no holder pattern needed
5. **Prevents multiple instantiation:** Guaranteed by the JVM specification
