# Spring Core DI — Structured Answers

> Deep-dive reference answers for the most commonly asked Spring DI interview questions.

---

## Q1: How does Spring resolve @Autowired dependencies?

Spring's dependency injection resolution follows a strict order:

```
Resolution Order:
1. Type Match  →  Find all beans matching the injection type
2. Qualifier   →  Filter by @Qualifier("name") if present
3. Primary     →  Choose @Primary bean if multiple remain
4. Name Match  →  Match field/parameter name to bean name
5. Fail        →  NoUniqueBeanDefinitionException if ambiguous
```

### Resolution Algorithm (Simplified)

```java
// Spring internally does something like:
public Object resolveAutowired(Class<?> requiredType, String fieldName) {
    List<String> candidates = beanFactory.getBeanNamesForType(requiredType);

    if (candidates.size() == 1) {
        return beanFactory.getBean(candidates.get(0));
    }

    // Multiple candidates — apply filters
    candidates = filterByQualifier(candidates);     // @Qualifier
    candidates = filterByPrimary(candidates);       // @Primary
    candidates = filterByName(candidates, fieldName); // name match

    if (candidates.size() == 1) {
        return beanFactory.getBean(candidates.get(0));
    }

    throw new NoUniqueBeanDefinitionException(requiredType, candidates);
}
```

### Example: Multiple Implementations

```java
public interface PaymentGateway {
    void process(Payment payment);
}

@Component("stripeGateway")
public class StripeGateway implements PaymentGateway { ... }

@Component("paypalGateway")
@Primary  // Used when no qualifier specified
public class PaypalGateway implements PaymentGateway { ... }

@Service
public class OrderService {

    // Gets PaypalGateway (because @Primary)
    @Autowired
    private PaymentGateway paymentGateway;

    // Gets StripeGateway (because @Qualifier)
    @Autowired
    @Qualifier("stripeGateway")
    private PaymentGateway stripeGateway;

    // Gets StripeGateway (because field name matches bean name)
    @Autowired
    private PaymentGateway stripeGateway;  // field name = bean name
}
```

---

## Q2: Constructor Injection vs Field Injection — Which to Use?

### Constructor Injection (Recommended)

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final AuditService auditService;

    // @Autowired optional in Spring 4.3+ (single constructor)
    public UserService(UserRepository userRepository,
                      EmailService emailService,
                      AuditService auditService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
        this.auditService = auditService;
    }
}
```

**Advantages:**
- Dependencies are `final` — immutable after construction
- Fails fast at startup if dependency missing
- Testable without Spring context (plain `new`)
- Makes circular dependencies immediately visible
- Explicit contract (class signature shows what it needs)

### Field Injection (Avoid)

```java
@Service
public class UserService {

    @Autowired  // DON'T DO THIS
    private UserRepository userRepository;

    @Autowired
    private EmailService emailService;
}
```

**Disadvantages:**
- Cannot make fields `final`
- Hidden dependencies — not visible in constructor
- Requires Spring context to test
- Hides violations of Single Responsibility Principle
- Breaks with serialization/deserialization in some edge cases

### Comparison Table

| Aspect | Constructor | Field | Setter |
|--------|------------|-------|--------|
| Immutability | Yes (final) | No | No |
| Testability | Without Spring | Needs Spring | Partial |
| Null safety | Guaranteed | Not guaranteed | Not guaranteed |
| Circular deps | Fails fast | May hide | May create |
| Verbosity | More | Less | Medium |
| Spring recommendation | Yes | No | Optional deps only |

---

## Q3: How Does Spring Resolve Circular Dependencies?

### Default Behavior (Spring 6+)

```java
// This will FAIL in Spring Boot 3.x by default
@Service
public class ServiceA {
    @Autowired
    private ServiceB serviceB;  // needs ServiceB
}

@Service
public class ServiceB {
    @Autowired
    private ServiceA serviceA;  // needs ServiceA
}
// BeanCurrentlyInCreationException
```

### Why Constructor Injection Fails Fast

```
Creating ServiceA
  → needs ServiceB
    → Creating ServiceB
      → needs ServiceA
        → ServiceA is currently being created!
          → BeanCurrentlyInCreationException ✗
```

### Solutions

**Solution 1: Redesign (Best)**
```java
// Extract common functionality into a third service
@Service
public class SharedService {
    // Common logic extracted here
}

@Service
public class ServiceA {
    private final SharedService sharedService;
    // No dependency on ServiceB
}

@Service
public class ServiceB {
    private final SharedService sharedService;
    // No dependency on ServiceA
}
```

**Solution 2: @Lazy**
```java
@Service
public class ServiceA {
    private final ServiceB serviceB;

