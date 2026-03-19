# Java Basics — Theory Questions

> 30 theory questions covering JVM architecture, String internals, primitives, keywords, and Java fundamentals.

---

## Q1: What is the difference between JVM, JDK, and JRE?

**Answer:**

- **JRE (Java Runtime Environment):** Provides everything needed to *run* Java programs. Includes the JVM, core libraries (java.lang, java.util, etc.), and supporting files. End-users need only the JRE.
- **JVM (Java Virtual Machine):** The runtime engine that executes Java bytecode. Platform-specific but provides a platform-independent execution environment. Handles memory management, GC, JIT compilation.
- **JDK (Java Development Kit):** Everything needed to *develop* Java programs. Includes the JRE plus development tools: `javac` compiler, `javadoc`, `jar`, debugger, profiler, `jstack`, `jmap`.

```
JDK  ⊃  JRE  ⊃  JVM

JDK = JRE + development tools (javac, javadoc, jar, jshell, jconsole)
JRE = JVM + standard libraries
JVM = execution engine (interpreter + JIT compiler + GC + class loader)
```

Since Java 9, the distinction is less sharp because of the module system — you can create custom runtime images with only the modules you need using `jlink`.

---

## Q2: How does Java achieve platform independence?

**Answer:**

Java achieves platform independence through a two-step compilation process:

1. **Compile time:** `javac` compiles `.java` source files into `.class` bytecode files. Bytecode is an intermediate representation — not machine code for any specific CPU.
2. **Runtime:** The JVM on each target platform interprets/JIT-compiles the bytecode into native machine instructions.

```
Source code (.java)
        │
        ▼ javac
Bytecode (.class)  ← platform-independent
        │
        ▼ JVM (platform-specific)
Native machine code  ← runs on specific OS + CPU
```

The JVM is platform-specific (there are different JVM implementations for Windows, Linux, macOS, ARM), but the bytecode it executes is universal. This is the "write once, run anywhere" principle.

**Important:** The JVM itself is NOT platform-independent — it is a native binary compiled for each OS/architecture.

---

## Q3: What are the different memory areas managed by the JVM?

**Answer:**

The JVM divides memory into several runtime data areas:

1. **Heap:** Stores all objects and arrays. Shared across all threads. Garbage collected. Divided into Young Generation (Eden + Survivor) and Old Generation (Tenured).

2. **Method Area (Metaspace in Java 8+):** Stores class metadata, method bytecodes, constant pool, static fields. Shared across threads. In Java 8+, lives in native memory (not the heap), sized with `-XX:MaxMetaspaceSize`.

3. **Stack (Thread Stack):** Each thread has its own stack. Stores stack frames for each method invocation. Each frame holds local variables, operand stack, and frame data. Fixed size — `StackOverflowError` if exhausted.

4. **PC Register:** Each thread has its own Program Counter. Holds the address of the currently executing JVM instruction.

5. **Native Method Stack:** Used for native (non-Java) method calls via JNI.

6. **Runtime Constant Pool:** Part of the Method Area. Per-class version of the constant pool from the .class file.

**Common flags:**
- `-Xmx` — max heap size
- `-Xms` — initial heap size
- `-Xss` — thread stack size
- `-XX:MaxMetaspaceSize` — max metaspace

---

## Q4: What is the difference between Stack and Heap memory in Java?

**Answer:**

| Feature | Stack | Heap |
|---------|-------|------|
| Stores | Primitive variables, references, method frames | All objects, arrays |
| Thread-sharing | Per-thread (private) | Shared across all threads |
| Allocation | LIFO (automatic) | Dynamic (GC managed) |
| Size | Smaller (default ~512KB–1MB) | Larger (configured via -Xmx) |
| Error | StackOverflowError | OutOfMemoryError |
| Speed | Faster (no GC) | Slower (GC overhead) |
| Lifetime | Tied to method scope | Until GC'd |

```java
void example() {
    int x = 10;                    // x is on the Stack
    String s = "hello";            // s (reference) is on Stack,
                                   // "hello" object is on Heap
    User user = new User("Alice");  // user reference on Stack,
                                   // User object on Heap
}  // x, s, user references popped off Stack when method returns
   // Objects on Heap become eligible for GC
```

---

## Q5: Explain the String Pool in Java.

**Answer:**

