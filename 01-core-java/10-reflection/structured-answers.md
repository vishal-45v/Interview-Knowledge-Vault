# Chapter 10: Reflection — Structured Answers

---

## Q1: What Can You Do With the Java Reflection API?

**One-line answer:** The Reflection API lets you inspect, create, and manipulate any Java class, method, field, or constructor at runtime — including private members — without knowing them at compile time.

**Core capabilities:**

```java
// 1. Inspect class metadata
Class<?> clazz = String.class;
System.out.println(clazz.getName());              // java.lang.String
System.out.println(clazz.getSuperclass());         // class java.lang.Object
System.out.println(Arrays.toString(clazz.getInterfaces())); // [Serializable, Comparable, CharSequence]
System.out.println(Modifier.isPublic(clazz.getModifiers())); // true
System.out.println(clazz.isRecord());             // false
System.out.println(clazz.isSealed());             // false

// 2. Inspect and access fields
class Config {
    private String dbUrl = "jdbc:postgresql://localhost/db";
    public int maxConnections = 10;
}

Config cfg = new Config();
Field privateField = Config.class.getDeclaredField("dbUrl");
privateField.setAccessible(true);
System.out.println(privateField.get(cfg)); // jdbc:postgresql://localhost/db
privateField.set(cfg, "jdbc:h2:mem:test"); // modify private field

// 3. Discover and invoke methods
Method[] methods = String.class.getDeclaredMethods();
System.out.println("String has " + methods.length + " methods");

Method substring = String.class.getMethod("substring", int.class, int.class);
String result = (String) substring.invoke("Hello World", 6, 11); // "World"

// 4. Create instances without knowing the class at compile time
String className = "java.util.ArrayList";
Class<?> cls = Class.forName(className);
List<?> list = (List<?>) cls.getDeclaredConstructor().newInstance();

// 5. Access constructor details and create instances
Constructor<?>[] constructors = StringBuilder.class.getDeclaredConstructors();
for (Constructor<?> ctor : constructors) {
    System.out.println(Arrays.toString(ctor.getParameterTypes()));
}

Constructor<StringBuilder> ctor = StringBuilder.class.getConstructor(int.class);
StringBuilder sb = ctor.newInstance(256); // new StringBuilder(256)

// 6. Read annotations at runtime
Method m = SomeService.class.getMethod("doWork");
Transactional tx = m.getAnnotation(Transactional.class);
if (tx != null) {
    System.out.println("timeout = " + tx.timeout());
}

// 7. Inspect generic types (partial — compiler-preserved metadata)
Field listField = MyClass.class.getDeclaredField("items"); // List<String>
ParameterizedType pt = (ParameterizedType) listField.getGenericType();
System.out.println(pt.getActualTypeArguments()[0]); // class java.lang.String
```

---

## Q2: How Does Spring Use Reflection Internally?

**One-line answer:** Spring uses reflection extensively at startup for component scanning (finding `@Component` classes), dependency injection (`@Autowired` field/constructor injection), AOP proxy creation, `@Value` binding, and `@Transactional` interception.

**1. Component scanning:**
```java
// Spring scans every class in your package and its subpackages
// For each class, it checks for @Component, @Service, @Repository, @Controller:

ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false);
scanner.addIncludeFilter(new AnnotationTypeFilter(Component.class));

Set<BeanDefinition> candidates = scanner.findCandidateComponents("com.myapp");
for (BeanDefinition bd : candidates) {
    String className = bd.getBeanClassName();
    Class<?> cls = Class.forName(className);
    // Register as a bean in the ApplicationContext
}
```

**2. Dependency injection via reflection:**
```java
@Service
class OrderService {
    @Autowired
    private PaymentService paymentService; // Spring injects this via reflection

    @Autowired
    public OrderService(UserRepository userRepo) { /* constructor injection */ }
}

// Simplified version of what Spring does for field injection:
Field[] fields = OrderService.class.getDeclaredFields();
for (Field field : fields) {
    if (field.isAnnotationPresent(Autowired.class)) {
        field.setAccessible(true);
        Object dependency = applicationContext.getBean(field.getType());
        field.set(orderServiceInstance, dependency);
    }
}
```