    public ServiceA(@Lazy ServiceB serviceB) {
        this.serviceB = serviceB;
        // Spring injects a CGLIB proxy initially
        // Real ServiceB instantiated on first use
    }
}
```

**Solution 3: Setter Injection for One Direction**
```java
@Service
public class ServiceA {
    private ServiceB serviceB;

    @Autowired  // setter injection breaks the cycle
    public void setServiceB(ServiceB serviceB) {
        this.serviceB = serviceB;
    }
}
```

**Solution 4: Enable Circular Refs (Last Resort)**
```yaml
# application.yml — NOT recommended
spring:
  main:
    allow-circular-references: true
```

---

## Q4: What Is the Difference Between @Bean and @Component?

```
@Component          @Bean
    │                   │
Auto-detected       Manually declared
by classpath scan   in @Configuration class
    │                   │
Applied to          Applied to
class definition    method definition
    │                   │
Less control        Full control over
over instantiation  instantiation logic
```

### @Component Example

```java
@Component  // Spring finds this via component scan
public class EmailNotifier implements Notifier {
    @Override
    public void notify(String message) {
        // send email
    }
}
```

### @Bean Example

```java
@Configuration
public class AppConfig {

    @Bean  // Full control: can configure, wrap, condition
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(jdbcUrl);
        ds.setMaximumPoolSize(10);
        ds.setConnectionTimeout(30000);
        return ds;
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .configure(FAIL_ON_UNKNOWN_PROPERTIES, false)
            .registerModule(new JavaTimeModule());
    }
}
```

### When to Use Each

| Use @Component | Use @Bean |
|---------------|-----------|
| Your own class | Third-party class |
| Simple instantiation | Complex initialization |
| Class can be annotated | Source code not available |
| Spring-managed | Need full control |

---

## Q5: How Does @Transactional Work with @Async and DI?

### The Proxy Problem

Spring wraps beans with proxies to intercept method calls for `@Transactional`, `@Async`, `@Cacheable`, etc.

```
Caller ──→ [Spring Proxy] ──→ [Real Bean]
              │
              ├── Opens transaction
              ├── Calls real method
              └── Commits/rolls back
```

### Self-Invocation Bypasses the Proxy

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder(Order order) {
        saveOrder(order);
        sendConfirmation(order);  // self-invocation!
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void sendConfirmation(Order order) {
        // This DOES NOT get its own transaction
        // because the proxy is bypassed!
        emailRepository.save(new EmailLog(order));
    }
}
```

### Fix: Inject Self or Extract Method

```java
@Service
public class OrderService {

    @Autowired
    private OrderService self;  // inject proxy reference

    @Transactional
    public void placeOrder(Order order) {
        saveOrder(order);
        self.sendConfirmation(order);  // goes through proxy ✓
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void sendConfirmation(Order order) {
        emailRepository.save(new EmailLog(order));
    }
}
```

---

## Q6: Prototype Bean in Singleton — The Scope Mismatch Problem

```java
@Component
@Scope("prototype")  // New instance each time
public class PrototypeBean {
    private int counter = 0;
    public void increment() { counter++; }
    public int getCounter() { return counter; }
}

@Service  // Singleton
public class SingletonService {

    @Autowired
    private PrototypeBean prototypeBean;
    // PROBLEM: Only ONE PrototypeBean is injected at startup
    // The singleton holds a fixed reference — prototype behavior is lost!

    public void process() {
        prototypeBean.increment();  // Always the SAME instance
    }
}
```

### Fix: ObjectProvider

```java
@Service
public class SingletonService {

    @Autowired
    private ObjectProvider<PrototypeBean> prototypeBeanProvider;

    public void process() {
        PrototypeBean bean = prototypeBeanProvider.getObject();
        // Gets a NEW PrototypeBean each time ✓
        bean.increment();
    }
}
```

### Fix: ApplicationContext.getBean()

```java
@Service
public class SingletonService implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.applicationContext = ctx;
    }

    public void process() {
        PrototypeBean bean = applicationContext.getBean(PrototypeBean.class);
        bean.increment();
    }
}
```

### Fix: @Lookup Method Injection

```java
@Service
public abstract class SingletonService {

    @Lookup  // Spring overrides this method to return a new prototype
    public abstract PrototypeBean getPrototypeBean();

    public void process() {
        PrototypeBean bean = getPrototypeBean();  // New instance each time ✓
        bean.increment();
    }
}
```

---

## Q7: @ConfigurationProperties vs @Value

### @Value — Simple Property Injection

```java
@Service
public class EmailService {

    @Value("${email.host}")
    private String host;

    @Value("${email.port:587}")  // default value
    private int port;

    @Value("${email.enabled:true}")
    private boolean enabled;

    @Value("${email.recipients}")  // comma-separated list
    private List<String> recipients;
}
```

### @ConfigurationProperties — Structured Binding