The String Pool (also called String Intern Pool or String Constant Pool) is a special cache area in the JVM's heap (since Java 7; was in PermGen before Java 7) that stores unique string literals.

**How it works:**
1. When you write `String s = "hello"`, the JVM checks the pool for an existing `"hello"` object.
2. If found, it returns a reference to the existing object.
3. If not found, it creates a new `String` object in the pool and returns its reference.

```java
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");   // Forces heap allocation outside pool
String s4 = s3.intern();           // Manually interns — gets pool reference

System.out.println(s1 == s2);      // true  — same pool object
System.out.println(s1 == s3);      // false — s3 is a separate heap object
System.out.println(s1 == s4);      // true  — intern() returns pool reference
System.out.println(s1.equals(s3)); // true  — content is the same
```

**Why String immutability is required for the pool:** If strings were mutable, one reference modifying the object would corrupt all other references pointing to the same pool entry.

---

## Q6: Why is String immutable in Java?

**Answer:**

String is immutable (its internal `char[]` / `byte[]` cannot be changed after construction) for several important reasons:

1. **String Pool safety:** The pool relies on multiple references pointing to the same object. If one reference could modify the string, it would affect all other references.

2. **Security:** Strings are used as class names, database connection URLs, file paths, credentials. If they were mutable, malicious code could modify them after security checks but before actual use.

3. **Thread safety:** Immutable objects are inherently thread-safe. A String can be shared across threads without synchronization.

4. **HashCode caching:** HashMap uses String keys heavily. String caches its `hashCode()` after the first computation — only safe because the value can't change.

5. **ClassLoader safety:** Class loading uses String class names. Mutable names could lead to loading a different class than intended.

```java
// String internals (simplified)
public final class String {            // final = cannot be subclassed
    private final byte[] value;        // final = reference cannot be reassigned
    private int hash;                  // cached hashCode

    // No setter for value — value is set in constructor and never changed
}
```

---

## Q7: What is the difference between `==` and `.equals()` for Strings?

**Answer:**

- **`==` (reference equality):** Compares memory addresses. Returns `true` only if both references point to the **same object** in memory.
- **`.equals()` (content equality):** Compares the actual character contents of two strings.

```java
String a = "hello";
String b = "hello";        // from pool — same reference as a
String c = new String("hello");  // new heap object

System.out.println(a == b);       // true  — both reference pool "hello"
System.out.println(a == c);       // false — different objects
System.out.println(a.equals(c));  // true  — same content

// Common trap:
String x = null;
x.equals("hello");   // NullPointerException
"hello".equals(x);   // false — safe pattern, use this
Objects.equals(x, "hello"); // false — null-safe comparison
```

**Interview trap:** Always use `.equals()` for string value comparison. Use `==` only when you intentionally want to check identity.

---

## Q8: What is autoboxing and unboxing? What are the pitfalls?

**Answer:**

**Autoboxing:** Automatic conversion of primitive types to their corresponding wrapper objects.
**Unboxing:** Automatic conversion of wrapper objects back to primitives.

```java
// Autoboxing
Integer i = 5;          // equivalent to Integer.valueOf(5)
List<Integer> list = new ArrayList<>();
list.add(42);           // int 42 is autoboxed to Integer(42)

// Unboxing
int x = i;              // equivalent to i.intValue()
int sum = list.get(0);  // unboxed automatically
```

**Pitfalls:**

1. **Integer cache (-128 to 127):**
```java
Integer a = 127;
Integer b = 127;
System.out.println(a == b);  // true — cached

Integer c = 128;
Integer d = 128;
System.out.println(c == d);  // false — different objects!
System.out.println(c.equals(d));  // true — always use equals()
```

2. **NullPointerException on unboxing:**
```java
Integer count = null;
int total = count + 1;  // NullPointerException! Unboxing null throws NPE
```

3. **Performance overhead:** Autoboxing in tight loops creates many short-lived objects, increasing GC pressure.

4. **`==` vs `.equals()` for wrappers:** Never use `==` to compare Integer/Long/etc. objects.

---

## Q9: Is Java pass-by-value or pass-by-reference?

**Answer:**

**Java is always pass-by-value.** However, the value being passed for objects is the *reference* (memory address), which causes confusion.

