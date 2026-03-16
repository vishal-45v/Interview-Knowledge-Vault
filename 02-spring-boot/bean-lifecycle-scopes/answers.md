# Bean Lifecycle & Scopes — Structured Answers

> Reference answers for bean lifecycle, BeanPostProcessor, and scope-related interview questions.

---

## Q1: What Is the Complete Spring Bean Lifecycle?

```
1.  Class discovery (component scan / @Bean method)
2.  BeanDefinition created and registered
3.  BeanFactoryPostProcessor runs (can modify BeanDefinitions)
4.  Bean instantiation (constructor called)
5.  Dependency injection (@Autowired, @Value populated)
6.  BeanNameAware.setBeanName() called
7.  BeanFactoryAware.setBeanFactory() called
8.  ApplicationContextAware.setApplicationContext() called
9.  BeanPostProcessor.postProcessBeforeInitialization()
10. @PostConstruct method called
11. InitializingBean.afterPropertiesSet() called
12. @Bean(initMethod="...") method called
13. BeanPostProcessor.postProcessAfterInitialization()
    ← AOP proxies created here (for @Transactional, @Async, etc.)
14. Bean is ready and in use

--- Application running ---

15. @PreDestroy method called
16. DisposableBean.destroy() called
17. @Bean(destroyMethod="...") method called
```

---

## Q2: Singleton vs Prototype — Key Differences

| Behavior | Singleton | Prototype |
|----------|-----------|-----------|
| Instances | 1 per ApplicationContext | New instance per request |
| Created at | Context startup (eager) | First `getBean()` call |
| `@PostConstruct` | Called once | Called for each instance |
| `@PreDestroy` | Called on context shutdown | **NOT called by Spring** |
| Stored in cache | Yes | No |
| Thread safety | Must handle yourself | Each thread can have its own |
| Use case | Services, repositories, controllers | Stateful objects, commands, non-shared resources |

```java
// Prototype bean — Spring never calls @PreDestroy!
@Component
@Scope("prototype")
public class StatefulProcessor {

    private final List<String> processedItems = new ArrayList<>();

    @PreDestroy  // ← NEVER CALLED for prototype beans!
    public void cleanup() {
        processedItems.clear();  // Won't run automatically
    }

    // You must call cleanup() manually when done with the bean
}
```

---

## Q3: What Does BeanPostProcessor Do?

`BeanPostProcessor` is a hook that intercepts every bean initialization. It wraps or replaces beans.

```java
public interface BeanPostProcessor {
    // Called BEFORE @PostConstruct
    default Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;  // Return the bean (or a replacement/wrapper)
    }

    // Called AFTER @PostConstruct (and afterPropertiesSet)
    // AOP proxies are created here
    default Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }
}
```

### Built-in BeanPostProcessors:

| PostProcessor | Purpose |
|---------------|---------|
| `AutowiredAnnotationBeanPostProcessor` | Processes `@Autowired`, `@Value`, `@Inject` |
| `CommonAnnotationBeanPostProcessor` | Processes `@PostConstruct`, `@PreDestroy`, `@Resource` |
| `AnnotationAwareAspectJAutoProxyCreator` | Creates AOP proxies for `@Transactional`, `@Async`, `@Cacheable` |
| `PersistenceAnnotationBeanPostProcessor` | Processes `@PersistenceContext`, `@PersistenceUnit` |

---

## Q4: What Happens If Two Beans Have the Same Name?

```java
@Configuration
public class Config1 {
    @Bean
    public PaymentService paymentService() {
        return new StripePaymentService();
    }
}

@Configuration
public class Config2 {
    @Bean
    public PaymentService paymentService() {  // Same name!
        return new PaypalPaymentService();
    }
}
```

**Result:** The **last registered** bean wins. Spring overrides earlier bean definitions with later ones.

In Spring Boot, bean overriding is **disabled by default** (since 2.1):

```yaml
spring:
  main:
    allow-bean-definition-overriding: true  # Enable only if needed
```

Without this, you get `BeanDefinitionOverrideException` at startup.

---

## Q5: How Do Scoped Proxies Work?

When a shorter-lived bean (request, session, prototype) is injected into a longer-lived bean (singleton), Spring creates a **scoped proxy**:

```java
// Without scoped proxy — won't work:
@Component
@Scope("request")
public class UserContext {
    private String userId;
}

@Service  // Singleton
public class AuditService {
    @Autowired
    private UserContext userContext;  // Gets ONE UserContext at startup
    // All requests see the same UserContext — WRONG!
}

// With scoped proxy — works:
@Component
@Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserContext {
    private String userId;
}

@Service  // Singleton
public class AuditService {
    @Autowired
    private UserContext userContext;  // Gets a CGLIB proxy
    // Proxy delegates to the correct request-scoped instance per request ✓
}
```

The proxy looks like `UserContext` but delegates to the actual request-scoped `UserContext` for each request thread.

---

## Q6: What Is the Difference Between @PostConstruct and afterPropertiesSet()?

Both are called after all properties are set. `@PostConstruct` is the modern, preferred approach.

| Aspect | @PostConstruct | afterPropertiesSet() |
|--------|---------------|---------------------|
| Mechanism | BeanPostProcessor (annotation) | InitializingBean interface |
| Spring coupling | No (JSR-250 standard) | Yes (Spring-specific) |
| Multiple methods | No (one method only) | No (one method only) |
| Order | Step 10 | Step 11 |
| Recommendation | Preferred | Avoid |

**Order when both present:**
```java
@Component
public class Example implements InitializingBean {

    @PostConstruct
    public void postConstruct() {
        System.out.println("1. @PostConstruct");  // Runs first
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("2. afterPropertiesSet");  // Runs second
    }
}
```
