# Reflection — Scenario Questions

> 14 real-world scenarios on reflection, dynamic proxies, annotation processing, and framework internals.

---

## Scenario 1: Custom Object Mapper

**Situation:** Build a simple CSV-to-POJO mapper using reflection that reads field names from a header row.

```java
public class CsvObjectMapper {

    public <T> List<T> parse(String csvContent, Class<T> targetClass) throws Exception {
        String[] lines = csvContent.split("\n");
        String[] headers = lines[0].split(",");

        // Build field map once — cache for performance
        Map<String, Field> fieldMap = Arrays.stream(targetClass.getDeclaredFields())
            .peek(f -> f.setAccessible(true))
            .collect(Collectors.toMap(
                f -> f.getName().toLowerCase(),
                f -> f
            ));

        List<T> results = new ArrayList<>();
        for (int i = 1; i < lines.length; i++) {
            String[] values = lines[i].split(",", -1);
            T instance = targetClass.getDeclaredConstructor().newInstance();

            for (int j = 0; j < headers.length && j < values.length; j++) {
                Field field = fieldMap.get(headers[j].trim().toLowerCase());
                if (field != null) {
                    setFieldValue(field, instance, values[j].trim());
                }
            }
            results.add(instance);
        }
        return results;
    }

    private void setFieldValue(Field field, Object instance, String value) throws Exception {
        Class<?> type = field.getType();
        Object converted;

        if (type == String.class) converted = value;
        else if (type == int.class || type == Integer.class) converted = Integer.parseInt(value);
        else if (type == long.class || type == Long.class) converted = Long.parseLong(value);
        else if (type == BigDecimal.class) converted = new BigDecimal(value);
        else if (type == boolean.class || type == Boolean.class) converted = Boolean.parseBoolean(value);
        else if (type.isEnum()) converted = Enum.valueOf((Class<Enum>) type, value);
        else converted = value;

        field.set(instance, converted);
    }
}
```

---

## Scenario 2: Spring-Like @Autowired Implementation

**Situation:** Build a simple annotation-based dependency injection using reflection.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Autowired {}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Component {}

public class MiniSpring {

    private final Map<Class<?>, Object> beans = new ConcurrentHashMap<>();

    // Register beans
    public void registerBean(Class<?> type, Object instance) {
        beans.put(type, instance);
        // Also register by interface
        for (Class<?> iface : type.getInterfaces()) {
            beans.putIfAbsent(iface, instance);
        }
    }

    // Inject dependencies into object
    public void inject(Object target) throws IllegalAccessException {
        Class<?> clazz = target.getClass();
        while (clazz != null && clazz != Object.class) {
            for (Field field : clazz.getDeclaredFields()) {
                if (field.isAnnotationPresent(Autowired.class)) {
                    Object dependency = beans.get(field.getType());
                    if (dependency == null) {
                        throw new RuntimeException("No bean of type: " + field.getType().getName());
                    }
                    field.setAccessible(true);
                    field.set(target, dependency);
                }
            }
            clazz = clazz.getSuperclass();  // Check parent class fields too
        }
    }

    // Scan and register all @Component classes
    public void scanPackage(String packageName) throws Exception {
        // In practice, use Reflections library or ClassLoader to find classes
        // Simplified: register manually
    }
}
```

---

## Scenario 3: Method Timing Proxy

**Situation:** Wrap any service with a logging proxy that records method execution times.

```java
public class TimingProxyFactory {

    @SuppressWarnings("unchecked")
    public static <T> T createProxy(T target, Class<T> iface) {
        return (T) Proxy.newProxyInstance(
            iface.getClassLoader(),
            new Class[]{iface},
            new TimingInvocationHandler<>(target)
        );
    }

    private static class TimingInvocationHandler<T> implements InvocationHandler {

        private final T target;
        private final Map<String, LongSummaryStatistics> stats = new ConcurrentHashMap<>();

        TimingInvocationHandler(T target) { this.target = target; }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            long start = System.nanoTime();
            try {
                return method.invoke(target, args);
            } catch (InvocationTargetException e) {
                throw e.getCause();
            } finally {
                long elapsedMs = (System.nanoTime() - start) / 1_000_000;
                stats.computeIfAbsent(method.getName(),
                    k -> new LongSummaryStatistics())
                    .accept(elapsedMs);
                log.debug("{}.{}: {}ms", target.getClass().getSimpleName(),
                          method.getName(), elapsedMs);
            }
        }

