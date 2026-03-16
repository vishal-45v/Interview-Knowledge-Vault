# Chapter 10: Reflection — Follow-Up Traps

Tricky questions interviewers ask after you describe the Reflection API.

---

## Trap 1: getDeclaredMethods vs getMethods — Which Includes Inherited?

**Interviewer:** "I have a Dog that extends Animal. I call `getDeclaredMethods()` on Dog. Will I get Animal's methods?"

**The Trap:** Candidates confuse "declared" with "all accessible."

**Answer:** No. `getDeclaredMethods()` returns only methods declared directly in `Dog` — including private ones but NOT inherited ones. `getMethods()` returns all public methods including inherited ones, but excludes private methods. To get all methods including private inherited ones, you must walk the hierarchy yourself.

```java
class Animal {
    public void breathe() {}
    protected void digest() {}
    private void circulateBlood() {}
}

class Dog extends Animal {
    public void bark() {}
    private void wag() {}
}

// getDeclaredMethods() — only Dog's own methods
Method[] declared = Dog.class.getDeclaredMethods();
// Returns: [bark, wag]   ← Dog's methods only (incl. private wag)
// Does NOT include: breathe, digest, circulateBlood  ← Animal's methods

// getMethods() — public methods from Dog AND inherited public
Method[] all = Dog.class.getMethods();
// Returns: [bark, breathe, toString, hashCode, equals, getClass, ...]
// Does NOT include: wag (private), digest (protected), circulateBlood (private)

// To get ALL methods including private in full hierarchy:
List<Method> allMethods = new ArrayList<>();
Class<?> cls = Dog.class;
while (cls != null) {
    allMethods.addAll(Arrays.asList(cls.getDeclaredMethods()));
    cls = cls.getSuperclass(); // walk up the hierarchy
}
// Now allMethods includes: bark, wag, breathe, digest, circulateBlood
```

---

## Trap 2: setAccessible(true) in Java 9+ Modules

**Interviewer:** "Your code uses `setAccessible(true)` to access a private field and works in Java 8. After upgrading to Java 17, it throws an exception. What happened?"

**The Trap:** Candidates say "setAccessible always works" or "just add --illegal-access=permit."

**Answer:** Java 9 introduced the module system with strong encapsulation. From Java 17, `--illegal-access` is removed entirely. If the target class is in a module that does not `open` its package, `setAccessible(true)` throws `InaccessibleObjectException`. The fix is to either open the package in `module-info.java` or use `--add-opens` as a JVM flag.

```java
// Java 8 — always worked
Field f = String.class.getDeclaredField("value");
f.setAccessible(true); // OK

// Java 17 without --add-opens — throws:
// java.lang.reflect.InaccessibleObjectException:
//   Unable to make field private final byte[] java.lang.String.value accessible:
//   module java.base does not "opens java.lang" to unnamed module

// Fix 1: JVM flag (for classpath apps, not recommended for production)
// --add-opens java.base/java.lang=ALL-UNNAMED

// Fix 2: module-info.java (for your own modules)
// module com.myapp {
//     opens com.myapp.internal to testing.framework;
// }

// Fix 3: check before accessing (graceful degradation)
try {
    Field field = SomeClass.class.getDeclaredField("privateField");
    field.setAccessible(true);
    Object value = field.get(instance);
} catch (InaccessibleObjectException e) {
    // Java 9+ module encapsulation
    log.warn("Cannot access field via reflection: {}", e.getMessage());
    // Use alternative approach
} catch (NoSuchFieldException | IllegalAccessException e) {
    log.error("Reflection error", e);
}
```

---

## Trap 3: Reflection and Performance — When Is It Too Slow?

**Interviewer:** "You are processing 10 million records using a reflection-based mapper. Is this a problem?"

**The Trap:** Candidates say "reflection is fast enough" without thinking about the hot path.

**Answer:** Yes. In a tight loop over 10 million records, the overhead of reflection (security checks, boxing of primitives, lack of JIT inlining) becomes significant — potentially 10-100x slower than direct calls. The fix is to cache `Method`/`Field` objects (not look them up per record), or to use `MethodHandle` which the JIT can optimize, or to use code generation (ByteBuddy, LambdaMetafactory).

