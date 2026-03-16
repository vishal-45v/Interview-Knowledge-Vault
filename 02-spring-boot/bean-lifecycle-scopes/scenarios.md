# Bean Lifecycle & Scopes — Scenarios

> 20+ real-world Spring bean lifecycle and scope scenarios for Senior Java interviews.

---

## Scenario 1: @PostConstruct vs Constructor for Initialization

**Problem:** You need to initialize a cache by loading data from the database after the bean is fully configured. A colleague does it in the constructor:

```java
@Service
public class ProductCacheService {

    @Autowired
    private ProductRepository productRepository;

    public ProductCacheService() {
        // This throws NullPointerException!
        List<Product> products = productRepository.findAll();
    }
}
```

**Why it fails:** The constructor runs before Spring injects dependencies. `productRepository` is `null` at constructor time.

**Fix — Use @PostConstruct:**

```java
@Service
public class ProductCacheService {

    @Autowired
    private ProductRepository productRepository;

    private final Map<Long, Product> cache = new ConcurrentHashMap<>();

    @PostConstruct
    public void initialize() {
        // Dependencies are injected — safe to use them
        productRepository.findAll()
            .forEach(p -> cache.put(p.getId(), p));
        log.info("Loaded {} products into cache", cache.size());
    }
}
```

**Fix — Constructor injection (better pattern):**

```java
@Service
public class ProductCacheService {

    private final ProductRepository productRepository;
    private final Map<Long, Product> cache = new ConcurrentHashMap<>();

    public ProductCacheService(ProductRepository productRepository) {
        this.productRepository = productRepository;
        // Still can't call productRepository here — Spring hasn't called the constructor yet
        // Actually: with constructor injection, productRepository IS available immediately
    }

    @PostConstruct
    public void initialize() {
        productRepository.findAll()
            .forEach(p -> cache.put(p.getId(), p));
    }
}
```

---

## Scenario 2: Bean Destruction — @PreDestroy for Cleanup

**Problem:** A connection pool or thread pool needs to be properly shut down when the application stops.

```java
@Component
public class BackgroundWorkerPool {

    private final ExecutorService executor = Executors.newFixedThreadPool(10);
    private volatile boolean running = true;

    @PostConstruct
    public void start() {
        for (int i = 0; i < 10; i++) {
            executor.submit(this::processQueue);
        }
        log.info("Worker pool started with 10 threads");
    }

    private void processQueue() {
        while (running) {
            // process tasks
        }
    }

    @PreDestroy
    public void shutdown() {
        log.info("Shutting down worker pool...");
        running = false;
        executor.shutdown();
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

**Note:** `@PreDestroy` is only called for singleton beans. Prototype beans are not tracked by Spring — you must manage their lifecycle manually.

---

## Scenario 3: BeanPostProcessor — Intercepting All Beans

**Problem:** You need to log all bean initializations with their initialization time in your Spring Boot application for performance monitoring.

```java
@Component
public class BeanInitializationTimeLogger implements BeanPostProcessor {

    private final Map<String, Long> startTimes = new ConcurrentHashMap<>();

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        startTimes.put(beanName, System.currentTimeMillis());
        return bean;  // Must return the bean (or a wrapper)!
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        Long startTime = startTimes.remove(beanName);
        if (startTime != null) {
            long elapsed = System.currentTimeMillis() - startTime;
            if (elapsed > 100) {  // Log slow beans
                log.warn("Slow bean initialization: {} took {}ms", beanName, elapsed);
            }
        }
        return bean;
    }
}
```

**Important:** `postProcessAfterInitialization()` is where Spring AOP creates proxies (for `@Transactional`, `@Async`, etc.). If you wrap/replace beans here, you must return the wrapped version.

---

## Scenario 4: BeanFactoryPostProcessor — Modifying Bean Definitions

**Problem:** Before beans are instantiated, you need to programmatically override a bean definition (e.g., change the class used for a bean based on an environment variable):

```java
@Component
public class PaymentGatewayBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        String gatewayType = System.getProperty("payment.gateway", "stripe");

        BeanDefinition beanDef = beanFactory.getBeanDefinition("paymentGateway");

        if ("paypal".equals(gatewayType)) {
            beanDef.setBeanClassName(PaypalGateway.class.getName());
        } else {
            beanDef.setBeanClassName(StripeGateway.class.getName());
        }

        log.info("Configured payment gateway: {}", gatewayType);
    }
}
```

---

## Scenario 5: Singleton Bean Thread Safety

**Problem:** A singleton `@Service` has mutable instance state:

```java
@Service  // Singleton — shared across all threads!
public class OrderProcessor {