**3. @Value binding:**
```java
@Component
class AppConfig {
    @Value("${server.port:8080}")
    private int serverPort;  // Spring reads @Value, resolves placeholder, sets via reflection
}

// Spring does roughly:
Field portField = AppConfig.class.getDeclaredField("serverPort");
portField.setAccessible(true);
String placeholder = portField.getAnnotation(Value.class).value(); // "${server.port:8080}"
String resolvedValue = environment.resolvePlaceholders(placeholder); // "8080"
portField.set(configInstance, Integer.parseInt(resolvedValue));
```

**4. AOP proxy creation:**
```java
// Spring wraps @Transactional beans in proxies
@Service
@Transactional
class OrderService {
    public void placeOrder(Order order) { /* ... */ }
}

// Spring detects @Transactional via reflection:
for (Method method : OrderService.class.getMethods()) {
    if (method.isAnnotationPresent(Transactional.class)
            || OrderService.class.isAnnotationPresent(Transactional.class)) {
        // Create proxy around this method
        // JDK Proxy if OrderService implements interface
        // CGLIB subclass if no interface
    }
}
```

---

## Q3: How Do You Invoke a Private Method Using Reflection?

**One-line answer:** Use `getDeclaredMethod()` (not `getMethod()`), call `setAccessible(true)`, then `invoke()` — and handle `InvocationTargetException` to get the real exception if the method throws.

```java
public class PasswordHasher {
    private String hashPassword(String password) {
        // Normally inaccessible from outside
        return BCrypt.hashpw(password, BCrypt.gensalt());
    }

    private static String generateSalt() {
        return BCrypt.gensalt(12);
    }
}

// Invoking a private instance method
PasswordHasher hasher = new PasswordHasher();

Method hashMethod = PasswordHasher.class.getDeclaredMethod("hashPassword", String.class);
hashMethod.setAccessible(true); // unlock the private method

try {
    String hashed = (String) hashMethod.invoke(hasher, "myPassword123");
    System.out.println("Hashed: " + hashed);
} catch (InvocationTargetException e) {
    // The method itself threw an exception — unwrap it
    throw new RuntimeException("hashPassword failed", e.getCause());
} catch (IllegalAccessException e) {
    // Should not happen after setAccessible(true) — unless module blocks it
    throw new RuntimeException("Cannot access method", e);
}

// Invoking a private STATIC method (pass null as the instance)
Method saltMethod = PasswordHasher.class.getDeclaredMethod("generateSalt");
saltMethod.setAccessible(true);
String salt = (String) saltMethod.invoke(null); // null for static methods

// Practical use case: testing private methods
// (though generally a design smell — prefer testing through public API)
@Test
void testHashPassword() throws Exception {
    PasswordHasher hasher = new PasswordHasher();
    Method method = PasswordHasher.class.getDeclaredMethod("hashPassword", String.class);
    method.setAccessible(true);
    String hash = (String) method.invoke(hasher, "testPassword");
    assertNotNull(hash);
    assertTrue(hash.startsWith("$2a$")); // BCrypt format
}
```

---

## Q4: What is the Performance Overhead of Reflection and How Can You Mitigate It?

**One-line answer:** Reflection method calls are roughly 5-100x slower than direct calls, primarily due to security checks, lack of JIT inlining, and boxing of primitives — mitigation: cache `Method` objects, use `MethodHandle`, or generate lambdas via `LambdaMetafactory`.