```java
// SLOW — Method lookup inside the loop (500+ ns per call vs 1 ns direct)
for (Record r : tenMillionRecords) {
    Method setter = r.getClass().getMethod("setValue", String.class); // EXPENSIVE lookup
    setter.invoke(r, "value");
}

// FASTER — cache the Method (5-10 ns per call)
Method setter = Record.class.getMethod("setValue", String.class);
setter.setAccessible(true);
for (Record r : tenMillionRecords) {
    setter.invoke(r, "value"); // still has security check overhead + boxing
}

// FAST — MethodHandle (2-3 ns per call, JIT-inlinable)
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle setterHandle = lookup.findVirtual(
    Record.class, "setValue",
    MethodType.methodType(void.class, String.class));

for (Record r : tenMillionRecords) {
    setterHandle.invoke(r, "value"); // JIT can inline this
}

// FASTEST — LambdaMetafactory (near-native, ~1 ns per call)
// Creates a functional interface backed by the target method
MethodHandles.Lookup lookup2 = MethodHandles.lookup();
MethodHandle target = lookup2.findVirtual(Record.class, "setValue",
    MethodType.methodType(void.class, String.class));

BiConsumer<Record, String> setter2 = (BiConsumer<Record, String>)
    LambdaMetafactory.metafactory(
        lookup2,
        "accept",
        MethodType.methodType(BiConsumer.class),
        MethodType.methodType(void.class, Object.class, Object.class),
        target,
        MethodType.methodType(void.class, Record.class, String.class)
    ).getTarget().invokeExact();

for (Record r : tenMillionRecords) {
    setter2.accept(r, "value"); // compiled to direct invocation by JIT
}
```

---

## Trap 4: getDeclaredField vs getField

**Interviewer:** "What is the difference between `getDeclaredField("name")` and `getField("name")`?"

**The Trap:** Same as the method trap — candidates assume they are symmetric.

**Answer:** `getDeclaredField()` finds any field (public, protected, private, package) declared directly in the class — but does NOT search superclasses. `getField()` searches the class AND all superclasses AND interfaces, but only for public fields.

```java
class Base {
    public String publicField = "base-public";
    private String privateField = "base-private";
}

class Child extends Base {
    public String childField = "child-public";
    private String childPrivate = "child-private";
}

// getDeclaredField — only Child's fields, any visibility
Child.class.getDeclaredField("childField");    // ✓ found
Child.class.getDeclaredField("childPrivate");  // ✓ found (private, but declared here)
Child.class.getDeclaredField("publicField");   // ✗ NoSuchFieldException — in Base, not Child

// getField — public fields from Child AND superclasses
Child.class.getField("childField");    // ✓ found (public, in Child)
Child.class.getField("publicField");   // ✓ found (public, inherited from Base)
Child.class.getField("childPrivate");  // ✗ NoSuchFieldException — not public
Child.class.getField("privateField");  // ✗ NoSuchFieldException — not public

// To find inherited private field:
Field f = Base.class.getDeclaredField("privateField"); // must go to the declaring class
f.setAccessible(true);
String val = (String) f.get(childInstance); // "base-private"
```

---

## Trap 5: Method.invoke() with null for Static Methods

**Interviewer:** "How do you invoke a static method using `Method.invoke()`? What do you pass as the first argument?"

**The Trap:** Candidates either say "pass an instance" (wrong for static) or are unsure about `null`.

**Answer:** For static methods, pass `null` as the first argument to `Method.invoke()`. There is no "instance" for a static method — `null` is the correct and expected value.