```java
// Primitives: pure pass-by-value
void increment(int x) {
    x++;  // modifies local copy only
}
int a = 5;
increment(a);
System.out.println(a);  // still 5

// Objects: pass-by-value of the reference
void modify(StringBuilder sb) {
    sb.append(" World");  // modifies the object the reference points to
}
StringBuilder sb = new StringBuilder("Hello");
modify(sb);
System.out.println(sb);  // "Hello World" — object was modified

// But reassigning the reference has no effect on the original
void reassign(StringBuilder sb) {
    sb = new StringBuilder("New");  // local reference now points elsewhere
}
StringBuilder sb2 = new StringBuilder("Original");
reassign(sb2);
System.out.println(sb2);  // "Original" — sb2 still points to original object
```

**Key point:** Java copies the reference value (the address), not the object. You can call methods on the object through the copied reference, but you cannot make the original variable point to a different object.

---

## Q10: What is the `static` keyword used for in Java?

**Answer:**

`static` means belonging to the class rather than to a specific instance.

**1. Static fields:** One copy shared across all instances.
```java
class Counter {
    static int count = 0;     // shared by all instances
    int id;

    Counter() { this.id = ++count; }
}
```

**2. Static methods:** Called on the class, not on an instance. Cannot access instance variables or `this`.
```java
Math.abs(-5);        // static method — no Math object needed
Integer.parseInt("42");
```

**3. Static blocks:** Runs once when the class is first loaded.
```java
class Config {
    static final Properties props;
    static {
        props = new Properties();
        props.load(...);  // initialization at class load time
    }
}
```

**4. Static inner classes:** Can be instantiated without an outer class instance.
```java
class Outer {
    static class Inner {  // can be used without Outer instance
        void doSomething() { }
    }
}
Outer.Inner inner = new Outer.Inner();  // no Outer instance needed
```

**5. Static imports:**
```java
import static java.lang.Math.PI;
import static java.util.Collections.sort;
```

---

## Q11: What is the `final` keyword in Java?

**Answer:**

`final` prevents modification depending on what it is applied to:

**1. final variable:** Can be assigned only once. Must be assigned before constructor completes.
```java
final int MAX = 100;          // must be assigned
final List<String> list = new ArrayList<>();
list.add("hello");            // OK — list contents can change
list = new ArrayList<>();     // ERROR — reference cannot be reassigned
```

**2. final method:** Cannot be overridden in subclasses.
```java
class Base {
    final void log() { System.out.println("Base log"); }
}
class Child extends Base {
    // void log() { }  // COMPILE ERROR — cannot override final method
}
```

**3. final class:** Cannot be subclassed. String, Integer, and most wrapper classes are final.
```java
public final class String { ... }
// class MyString extends String { }  // COMPILE ERROR
```

**Interview point:** `final` does NOT make an object immutable — it only prevents reassignment of the reference. The object's mutable state can still change.

---

## Q12: What are access modifiers and what do they control?

**Answer:**

| Modifier | Same Class | Same Package | Subclass (diff pkg) | Other Packages |
|----------|-----------|--------------|---------------------|----------------|
| `public` | ✓ | ✓ | ✓ | ✓ |
| `protected` | ✓ | ✓ | ✓ | ✗ |
| (package-private) | ✓ | ✓ | ✗ | ✗ |
| `private` | ✓ | ✗ | ✗ | ✗ |

```java
public class BankAccount {
    private double balance;        // only this class
    protected String accountType;  // subclasses + package
    String branch;                 // package-private
    public String owner;           // everyone

    protected void audit() { }    // subclasses can call
    private void encrypt() { }    // internal use only
}
```

**Best practice:** Default to the most restrictive access. Use `private` for all fields, expose only through getter/setter methods.

---

## Q13: What is the `volatile` keyword and when should you use it?

**Answer:**

`volatile` ensures that reads and writes to a variable are always done directly from/to main memory, not from a thread's CPU cache. It provides:
1. **Visibility guarantee:** Changes made by one thread are immediately visible to other threads.
2. **Prevents reordering:** Volatile reads/writes cannot be reordered with other volatile reads/writes.

`volatile` does NOT provide atomicity for compound operations (check-then-act, increment, etc.).

```java
class Singleton {
    private static volatile Singleton instance;  // volatile needed here

    public static Singleton getInstance() {
        if (instance == null) {                    // First check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {            // Second check (with lock)
                    instance = new Singleton();    // volatile ensures partial
                }                                  // construction not visible
            }
        }
        return instance;
    }
}
```