**Understanding the cost breakdown:**
```java
// 1. getMethod() / getDeclaredMethod() — VERY EXPENSIVE (never call in loops)
//    - Scans the class hierarchy
//    - Allocates new Method objects
//    - Does security checks
//    Cost: ~500 ns to 5000 ns

// 2. method.invoke() with cached Method — MODERATELY EXPENSIVE
//    - Access control check per invocation
//    - Arguments boxed into Object[]
//    - Reflective dispatch (no JIT inlining)
//    Cost: ~5-15 ns (vs ~1 ns for direct call)

// 3. MethodHandle.invoke() — CHEAP (JIT can optimize)
//    - Access checked once at creation time
//    - JIT can inline and optimize
//    Cost: ~2-3 ns

// 4. Lambda via LambdaMetafactory — NATIVE SPEED
//    - Compiled to direct method call by JIT
//    Cost: ~1 ns (same as direct call)
```

**Mitigation strategies:**

```java
// Strategy 1: Cache Method objects in a static or instance field
public class ReflectiveMapper {
    // Cache at startup — never look up in hot path
    private static final Map<String, Method> METHOD_CACHE = new ConcurrentHashMap<>();

    public static Object invoke(Object target, String methodName, Object... args)
            throws Exception {
        String key = target.getClass().getName() + "." + methodName;
        Method method = METHOD_CACHE.computeIfAbsent(key, k -> {
            try {
                Method m = target.getClass().getMethod(methodName,
                    Arrays.stream(args).map(Object::getClass).toArray(Class[]::new));
                m.setAccessible(true);
                return m;
            } catch (NoSuchMethodException e) {
                throw new RuntimeException(e);
            }
        });
        return method.invoke(target, args);
    }
}

// Strategy 2: MethodHandle — JIT-friendly
public class MethodHandleCache {
    private final MethodHandle handle;

    public MethodHandleCache(Class<?> cls, String name, MethodType type) throws Exception {
        MethodHandles.Lookup lookup = MethodHandles.lookup();
        this.handle = lookup.findVirtual(cls, name, type);
        // Access check done ONCE at creation
    }

    public Object invoke(Object target, Object... args) throws Throwable {
        return handle.invokeWithArguments(target, args); // JIT can inline
    }
}

// Strategy 3: LambdaMetafactory — near-native speed for known interfaces
// (Example: converting a Method to a Function<String, Integer>)
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle target = lookup.findVirtual(String.class, "length",
    MethodType.methodType(int.class));

Function<String, Integer> lengthFn = (Function<String, Integer>)
    LambdaMetafactory.metafactory(
        lookup, "apply",
        MethodType.methodType(Function.class),
        MethodType.methodType(Object.class, Object.class),
        target,
        MethodType.methodType(Integer.class, String.class)
    ).getTarget().invokeExact();

// Now as fast as a direct call
int len = lengthFn.apply("Hello"); // 5, compiled to direct String.length() call by JIT

// Strategy 4: Disable per-invocation access checks (Java 9+)
Method m = MyClass.class.getDeclaredMethod("privateMethod");
m.setAccessible(true);
// After setAccessible(true), JVM skips per-invocation access checks (in some JVMs)
// Slightly faster for frequently called methods
```

---

## Q5: How Do You Create an Object Using Reflection Without Calling new?

**One-line answer:** Use `Class.getDeclaredConstructor(...).newInstance(args)` — or `sun.misc.Unsafe.allocateInstance()` to create an instance without calling any constructor at all (for deserialization frameworks).

**Standard approach:**
```java
// No-arg constructor
Class<?> cls = Class.forName("com.example.UserService");
Object instance = cls.getDeclaredConstructor().newInstance();

// Constructor with arguments
Constructor<StringBuilder> ctor = StringBuilder.class.getConstructor(int.class);
StringBuilder sb = ctor.newInstance(256); // equivalent to new StringBuilder(256)

// Private constructor (e.g., Singleton pattern)
class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() { System.out.println("Singleton created"); }
    public static Singleton getInstance() { return INSTANCE; }
}

Constructor<Singleton> privateCtor = Singleton.class.getDeclaredConstructor();
privateCtor.setAccessible(true);
Singleton newInstance = privateCtor.newInstance();
// Creates a SECOND Singleton instance — breaks the singleton guarantee!
System.out.println(newInstance == Singleton.getInstance()); // false!
```