```java
class MathUtils {
    public static int square(int n) { return n * n; }
    public int cube(int n) { return n * n * n; }
}

// Invoking STATIC method — pass null as the object
Method squareMethod = MathUtils.class.getMethod("square", int.class);
int result = (int) squareMethod.invoke(null, 5); // null for static — returns 25

// Invoking INSTANCE method — pass an instance
MathUtils instance = new MathUtils();
Method cubeMethod = MathUtils.class.getMethod("cube", int.class);
int cubeResult = (int) cubeMethod.invoke(instance, 3); // must pass instance — returns 27

// What happens if you pass null for an instance method?
try {
    cubeMethod.invoke(null, 3); // NullPointerException (or InvocationTargetException wrapping it)
} catch (InvocationTargetException e) {
    System.out.println("Cause: " + e.getCause()); // NullPointerException
}

// Always check if static before deciding:
if (Modifier.isStatic(method.getModifiers())) {
    method.invoke(null, args);
} else {
    method.invoke(targetInstance, args);
}
```

---

## Trap 6: Arrays and Reflection — getClass().isArray()

**Interviewer:** "You receive an `Object` and want to check if it's an array. How do you do it?"

**The Trap:** Candidates use `instanceof Object[]` (misses primitive arrays).

**Answer:** Use `obj.getClass().isArray()`. The `instanceof Object[]` check only works for reference type arrays — it misses `int[]`, `double[]`, `boolean[]`, etc. Reflection's `isArray()` handles all array types.

```java
Object intArray = new int[]{1, 2, 3};
Object strArray = new String[]{"a", "b"};
Object plain    = "hello";

// WRONG — misses primitive arrays
System.out.println(intArray instanceof Object[]);  // FALSE! int[] is not Object[]
System.out.println(strArray instanceof Object[]);  // true

// CORRECT — handles all array types
System.out.println(intArray.getClass().isArray()); // true
System.out.println(strArray.getClass().isArray()); // true
System.out.println(plain.getClass().isArray());    // false

// Getting the component type
System.out.println(intArray.getClass().getComponentType()); // int
System.out.println(strArray.getClass().getComponentType()); // class java.lang.String

// Accessing elements of an unknown array
import java.lang.reflect.Array;
Object arr = intArray;
int length = Array.getLength(arr);      // 3 (works for any array type)
Object element = Array.get(arr, 0);     // 1 (autoboxed from int)

// Dynamic array creation
Object newArr = Array.newInstance(int.class, 5); // creates int[5]
Array.set(newArr, 0, 42);
System.out.println(Array.get(newArr, 0)); // 42
```

---

## Trap 7: Reflection and Generics — Type Erasure Limits

**Interviewer:** "Can you use reflection to find out that a field is `List<String>` instead of just `List`?"

**The Trap:** Candidates say "no, type erasure removes all generic info."

**Answer:** Partially. The JVM erases type arguments at runtime, so `field.getType()` returns the raw type (`List`). However, the compiler stores generic type information in the class file as metadata (in the `Signature` attribute). You can access this through `field.getGenericType()`, which returns a `ParameterizedType` you can inspect.

```java
class Container {
    private List<String> items;
    private Map<String, List<Integer>> complexMap;
}

Field itemsField = Container.class.getDeclaredField("items");

// Raw type — type erasure
System.out.println(itemsField.getType()); // class java.util.List (raw!)

// Generic type — compiler-preserved metadata
Type genericType = itemsField.getGenericType();
System.out.println(genericType); // java.util.List<java.lang.String>

if (genericType instanceof ParameterizedType pt) {
    System.out.println(pt.getRawType());         // interface java.util.List
    System.out.println(pt.getActualTypeArguments()[0]); // class java.lang.String
}

// Deeply nested type
Field mapField = Container.class.getDeclaredField("complexMap");
ParameterizedType mapType = (ParameterizedType) mapField.getGenericType();
Type valueType = mapType.getActualTypeArguments()[1]; // List<Integer>
if (valueType instanceof ParameterizedType listType) {
    System.out.println(listType.getActualTypeArguments()[0]); // Integer
}

// LIMIT: generic info for local variables and method bodies is ALWAYS erased
// Only field declarations, method parameter/return types, class declarations
// retain generic info in Signature attributes
```

---

## Trap 8: Constructor.newInstance() vs Class.newInstance() (Deprecated)

**Interviewer:** "What is wrong with `MyClass.class.newInstance()`?"