**When to use volatile:**
- Simple flags (stop flag for a thread)
- Status indicators
- Double-checked locking pattern
- **Not for** incrementing counters (use `AtomicInteger` instead)

---

## Q14: What is the difference between `transient` and `volatile`?

**Answer:**

- **`transient`:** Applies to fields that should be excluded from Java serialization. When an object is serialized, transient fields are not written to the output stream and will have their default values when deserialized.

- **`volatile`:** A threading keyword that ensures visibility of changes across threads (see Q13).

```java
class User implements Serializable {
    String username;
    transient String password;    // not serialized — security best practice
    transient Connection conn;    // not serializable — skip it
    volatile boolean active;      // threading visibility
}
```

---

## Q15: What is type casting in Java and what are its risks?

**Answer:**

**Widening conversion (implicit):** No data loss possible.
```java
int i = 42;
long l = i;    // implicit widening — always safe
double d = i;  // implicit widening
```

**Narrowing conversion (explicit):** May lose data.
```java
double d = 9.99;
int i = (int) d;   // explicit — i = 9 (truncated, not rounded!)
long big = 1234567890123L;
int small = (int) big;  // data loss if value doesn't fit
```

**Object casting:**
```java
Object obj = "hello";
String s = (String) obj;    // ClassCastException if obj is not String

// Safe casting with instanceof:
if (obj instanceof String str) {  // Java 16+ pattern matching
    System.out.println(str.length());
}
```

**`instanceof` operator:**
```java
Animal a = new Dog();
System.out.println(a instanceof Dog);    // true
System.out.println(a instanceof Animal); // true
System.out.println(a instanceof Cat);    // false

// Java 16+ pattern matching:
if (a instanceof Dog d) {
    d.bark();  // d is already cast
}
```

---

## Q16: What is the difference between `int` and `Integer`?

**Answer:**

| Feature | `int` | `Integer` |
|---------|-------|-----------|
| Type | Primitive | Object (wrapper class) |
| Memory | 4 bytes on stack | ~16 bytes on heap |
| Default value | 0 | null |
| Can be null | No | Yes |
| Generic type parameter | No (use Integer) | Yes |
| Arithmetic | Direct | Unboxing required |
| Comparison | == is safe | Must use .equals() |
| Methods | None | parseInt, valueOf, compareTo, etc. |

```java
int a = 0;        // default is 0, never null
Integer b = null; // can be null
Integer c = 5;
int d = c;        // unboxing — NPE if c is null!

// In generics, must use wrapper:
List<Integer> list = new ArrayList<>();  // List<int> is invalid
Map<String, Integer> map = new HashMap<>();
```

---

## Q17: Explain the `instanceof` operator and pattern matching (Java 16+).

**Answer:**

`instanceof` checks if an object is an instance of a given class or implements an interface.

```java
// Traditional usage
Object obj = getSomeObject();
if (obj instanceof String) {
    String s = (String) obj;  // explicit cast still needed
    System.out.println(s.length());
}

// Java 16+ Pattern Matching for instanceof
if (obj instanceof String s) {  // declares and casts in one step
    System.out.println(s.length());  // s is in scope here
}

// Useful with sealed classes (Java 17+)
sealed interface Shape permits Circle, Rectangle, Triangle {}

double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
};
```

**Interview note:** `instanceof` returns false if the object is `null` — it does not throw NPE.
```java
String s = null;
System.out.println(s instanceof String);  // false, not NPE
```

---

## Q18: What is the `var` keyword introduced in Java 10?

**Answer:**

`var` enables local variable type inference — the compiler infers the type from the right-hand side of the assignment.

```java
// Instead of:
ArrayList<String> list = new ArrayList<String>();

// You can write:
var list = new ArrayList<String>();  // type inferred as ArrayList<String>

// More examples:
var map = new HashMap<String, List<Integer>>();
var stream = list.stream().filter(s -> s.startsWith("A"));
var entry = map.entrySet().iterator().next();

// In for-each:
for (var entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}
```

**Restrictions:**
- Only for **local variables** — not for fields, method parameters, or return types
- **Cannot** be used with `null` (no type to infer from)
- **Cannot** be used for array initializers without explicit type: `var arr = {1, 2, 3}` is invalid