**Framework-style dynamic instantiation:**
```java
// Spring/Guice style: create from class name in configuration
public <T> T createBean(String className, Class<T> type) throws Exception {
    Class<?> cls = Class.forName(className);

    // Try @Inject or @Autowired constructor first
    Constructor<?> bestCtor = findAnnotatedConstructor(cls);
    if (bestCtor == null) {
        bestCtor = cls.getDeclaredConstructor(); // fall back to no-arg
    }
    bestCtor.setAccessible(true);

    Object[] args = resolveConstructorArgs(bestCtor);
    return type.cast(bestCtor.newInstance(args));
}

private Constructor<?> findAnnotatedConstructor(Class<?> cls) {
    for (Constructor<?> ctor : cls.getDeclaredConstructors()) {
        if (ctor.isAnnotationPresent(Autowired.class)
                || ctor.isAnnotationPresent(Inject.class)) {
            return ctor;
        }
    }
    return null;
}
```

**Unsafe allocation (bypass constructor entirely):**
```java
// Used by Java serialization and some ORMs
// Creates instance WITHOUT calling any constructor
// Fields get default values (null, 0, false)
import sun.misc.Unsafe;
Field f = Unsafe.class.getDeclaredField("theUnsafe");
f.setAccessible(true);
Unsafe unsafe = (Unsafe) f.get(null);

SomeClass instance = (SomeClass) unsafe.allocateInstance(SomeClass.class);
// SomeClass() constructor was NEVER called
// Use case: deserializing objects where no appropriate constructor exists
```

---

## Q6: How Do You Read Annotation Values at Runtime?

**One-line answer:** Call `element.getAnnotation(AnnotationType.class)` on any `AnnotatedElement` (Class, Method, Field, Constructor, Parameter) to get the annotation proxy, then read its values through the accessor methods.

```java
// Define annotation
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD})
@interface Validate {
    int minLength() default 0;
    int maxLength() default Integer.MAX_VALUE;
    boolean notNull() default false;
    String message() default "";
}

// Apply annotation
@Validate(notNull = true, message = "User cannot be null")
public class User {
    @Validate(minLength = 2, maxLength = 50, notNull = true, message = "Name is required")
    private String name;

    @Validate(minLength = 5, maxLength = 100)
    private String email;
}

// Read class-level annotation
Class<?> clazz = User.class;
Validate classAnnotation = clazz.getAnnotation(Validate.class);
if (classAnnotation != null) {
    System.out.println("Class message: " + classAnnotation.message());
    System.out.println("notNull: " + classAnnotation.notNull());
}

// Read field-level annotations
for (Field field : clazz.getDeclaredFields()) {
    Validate fieldAnnotation = field.getAnnotation(Validate.class);
    if (fieldAnnotation != null) {
        System.out.printf("Field: %-10s  minLength=%d  maxLength=%d  notNull=%b  message='%s'%n",
            field.getName(),
            fieldAnnotation.minLength(),
            fieldAnnotation.maxLength(),
            fieldAnnotation.notNull(),
            fieldAnnotation.message());
    }
}

// Build a runtime validator using annotation data
public class AnnotationValidator {

    public List<String> validate(Object obj) throws IllegalAccessException {
        List<String> errors = new ArrayList<>();
        Class<?> clazz = obj.getClass();

        for (Field field : clazz.getDeclaredFields()) {
            Validate v = field.getAnnotation(Validate.class);
            if (v == null) continue;

            field.setAccessible(true);
            Object value = field.get(obj);

            if (v.notNull() && value == null) {
                errors.add(field.getName() + ": " +
                    (v.message().isEmpty() ? "must not be null" : v.message()));
                continue;
            }

            if (value instanceof String s) {
                if (s.length() < v.minLength()) {
                    errors.add(field.getName() + ": too short (min " + v.minLength() + ")");
                }
                if (s.length() > v.maxLength()) {
                    errors.add(field.getName() + ": too long (max " + v.maxLength() + ")");
                }
            }
        }
        return errors;
    }
}
```

