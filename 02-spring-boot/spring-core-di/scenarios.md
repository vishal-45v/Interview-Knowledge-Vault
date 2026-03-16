# Spring Core DI — Scenarios

> 22 scenarios covering Spring dependency injection, bean configuration, and common pitfalls.

---

## Scenario 1: Constructor vs Field Injection — Which to Use?

**Context:** A new Spring Boot project. The team is debating injection styles.

**Question:** Compare all three injection types and recommend the best.

**Answer:**

**Field Injection (avoid):**
```java
@Service
public class OrderService {
    @Autowired
    private OrderRepository orderRepository;  // field injection

    // Problems:
    // 1. Cannot be tested without Spring context
    // 2. Dependencies are hidden — not visible in constructor
    // 3. Can lead to NullPointerException in non-Spring contexts
    // 4. Prevents making fields final
    // 5. Encourages too many dependencies (no friction for adding more)
}
```

**Setter Injection (occasional use):**
```java
@Service
public class OrderService {
    private OrderRepository orderRepository;

    @Autowired  // optional dependency — can be null
    public void setOrderRepository(OrderRepository repo) {
        this.orderRepository = repo;
    }
    // Use when: optional dependency, or circular dependency that can't be resolved otherwise
}
```

**Constructor Injection (recommended):**
```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;  // final — truly immutable
    private final PaymentService paymentService;

    public OrderService(OrderRepository orderRepository, PaymentService paymentService) {
        this.orderRepository = orderRepository;  // fail-fast: NPE at startup if null
        this.paymentService = paymentService;
    }
    // Benefits: testable without Spring, dependencies visible, final fields, fail-fast
}

// Since Spring 4.3, single constructor doesn't need @Autowired:
@Service
@RequiredArgsConstructor  // Lombok generates constructor for final fields
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
}
```

---

## Scenario 2: Circular Dependency Problem

**Context:**
```java
@Service
public class ServiceA {
    @Autowired ServiceB serviceB;
}

@Service
public class ServiceB {
    @Autowired ServiceA serviceA;
}
```

**Problem:** `BeanCurrentlyInCreationException` at startup.

**Question:** Why and how to fix?

**Answer:**

With constructor injection, Spring cannot resolve circular dependencies — A needs B to be created, B needs A. With field injection, Spring can use a two-phase approach (create all beans, then inject) but it's still a design problem.

**Root cause:** Usually a design issue — too tight coupling between services.

**Fix 1 — Break the cycle with an event:**
```java
@Service
public class ServiceA {
    private final ApplicationEventPublisher eventPublisher;
    // Don't inject ServiceB — publish events instead
}

@Service
public class ServiceB implements ApplicationListener<SomeEvent> {
    // React to events from ServiceA
}
```

**Fix 2 — Extract shared logic to a third service:**
```java
@Service
public class SharedService { /* common logic */ }

@Service
public class ServiceA {
    private final SharedService sharedService;
}
@Service
public class ServiceB {
    private final SharedService sharedService;
}
```

**Fix 3 — @Lazy (last resort):**
```java
@Service
public class ServiceA {
    @Autowired @Lazy ServiceB serviceB;  // creates a proxy, resolved lazily
}
```

**Fix 4 — Setter injection (breaks cycle for field injection):**
```java
@Service
public class ServiceA {
    private ServiceB serviceB;
    @Autowired public void setServiceB(ServiceB serviceB) { this.serviceB = serviceB; }
}
```

---

## Scenario 3: @Qualifier and @Primary

**Context:**
```java
@Repository
public class MySqlUserRepo implements UserRepository { ... }

@Repository
public class MongoUserRepo implements UserRepository { ... }

@Service
public class UserService {
    @Autowired
    UserRepository userRepository;  // Which one gets injected?
}
```

**Question:** How does Spring resolve which implementation to inject?

**Answer:**

Spring throws `NoUniqueBeanDefinitionException` if two beans qualify for the same type without disambiguation.

**Fix 1 — @Primary:** Mark the default implementation:
```java
@Repository
@Primary
public class MySqlUserRepo implements UserRepository { ... }
// UserService gets MySqlUserRepo by default
```

