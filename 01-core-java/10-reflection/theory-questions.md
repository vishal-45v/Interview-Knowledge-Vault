# Reflection — Theory Questions

> 18 theory questions on Java Reflection, dynamic proxies, annotations processing, and framework internals.

---

## Core Reflection API

**Q1. What is Java Reflection? What can you do with it?**

Reflection allows inspecting and manipulating class structure and objects at runtime, bypassing compile-time type checking.

```java
Class<?> clazz = Order.class;
// Or:
Class<?> clazz = Class.forName("com.example.Order");  // Dynamic class loading

// Inspect structure
clazz.getDeclaredFields();        // All fields (including private)
clazz.getDeclaredMethods();       // All methods
clazz.getDeclaredConstructors();  // All constructors
clazz.getDeclaredAnnotations();   // Annotations on class
clazz.getInterfaces();            // Implemented interfaces
clazz.getSuperclass();            // Parent class
clazz.getGenericSuperclass();     // Parent with generic info

// Access and modify fields
Field field = clazz.getDeclaredField("status");
field.setAccessible(true);             // Bypass private!
String value = (String) field.get(order);  // Read
field.set(order, "CANCELLED");             // Write

// Invoke methods
Method method = clazz.getDeclaredMethod("cancel", String.class);
method.setAccessible(true);
method.invoke(order, "User requested");    // Call private method

// Create instances
Constructor<?> ctor = clazz.getDeclaredConstructor();
ctor.setAccessible(true);
Order newOrder = (Order) ctor.newInstance();
```

**Use cases:** Framework code (Spring DI, Hibernate ORM), serialization (Jackson), testing (Mockito), code generation tools.

---

**Q2. What is `setAccessible(true)` and why is it dangerous?**

`setAccessible(true)` suppresses Java's access control checks, allowing access to private/protected members.

**Dangers:**
- Breaks encapsulation — internal implementation details become accessible
- Can corrupt object state (bypassing invariants enforced in constructors/setters)
- Security concern in sandboxed environments
- In Java 9+ module system: may fail with `InaccessibleObjectException` for classes in other modules

```java
// Java 9+ requires module opens
// module-info.java:
// module myapp {
//     opens com.example.internal to mylib;  // Allow reflection on this package
// }

// Or at JVM startup:
// --add-opens java.base/java.lang=ALL-UNNAMED
```

**Performance cost:** Access control check bypass is fast, but reflection overall is ~10-100x slower than direct calls due to method lookup and boxing of primitives.

---

**Q3. How do you inspect annotations at runtime?**

```java
// Check if annotation present
if (method.isAnnotationPresent(Transactional.class)) {
    Transactional tx = method.getAnnotation(Transactional.class);
    System.out.println("Propagation: " + tx.propagation());
}

// Get all annotations
for (Annotation annotation : method.getDeclaredAnnotations()) {
    System.out.println(annotation.annotationType().getSimpleName());
}

// Inherited vs declared
clazz.getAnnotations();         // Includes @Inherited annotations from superclass
clazz.getDeclaredAnnotations(); // Only annotations directly on this class

// Repeatable annotations (Java 8+)
Scheduled.List scheduledList = method.getAnnotation(Scheduled.List.class);
// Or:
method.getAnnotationsByType(Scheduled.class);  // Handles repeatable annotations

// Example: build annotation processor
public void processAnnotatedMethods(Object bean) {
    Class<?> clazz = bean.getClass();
    for (Method method : clazz.getDeclaredMethods()) {
        Audit audit = method.getAnnotation(Audit.class);
        if (audit != null) {
            registerAuditHandler(bean, method, audit);
        }
    }
}
```

---

**Q4. What is the difference between `getMethod()` and `getDeclaredMethod()`?**