        public void printStats() {
            stats.forEach((method, stat) -> {
                System.out.printf("%s: count=%d, avg=%.1fms, max=%dms%n",
                    method, stat.getCount(), stat.getAverage(), stat.getMax());
            });
        }
    }
}

// Usage
OrderService proxy = TimingProxyFactory.createProxy(
    new OrderServiceImpl(), OrderService.class);
```

---

## Scenario 4: Generic DAO Implementation with Reflection

**Situation:** Build a generic JDBC DAO that maps ResultSets to POJOs using reflection.

```java
public abstract class GenericJdbcDao<T, ID> {

    private final Class<T> entityClass;
    private final Map<String, Field> columnFieldMap;

    @SuppressWarnings("unchecked")
    protected GenericJdbcDao() {
        // Capture T from subclass generic superclass
        ParameterizedType type =
            (ParameterizedType) getClass().getGenericSuperclass();
        this.entityClass = (Class<T>) type.getActualTypeArguments()[0];

        // Build column -> field mapping from @Column annotations
        this.columnFieldMap = Arrays.stream(entityClass.getDeclaredFields())
            .peek(f -> f.setAccessible(true))
            .filter(f -> f.isAnnotationPresent(Column.class))
            .collect(Collectors.toMap(
                f -> f.getAnnotation(Column.class).name(),
                f -> f
            ));
    }

    protected RowMapper<T> rowMapper() {
        return (rs, rowNum) -> {
            try {
                T instance = entityClass.getDeclaredConstructor().newInstance();
                ResultSetMetaData meta = rs.getMetaData();
                for (int i = 1; i <= meta.getColumnCount(); i++) {
                    String colName = meta.getColumnName(i).toLowerCase();
                    Field field = columnFieldMap.get(colName);
                    if (field != null) {
                        field.set(instance, rs.getObject(i));
                    }
                }
                return instance;
            } catch (Exception e) {
                throw new RuntimeException("Failed to map ResultSet to " + entityClass, e);
            }
        };
    }
}
```

---

## Scenario 5: Annotation-Driven Event System

**Situation:** Allow classes to annotate methods as event handlers using `@Subscribe(EventType.class)`.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Subscribe {
    Class<? extends Event> value();
}

public class AnnotationEventBus {

    // Map: event class → list of (object, method) pairs
    private final Map<Class<?>, List<MethodHandler>> handlers = new ConcurrentHashMap<>();

    record MethodHandler(Object target, Method method) {}

    public void register(Object listener) {
        for (Method method : listener.getClass().getDeclaredMethods()) {
            Subscribe sub = method.getAnnotation(Subscribe.class);
            if (sub != null) {
                method.setAccessible(true);
                handlers.computeIfAbsent(sub.value(), k -> new CopyOnWriteArrayList<>())
                        .add(new MethodHandler(listener, method));
            }
        }
    }

    public void publish(Event event) {
        List<MethodHandler> eventHandlers = handlers.getOrDefault(
            event.getClass(), List.of());

        for (MethodHandler handler : eventHandlers) {
            try {
                handler.method().invoke(handler.target(), event);
            } catch (InvocationTargetException e) {
                log.error("Handler threw exception", e.getCause());
            } catch (IllegalAccessException e) {
                log.error("Cannot invoke handler", e);
            }
        }
    }
}

// Usage
class OrderEventListener {
    @Subscribe(OrderCreatedEvent.class)
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("Order created: {}", event.orderId());
    }

    @Subscribe(OrderShippedEvent.class)
    public void onOrderShipped(OrderShippedEvent event) {
        notificationService.sendShippingEmail(event.orderId());
    }
}
```

---

## Scenario 6: Copy Properties Between Objects

**Situation:** Implement a utility similar to BeanUtils.copyProperties() that copies matching field values between objects.

