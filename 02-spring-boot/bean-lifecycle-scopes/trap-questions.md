# Bean Lifecycle & Scopes — Trap Questions

> Tricky lifecycle questions that reveal whether candidates truly understand Spring internals.

---

## Trap 1: @PostConstruct in @Configuration Class

**Question:** Does `@PostConstruct` work in a `@Configuration` class?

```java
@Configuration
public class DatabaseConfig {

    @PostConstruct
    public void init() {
        // Is this called?
        validateDatabaseConnection();
    }

    @Bean
    public DataSource dataSource() { ... }
}
```

**Answer:** Yes, `@PostConstruct` works in `@Configuration` classes. The `@Configuration` class itself is a Spring bean (a CGLIB-proxied singleton). After it's fully instantiated and dependencies are injected, `@PostConstruct` methods are called.

However, using `@PostConstruct` in `@Configuration` is unusual and not recommended. Put initialization logic in `@PostConstruct` on the beans themselves.

---

## Trap 2: Prototype Bean @PreDestroy

**Question:** Will the `@PreDestroy` method be called for this prototype bean?

```java
@Component
@Scope("prototype")
public class DatabaseMigration {

    @PreDestroy
    public void cleanup() {
        log.info("Cleaning up migration resources");
    }
}
```

**Answer:** No. Spring **never calls `@PreDestroy`** for prototype-scoped beans. Spring creates prototype beans on demand but doesn't manage their lifecycle after creation. The container doesn't track prototype instances — it's your responsibility to call cleanup methods.

If you need cleanup for prototype beans, inject a `DisposableBeanAdapter` or call the cleanup method manually when you're done with the bean.

---

## Trap 3: postProcessAfterInitialization Returns Null

**Question:** What happens if `postProcessAfterInitialization()` returns `null`?

```java
@Component
public class BrokenBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (beanName.equals("userService")) {
            return null;  // What happens?
        }
        return bean;
    }
}
```

**Answer:** The behavior changed across Spring versions. In older versions, returning `null` skipped further processing but could cause `NullPointerException` elsewhere. In current Spring, returning `null` from `postProcessAfterInitialization()` means "use the original bean" — but this is implementation-specific behavior.

**The correct contract:** Always return a non-null value. Return `bean` unchanged if you don't want to modify it. Returning `null` is contract violation and leads to undefined behavior.

---

## Trap 4: @Autowired in @PostConstruct

**Question:** Can you use an `@Autowired` dependency in `@PostConstruct`?

```java
@Service
public class CacheService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @PostConstruct
    public void warmUp() {
        redisTemplate.opsForValue().set("status", "ready");
        // Is redisTemplate available here?
    }
}
```

**Answer:** Yes. `@PostConstruct` is called **after** all dependencies are injected (that's the whole point of it). The injection lifecycle is:

1. Constructor called
2. `@Autowired` dependencies injected
3. `@PostConstruct` called

So `redisTemplate` is available in `@PostConstruct`. This is safe.

---

## Trap 5: Context Refresh and BeanPostProcessor Re-execution

**Question:** If you call `applicationContext.refresh()`, are `@PostConstruct` methods called again?

**Answer:** Yes. `refresh()` re-initializes the entire context — all singleton beans are destroyed and recreated from scratch. All `@PostConstruct` methods run again. This can cause:
1. Double initialization
2. Memory leaks (old beans not properly cleaned up)
3. Port binding errors (if beans start servers)

In production, `refresh()` is rarely called directly. Spring Cloud Config's `/actuator/refresh` only refreshes `@RefreshScope` beans, not the entire context.

---

## Trap 6: Order of Multiple @PostConstruct Methods in Inheritance

**Question:** Which `@PostConstruct` runs first?

```java
@Service
public class BaseService {

    @PostConstruct
    public void baseInit() {
        System.out.println("Base @PostConstruct");
    }
}

@Service
public class ChildService extends BaseService {

    @PostConstruct
    public void childInit() {
        System.out.println("Child @PostConstruct");
    }
}
```

**Answer:** The parent class `@PostConstruct` (`BaseInit`) runs first, then the child class (`childInit`). Spring processes annotations from the most superclass downward — same as Java's constructor chaining order.

---

## Trap 7: Singleton Bean with Request-Scoped Proxy — Thread Safety

**Question:** Is this code thread-safe?

```java
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart {
    private final List<Item> items = new ArrayList<>();
    public void addItem(Item item) { items.add(item); }
    public List<Item> getItems() { return items; }
}

@Service
public class CheckoutService {
    @Autowired
    private ShoppingCart cart;  // Request-scoped proxy in singleton
}
```

**Answer:** Yes, it's thread-safe from a Spring perspective. Each HTTP request gets its own `ShoppingCart` instance (new `ArrayList`). Concurrent requests don't share the same `ShoppingCart`. The proxy delegates to the current request's instance.

However, within a single request with multiple threads (e.g., `parallelStream()`), those threads share the same `ShoppingCart` for that request. The `ArrayList` is not thread-safe in that case.

---

## Trap 8: @Bean Method Calling Another @Bean Method

**Question:** How many `DataSource` instances are created?

```java
@Configuration
public class Config {

    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }

    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource());  // Calls dataSource()
    }

    @Bean
    public NamedParameterJdbcTemplate namedJdbcTemplate() {
        return new NamedParameterJdbcTemplate(dataSource());  // Calls dataSource() again
    }
}
```

**Answer:** Only **one** `DataSource` instance is created. `@Configuration` is CGLIB-proxied. Every call to `dataSource()` goes through the proxy, which checks if a `DataSource` bean already exists in the context. If it does, the existing bean is returned rather than calling the actual method again.

If this were a `@Component` class (lite mode), two separate `HikariDataSource` instances would be created.