**Fix 2 — @Qualifier:** Explicitly name the bean:
```java
@Repository("mysqlRepo")
public class MySqlUserRepo implements UserRepository { ... }

@Repository("mongoRepo")
public class MongoUserRepo implements UserRepository { ... }

@Service
public class UserService {
    @Autowired
    @Qualifier("mongoRepo")
    UserRepository userRepository;  // explicitly choose MongoUserRepo
}
```

**Fix 3 — Match by bean name:**
```java
// Spring autowires by type first, then by name if multiple candidates
@Autowired
UserRepository mysqlUserRepo;  // field name matches bean name
```

**Fix 4 — @Profile-based:**
```java
@Repository
@Profile("mysql")
public class MySqlUserRepo implements UserRepository { ... }

@Repository
@Profile("mongo")
public class MongoUserRepo implements UserRepository { ... }
// Only one active per profile
```

---

## Scenario 4: Prototype Bean in Singleton Context

**Context:**
```java
@Component
@Scope("prototype")  // new instance per injection
public class RequestLogger { ... }

@Service
public class OrderService {
    @Autowired
    RequestLogger logger;  // injected once at startup — same instance forever!
}
```

**Problem:** Even though `RequestLogger` is prototype-scoped, `OrderService` gets the same instance every time because singleton beans are only created once.

**Question:** How to get a fresh prototype instance each time?

**Answer:**

```java
// Option 1 — ObjectProvider (recommended Spring 4.3+):
@Service
public class OrderService {
    private final ObjectProvider<RequestLogger> loggerProvider;

    OrderService(ObjectProvider<RequestLogger> loggerProvider) {
        this.loggerProvider = loggerProvider;
    }

    void processOrder(Order order) {
        RequestLogger logger = loggerProvider.getObject();  // new instance each call
        // use logger
    }
}

// Option 2 — ApplicationContext lookup:
@Service
public class OrderService {
    private final ApplicationContext context;

    void processOrder(Order order) {
        RequestLogger logger = context.getBean(RequestLogger.class);  // new instance
    }
}

// Option 3 — @Lookup method injection:
@Service
public abstract class OrderService {
    @Lookup
    protected abstract RequestLogger createLogger();  // Spring provides implementation

    void processOrder(Order order) {
        RequestLogger logger = createLogger();  // Spring returns new prototype
    }
}
```

---

## Scenario 5: @Autowired on Collection

**Context:** You want to inject all implementations of an interface.

**Question:** How does Spring handle this?

**Answer:**

Spring automatically injects all beans of a given type into a `List<T>`, `Set<T>`, or `Map<String, T>`:

```java
interface NotificationSender { void send(Notification n); }

@Component class EmailSender implements NotificationSender { ... }
@Component class SmsSender implements NotificationSender { ... }
@Component class PushSender implements NotificationSender { ... }

@Service
public class NotificationService {
    private final List<NotificationSender> senders;

    // Spring injects all NotificationSender implementations!
    public NotificationService(List<NotificationSender> senders) {
        this.senders = senders;
    }

    public void notify(Notification n) {
        senders.forEach(sender -> sender.send(n));
    }
}

// Map injection — bean name as key:
public NotificationService(Map<String, NotificationSender> senderMap) {
    // senderMap: {"emailSender": EmailSender, "smsSender": SmsSender, ...}
}

// Control ordering with @Order or @Priority:
@Component
@Order(1)  // first in the list
public class EmailSender implements NotificationSender { ... }
```

---

## Scenario 6: Conditional Bean Registration

**Context:** A service should only be created in production profile, with a mock in test.

**Question:** Implement profile-based bean switching.

**Answer:**

```java
// Production bean:
@Service
@Profile("production")
public class RealPaymentService implements PaymentService {
    public PaymentResult process(Payment payment) { /* real payment gateway */ }
}

// Test/dev bean:
@Service
@Profile("!production")  // all non-production profiles
public class MockPaymentService implements PaymentService {
    public PaymentResult process(Payment payment) {
        return PaymentResult.success("mock-transaction-id");
    }
}

// Activate in application.properties:
// spring.profiles.active=production

// Or with @ConditionalOnProperty:
@Service
@ConditionalOnProperty(name = "feature.newPaymentSystem", havingValue = "true")
public class NewPaymentService implements PaymentService { ... }

@Service
@ConditionalOnMissingBean(PaymentService.class)
public class DefaultPaymentService implements PaymentService { ... }
```