```java
// getMethod() — public methods only, including inherited from superclasses
// getDeclaredMethod() — any access modifier, only in THIS class (not inherited)

class Parent {
    public void parentPublic() {}
    private void parentPrivate() {}
}

class Child extends Parent {
    public void childPublic() {}
    private void childPrivate() {}
    @Override
    public void parentPublic() {}
}

Child.class.getMethod("parentPublic")       // Found (public, including inherited)
Child.class.getMethod("childPublic")        // Found
Child.class.getMethod("parentPrivate")      // NoSuchMethodException (not public)
Child.class.getDeclaredMethod("childPrivate")  // Found (declared in Child)
Child.class.getDeclaredMethod("parentPrivate") // NoSuchMethodException (declared in Parent)
```

**Rule of thumb:**
- `getDeclared*()` → current class only, all access modifiers
- `get*()` → public only, includes inherited

---

**Q5. How do you read generic type information at runtime?**

Type erasure removes most generic info, but some is retained in class metadata:

```java
// Type info on fields
class Repository<T> {
    private List<T> items;  // T is erased at runtime
}

// BUT: if you subclass with explicit type, it's retained:
class OrderRepository extends Repository<Order> {}
// This stores "Repository<Order>" as the generic superclass

Class<?> cls = OrderRepository.class;
ParameterizedType superclass = (ParameterizedType) cls.getGenericSuperclass();
Type[] typeArgs = superclass.getActualTypeArguments();
Class<?> entityType = (Class<?>) typeArgs[0];  // Order.class

// Generic type on fields (not type parameters)
Field field = MyClass.class.getDeclaredField("orders");
ParameterizedType fieldType = (ParameterizedType) field.getGenericType();
Class<?> typeArg = (Class<?>) fieldType.getActualTypeArguments()[0];

// Generic return type on methods
Method method = cls.getDeclaredMethod("findAll");
ParameterizedType returnType = (ParameterizedType) method.getGenericReturnType();
```

**TypeToken pattern:** Anonymous subclasses capture type information in the generic superclass, allowing Jackson and similar libraries to deserialize `List<Order>`.

---

## Dynamic Proxies

**Q6. What is a JDK dynamic proxy?**

JDK proxies create runtime implementations of interfaces, intercepting method calls:

```java
// Proxy must implement interface(s)
interface OrderService {
    Order createOrder(OrderRequest request);
    Order getOrder(Long id);
}

// Create proxy
OrderService proxy = (OrderService) Proxy.newProxyInstance(
    OrderService.class.getClassLoader(),
    new Class[]{OrderService.class},
    new InvocationHandler() {
        private final OrderService delegate = new OrderServiceImpl();

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            long start = System.nanoTime();
            try {
                log.info("Calling: {}", method.getName());
                return method.invoke(delegate, args);
            } catch (InvocationTargetException e) {
                throw e.getCause();  // Unwrap reflection exception
            } finally {
                long elapsed = System.nanoTime() - start;
                log.info("{} took {}ms", method.getName(), elapsed / 1_000_000);
            }
        }
    }
);

proxy.createOrder(request);  // Intercepted by InvocationHandler
```

**Limitation:** Can only proxy interfaces, not classes.

---

**Q7. How does CGLIB proxying differ from JDK proxies?**

CGLIB (Code Generation Library) creates subclass-based proxies, allowing proxying of classes:

```java
// JDK Proxy: interface-based, cannot proxy final classes or methods
// CGLIB: creates subclass, can proxy any non-final class/method

// Spring uses JDK proxy when bean implements interfaces
// Spring uses CGLIB when bean is a class (or @EnableAspectJAutoProxy(proxyTargetClass=true))

// CGLIB proxy example (simplified):
Enhancer enhancer = new Enhancer();
enhancer.setSuperclass(OrderService.class);  // Can proxy concrete class!
enhancer.setCallback((MethodInterceptor) (obj, method, args, proxy) -> {
    log.info("Before: {}", method.getName());
    Object result = proxy.invokeSuper(obj, args);  // Call original method
    log.info("After: {}", method.getName());
    return result;
});
OrderService cgLibProxy = (OrderService) enhancer.create();
```

**Key difference for Spring:**
- `@Transactional` on a `final` method → won't work (CGLIB can't override)
- `@Transactional` on a method calling another `@Transactional` method in the same class → self-invocation bypass, proxy not invoked