```java
public class BeanCopier {

    // Cache field maps for performance
    private static final Map<Class<?>, Map<String, Field>> fieldCache = new ConcurrentHashMap<>();

    public static void copyProperties(Object source, Object target) {
        Map<String, Field> sourceFields = getFieldMap(source.getClass());
        Map<String, Field> targetFields = getFieldMap(target.getClass());

        sourceFields.forEach((name, srcField) -> {
            Field tgtField = targetFields.get(name);
            if (tgtField != null && tgtField.getType().isAssignableFrom(srcField.getType())) {
                try {
                    tgtField.set(target, srcField.get(source));
                } catch (IllegalAccessException e) {
                    log.warn("Cannot copy field: {}", name);
                }
            }
        });
    }

    private static Map<String, Field> getFieldMap(Class<?> clazz) {
        return fieldCache.computeIfAbsent(clazz, c -> {
            Map<String, Field> map = new LinkedHashMap<>();
            Class<?> current = c;
            while (current != null && current != Object.class) {
                for (Field field : current.getDeclaredFields()) {
                    if (!Modifier.isStatic(field.getModifiers())) {
                        field.setAccessible(true);
                        map.putIfAbsent(field.getName(), field);  // Child fields override parent
                    }
                }
                current = current.getSuperclass();
            }
            return Collections.unmodifiableMap(map);
        });
    }
}
```

---

## Scenario 7: Configuration Binding via Reflection

**Situation:** Read environment variables and bind them to a configuration class based on field names.

```java
public class EnvironmentConfigBinder {

    public static <T> T bind(Class<T> configClass, String prefix) throws Exception {
        T instance = configClass.getDeclaredConstructor().newInstance();

        for (Field field : configClass.getDeclaredFields()) {
            String envKey = toEnvKey(prefix, field.getName());
            String value = System.getenv(envKey);

            if (value == null) {
                ConfigDefault defaultAnnotation = field.getAnnotation(ConfigDefault.class);
                if (defaultAnnotation != null) value = defaultAnnotation.value();
            }

            if (value != null) {
                field.setAccessible(true);
                field.set(instance, convertValue(value, field.getType()));
            }
        }

        return instance;
    }

    // MY_APP_DATABASE_HOST → MYAPP_DATABASE_HOST
    private static String toEnvKey(String prefix, String fieldName) {
        String snake = fieldName.replaceAll("([A-Z])", "_$1").toUpperCase();
        return prefix.isEmpty() ? snake : prefix + "_" + snake;
    }

    private static Object convertValue(String value, Class<?> type) {
        if (type == String.class) return value;
        if (type == int.class || type == Integer.class) return Integer.parseInt(value);
        if (type == boolean.class || type == Boolean.class) return Boolean.parseBoolean(value);
        if (type == Duration.class) return Duration.parse(value);
        throw new UnsupportedOperationException("No converter for: " + type);
    }
}
```

---

## Scenario 8: Validation Framework using Reflection

**Situation:** Build a simple validation framework that uses annotations and reflection.

```java
// Annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface NotNull { String message() default "must not be null"; }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Min { long value(); String message() default "must be >= {value}"; }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Pattern { String regexp(); String message() default "must match pattern"; }

// Validator
public class ReflectionValidator {

    public List<String> validate(Object object) {
        List<String> errors = new ArrayList<>();
        Class<?> clazz = object.getClass();

        for (Field field : getAllFields(clazz)) {
            field.setAccessible(true);
            Object value;
            try { value = field.get(object); }
            catch (IllegalAccessException e) { continue; }

            String fieldName = field.getName();

            NotNull notNull = field.getAnnotation(NotNull.class);
            if (notNull != null && value == null) {
                errors.add(fieldName + ": " + notNull.message());
                continue;  // Skip other validations if null
            }

            if (value == null) continue;

            Min min = field.getAnnotation(Min.class);
            if (min != null && value instanceof Number num) {
                if (num.longValue() < min.value()) {
                    errors.add(fieldName + ": " + min.message().replace("{value}", "" + min.value()));
                }
            }

            Pattern pattern = field.getAnnotation(Pattern.class);
            if (pattern != null && value instanceof String str) {
                if (!str.matches(pattern.regexp())) {
                    errors.add(fieldName + ": " + pattern.message());
                }
            }
        }
        return errors;
    }

    private List<Field> getAllFields(Class<?> clazz) {
        List<Field> fields = new ArrayList<>();
        while (clazz != null && clazz != Object.class) {
            fields.addAll(Arrays.asList(clazz.getDeclaredFields()));
            clazz = clazz.getSuperclass();
        }
        return fields;
    }
}
```

---

## Scenario 9: Dynamic Method Dispatch / Command Pattern

**Situation:** Route incoming command strings to handler methods using reflection.