    private int processedCount = 0;  // SHARED MUTABLE STATE — BUG!

    @Transactional
    public void processOrder(Order order) {
        // ... process order ...
        processedCount++;  // Race condition! Multiple threads increment concurrently
    }

    public int getProcessedCount() {
        return processedCount;  // Reads stale value
    }
}
```

**Why it's wrong:** Singletons are shared across all request threads. Mutable instance variables cause race conditions.

**Fix — Use atomic types or redesign:**

```java
@Service
public class OrderProcessor {

    // Thread-safe atomic counter
    private final AtomicInteger processedCount = new AtomicInteger(0);

    @Transactional
    public void processOrder(Order order) {
        // ... process order ...
        processedCount.incrementAndGet();  // Thread-safe
    }

    public int getProcessedCount() {
        return processedCount.get();
    }
}
```

**Better Fix — Stateless service, use DB/metrics for counts:**

```java
@Service  // Fully stateless — no instance variables
public class OrderProcessor {

    private final OrderRepository orderRepository;
    private final MeterRegistry meterRegistry;

    @Transactional
    public void processOrder(Order order) {
        orderRepository.save(order.markProcessed());
        meterRegistry.counter("orders.processed").increment();
    }
}
```

---

## Scenario 6: @Scope("request") — Per-Request State

**Problem:** You need to track the current user's context (userId, correlationId, locale) throughout a single request without passing it through every method:

```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {

    private String userId;
    private String correlationId;
    private Locale locale;

    // getters and setters
}
```

```java
@RestController
public class OrderController {

    @Autowired
    private RequestContext requestContext;  // Injected into singleton controller
    // proxyMode = TARGET_CLASS creates a proxy that delegates to the request-scoped bean

    @PostMapping("/orders")
    public ResponseEntity<Order> createOrder(@RequestBody OrderRequest request) {
        // requestContext is specific to THIS request
        requestContext.setUserId(getCurrentUserId());
        requestContext.setCorrelationId(UUID.randomUUID().toString());
        return ResponseEntity.ok(orderService.create(request));
    }
}
```

**Key:** `proxyMode = ScopedProxyMode.TARGET_CLASS` is required when injecting a request-scoped bean into a singleton. Without it, Spring throws an error because a singleton bean cannot hold a direct reference to a request-scoped bean.

---

## Scenario 7: Startup Failure from @PostConstruct

**Problem:** `@PostConstruct` connects to a database to preload data. In production, if the DB is momentarily unavailable during deployment, the app fails to start entirely:

```java
@Component
public class ConfigurationLoader {

    @PostConstruct
    public void load() {
        // Throws if DB is down — PREVENTS APP FROM STARTING
        List<Config> configs = configRepository.findAll();
        configCache.putAll(configs);
    }
}
```

**Better approach — Use ApplicationReadyEvent:**

```java
@Component
public class ConfigurationLoader {