```java
var x = null;              // COMPILE ERROR — cannot infer type
var arr = {1, 2, 3};       // COMPILE ERROR — array initializer needs type
var arr = new int[]{1, 2}; // OK — type inferred as int[]
```

**Not a dynamic type:** `var` is still strongly typed. The type is determined at compile time and cannot change.

---

## Q19: What is the difference between `==` for primitives vs objects?

**Answer:**

For **primitives**, `==` compares values directly:
```java
int a = 5;
int b = 5;
System.out.println(a == b);  // true — same value
```

For **objects**, `==` compares references (memory addresses):
```java
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1 == s2);      // false — different heap objects
System.out.println(s1.equals(s2)); // true  — same content

// Exception: String literals from the pool
String s3 = "hello";
String s4 = "hello";
System.out.println(s3 == s4);  // true — same pool object
```

**Important classes where `==` is deceptive:**
- `String` (pool caching)
- `Integer`, `Long`, etc. (cached -128 to 127)
- Enums (enums are singletons — `==` and `.equals()` both work)

**Rule of thumb:** Use `==` for primitives and enums. Use `.equals()` for all objects.

---

## Q20: What happens when `hashCode()` is not overridden but `equals()` is?

**Answer:**

The `hashCode()` contract states: if `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` **must** be true.

If you override `equals()` without `hashCode()`, objects that are logically equal will have different hash codes. This breaks HashMap and HashSet behavior:

```java
class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        Point p = (Point) o;
        return this.x == p.x && this.y == p.y;
    }
    // NO hashCode override!
}

Map<Point, String> map = new HashMap<>();
Point p1 = new Point(1, 2);
map.put(p1, "origin");

Point p2 = new Point(1, 2);
System.out.println(p1.equals(p2)); // true — custom equals
System.out.println(map.get(p2));   // null! — different hashCodes,
                                   // so looked in wrong bucket
```

**Always override both or neither.** IDEs and Lombok can generate correct implementations.

---

## Q21: What is classloading and what are the classloader types?

**Answer:**

Classloading is the process of finding a class definition, loading it into the JVM, and making it available. It has three phases: Loading, Linking (Verification, Preparation, Resolution), and Initialization.

**Three built-in classloaders:**

1. **Bootstrap ClassLoader:** The parent of all. Loads core Java classes (`java.lang.*`, `java.util.*`) from `rt.jar` / modules. Written in native code.

2. **Extension ClassLoader (Platform ClassLoader in Java 9+):** Loads classes from `$JAVA_HOME/lib/ext` or platform modules.

3. **Application ClassLoader (System ClassLoader):** Loads classes from the application's classpath (`-classpath`). This is the classloader for most user code.

**Delegation model (Parent First):**
```
Application ClassLoader
        │ delegates to parent first
        ▼
Extension/Platform ClassLoader
        │ delegates to parent first
        ▼
Bootstrap ClassLoader
        │ if not found, returns to child
        ▼
  ClassNotFoundException
```

Classes loaded by parent classloaders are always preferred — prevents malicious code from replacing java.lang.String.

---

## Q22: What is JIT compilation and how does it work?

**Answer:**

**JIT (Just-In-Time) compiler** converts frequently executed bytecode into native machine code at runtime to improve performance.

**Process:**
1. JVM starts by interpreting bytecode (slow but immediate)
2. JIT monitors which methods are called frequently (hot spots)
3. Hot methods are compiled to native code and cached
4. Subsequent calls use native code (much faster)

**Two-tier compilation (Java HotSpot):**
- **C1 (Client compiler):** Fast compilation, basic optimizations. Used initially.
- **C2 (Server compiler):** Slower compilation, aggressive optimizations (inlining, escape analysis, loop unrolling). Used for very hot code.

**Key optimizations JIT performs:**
- **Method inlining:** Replaces method call with method body
- **Dead code elimination:** Removes code that can never execute
- **Loop unrolling:** Expands loop bodies to reduce iteration overhead
- **Escape analysis:** Allocates objects on stack instead of heap if they don't escape

---

## Q23: What is Garbage Collection and what are the main GC algorithms?

**Answer:**

Garbage Collection automatically reclaims heap memory occupied by objects that are no longer reachable (no live reference path from GC roots).

**GC Roots:** Static variables, local variables in running threads, JNI references.

**Main GC algorithms in HotSpot JVM:**