```java
// Command routing via reflection
public class CommandDispatcher {

    private final Map<String, Method> commandHandlers;
    private final Object handlerInstance;

    public CommandDispatcher(Object handler) {
        this.handlerInstance = handler;
        this.commandHandlers = new HashMap<>();

        for (Method method : handler.getClass().getDeclaredMethods()) {
            CommandHandler annotation = method.getAnnotation(CommandHandler.class);
            if (annotation != null) {
                method.setAccessible(true);
                commandHandlers.put(annotation.command(), method);
            }
        }
    }

    public Object dispatch(String command, Object... args) throws Exception {
        Method method = commandHandlers.get(command);
        if (method == null) {
            throw new UnknownCommandException("No handler for: " + command);
        }
        try {
            return method.invoke(handlerInstance, args);
        } catch (InvocationTargetException e) {
            throw (Exception) e.getCause();
        }
    }
}

// Handler class
class OrderCommandHandler {
    @CommandHandler(command = "create")
    public Order createOrder(CreateOrderRequest req) { ... }

    @CommandHandler(command = "cancel")
    public void cancelOrder(Long orderId, String reason) { ... }

    @CommandHandler(command = "ship")
    public void shipOrder(Long orderId, String trackingNumber) { ... }
}

// Usage:
dispatcher.dispatch("create", createRequest);
dispatcher.dispatch("cancel", orderId, "Customer request");
```

---

## Scenario 10: Deep Clone via Reflection

**Situation:** Implement deep clone for any object graph without requiring Cloneable.

```java
public class DeepCloner {

    public static <T> T deepClone(T original) throws Exception {
        if (original == null) return null;

        Class<?> clazz = original.getClass();

        // Handle primitives and immutable types
        if (isImmutable(clazz)) return original;

        // Handle arrays
        if (clazz.isArray()) {
            int length = Array.getLength(original);
            Object clone = Array.newInstance(clazz.getComponentType(), length);
            for (int i = 0; i < length; i++) {
                Array.set(clone, i, deepClone(Array.get(original, i)));
            }
            return (T) clone;
        }

        // Create new instance without calling constructor
        // (use Unsafe or Objenesis in production)
        T clone = (T) clazz.getDeclaredConstructor().newInstance();

        // Copy all fields recursively
        Class<?> current = clazz;
        while (current != null && current != Object.class) {
            for (Field field : current.getDeclaredFields()) {
                if (Modifier.isStatic(field.getModifiers())) continue;
                field.setAccessible(true);
                Object value = field.get(original);
                field.set(clone, deepClone(value));
            }
            current = current.getSuperclass();
        }

        return clone;
    }

    private static boolean isImmutable(Class<?> cls) {
        return cls.isPrimitive() || cls == String.class || cls == Integer.class
            || cls == Long.class || cls == Double.class || cls == Boolean.class
            || cls == BigDecimal.class || cls.isEnum();
    }
}
```

---

## Scenario 11: Method Signature Comparison

**Situation:** Check if two methods have the same signature (name, return type, parameters).

```java
public class MethodUtils {

    public static boolean hasSameSignature(Method m1, Method m2) {
        if (!m1.getName().equals(m2.getName())) return false;
        if (!m1.getReturnType().equals(m2.getReturnType())) return false;
        return Arrays.equals(m1.getParameterTypes(), m2.getParameterTypes());
    }

    // Find method in class hierarchy (including interfaces)
    public static Optional<Method> findMethod(Class<?> clazz, String name, Class<?>... paramTypes) {
        Class<?> current = clazz;
        while (current != null) {
            try {
                Method m = current.getDeclaredMethod(name, paramTypes);
                m.setAccessible(true);
                return Optional.of(m);
            } catch (NoSuchMethodException e) {
                // Try parent class
            }
            current = current.getSuperclass();
        }
        // Check interfaces
        for (Class<?> iface : getAllInterfaces(clazz)) {
            try {
                return Optional.of(iface.getMethod(name, paramTypes));
            } catch (NoSuchMethodException e) { /* continue */ }
        }
        return Optional.empty();
    }

    private static Set<Class<?>> getAllInterfaces(Class<?> clazz) {
        Set<Class<?>> interfaces = new LinkedHashSet<>();
        for (Class<?> iface : clazz.getInterfaces()) {
            interfaces.add(iface);
            interfaces.addAll(getAllInterfaces(iface));
        }
        if (clazz.getSuperclass() != null) {
            interfaces.addAll(getAllInterfaces(clazz.getSuperclass()));
        }
        return interfaces;
    }
}
```

---

## Scenario 12: Classpath Scanner

**Situation:** Find all classes in a package that implement a specific interface.