---

## Scenario 7: Bean Naming Conflicts

**Context:** Two `@Configuration` classes define a `DataSource` bean.

**Question:** What happens? How to resolve?

**Answer:**

If two beans have the same name (class name with lowercase first letter by default), the second definition **overrides** the first. Spring Boot disables bean overriding by default since 2.1.

```java
// Both define a "dataSource" bean:
@Configuration
class PrimaryConfig {
    @Bean DataSource dataSource() { return primaryDS; }
}

@Configuration
class TestConfig {
    @Bean DataSource dataSource() { return testDS; }  // overrides primary!
}

// Exception with Spring Boot 2.1+:
// BeanDefinitionOverrideException: Invalid bean definition with name 'dataSource'

// Fix 1 — use different names:
@Bean("primaryDataSource")
DataSource primaryDataSource() { ... }

@Bean("testDataSource")
DataSource testDataSource() { ... }

// Fix 2 — allow overriding (risky but sometimes needed):
// spring.main.allow-bean-definition-overriding=true

// Fix 3 — use @ConditionalOnMissingBean:
@Bean
@ConditionalOnMissingBean(DataSource.class)
DataSource defaultDataSource() { ... }  // only created if no other DataSource
```

---

## Scenario 8: Injecting values with @Value

**Context:** Inject configuration properties into beans.

**Answer:**

```java
@Service
public class EmailService {
    @Value("${mail.host:smtp.gmail.com}")      // with default value
    private String mailHost;

    @Value("${mail.port:587}")
    private int mailPort;

    @Value("${mail.recipients}")               // required — no default
    private List<String> recipients;           // automatically splits comma-separated values

    @Value("#{systemProperties['java.home']}") // SpEL expression
    private String javaHome;

    @Value("#{T(java.lang.Math).PI}")          // SpEL with method/constant
    private double pi;

    @Value("#{@otherBean.someProperty}")       // inject from another bean
    private String otherProperty;
}
```

**Better alternative — @ConfigurationProperties:**
```java
@ConfigurationProperties(prefix = "mail")
@Validated
public class MailProperties {
    @NotBlank private String host = "smtp.gmail.com";
    @Min(1) @Max(65535) private int port = 587;
    private List<String> recipients = new ArrayList<>();

    // getters/setters
}

@Service
@RequiredArgsConstructor
public class EmailService {
    private final MailProperties mailProperties;  // type-safe, validated
}
```

---

## Scenario 9: Spring IoC Container Startup Failure

**Context:** Application fails to start with `BeanCreationException`.

**Question:** How do you diagnose and fix bean creation failures?

**Answer:**

Common causes and fixes:

```
BeanCreationException: Error creating bean with name 'orderService'
→ Look at the "Caused by:" chain — root cause is usually lower in the stack

Common causes:
1. Missing @Component/@Service/@Repository on a dependency
2. Missing dependency in classpath (class not found)
3. Ambiguous bean (NoUniqueBeanDefinitionException)
4. Circular dependency (BeanCurrentlyInCreationException)
5. Failed @PostConstruct (initialization exception)
6. Missing configuration property (@Value for required property)
7. Failed condition (@Conditional not met as expected)
```

```java
// Diagnose with actuator:
// GET /actuator/beans — list all beans and their dependencies
// GET /actuator/conditions — show @Conditional evaluation results
// GET /actuator/configprops — show configuration properties

// Debug logging:
logging.level.org.springframework=DEBUG  // see all bean creation steps
```

---

## Scenario 10: ApplicationContext vs BeanFactory

**Question:** What is the difference?

**Answer:**

`BeanFactory` is the basic IoC container. `ApplicationContext` extends `BeanFactory` with enterprise features:

```
BeanFactory:
- Basic DI
- Lazy initialization by default

ApplicationContext (extends BeanFactory):
- Eager initialization of singleton beans
- ApplicationEvent publishing (ApplicationEventPublisher)
- MessageSource (internationalization)
- AOP integration
- @Transactional support
- @Async support
- Environment abstraction (profiles, properties)
- Resource loading
```