---

## Q7: What is the Difference Between a Dynamic Proxy and a CGLIB Proxy?

**One-line answer:** JDK dynamic proxies work only with interfaces (generate an interface-implementing class at runtime); CGLIB proxies work with concrete classes (generate a subclass at runtime). Spring uses JDK proxy when an interface is available, CGLIB otherwise.

```java
// JDK Dynamic Proxy — interface required
interface OrderService {
    void placeOrder(Order order);
}

class OrderServiceImpl implements OrderService {
    public void placeOrder(Order order) { System.out.println("Placing: " + order); }
}

// Proxy generation: JVM creates a class that implements OrderService
OrderService jdkProxy = (OrderService) Proxy.newProxyInstance(
    OrderService.class.getClassLoader(),
    new Class[]{OrderService.class},
    (proxy, method, args) -> {
        System.out.println("Before " + method.getName());
        Object result = method.invoke(new OrderServiceImpl(), args);
        System.out.println("After " + method.getName());
        return result;
    }
);

System.out.println(jdkProxy.getClass().getName()); // $Proxy0 or similar
System.out.println(jdkProxy instanceof OrderService); // true (implements the interface)
System.out.println(jdkProxy instanceof OrderServiceImpl); // false (not a subclass)
```

```java
// CGLIB Proxy — works with concrete classes (no interface needed)
class PaymentService {  // NO interface
    public void processPayment(BigDecimal amount) {
        System.out.println("Processing: " + amount);
    }
    public final void validatePayment(BigDecimal amount) { /* ... */ }
}

// Using CGLIB (via Spring's dependency on cglib)
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(PaymentService.class);  // creates a SUBCLASS
enhancer.setCallback((MethodInterceptor) (obj, method, args, methodProxy) -> {
    System.out.println("Before " + method.getName());
    Object result = methodProxy.invokeSuper(obj, args); // calls super (real method)
    System.out.println("After " + method.getName());
    return result;
});

PaymentService cglibProxy = (PaymentService) enhancer.create();
System.out.println(cglibProxy.getClass().getName());
// PaymentService$$EnhancerByCGLIB$$abc123

System.out.println(cglibProxy instanceof PaymentService); // true (IS a subclass)
cglibProxy.processPayment(BigDecimal.TEN); // intercepted ✓
cglibProxy.validatePayment(BigDecimal.TEN); // NOT intercepted — final method!
```

**Comparison table:**

| Feature | JDK Proxy | CGLIB |
|---|---|---|
| Requires interface | Yes | No |
| Works with final classes | No | No |
| Works with final methods | N/A | No (cannot override) |
| Generated type | Implements interface | Subclass of target |
| Performance | Similar to CGLIB | Similar to JDK Proxy |
| Spring behavior | Used when interface exists | Used when no interface |
| Can intercept `toString()` | Yes | Yes |
| Can intercept constructors | No | No (only methods) |

---

## Q8: How Do You Get All Fields of a Class Including Inherited Ones?

**One-line answer:** `getDeclaredFields()` only returns fields from the specific class — to get all including inherited, walk up the class hierarchy calling `getDeclaredFields()` at each level until you reach `Object`.