```java
public class ClasspathScanner {

    public List<Class<?>> findImplementations(String packageName, Class<?> interfaceType) {
        List<Class<?>> implementations = new ArrayList<>();

        String path = packageName.replace('.', '/');
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();

        try {
            Enumeration<URL> resources = classLoader.getResources(path);
            while (resources.hasMoreElements()) {
                URL resource = resources.nextElement();
                File directory = new File(resource.getFile());
                if (directory.isDirectory()) {
                    scanDirectory(directory, packageName, interfaceType, implementations);
                }
            }
        } catch (IOException e) {
            throw new RuntimeException("Classpath scan failed", e);
        }

        return implementations;
    }

    private void scanDirectory(File directory, String packageName,
                               Class<?> interfaceType, List<Class<?>> found) {
        for (File file : directory.listFiles()) {
            if (file.isDirectory()) {
                scanDirectory(file, packageName + "." + file.getName(), interfaceType, found);
            } else if (file.getName().endsWith(".class")) {
                String className = packageName + "." +
                    file.getName().replace(".class", "");
                try {
                    Class<?> clazz = Class.forName(className);
                    if (interfaceType.isAssignableFrom(clazz)
                            && !clazz.isInterface()
                            && !Modifier.isAbstract(clazz.getModifiers())) {
                        found.add(clazz);
                    }
                } catch (ClassNotFoundException | Error e) {
                    log.debug("Skipping class: {}", className);
                }
            }
        }
    }
}
```

---

## Scenario 13: Builder Generation Helper

**Situation:** Dynamically generate a Builder for any data class using reflection.

```java
// Runtime builder — sets fields via reflection
public class DynamicBuilder<T> {

    private final Class<T> targetClass;
    private final Map<String, Object> fieldValues = new LinkedHashMap<>();

    public static <T> DynamicBuilder<T> of(Class<T> clazz) {
        return new DynamicBuilder<>(clazz);
    }

    private DynamicBuilder(Class<T> clazz) {
        this.targetClass = clazz;
    }

    public DynamicBuilder<T> set(String fieldName, Object value) {
        fieldValues.put(fieldName, value);
        return this;
    }

    public T build() throws Exception {
        T instance = targetClass.getDeclaredConstructor().newInstance();
        for (Map.Entry<String, Object> entry : fieldValues.entrySet()) {
            Field field = findField(targetClass, entry.getKey());
            if (field != null) {
                field.setAccessible(true);
                field.set(instance, entry.getValue());
            } else {
                throw new NoSuchFieldException(entry.getKey() + " in " + targetClass);
            }
        }
        return instance;
    }

    private Field findField(Class<?> clazz, String name) {
        Class<?> current = clazz;
        while (current != null && current != Object.class) {
            try { return current.getDeclaredField(name); }
            catch (NoSuchFieldException e) { current = current.getSuperclass(); }
        }
        return null;
    }
}

// Usage in tests:
Order order = DynamicBuilder.of(Order.class)
    .set("id", 1L)
    .set("status", OrderStatus.PENDING)
    .set("total", new BigDecimal("99.99"))
    .build();
```

---

## Scenario 14: Intercepting Java Serialization

**Situation:** Add encryption to serialized objects without changing the class hierarchy.

```java
// Custom ObjectOutputStream that encrypts the serialized bytes
public class EncryptedObjectOutputStream extends ObjectOutputStream {

    private final Cipher cipher;

    public EncryptedObjectOutputStream(OutputStream out, SecretKey key) throws Exception {
        super(out);
        this.cipher = Cipher.getInstance("AES/GCM/NoPadding");
        GCMParameterSpec spec = new GCMParameterSpec(128, generateIV());
        cipher.init(Cipher.ENCRYPT_MODE, key, spec);
        // Write IV to stream so decryptor can initialize
        out.write(spec.getIV());
    }

    @Override
    protected void writeObjectOverride(Object obj) throws IOException {
        try {
            // Serialize to bytes using reflection
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            try (ObjectOutputStream innerOos = new ObjectOutputStream(baos)) {
                innerOos.writeObject(obj);
            }
            byte[] serialized = baos.toByteArray();
            byte[] encrypted = cipher.doFinal(serialized);

            DataOutputStream dos = new DataOutputStream(this.out);
            dos.writeInt(encrypted.length);
            dos.write(encrypted);
        } catch (Exception e) {
            throw new IOException("Encryption failed", e);
        }
    }
}
```
