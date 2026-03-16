# Spring Core DI — Trap Questions

> Subtle questions designed to catch candidates who learned Spring from tutorials but haven't used it in production.

---

## Trap 1: @Transactional on a Private Method

**Question:** Will this work?

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        saveOrderItems(order);
    }

    @Transactional  // Will this transaction be applied?
    private void saveOrderItems(Order order) {
        itemRepository.saveAll(order.getItems());
    }
}
```

**Answer:** No — it will be completely ignored.

Spring creates a CGLIB proxy that wraps `OrderService`. The proxy can only intercept public methods called through the proxy. Private methods:
1. Cannot be overridden by CGLIB
2. Are called directly (not through the proxy) even when called from within the class

The `@Transactional` annotation on `saveOrderItems()` has zero effect.

**Fix:** Make the method public and inject self, or extract to a separate `@Service` class.

---

## Trap 2: @Autowired on a Final Field

**Question:** Will this compile and work?

```java
@Service
public class UserService {

    @Autowired
    private final UserRepository userRepository;  // final + @Autowired
}
```

**Answer:** It compiles, but it won't work the way you expect.

- `final` fields must be initialized in the constructor
- `@Autowired` on a field means Spring injects after construction via reflection
- The field must have an initial value (usually `null`) set at declaration
- Spring can inject into `final` fields via `Field.set()` with `setAccessible(true)`, but this is fragile

In practice, this is considered bad practice. Use constructor injection:

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;  // truly final
    }
}
```

---

## Trap 3: @Component vs @Bean — Third-Party Class

**Question:** You want to use `ObjectMapper` (Jackson) as a Spring bean. Which annotation works?

```java
// Option A
@Component
public class ObjectMapper { ... }  // Jackson class — can't add @Component

// Option B
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}
```

**Answer:** Only Option B works. You cannot add `@Component` to a third-party class you don't own. `@Bean` is used in `@Configuration` classes to register beans you instantiate manually, which is the only way to register third-party classes.

---

## Trap 4: @Primary in Multiple Modules

**Question:** Two separate `@Configuration` classes in different modules both define a `@Primary` bean of the same type. What happens?

```java
// Module A
@Configuration
public class ModuleAConfig {
    @Primary @Bean
    public PaymentGateway stripeGateway() { return new StripeGateway(); }
}

// Module B
@Configuration
public class ModuleBConfig {
    @Primary @Bean
    public PaymentGateway paypalGateway() { return new PaypalGateway(); }
}
```

**Answer:** `NoUniqueBeanDefinitionException` — Spring finds two `@Primary` beans and cannot decide between them. Only one `@Primary` bean per type is allowed. The fix is to remove `@Primary` from one and use `@Qualifier` at injection points.

---

## Trap 5: @Autowired Collection Injection Order

**Question:** In what order are beans injected in a list?

```java
public interface Plugin { void execute(); }

@Component @Order(3) public class PluginC implements Plugin { ... }
@Component @Order(1) public class PluginA implements Plugin { ... }
@Component @Order(2) public class PluginB implements Plugin { ... }

@Service
public class PluginRunner {
    @Autowired
    private List<Plugin> plugins;  // What order?
}
```

**Answer:** The list is sorted by `@Order` value: `[PluginA, PluginB, PluginC]` (ascending order, lowest value first).

**Trap follow-up:** What if you use `@Autowired private Set<Plugin>` instead?

**Answer:** A `Set` has no defined ordering — `@Order` is ignored. Use `List` when order matters.

---

## Trap 6: @Conditional Bean and Test Context

**Question:** You have:

```java
@Bean
@ConditionalOnProperty(name = "payment.enabled", havingValue = "true")
public PaymentService paymentService() { ... }
```

Your test doesn't set `payment.enabled`. Another bean `OrderService` autowires `PaymentService`. What happens in the test?

**Answer:** `UnsatisfiedDependencyException` — `PaymentService` is not registered (condition is false), so `OrderService` cannot be autowired.

Fix options:
1. Add `@TestPropertySource(properties = "payment.enabled=true")` to enable it
2. Add `required = false` to `@Autowired` in `OrderService`
3. Use `@ConditionalOnProperty(matchIfMissing = false)` and mock `PaymentService` in test

---

## Trap 7: @PostConstruct and Exception Handling

**Question:** What happens if `@PostConstruct` throws a `RuntimeException`?

```java
@Component
public class CacheWarmer {

    @Autowired
    private CacheService cacheService;

    @PostConstruct
    public void warmUp() {
        cacheService.loadAllProducts();  // throws RuntimeException if DB is down
    }
}
```

**Answer:** The entire Spring application context fails to start. The exception propagates up through `BeanCreationException`, and the application exits with an error.

This is a common production problem: if any `@PostConstruct` method fails, the whole app won't start, even if the affected bean is not critical.