    @EventListener(ApplicationReadyEvent.class)
    public void load(ApplicationReadyEvent event) {
        try {
            List<Config> configs = configRepository.findAll();
            configCache.putAll(configs);
        } catch (Exception e) {
            log.warn("Config preload failed, will retry on first access: {}", e.getMessage());
        }
        // App starts regardless — cache population is best-effort
    }
}
```

---

## Scenario 8: DisposableBean vs @PreDestroy vs destroyMethod

**Problem:** Which lifecycle hook should you use for bean cleanup?

```java
// Option 1: @PreDestroy — preferred for your own classes
@Component
public class ResourceManager {
    @PreDestroy
    public void cleanup() {
        // Called before context is destroyed
        releaseResources();
    }
}

// Option 2: DisposableBean interface — avoid (Spring-specific coupling)
@Component
public class ResourceManager implements DisposableBean {
    @Override
    public void destroy() throws Exception {
        releaseResources();
    }
}

// Option 3: @Bean destroyMethod — for third-party classes
@Configuration
public class Config {
    @Bean(destroyMethod = "close")  // Calls close() on shutdown
    public HikariDataSource dataSource() {
        return new HikariDataSource(hikariConfig);
    }
}

// Option 4: Auto-detected destroyMethod (default for @Bean)
// If the class has a public close() or shutdown() method,
// Spring automatically calls it! This is the default behavior.
@Bean
public SomeExternalLibrary externalLib() {
    return new SomeExternalLibrary();
    // If SomeExternalLibrary.close() exists, it's called automatically
}
```

---

## Scenario 9: Lazy Initialization for Optional Features

**Problem:** An optional feature (PDF generation) requires loading a large library and is rarely used. It slows down startup:

```java
// Without @Lazy: PdfService always created at startup (slow)
@Service
public class PdfService {

    private final PdfLibrary pdfLibrary;

    public PdfService() {
        this.pdfLibrary = new PdfLibrary();  // Takes 3 seconds to initialize!
    }
}

// With @Lazy: Only created when first injected/used
@Service
@Lazy
public class PdfService {

    private final PdfLibrary pdfLibrary;

    public PdfService() {
        this.pdfLibrary = new PdfLibrary();
    }
}

// Inject with @Lazy to defer proxy creation:
@RestController
public class ReportController {

    @Autowired
    @Lazy  // Proxy injected immediately, real bean created on first use
    private PdfService pdfService;
}
```

---

## Scenario 10: ApplicationListener for Startup Events

**Problem:** You need to run initialization tasks at different points in the startup sequence:

```java
@Component
public class AppStartupListener {

    // Fires after context is refreshed, before CommandLineRunner
    @EventListener(ContextRefreshedEvent.class)
    public void onContextRefreshed() {
        log.info("Context refreshed — all beans available");
        // Caution: fires on EVERY refresh (including child contexts)
    }

    // Fires when app is ready to serve requests (after all CommandLineRunners)
    @EventListener(ApplicationReadyEvent.class)
    public void onApplicationReady() {
        log.info("Application ready — all initializations complete");
        // Safe to warm caches, start background jobs here
    }

    // Fires when context is shutting down
    @EventListener(ContextClosedEvent.class)
    public void onContextClosed() {
        log.info("Context closing — perform cleanup");
    }
}
```

---

## Scenario 11: Bean Initialization Order with @DependsOn

**Problem:** `CacheService` must be fully initialized before `ProductService` starts. By default, Spring doesn't guarantee initialization order between unrelated beans:

```java
@Service
@DependsOn("cacheService")  // Ensures cacheService initializes first
public class ProductService {

    @Autowired
    private CacheService cacheService;

    @PostConstruct
    public void init() {
        // cacheService is guaranteed to be fully initialized ✓
        cacheService.preload("products");
    }
}

@Service("cacheService")
public class CacheService {
    @PostConstruct
    public void init() {
        log.info("CacheService initialized");
    }
}
```

**Note:** If `ProductService` injects `CacheService` directly, Spring already handles the dependency order. `@DependsOn` is mainly useful for non-injection dependencies (e.g., when a bean relies on a side effect of another bean's initialization).

---

## Scenario 12: Testing Bean Lifecycle with Mocks

**Problem:** You need to test that `@PostConstruct` runs correctly in a Spring Boot test:

```java
@SpringBootTest
class ProductCacheServiceTest {