```java
public class ReflectionUtils {

    /**
     * Returns all declared fields from the class and all its superclasses,
     * stopping at (but not including) stopClass.
     */
    public static List<Field> getAllFields(Class<?> cls, Class<?> stopClass) {
        List<Field> fields = new ArrayList<>();
        while (cls != null && cls != stopClass) {
            fields.addAll(Arrays.asList(cls.getDeclaredFields()));
            cls = cls.getSuperclass();
        }
        return fields;
    }

    public static List<Field> getAllFields(Class<?> cls) {
        return getAllFields(cls, Object.class);
    }
}

// Example hierarchy
class Entity {
    private Long id;
    private LocalDateTime createdAt;
}

class User extends Entity {
    private String username;
    private String email;
}

class AdminUser extends User {
    private Set<String> permissions;
}

// getDeclaredFields() — only this class
System.out.println(Arrays.asList(AdminUser.class.getDeclaredFields()));
// [permissions] — only AdminUser's own fields

// ReflectionUtils.getAllFields() — full hierarchy
List<Field> allFields = ReflectionUtils.getAllFields(AdminUser.class);
allFields.forEach(f -> System.out.println(f.getDeclaringClass().getSimpleName() + "." + f.getName()));
// AdminUser.permissions
// User.username
// User.email
// Entity.id
// Entity.createdAt

// Practical use: building a toString() via reflection
public static String reflectiveToString(Object obj) throws IllegalAccessException {
    StringBuilder sb = new StringBuilder(obj.getClass().getSimpleName()).append("{");
    boolean first = true;
    for (Field field : ReflectionUtils.getAllFields(obj.getClass())) {
        field.setAccessible(true);
        if (!first) sb.append(", ");
        sb.append(field.getName()).append("=").append(field.get(obj));
        first = false;
    }
    return sb.append("}").toString();
}
```

---

## Q9: What Are the Security Implications of setAccessible(true)?

**One-line answer:** `setAccessible(true)` bypasses Java's access control entirely — it can read/modify private fields (including sensitive data like passwords), break immutability of `final` fields, and violate class invariants, enabling attacks if untrusted code can call it.

**Breaking immutability:**
```java
// String is supposed to be immutable — setAccessible breaks this
String secret = "password123";
System.out.println(secret); // password123

Field valueField = String.class.getDeclaredField("value");
valueField.setAccessible(true);
// In Java 9+, this may throw InaccessibleObjectException for java.base

// Even if it works (in older Java or with --add-opens):
byte[] bytes = (byte[]) valueField.get(secret);
bytes[0] = 'X'; // mutate "immutable" String's backing array
System.out.println(secret); // Xassword123 — String is now corrupted!
System.out.println("password123"); // might also print "Xassword123"
// because String literals are interned — same object!
```

**Breaking singleton pattern:**
```java
public enum DatabaseType { POSTGRES, MYSQL, ORACLE }

// Enums are supposed to be singletons — you cannot normally create new instances
// With reflection: (this actually fails for enums in modern Java, but illustrates the concept)
try {
    Constructor<DatabaseType> ctor = DatabaseType.class.getDeclaredConstructor(String.class, int.class);
    ctor.setAccessible(true);
    DatabaseType fake = ctor.newInstance("FAKE", 3); // throws on modern JVMs
} catch (Exception e) {
    System.out.println("Enum protection: " + e.getMessage());
    // "Cannot reflectively create enum objects" — JVM explicitly blocks this
}
```

**Reading sensitive data:**
```java
// Stealing passwords from a security-conscious class
public class EncryptedConfig {
    private final char[] masterPassword = "secret123".toCharArray();
    private final byte[] encryptionKey = generateKey();
}

// With reflection:
EncryptedConfig config = new EncryptedConfig();
Field pwField = EncryptedConfig.class.getDeclaredField("masterPassword");
pwField.setAccessible(true);
char[] stolen = (char[]) pwField.get(config);
System.out.println(new String(stolen)); // "secret123" — stolen!
```