---

**Q8. How does Spring use reflection and proxies internally?**

Spring's core mechanisms:
1. **Component scan**: reflects over classpath to find `@Component`, `@Bean`, etc.
2. **Dependency injection**: uses `setAccessible(true)` on `@Autowired` fields
3. **`@Transactional`**: wraps bean in proxy (JDK or CGLIB); `TransactionInterceptor.invoke()` called for each method
4. **`@Cacheable`**: `CacheInterceptor` wraps method in proxy
5. **AOP**: `JdkDynamicAopProxy` or `CglibAopProxy` dispatches to advisor chain

```java
// Spring's @Autowired uses reflection:
// AutowiredAnnotationBeanPostProcessor does roughly:
Field[] fields = bean.getClass().getDeclaredFields();
for (Field field : fields) {
    if (field.isAnnotationPresent(Autowired.class)) {
        field.setAccessible(true);
        Object dependency = applicationContext.getBean(field.getType());
        field.set(bean, dependency);
    }
}
```

---

**Q9. What are the performance implications of reflection and how do you mitigate them?**

**Costs:**
- Method lookup: comparing method signatures (faster with caching)
- Security check bypass: `setAccessible()` has overhead
- Boxing/unboxing: reflection passes `Object[]`, primitives get boxed
- No JIT optimization: JIT can't inline reflected calls (in general)

**Mitigation strategies:**
```java
// 1. Cache Method/Field objects — lookup is expensive, invocation less so
private static final Method PROCESS_METHOD;
static {
    try {
        PROCESS_METHOD = OrderService.class.getDeclaredMethod("process", Order.class);
        PROCESS_METHOD.setAccessible(true);
    } catch (NoSuchMethodException e) {
        throw new RuntimeException(e);
    }
}

// 2. Use MethodHandle (Java 7+) — JIT-friendly, closer to direct call
MethodHandle handle = MethodHandles.lookup()
    .findVirtual(OrderService.class, "process",
                 MethodType.methodType(void.class, Order.class));
handle.invoke(service, order);  // Can be inlined by JIT!

// 3. Use LambdaMetafactory for fastest reflective invocation
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle methodHandle = lookup.findVirtual(
    OrderService.class, "process", MethodType.methodType(void.class, Order.class));
// Generates a lambda that directly invokes the method — near-native performance
BiConsumer<OrderService, Order> invoker = (BiConsumer<OrderService, Order>)
    LambdaMetafactory.metafactory(
        lookup,
        "accept",
        MethodType.methodType(BiConsumer.class),
        MethodType.methodType(void.class, Object.class, Object.class),
        methodHandle,
        MethodType.methodType(void.class, OrderService.class, Order.class))
    .getTarget().invokeExact();
```

---

**Q10. What is the module system (JPMS) and how does it affect reflection?**

Java 9 introduced the module system (`module-info.java`). By default, strong encapsulation prevents reflection on internal module members:

```java
// module-info.java of a module
module com.example.core {
    exports com.example.core.api;     // Exported to all
    exports com.example.core.impl to com.example.web;  // Only to specific module

    opens com.example.core.model;     // Allow reflection from any module
    opens com.example.core.impl to mylib;  // Allow reflection only from mylib

    requires java.sql;
    requires spring.core;
}
```

**Without `opens`:** Reflection on module-private packages throws `InaccessibleObjectException`.

**Workaround for legacy code or testing:**
```bash
# JVM startup flag
--add-opens java.base/java.lang.reflect=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
```

---

## Annotation Processing

**Q11. What is compile-time annotation processing vs runtime annotation processing?**

**Compile-time (APT — Annotation Processing Tool):**
- Runs during `javac` compilation
- Cannot access instance state — works with source/class file structure only
- Used for: Lombok (generates getters/setters), MapStruct (generates mappers), AutoValue, Dagger
- Implements `javax.annotation.processing.Processor`
- Generates new source files — processed in subsequent compilation rounds