1. **Serial GC (`-XX:+UseSerialGC`):** Single-threaded. Stop-the-world. Best for small heaps, single-core.

2. **Parallel GC (`-XX:+UseParallelGC`):** Multi-threaded throughput-focused GC. Default in Java 8. Stop-the-world but faster than Serial.

3. **G1 GC (`-XX:+UseG1GC`):** Default since Java 9. Divides heap into regions. Predictable pause times. Concurrent marking, incremental collection.

4. **ZGC (`-XX:+UseZGC`):** Available Java 11+. Near-pause-free (<10ms pauses regardless of heap size). Concurrent compaction. Good for large heaps.

5. **Shenandoah:** Similar to ZGC, concurrent compaction. Available in OpenJDK.

**GC tuning flags:**
```bash
-Xmx4g                        # max heap
-XX:+UseG1GC                  # use G1
-XX:MaxGCPauseMillis=200       # target max pause
-XX:GCTimeRatio=4              # target throughput: 80% application time
-Xlog:gc*:file=gc.log          # GC logging
```

---

## Q24: What are Java records (Java 16+)?

**Answer:**

Records are a concise way to declare immutable data carrier classes. The compiler auto-generates constructor, `equals()`, `hashCode()`, `toString()`, and accessor methods.

```java
// Before records:
public final class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) { this.x = x; this.y = y; }
    public int x() { return x; }
    public int y() { return y; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// With records (Java 16+):
public record Point(int x, int y) {}

// Usage:
Point p = new Point(3, 4);
System.out.println(p.x());         // 3
System.out.println(p);             // Point[x=3, y=4]
System.out.println(new Point(3,4).equals(new Point(3,4)));  // true
```

**Custom compact constructor:**
```java
public record Range(int min, int max) {
    Range {  // compact constructor — no parentheses
        if (min > max) throw new IllegalArgumentException("min > max");
    }
}
```

**Limitations:**
- Cannot extend other classes (implicitly extends `java.lang.Record`)
- All components are `final` — truly immutable
- Cannot declare instance fields outside the record header

---

## Q25: What are sealed classes (Java 17+)?

**Answer:**

Sealed classes restrict which classes can extend or implement them. Combined with `pattern matching switch`, they enable exhaustive type checking.

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double width, double height) implements Shape {}
public record Triangle(double base, double height) implements Shape {}

// Exhaustive switch — compiler enforces all cases covered:
double area = switch (shape) {
    case Circle c    -> Math.PI * c.radius() * c.radius();
    case Rectangle r -> r.width() * r.height();
    case Triangle t  -> 0.5 * t.base() * t.height();
    // No default needed — compiler knows all subtypes!
};
```

**permits clause rules:**
- Permitted subclasses must be in the same package or module
- Permitted subclasses must `extends`/`implements` the sealed type
- Permitted subclasses must be: `final`, `sealed`, or `non-sealed`

**Benefits:**
- Better modeling of algebraic data types (like sum types)
- Compiler-enforced exhaustiveness in pattern matching
- Clear documentation of intended type hierarchy

---

## Q26: What is the difference between `String`, `StringBuilder`, and `StringBuffer`?

**Answer:**

| Feature | String | StringBuilder | StringBuffer |
|---------|--------|--------------|--------------|
| Mutability | Immutable | Mutable | Mutable |
| Thread safety | Thread-safe (immutable) | Not thread-safe | Thread-safe (synchronized) |
| Performance | Slow for concatenation | Fast | Slower than StringBuilder |
| Use case | String literals, keys | Single-threaded string building | Multi-threaded string building |
| Since | Java 1.0 | Java 1.5 | Java 1.0 |

```java
// String — new object for each concat:
String s = "Hello";
s += " World";  // creates new String object, old one becomes garbage

// StringBuilder — same buffer, efficient:
StringBuilder sb = new StringBuilder("Hello");
sb.append(" World");
sb.insert(0, "Say: ");
sb.delete(0, 5);
String result = sb.toString();

// StringBuffer — same as StringBuilder but synchronized:
StringBuffer sbuf = new StringBuffer();
sbuf.append("thread-safe");
```

**Rule:** Use `StringBuilder` for single-threaded string building. The compiler automatically uses `StringBuilder` for simple `+` concatenation in a single expression, but a loop with `+` creates many intermediate objects.

---

## Q27: What is autoboxing's performance impact and how can you mitigate it?

**Answer:**

Autoboxing creates wrapper objects on the heap, causing GC pressure in hot paths.

```java
// Bad — autoboxing on every iteration
long sum = 0;
for (Integer i : largeList) {  // unboxing Integer → int on each access
    sum += i;                   // boxing if sum were Long
}