**The Trap:** Candidates use it without knowing why it was deprecated.

**Answer:** `Class.newInstance()` (deprecated in Java 9, removed conceptually in 17+) has two critical flaws: it only calls the no-argument constructor, and it throws checked exceptions (from the constructor) as if they were unchecked — bypassing Java's checked exception system. `Constructor.newInstance()` is the correct replacement.

```java
// DEPRECATED and broken — Class.newInstance()
class ServiceWithChecked {
    public ServiceWithChecked() throws IOException { // checked exception
        throw new IOException("startup failed");
    }
}

try {
    ServiceWithChecked s = ServiceWithChecked.class.newInstance();
    // PROBLEM: IOException is thrown, but declared as InstantiationException
    // The checked exception is re-thrown as InstantiationException
    // This bypasses Java's type system — the caller may not know to catch IOException
} catch (InstantiationException | IllegalAccessException e) {
    // e.getCause() might be an IOException — hidden!
}

// CORRECT — Constructor.newInstance()
try {
    Constructor<ServiceWithChecked> ctor =
        ServiceWithChecked.class.getDeclaredConstructor();
    ctor.setAccessible(true);
    ServiceWithChecked s = ctor.newInstance();
} catch (InvocationTargetException e) {
    // The actual exception is wrapped in InvocationTargetException
    Throwable cause = e.getCause(); // the real IOException
    if (cause instanceof IOException ioe) {
        System.out.println("Startup failed: " + ioe.getMessage());
    }
} catch (NoSuchMethodException | InstantiationException | IllegalAccessException e) {
    System.out.println("Reflection error: " + e.getMessage());
}

// Constructor with arguments — only Constructor.newInstance() can do this
Constructor<MyService> ctor = MyService.class.getDeclaredConstructor(
    Repository.class, Validator.class);
MyService service = ctor.newInstance(repo, validator);
```

---

## Trap 9: Proxy.newProxyInstance() Limitations

**Interviewer:** "Can I use `Proxy.newProxyInstance()` to intercept calls on a class that does not implement any interface?"

**The Trap:** Candidates say "yes" or "use any ClassLoader."

**Answer:** No. JDK's `Proxy.newProxyInstance()` strictly requires that the target type is an interface (or a list of interfaces). If you pass a class (not an interface) in the `interfaces` array, it throws `IllegalArgumentException`. For concrete class proxying, you need CGLIB or ByteBuddy.

```java
interface Auditable { void save(Object entity); }
class Repository implements Auditable { /* ... */ }
class PlainRepository { /* does NOT implement Auditable or any interface */ }

// ✓ WORKS — Auditable is an interface
Auditable proxy1 = (Auditable) Proxy.newProxyInstance(
    Auditable.class.getClassLoader(),
    new Class[]{Auditable.class},          // ← interface ✓
    handler);

// ✗ FAILS — PlainRepository is not an interface
try {
    Object proxy2 = Proxy.newProxyInstance(
        PlainRepository.class.getClassLoader(),
        new Class[]{PlainRepository.class}, // ← class, not interface ✗
        handler);
} catch (IllegalArgumentException e) {
    System.out.println(e.getMessage()); // "PlainRepository is not an interface"
}

// ✗ ALSO FAILS — final class, even if interface was available
final class FinalService implements Auditable { /* ... */ }
// CGLIB cannot subclass final classes either
// Only solution: make it non-final OR use JDK Proxy via its interface

// CGLIB approach for concrete (non-final) class
// Enhancer enhancer = new Enhancer();
// enhancer.setSuperclass(PlainRepository.class);
// enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
//     System.out.println("Before: " + method.getName());
//     return proxy.invokeSuper(obj, args);
// });
// PlainRepository cgProxy = (PlainRepository) enhancer.create();
```

---

## Trap 10: Reflection Security Manager (Now Largely Removed in Modern Java)

**Interviewer:** "Can you use a SecurityManager to block reflection-based access to private fields?"

**The Trap:** Candidates describe SecurityManager as the defense for reflection.