```java
// BeanFactory (rarely used directly):
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));

// ApplicationContext variants:
ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);
// Spring Boot creates a web-aware version automatically

// Programmatic access (if needed):
@Service
public class MyService implements ApplicationContextAware {
    private ApplicationContext context;

    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        this.context = ctx;
    }
}
// Or inject directly:
@Autowired
ApplicationContext applicationContext;
```

---

## Scenario 11: @Bean vs @Component

**Question:** When do you use @Bean in @Configuration vs @Component annotations?

**Answer:**

```java
// @Component (and @Service, @Repository, @Controller) — component scanning:
@Service
public class UserService { ... }
// Use when: you OWN the class (source code)

// @Bean in @Configuration — explicit bean declaration:
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();  // third-party class!
    }

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}
// Use when: you DON'T own the class (library class), need custom initialization,
//           or need multiple beans of the same type with different configs
```

**@Bean lifecycle methods:**
```java
@Bean(initMethod = "init", destroyMethod = "cleanup")
public SomeService someService() { ... }
// Or:
@Bean
@Scope("prototype")
public ExpensiveObject expensiveObject() { ... }
```

---

## Scenario 12: Injecting List of Beans with Ordering

**Context:** Multiple validators need to run in a specific order.

**Answer:**

```java
@Component
@Order(1)
public class NotNullValidator implements Validator<Order> { ... }

@Component
@Order(2)
public class BusinessRulesValidator implements Validator<Order> { ... }

@Component
@Order(3)
public class StockValidator implements Validator<Order> { ... }

@Service
public class OrderValidationService {
    private final List<Validator<Order>> validators;

    public OrderValidationService(List<Validator<Order>> validators) {
        this.validators = validators;  // ordered: NotNull, BusinessRules, Stock
    }

    public ValidationResult validate(Order order) {
        return validators.stream()
            .map(v -> v.validate(order))
            .filter(r -> !r.isValid())
            .findFirst()
            .orElse(ValidationResult.valid());
    }
}
```

---

## Scenario 13: Lazy Bean Initialization

**Context:** Some beans are expensive to create and not always needed.

**Answer:**

```java
// Globally lazy (not recommended — delays failure detection):
spring.main.lazy-initialization=true

// Per-bean lazy:
@Service
@Lazy  // only created when first accessed
public class ExpensiveAnalyticsService {
    public ExpensiveAnalyticsService() {
        // expensive initialization — only paid when first needed
        loadMLModel();
    }
}

// Lazy injection:
@Service
public class OrderService {
    @Autowired
    @Lazy
    private ExpensiveAnalyticsService analytics;
    // analytics initialized only on first call to analytics.something()
}

// ObjectProvider for lazy access:
@Service
public class OrderService {
    private final ObjectProvider<ExpensiveAnalyticsService> analyticsProvider;

    void analyzeOrder(Order order) {
        if (featureFlags.isAnalyticsEnabled()) {
            analyticsProvider.getIfAvailable();  // null if not available
            analyticsProvider.getObject();       // throws if not available
        }
    }
}
```

---

## Scenario 14: Environment and Profiles

**Context:** Different configuration for dev, staging, and production.

**Answer:**

```yaml
# application.yml (common)
app:
  name: MyApp

---
spring:
  config:
    activate:
      on-profile: dev
app:
  database: h2
  debug: true

---
spring:
  config:
    activate:
      on-profile: production
app:
  database: postgres
  debug: false
```

```java
@Configuration
public class DataSourceConfig {
    @Bean
    @Profile("dev")
    DataSource h2DataSource() {
        return new EmbeddedDatabaseBuilder().setType(EmbeddedDatabaseType.H2).build();
    }

    @Bean
    @Profile({"staging", "production"})
    DataSource postgresDataSource(@Value("${db.url}") String url) {
        return DataSourceBuilder.create().url(url).build();
    }
}

// Programmatic profile check:
@Service
public class FeatureService {
    @Autowired Environment env;

    boolean isDebugMode() {
        return env.matchesProfiles("dev");
    }
}
```