```java
@ConfigurationProperties(prefix = "email")
@Component
public class EmailProperties {

    private String host;
    private int port = 587;  // default
    private boolean enabled = true;
    private List<String> recipients = new ArrayList<>();
    private Retry retry = new Retry();

    @Data
    public static class Retry {
        private int maxAttempts = 3;
        private Duration delay = Duration.ofSeconds(1);
    }

    // getters/setters
}
```

```yaml
# application.yml
email:
  host: smtp.gmail.com
  port: 587
  enabled: true
  recipients:
    - admin@company.com
    - ops@company.com
  retry:
    max-attempts: 5
    delay: 2s
```

### Comparison

| Feature | @Value | @ConfigurationProperties |
|---------|--------|--------------------------|
| Complex types | Limited | Full support |
| Validation | No | Yes (@Validated) |
| Relaxed binding | No | Yes (camelCase = kebab-case) |
| IDE support | Limited | Full type-safe |
| Nested objects | No | Yes |
| Lists/Maps | Manual | Automatic |
| Refactoring | Hard | Easy |

### With Validation

```java
@ConfigurationProperties(prefix = "email")
@Validated
@Component
public class EmailProperties {

    @NotBlank
    private String host;

    @Min(1) @Max(65535)
    private int port;

    @Email
    private String fromAddress;

    @NotEmpty
    private List<String> recipients;
}
```

---

## Q8: Bean Lifecycle — Complete Sequence

```
┌─────────────────────────────────────────────────────────────┐
│                    Bean Lifecycle                            │
│                                                             │
│  1. Instantiate (constructor called)                        │
│  2. Populate Properties (@Autowired, @Value)                │
│  3. setBeanName() [if BeanNameAware]                        │
│  4. setBeanFactory() [if BeanFactoryAware]                  │
│  5. setApplicationContext() [if ApplicationContextAware]    │
│  6. postProcessBeforeInitialization() [BeanPostProcessor]   │
│  7. @PostConstruct method                                   │
│  8. afterPropertiesSet() [if InitializingBean]              │
│  9. @Bean(initMethod="...")                                 │
│ 10. postProcessAfterInitialization() [BeanPostProcessor]    │
│     → Bean is READY                                         │
│                                                             │
│     ... Application runs ...                               │
│                                                             │
│ 11. @PreDestroy method                                      │
│ 12. destroy() [if DisposableBean]                           │
│ 13. @Bean(destroyMethod="...")                              │
└─────────────────────────────────────────────────────────────┘
```

### Example: Full Lifecycle

```java
@Component
public class DatabaseConnectionPool
        implements BeanNameAware, ApplicationContextAware,
                   InitializingBean, DisposableBean {

    private String beanName;
    private ApplicationContext context;

    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("3. setBeanName: " + name);
    }

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
        System.out.println("5. setApplicationContext");
    }

    @PostConstruct
    public void postConstruct() {
        System.out.println("7. @PostConstruct - init pool");
        // Initialize connection pool here
    }

    @Override
    public void afterPropertiesSet() {
        System.out.println("8. afterPropertiesSet");
    }

    @PreDestroy
    public void preDestroy() {
        System.out.println("11. @PreDestroy - closing connections");
        // Close all connections
    }

    @Override
    public void destroy() {
        System.out.println("12. destroy()");
    }
}
```

---

## Q9: Spring Events — @EventListener vs ApplicationListener

### Publishing Events

```java
@Service
public class UserService {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    public void registerUser(User user) {
        userRepository.save(user);
        // Publish event after successful save
        eventPublisher.publishEvent(new UserRegisteredEvent(this, user));
    }
}
```

### Listening: @EventListener

```java
@Component
public class EmailNotificationListener {

    @EventListener
    public void handleUserRegistered(UserRegisteredEvent event) {
        // Called synchronously in same thread by default
        emailService.sendWelcomeEmail(event.getUser());
    }

    @EventListener
    @Async  // Runs in async thread pool
    public void handleUserRegisteredAsync(UserRegisteredEvent event) {
        analyticsService.track(event.getUser());
    }
}
```

### @TransactionalEventListener — Runs After Commit

```java
@Component
public class AuditListener {

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onUserRegistered(UserRegisteredEvent event) {
        // Runs AFTER the publishing transaction commits
        // Safe to read the saved user from DB
        auditService.log("User registered: " + event.getUser().getId());
    }

    @TransactionalEventListener(phase = AFTER_ROLLBACK)
    public void onUserRegistrationFailed(UserRegisteredEvent event) {
        // Runs if the transaction rolls back
        notificationService.alert("Registration failed for: " + event.getUser().getEmail());
    }
}
```

### Custom Event

```java
public class UserRegisteredEvent extends ApplicationEvent {

    private final User user;

    public UserRegisteredEvent(Object source, User user) {
        super(source);
        this.user = user;
    }

    public User getUser() { return user; }
}
```