// Better — use primitive streams
OptionalLong sum = largeList.stream()
    .mapToLong(Integer::longValue)
    .sum();

// Best for large numeric collections — use int[] or primitive arrays
int[] array = new int[1000];
long sum = 0;
for (int i : array) {  // no boxing/unboxing
    sum += i;
}
```

**Third-party solutions:**
- Eclipse Collections (primitives collections)
- Trove4j (primitive hash maps/lists)
- Agrona (high-performance primitive data structures)

---

## Q28: What are the rules for `hashCode()` and `equals()` contracts?

**Answer:**

**equals() contract (from Object javadoc):**
1. **Reflexive:** `x.equals(x)` must return `true`
2. **Symmetric:** if `x.equals(y)` then `y.equals(x)`
3. **Transitive:** if `x.equals(y)` and `y.equals(z)` then `x.equals(z)`
4. **Consistent:** multiple calls return same result if object state unchanged
5. **Null check:** `x.equals(null)` must return `false`, not NPE

**hashCode() contract:**
1. If `a.equals(b)` → `a.hashCode() == b.hashCode()` (mandatory)
2. If `a.hashCode() == b.hashCode()` → `a.equals(b)` is NOT required (hash collision is OK)

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;           // reflexive + identity optimization
    if (!(o instanceof MyClass)) return false; // null-safe + type check
    MyClass that = (MyClass) o;
    return this.id == that.id && Objects.equals(this.name, that.name);
}

@Override
public int hashCode() {
    return Objects.hash(id, name);  // uses same fields as equals!
}
```

---

## Q29: What is the difference between primitive types in terms of memory and range?

**Answer:**

| Type | Size | Range |
|------|------|-------|
| `boolean` | JVM-dependent (typically 1 byte) | true / false |
| `byte` | 1 byte (8 bits) | -128 to 127 |
| `short` | 2 bytes (16 bits) | -32,768 to 32,767 |
| `char` | 2 bytes (16 bits) | 0 to 65,535 (Unicode code units) |
| `int` | 4 bytes (32 bits) | -2,147,483,648 to 2,147,483,647 |
| `long` | 8 bytes (64 bits) | -9.2 × 10^18 to 9.2 × 10^18 |
| `float` | 4 bytes (32 bits) | ~1.4 × 10^-45 to ~3.4 × 10^38 |
| `double` | 8 bytes (64 bits) | ~4.9 × 10^-324 to ~1.8 × 10^308 |

```java
// Common pitfalls:
int maxInt = Integer.MAX_VALUE;
int overflow = maxInt + 1;  // -2147483648 (overflows silently!)

long l = 9999999999L;  // need L suffix for long literals > Integer.MAX_VALUE
float f = 3.14f;       // need f suffix, otherwise interpreted as double
double d = 3.14;       // default decimal type is double

// Integer overflow detection (Java 8+):
Math.addExact(Integer.MAX_VALUE, 1);  // throws ArithmeticException
```

---

## Q30: What is the difference between compile-time and runtime polymorphism?

**Answer:**

**Compile-time polymorphism (static binding / method overloading):**
The method to call is determined at compile time based on the declared type of the reference and the method signature.

```java
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }  // overloaded
    int add(int a, int b, int c) { return a + b + c; } // overloaded
}
Calculator calc = new Calculator();
calc.add(1, 2);        // compiler picks int version at compile time
calc.add(1.0, 2.0);    // compiler picks double version at compile time
```

**Runtime polymorphism (dynamic binding / method overriding):**
The method to call is determined at runtime based on the actual type of the object, not the declared type of the reference.

```java
class Animal {
    void sound() { System.out.println("..."); }
}
class Dog extends Animal {
    @Override void sound() { System.out.println("Woof"); }
}
class Cat extends Animal {
    @Override void sound() { System.out.println("Meow"); }
}

Animal a = new Dog();  // declared as Animal, actual type is Dog
a.sound();             // "Woof" — runtime determines Dog.sound() is called
```

The JVM uses a **vtable (virtual method table)** per class to dispatch overridden methods at runtime.