---

## Scenario 15: Custom Annotation for Bean Injection

**Context:** Create a custom qualifier annotation.

**Answer:**

```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface PrimaryCache { }

@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface SecondaryCache { }

@Component
@PrimaryCache
public class RedisCacheService implements CacheService { ... }

@Component
@SecondaryCache
public class EhCacheCacheService implements CacheService { ... }

@Service
public class ProductService {
    @Autowired @PrimaryCache CacheService primaryCache;
    @Autowired @SecondaryCache CacheService secondaryCache;
}
```

---

## Scenario 16: Understanding @Autowired Resolution Order

**Context:** Multiple UserRepository beans. Which one gets autowired?

**Question:** Explain the resolution order.

**Answer:**

Spring resolves @Autowired in this order:
1. **By type:** If exactly one bean matches the required type → inject it
2. **By @Primary:** If multiple match and one has @Primary → inject @Primary
3. **By @Qualifier:** If @Qualifier specified → inject matching named bean
4. **By name:** If no Qualifier, check if field/parameter name matches a bean name
5. **Failure:** If still ambiguous → `NoUniqueBeanDefinitionException`

```java
@Repository
public class MySqlUserRepo implements UserRepository { ... }

@Repository
@Primary
public class JpaUserRepo implements UserRepository { ... }

@Service
public class UserService {
    @Autowired
    UserRepository userRepository;  // gets JpaUserRepo (primary)

    @Autowired
    @Qualifier("mySqlUserRepo")     // gets MySqlUserRepo (explicit qualifier)
    UserRepository legacyRepository;

    @Autowired
    UserRepository mySqlUserRepo;   // gets MySqlUserRepo (name match)
}
```

---

## Scenario 17: Dependency Injection in Tests

**Context:** How to write unit tests without loading the Spring context?

**Answer:**

```java
// Unit test — NO Spring context, fast:
class OrderServiceTest {
    private OrderRepository mockRepo = mock(OrderRepository.class);
    private PaymentService mockPayment = mock(PaymentService.class);
    private OrderService orderService;  // construct manually

    @BeforeEach void setUp() {
        orderService = new OrderService(mockRepo, mockPayment);  // constructor injection!
    }

    @Test void testPlaceOrder() {
        Order order = new Order(/* ... */);
        when(mockRepo.save(any())).thenReturn(order);
        when(mockPayment.charge(any())).thenReturn(PaymentResult.success("txn123"));

        OrderResult result = orderService.placeOrder(order);
        assertThat(result.isSuccess()).isTrue();
    }
}

// Integration test — WITH Spring context:
@SpringBootTest
class OrderServiceIntegrationTest {
    @Autowired OrderService orderService;  // full Spring context

    @Test void testEndToEnd() { ... }
}

// Slice test — partial Spring context:
@DataJpaTest  // only JPA-related beans
class OrderRepositoryTest {
    @Autowired OrderRepository repo;
    @Test void testFindByStatus() { ... }
}
```

This is why constructor injection makes tests so much easier — you don't need Spring at all for unit tests.

---

## Scenario 18: @EventListener vs ApplicationListener

**Context:** Reacting to application events.

**Answer:**

```java
// ApplicationListener<E> interface approach:
@Component
public class UserEventListener implements ApplicationListener<UserRegisteredEvent> {
    @Override
    public void onApplicationEvent(UserRegisteredEvent event) {
        sendWelcomeEmail(event.getUser());
    }
}

// @EventListener annotation (simpler, preferred):
@Component
public class UserEventHandlers {
    @EventListener
    public void onUserRegistered(UserRegisteredEvent event) {
        sendWelcomeEmail(event.getUser());
    }

    @EventListener
    @Async  // run in a separate thread
    public void onUserRegisteredAsync(UserRegisteredEvent event) {
        triggerAnalytics(event.getUser());
    }

    @EventListener(condition = "#event.user.premium")  // SpEL condition
    public void onPremiumUserRegistered(UserRegisteredEvent event) {
        assignPremiumBenefits(event.getUser());
    }

    // Listen for multiple event types:
    @EventListener({UserRegisteredEvent.class, UserUpdatedEvent.class})
    public void onUserChanged(ApplicationEvent event) { ... }
}

// Publishing events:
@Service
public class UserService {
    private final ApplicationEventPublisher publisher;

    void registerUser(User user) {
        save(user);
        publisher.publishEvent(new UserRegisteredEvent(this, user));
    }
}
```