**Runtime (Reflection-based):**
- Runs when application starts or when annotation is accessed
- Can access instance state
- Used by: Spring, Hibernate, Jackson, JUnit
- Slower startup, but very flexible

```java
// Compile-time annotation processor (simplified)
@SupportedAnnotationTypes("com.example.Builder")
@SupportedSourceVersion(SourceVersion.RELEASE_17)
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        for (Element element : roundEnv.getElementsAnnotatedWith(Builder.class)) {
            TypeElement typeElement = (TypeElement) element;
            generateBuilder(typeElement);  // Write a new .java file
        }
        return true;
    }
}
```

---

**Q12. What is `instanceof` and type checking with reflection?**

```java
// Regular instanceof — compile-time type check
if (obj instanceof String s) { ... }  // Pattern matching (Java 16+)

// Reflection-based type check — useful when type not known at compile time
Class<?> targetType = String.class;
boolean isInstance = targetType.isInstance(obj);  // Equivalent to obj instanceof String

// Check assignability
Integer.class.isAssignableFrom(Number.class);  // false — Integer is not a supertype of Number
Number.class.isAssignableFrom(Integer.class);  // true — Number is supertype of Integer

// Check if class is interface, abstract, enum, etc.
clazz.isInterface();
clazz.isArray();
clazz.isEnum();
Modifier.isAbstract(clazz.getModifiers());
Modifier.isPublic(clazz.getModifiers());
```

---

**Q13. How do Spring's `BeanDefinitionRegistry` and reflection work together?**

```java
// Spring's component scan (simplified) uses reflection:
// 1. Scan classpath for .class files
// 2. Load each class and check for @Component (and meta-annotations)
// 3. Register as BeanDefinition

// How meta-annotations work:
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Component  // @Service is meta-annotated with @Component
public @interface Service {}

// Spring checks: clazz.getAnnotation(Service.class) returns @Service
// Then: Service.class.getAnnotation(Component.class) returns @Component
// This meta-annotation chain is why @Service, @Repository, @Controller work as @Component

// Detecting annotations recursively (Spring's AnnotationUtils)
AnnotationUtils.findAnnotation(method, Transactional.class);  // Checks superclasses too
AnnotatedElementUtils.findMergedAnnotation(method, Transactional.class);  // Handles composed
```

---

**Q14. What is method interception and how do proxies implement cross-cutting concerns?**

```java
// AOP proxy intercepts method calls for cross-cutting concerns
// Spring's MethodInterceptor:
public class LoggingInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        Object[] args = invocation.getArguments();

        log.info("→ {}.{}", method.getDeclaringClass().getSimpleName(), method.getName());
        try {
            Object result = invocation.proceed();  // Call actual method (or next interceptor)
            log.info("← {} = {}", method.getName(), result);
            return result;
        } catch (Exception e) {
            log.error("✗ {} threw {}", method.getName(), e.getClass().getSimpleName());
            throw e;
        }
    }
}

// Chain of interceptors (like middleware):
// Request → SecurityInterceptor → TransactionInterceptor → CacheInterceptor → Method
// Each interceptor calls proceed() to invoke the next one
```

---

**Q15. What are the limits of reflection — what can't you do?**

1. **Cannot create instances of abstract classes or interfaces** via reflection
2. **Cannot invoke constructors with autoboxing** — must pass exact types
3. **Cannot access local classes' outer variables** (even with reflection)
4. **Module system barriers** (Java 9+) — `InaccessibleObjectException` without `opens`
5. **`final` fields**: can set non-static final fields (but behavior undefined after JVM optimizations)
6. **Cannot get generic type info at runtime** for most generic types (erasure)
7. **Cannot reflect on lambda bodies** — lambdas are anonymous, generated classes
8. **Performance**: cannot be inlined by JIT; each call has overhead

```java
// final field reflection — technically possible but fragile:
Field field = Order.class.getDeclaredField("id");
field.setAccessible(true);
// field is final — using Unsafe or Field.set() bypasses the final check
// But JIT may have cached the value, so the change might not be visible
```

---

**Q16. How does Mockito use reflection to create mocks?**