    @MockBean  // Replaces real repository with mock
    private ProductRepository productRepository;

    @Autowired
    private ProductCacheService productCacheService;

    @Test
    void shouldLoadProductsOnStartup() {
        // Verify that @PostConstruct loaded data from repository
        verify(productRepository, times(1)).findAll();
        // ProductCacheService was initialized with mock data
    }

    @Test
    void shouldReturnCachedProduct() {
        when(productRepository.findAll()).thenReturn(List.of(
            new Product(1L, "Widget", 9.99)
        ));
        // Re-initialize with mock data
        productCacheService.initialize();

        Optional<Product> product = productCacheService.findById(1L);
        assertThat(product).isPresent();
        assertThat(product.get().getName()).isEqualTo("Widget");
    }
}
```

---

## Scenario 13: SmartLifecycle for Controlled Startup/Shutdown

**Problem:** You need fine-grained control over when a component starts and stops relative to other components (e.g., start consuming from a message queue only after all other beans are ready):

```java
@Component
public class MessageQueueConsumer implements SmartLifecycle {

    private volatile boolean running = false;
    private ExecutorService executor;

    @Override
    public void start() {
        executor = Executors.newSingleThreadExecutor();
        executor.submit(this::consumeMessages);
        running = true;
        log.info("Message queue consumer started");
    }

    @Override
    public void stop() {
        running = false;
        executor.shutdown();
        log.info("Message queue consumer stopped");
    }

    @Override
    public boolean isRunning() {
        return running;
    }

    @Override
    public int getPhase() {
        return Integer.MAX_VALUE;  // Start last, stop first
    }

    @Override
    public boolean isAutoStartup() {
        return true;  // Start automatically with context
    }

    private void consumeMessages() {
        while (running) {
            // consume and process messages
        }
    }
}
```

---

## Scenario 14: @RefreshScope — Beans That Reload Configuration

**Problem:** In a Spring Cloud environment, you need a bean to pick up changed configuration properties without restarting the app:

```java
@Component
@RefreshScope  // Bean is re-created when /actuator/refresh is called
public class PaymentConfig {

    @Value("${payment.maxAmount}")
    private BigDecimal maxAmount;

    @Value("${payment.currency}")
    private String currency;

    public boolean isAmountValid(BigDecimal amount) {
        return amount.compareTo(maxAmount) <= 0;
    }
}
```

When `/actuator/refresh` endpoint is called:
1. Configuration properties are reloaded from config source
2. `@RefreshScope` beans are destroyed
3. They are re-created with new property values on next access

**Note:** Singleton beans without `@RefreshScope` keep their original property values. Only `@RefreshScope` beans pick up changes.

---

## Scenario 15: CommandLineRunner and ApplicationRunner

**Problem:** You need to run code after the Spring context is fully started (e.g., initialize default data):

```java
@Component
@Order(1)  // Runs first
public class DatabaseSeeder implements CommandLineRunner {

    @Autowired
    private UserRepository userRepository;

    @Override
    public void run(String... args) throws Exception {
        if (userRepository.count() == 0) {
            userRepository.save(new User("admin@company.com", "ADMIN"));
            log.info("Default admin user created");
        }
    }
}

@Component
@Order(2)  // Runs second
public class CacheWarmer implements ApplicationRunner {

    @Autowired
    private CacheService cacheService;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // ApplicationArguments provides parsed command-line arguments
        boolean forceFresh = args.containsOption("force-cache-refresh");
        cacheService.warmUp(forceFresh);
    }
}
```

**Difference:** `CommandLineRunner.run(String... args)` receives raw args. `ApplicationRunner.run(ApplicationArguments args)` receives parsed args with `--key=value` support.