**Fix:** Wrap with try-catch and log, or make the warm-up asynchronous via `@EventListener(ApplicationReadyEvent.class)`:

```java
@EventListener(ApplicationReadyEvent.class)
public void warmUp() {
    try {
        cacheService.loadAllProducts();
    } catch (Exception e) {
        log.warn("Cache warm-up failed, will serve cold cache", e);
    }
}
```

---

## Trap 8: @Scope("prototype") with @Configuration

**Question:** Does this work as expected?

```java
@Configuration
public class AppConfig {

    @Bean
    @Scope("prototype")
    public ExpensiveObject expensiveObject() {
        return new ExpensiveObject();
    }

    @Bean
    public ServiceA serviceA() {
        return new ServiceA(expensiveObject());  // calls @Bean method
    }

    @Bean
    public ServiceB serviceB() {
        return new ServiceB(expensiveObject());  // calls @Bean method
    }
}
```

**Answer:** In `@Configuration` (full mode), Spring intercepts calls to `@Bean` methods. However, for prototype beans, Spring creates a new instance each time the `@Bean` method is called (that's the point of prototype scope).

`ServiceA` and `ServiceB` will each get a different `ExpensiveObject` instance — this is the correct behavior. Each call to `expensiveObject()` goes through the Spring proxy and returns a new prototype instance.

---

## Trap 9: Field Injection and Immutability

**Question:** A colleague says: "I use field injection to keep my code clean and simple. What's wrong with that?"

**Answer:** Beyond style, there are concrete technical issues:

1. **No immutability**: Cannot declare field as `final`, so field can be reassigned accidentally
2. **Hidden dependencies**: Constructor signature doesn't show what the class needs — violates good OOP design
3. **Test difficulty**: To test without Spring, you must use reflection (`ReflectionTestUtils.setField()`) or `@InjectMocks` with Mockito — messy and error-prone
4. **NullPointerException in tests**: If you forget to inject, field is `null` — constructor injection fails at construction time instead
5. **Violates Dependency Inversion**: You're depending on Spring's DI mechanism rather than explicitly declaring dependencies

---

## Trap 10: @Autowired vs @Inject vs @Resource

**Question:** What's the difference between `@Autowired`, `@Inject`, and `@Resource`?

| Annotation | Package | Match Strategy |
|-----------|---------|----------------|
| `@Autowired` | Spring (`org.springframework`) | By type, then by name (with @Qualifier) |
| `@Inject` | Jakarta EE (`jakarta.inject`) | By type, then by name (with @Named) |
| `@Resource` | Jakarta EE (`jakarta.annotation`) | By name first, then by type |

```java
// @Autowired — Spring-specific, by type
@Autowired
@Qualifier("stripeGateway")
private PaymentGateway gateway;

// @Inject — JSR-330 standard, by type
@Inject
@Named("stripeGateway")
private PaymentGateway gateway;

// @Resource — JSR-250, by name first
@Resource(name = "stripeGateway")
private PaymentGateway gateway;
```

Spring supports all three. `@Autowired` is most commonly used in Spring projects. `@Inject` is preferred when you want to avoid Spring coupling in domain classes. `@Resource` is useful when name-based injection is clearer.

---

## Trap 11: Eager vs Lazy Initialization

**Question:** When are Spring singleton beans initialized by default?

**Answer:** Singleton beans are initialized **eagerly** at context startup by default. All singleton beans are instantiated when the `ApplicationContext` is created.

This means:
- Errors in beans fail fast at startup
- `@PostConstruct` methods run at startup
- Any bean that needs a network connection, DB, etc., may fail if those aren't available at startup

```java
// Make a bean lazy (only created on first use)
@Component
@Lazy
public class ExpensiveService {
    // Not created until first @Autowired injection is used
}

// Or make ALL beans lazy (Spring Boot 2.2+)
spring.main.lazy-initialization=true
```

**Trap follow-up:** What's a downside of `spring.main.lazy-initialization=true`?

**Answer:** Configuration errors are hidden until the bean is first used, potentially causing failures at runtime under load rather than at startup. Also, first request latency increases because beans are created on demand.

---

## Trap 12: @ComponentScan Scope

**Question:** Your new `@Service` class isn't being picked up by Spring. What's the most likely reason?

**Answer:** The class is in a package that isn't scanned.

`@SpringBootApplication` (which includes `@ComponentScan`) scans the package of the annotated class and all its sub-packages. If your service is in a different package tree, it won't be found.

```
com.example.myapp
├── MyApplication.java       ← @SpringBootApplication scans here
├── service
│   └── UserService.java     ← ✓ Found (sub-package)
└── ...

com.example.payments          ← ✗ Not found (different package tree)
└── PaymentService.java
```

Fix: Either move the class into the scanned package tree, or add `@ComponentScan(basePackages = {"com.example.myapp", "com.example.payments"})`.