Mockito creates mock objects using:
1. **CGLIB/ByteBuddy** to create subclasses of the class to mock
2. **`Objenesis`** to instantiate without calling constructors
3. **`InvocationHandler`** equivalent to intercept all method calls

```java
// What Mockito.mock(OrderService.class) roughly does:
// 1. Create a subclass of OrderService using ByteBuddy
// 2. Override all non-final methods to go through an interceptor
// 3. Instantiate without constructor (Objenesis)

@Test
void testOrderService() {
    OrderRepository mockRepo = Mockito.mock(OrderRepository.class);

    // Stubbing: when() uses InvocationHandler to record the call
    when(mockRepo.findById(1L)).thenReturn(Optional.of(testOrder));

    // Verification: verify() checks the method was called
    verify(mockRepo, times(1)).findById(1L);
    verify(mockRepo, never()).deleteById(any());
}

// @MockBean in Spring:
// - Replaces the actual bean in ApplicationContext with a Mockito mock
// - Spring uses BeanDefinitionRegistry.registerBeanDefinition() to replace it
```

---

**Q17. How can you use reflection to implement a simple dependency injection container?**

```java
// Simplified DI container using reflection
public class SimpleContainer {

    private final Map<Class<?>, Object> singletons = new ConcurrentHashMap<>();

    @SuppressWarnings("unchecked")
    public <T> T get(Class<T> type) {
        return (T) singletons.computeIfAbsent(type, this::createInstance);
    }

    private Object createInstance(Class<?> type) {
        try {
            // Find constructor annotated with @Inject (or use no-arg)
            Constructor<?> constructor = findInjectableConstructor(type);
            constructor.setAccessible(true);

            // Resolve constructor parameters recursively
            Object[] args = Arrays.stream(constructor.getParameterTypes())
                .map(this::get)  // Recursive dependency resolution
                .toArray();

            Object instance = constructor.newInstance(args);

            // Field injection
            for (Field field : type.getDeclaredFields()) {
                if (field.isAnnotationPresent(Inject.class)) {
                    field.setAccessible(true);
                    field.set(instance, get(field.getType()));
                }
            }

            return instance;

        } catch (Exception e) {
            throw new RuntimeException("Cannot create " + type.getName(), e);
        }
    }

    private Constructor<?> findInjectableConstructor(Class<?> type) throws NoSuchMethodException {
        return Arrays.stream(type.getDeclaredConstructors())
            .filter(c -> c.isAnnotationPresent(Inject.class))
            .findFirst()
            .orElse(type.getDeclaredConstructor());  // Fallback to no-arg
    }
}
```

---

**Q18. What are method handles and how do they compare to reflection?**

`MethodHandle` (Java 7+) in `java.lang.invoke` — more JIT-friendly alternative to reflection:

```java
MethodHandles.Lookup lookup = MethodHandles.lookup();

// Virtual method handle
MethodHandle handle = lookup.findVirtual(
    String.class, "toUpperCase",
    MethodType.methodType(String.class));  // Return type, then parameter types

String result = (String) handle.invoke("hello");  // "HELLO"

// Static method
MethodHandle parseIntHandle = lookup.findStatic(
    Integer.class, "parseInt",
    MethodType.methodType(int.class, String.class));

int num = (int) parseIntHandle.invoke("42");

// Constructor
MethodHandle newOrder = lookup.findConstructor(
    Order.class, MethodType.methodType(void.class, Long.class, String.class));
Order order = (Order) newOrder.invoke(1L, "PENDING");

// vs Reflection:
// Method.invoke() — boxing, security checks
// MethodHandle.invoke() — JIT can inline, no boxing for exact invocation
// invokeExact() — exact types required, fastest (can be inlined by JIT)

double d = (double) handle.invokeExact(42.0, 2.0);  // invokeExact — no boxing
```

**Performance comparison:**
- Direct method call: baseline
- `MethodHandle.invokeExact()`: near-direct after JIT
- `MethodHandle.invoke()`: ~2x slower (type conversion)
- `Method.invoke()`: ~10-50x slower