**Java 9+ mitigations:**
```java
// Module system restricts setAccessible across module boundaries
// Without opens directive → InaccessibleObjectException
// This is the effective security boundary in modern Java

// To completely prevent reflection-based access to a specific class (within modules):
// 1. Don't include opens in module-info.java
// 2. The module system prevents cross-module setAccessible

// For non-modular code, the SecurityManager was the traditional defense
// but it's now deprecated/removed. The recommendation today:
// - Use the module system for true encapsulation
// - Never put secrets in fields (use hardware security modules, secret vaults)
// - Audit third-party library use of setAccessible
```

---

## Q10: How Does Java 9 Module System Restrict Reflection?

**One-line answer:** In the module system, a class's package must be explicitly `open`ed (in `module-info.java`) for external code to use `setAccessible(true)` on it — otherwise `InaccessibleObjectException` is thrown, even for classes that were previously accessible.

**The problem it solves:**
```java
// Before Java 9: anyone could access any internal JDK class via reflection
// e.g., sun.misc.Unsafe, com.sun.crypto.provider.*, etc.
// This made JVM internals de facto public API — impossible to change them

// Java 9+: JDK internals are in modules with no opens directives
// setAccessible(true) respects these boundaries
```

**The module directives:**
```java
// module-info.java — controls what is reflectively accessible

module com.myapp {
    requires spring.core;
    requires spring.beans;

    // EXPORT only: code can call public API, but NO deep reflection
    exports com.myapp.api;

    // OPEN: allows deep reflection (setAccessible) but NOT just API use
    opens com.myapp.model;               // open to ALL other modules (for frameworks)
    opens com.myapp.service to spring.beans;  // open ONLY to spring.beans

    // EXPORT + OPEN: both public API access AND reflection
    exports com.myapp.dto;
    opens com.myapp.dto to jackson.databind;  // Jackson needs to reflect on fields
}
```

**Behavior matrix:**

```java
// Scenario 1: unrestricted access (same module or opens to ALL)
// module com.myapp opens com.myapp.model;
Field f = UserModel.class.getDeclaredField("name");
f.setAccessible(true); // ✓ works — package is opened

// Scenario 2: restricted access (module does not open package)
// module java.base (does NOT open java.lang)
Field stringValue = String.class.getDeclaredField("value");
try {
    stringValue.setAccessible(true); // ✗ InaccessibleObjectException
} catch (InaccessibleObjectException e) {
    // "Unable to make field private final byte[] java.lang.String.value accessible:
    //  module java.base does not 'opens java.lang' to module com.myapp"
}

// Fix for tests: --add-opens JVM flag
// java --add-opens java.base/java.lang=ALL-UNNAMED -jar myapp.jar

// Fix for production: add opens to module-info.java (for your own modules)
// module com.myapp {
//     opens com.myapp.internal to testing.framework;
// }

// Checking programmatically before accessing
Module targetModule = String.class.getModule();
Module callerModule = MyClass.class.getModule();

if (!targetModule.isOpen("java.lang", callerModule)) {
    System.out.println("Cannot reflectively access java.lang from " + callerModule.getName());
    // Use alternative approach (MethodHandles with lookup in the target module,
    // or avoid reflection entirely)
}
```

**Practical implications for frameworks:**

```java
// Spring Boot 3+ and Java 17+ require explicit opens for DI to work:
// In a fully modular app, Spring's reflection-based injection needs:

module com.myapp {
    requires spring.core;
    requires spring.beans;
    requires spring.context;

    // Allow Spring to inject into your beans' fields and constructors
    opens com.myapp.service to spring.beans, spring.core;
    opens com.myapp.controller to spring.beans, spring.core;
    opens com.myapp.repository to spring.beans, spring.core;

    // Allow Jackson to serialize/deserialize your DTOs
    opens com.myapp.dto to jackson.databind;

    // Allow JPA/Hibernate to access your entity fields
    opens com.myapp.entity to org.hibernate.orm.core;
}

// Alternatively: run on unnamed module (classpath) — no module-info.java
// Spring Boot apps on classpath are "unnamed module" — setAccessible still works
// Module restrictions only apply to NAMED modules with explicit module-info.java
```