---

## Scenario 19: @ConditionalOnBean

**Context:** Configure a bean only if another bean is present.

**Answer:**

```java
@Configuration
public class MetricsConfig {
    // Only configure Prometheus if Micrometer is on classpath:
    @Bean
    @ConditionalOnClass(name = "io.micrometer.core.instrument.MeterRegistry")
    public MetricsCollector metricsCollector(MeterRegistry registry) {
        return new PrometheusMetricsCollector(registry);
    }

    // Only if a DataSource bean exists:
    @Bean
    @ConditionalOnBean(DataSource.class)
    public DatabaseHealthIndicator dbHealthIndicator(DataSource dataSource) {
        return new DatabaseHealthIndicator(dataSource);
    }

    // Only if a property is set:
    @Bean
    @ConditionalOnProperty(name = "metrics.enabled", havingValue = "true", matchIfMissing = false)
    public DetailedMetricsService detailedMetrics() {
        return new DetailedMetricsService();
    }
}
```

---

## Scenario 20: Spring DI in Multi-Module Applications

**Context:** A large application split into multiple Maven modules. How does Spring handle beans across modules?

**Answer:**

Spring scans components based on `@ComponentScan` paths. In a multi-module setup:

```java
// Module A: common
@Component
public class EmailValidator { ... }

// Module B: service (depends on Module A)
@SpringBootApplication
@ComponentScan(basePackages = {
    "com.example.service",    // this module
    "com.example.common"      // module A package
})
public class ServiceApplication { ... }

// Or use @Import:
@Configuration
@Import({CommonModuleConfig.class, PersistenceModuleConfig.class})
public class ServiceConfig { ... }

// Or Spring Boot auto-configuration via META-INF/spring.factories:
// org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
//   com.example.common.CommonAutoConfiguration
```

---

## Scenario 21: Injecting Optional Dependencies

**Context:** A feature requires an optional third-party service.

**Answer:**

```java
// Option 1 — @Autowired with required=false:
@Service
public class OrderService {
    @Autowired(required = false)
    private FraudDetectionService fraudDetectionService;  // null if not available

    void placeOrder(Order order) {
        if (fraudDetectionService != null) {
            fraudDetectionService.check(order);
        }
        // continue regardless
    }
}

// Option 2 — Optional<T> injection (Java 8 style):
@Service
public class OrderService {
    private final Optional<FraudDetectionService> fraudDetection;

    OrderService(Optional<FraudDetectionService> fraudDetection) {
        this.fraudDetection = fraudDetection;
    }

    void placeOrder(Order order) {
        fraudDetection.ifPresent(service -> service.check(order));
    }
}

// Option 3 — Default no-op implementation:
@Bean
@ConditionalOnMissingBean(FraudDetectionService.class)
public FraudDetectionService noOpFraudDetection() {
    return order -> {};  // no-op lambda
}
```

---

## Scenario 22: Singleton Bean Thread Safety

**Context:** A singleton Spring bean has mutable state. Is it thread-safe?

**Answer:**

Singleton beans are NOT thread-safe by default. Spring creates ONE instance shared across all threads. If the bean has mutable instance variables, concurrent access can cause race conditions.

```java
// UNSAFE — singleton bean with mutable state:
@Service
public class RequestCounter {
    private int count = 0;  // shared among all threads!

    public void increment() { count++; }  // race condition
    public int getCount() { return count; }
}

// SAFE — use atomic types for thread-safe mutation:
@Service
public class RequestCounter {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() { count.incrementAndGet(); }
    public int getCount() { return count.get(); }
}

// SAFE — stateless (best design for singletons):
@Service
public class OrderCalculator {
    // No mutable state — all data comes through method parameters
    public BigDecimal calculateTotal(List<OrderItem> items) {
        return items.stream()
            .map(item -> item.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

**Golden rule:** Singleton Spring beans should be stateless (or use thread-safe state management).