**Answer:** `SecurityManager` was deprecated in Java 17 and removed in Java 24. It was never particularly effective as a reflection defense because of its complex configuration and bypass vulnerabilities. The modern defense against unwanted reflection is the **Java Module System** (JPMS) — packages not opened in `module-info.java` are inaccessible via reflection, and this cannot be bypassed without JVM flags.

```java
// Java 8: SecurityManager could block reflection
System.setSecurityManager(new SecurityManager() {
    @Override
    public void checkMemberAccess(Class<?> clazz, int which) {
        if (which == Member.DECLARED) {
            throw new SecurityException("Reflection access denied");
        }
    }
});
// Deprecated in Java 17, no-op in Java 18+, removed in Java 24

// Modern Java (9+): Module system IS the security boundary
// module-info.java controls what is accessible:
//
// module com.myapp.secure {
//     exports com.myapp.api;
//     // com.myapp.internal is NOT exported — inaccessible AND non-reflectable
//     // to other modules unless explicitly opened
// }

// Checking module accessibility programmatically
Module targetModule = String.class.getModule();
Module callerModule = MyClass.class.getModule();

System.out.println(targetModule.isOpen("java.lang", callerModule));
// false — java.lang is not opened to your module

// For testing frameworks that need deep access, use --add-opens:
// --add-opens java.base/java.lang=ALL-UNNAMED
// This is acceptable in test configurations but should never be in production
```

---

## Trap 11: InvocationTargetException — Unwrapping the Real Cause

**Interviewer:** "A method invoked via reflection throws a `NullPointerException`. What exception does the `invoke()` call throw?"

**The Trap:** Candidates say `NullPointerException`.

**Answer:** `Method.invoke()` wraps any exception thrown by the target method in `InvocationTargetException`. To get the original `NullPointerException`, you must call `e.getCause()`.

```java
class BuggyService {
    public void process(String s) {
        System.out.println(s.toUpperCase()); // NPE if s is null
    }
}

Method m = BuggyService.class.getMethod("process", String.class);
BuggyService svc = new BuggyService();

try {
    m.invoke(svc, (Object) null); // passing null as the String argument
} catch (InvocationTargetException e) {
    Throwable cause = e.getCause();
    System.out.println(cause.getClass().getName()); // java.lang.NullPointerException
    // NOT InvocationTargetException — that's just the wrapper

    // Proper re-throwing pattern in frameworks:
    if (cause instanceof RuntimeException re) throw re;
    if (cause instanceof Error err) throw err;
    throw new RuntimeException("Invocation failed", cause);
} catch (IllegalAccessException | IllegalArgumentException e) {
    // These are reflection-level errors, not target method errors
    throw new RuntimeException("Reflection failed", e);
}
```

---

## Trap 12: Reflection on Records — Component vs Field Access

**Interviewer:** "A record has a component `name`. Can you access it via `getDeclaredField("name")`?"

**The Trap:** Candidates say "yes, just like a regular field."

**Answer:** Yes, `getDeclaredField("name")` works — the backing field is named after the component. But the recommended and more semantic approach for records is `getRecordComponents()`, which gives you the `RecordComponent` objects that include the component name, type, accessor method, and annotations.

```java
public record Person(String name, int age) {}

// Works but not record-specific:
Field nameField = Person.class.getDeclaredField("name");
nameField.setAccessible(true);
String nameValue = (String) nameField.get(new Person("Alice", 30)); // "Alice"

// Preferred — record-aware API
if (Person.class.isRecord()) {
    RecordComponent[] components = Person.class.getRecordComponents();
    for (RecordComponent comp : components) {
        System.out.println("Component: " + comp.getName());          // "name"
        System.out.println("Type: " + comp.getType());               // class java.lang.String
        System.out.println("Accessor: " + comp.getAccessor());       // public String Person.name()

        // Invoke the accessor (not field access — respects record's accessor)
        Method accessor = comp.getAccessor();
        Object value = accessor.invoke(new Person("Alice", 30));
        System.out.println("Value: " + value);                        // "Alice"

        // Get annotations on the component
        Annotation[] annotations = comp.getAnnotations();
    }
}
```
