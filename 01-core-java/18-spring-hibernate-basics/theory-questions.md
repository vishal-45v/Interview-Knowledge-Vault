# Spring & Hibernate Basics — Theory Questions

---

## Question 1: How does the Spring IoC container work? What is BeanFactory vs ApplicationContext?

**IoC (Inversion of Control):** Objects don't create their dependencies — the container creates and injects them. This is Dependency Injection (DI).

**BeanFactory:** Basic container. Lazy initialization by default. Minimal features.

**ApplicationContext:** Extends BeanFactory. Eager initialization, internationalization, event publishing, AOP, environment abstraction. Use ApplicationContext in all Spring applications.

```
ApplicationContext initialization sequence:
1. Read @Configuration classes / XML / component scan
2. Create BeanDefinitions (metadata, not instances yet)
3. Run BeanFactoryPostProcessors (e.g., PropertySourcesPlaceholderConfigurer)
4. Create and initialize beans (singleton scope: eagerly)
5. Run BeanPostProcessors (AOP proxies created here)
6. Publish ContextRefreshedEvent
```

```java
@Configuration
@ComponentScan("com.example")
@PropertySource("classpath:application.properties")
public class AppConfig {

    @Bean
    public OrderService orderService(OrderRepository repo, PaymentService payment) {
        return new OrderService(repo, payment);
        // Spring manages lifecycle, injects dependencies
    }
}

// Container lifecycle awareness
@Component
public class CacheInitializer {

    @PostConstruct  // called after dependency injection
    public void warmupCache() { ... }

    @PreDestroy  // called before container shutdown
    public void flushCache() { ... }
}
```

---

## Question 2: What are the different Spring bean scopes?

| Scope | Description | Use Case |
|---|---|---|
| `singleton` | One instance per container (default) | Stateless services |
| `prototype` | New instance per injection request | Stateful objects, expensive objects used briefly |
| `request` | One per HTTP request | Request-scoped data (e.g., request context) |
| `session` | One per HTTP session | User-specific state (shopping cart) |
| `application` | One per ServletContext | Shared web app-level state |

```java
@Service
@Scope("prototype")  // or: @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class CsvParser {
    private final List<String[]> rows = new ArrayList<>();  // mutable state — needs prototype
}

// Injecting prototype into singleton — PITFALL!
@Service  // singleton
public class OrderProcessor {
    @Autowired
    private CsvParser parser;  // prototype injected ONCE at construction — acts like singleton!

    // Fix: inject ApplicationContext and call getBean() each time
    // Or use @Lookup to generate a proxy method
    @Lookup
    public CsvParser getCsvParser() { return null; }  // Spring overrides with prototype lookup
}
```

**`@Scope("request")` in production:** Must be proxied to be injectable into singletons:
```java
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { ... }
```

---

## Question 3: Explain @Autowired, @Qualifier, and @Primary. What injection type is preferred?

**Three DI types:**
```java
// 1. Constructor injection (PREFERRED)
@Service
public class OrderService {
    private final OrderRepository repo;  // final — guaranteed non-null, testable

    public OrderService(OrderRepository repo) {  // @Autowired implicit if single constructor
        this.repo = repo;
    }
}

// 2. Setter injection (optional dependencies)
@Service
public class OrderService {
    private MetricsCollector metrics;

    @Autowired(required = false)  // optional
    public void setMetrics(MetricsCollector metrics) {
        this.metrics = metrics;
    }
}

// 3. Field injection (avoid — not testable without Spring)
@Service
public class OrderService {
    @Autowired private OrderRepository repo;  // can't be final, requires Spring for testing
}
```

**`@Qualifier` and `@Primary`:**
```java
@Service
@Primary  // default if no @Qualifier specified
public class PostgresOrderRepository implements OrderRepository { ... }

@Service
@Qualifier("readOnly")
public class ReadReplicaOrderRepository implements OrderRepository { ... }

// Injection with qualifier
@Service
public class ReportService {
    @Autowired
    @Qualifier("readOnly")
    private OrderRepository readRepo;  // specifically uses read replica
}
```

---

## Question 4: How does Spring AOP work? What is a pointcut, advice, and join point?

```
Join Point   — a point in program execution (method call, exception throw)
Pointcut     — predicate that matches join points (expression that selects which methods)
Advice       — action to take at matched join point (before, after, around)
Aspect       — Pointcut + Advice together
```

```java
@Aspect
@Component
public class PerformanceAspect {

    // Pointcut: any method in service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    // Pointcut: methods annotated with @Timed
    @Pointcut("@annotation(com.example.Timed)")
    public void timedMethods() {}

    // Around advice — most powerful, can modify args and return value
    @Around("serviceLayer() && timedMethods()")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            Object result = pjp.proceed();  // invoke actual method
            return result;
        } finally {
            long elapsed = (System.nanoTime() - start) / 1_000_000;
            log.info("{}.{} took {}ms",
                pjp.getTarget().getClass().getSimpleName(),
                pjp.getSignature().getName(),
                elapsed);
        }
    }

    // Before advice
    @Before("execution(* com.example.*.delete*(..))")
    public void logDeletion(JoinPoint jp) {
        log.info("Deleting: {} with args {}", jp.getSignature(), jp.getArgs());
    }

    // AfterReturning — access return value
    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void logResult(Object result) { ... }

    // AfterThrowing — handle exceptions
    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    public void logException(Exception ex) { ... }
}
```

**AOP limitations:**
- Only intercepts Spring-managed beans
- Does not intercept `private` or `final` methods (CGLIB can't subclass final)
- Self-invocation bypasses the proxy: `this.method()` inside the bean skips AOP

---

## Question 5: How does @Transactional work? Explain propagation levels.

**How it works:**
1. Spring wraps the bean in a proxy (CGLIB or JDK dynamic proxy)
2. Before the method: `TransactionManager.getTransaction()` — starts or joins transaction
3. After the method (no exception): `commit()`
4. After a `RuntimeException`: `rollback()` (by default)
5. After a checked exception: **no rollback by default** (configure with `rollbackFor`)

```java
@Transactional(
    isolation = Isolation.READ_COMMITTED,
    propagation = Propagation.REQUIRED,     // default
    rollbackFor = {BusinessException.class},
    timeout = 30,
    readOnly = false
)
public Order placeOrder(CreateOrderRequest request) { ... }
```

**Propagation levels (most important):**

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Use existing transaction; create new if none |
| `REQUIRES_NEW` | Always create new transaction; suspend existing |
| `SUPPORTS` | Use existing if present; no transaction if none |
| `NOT_SUPPORTED` | Run without transaction; suspend existing |
| `MANDATORY` | Must have existing transaction; throw if none |
| `NEVER` | Must NOT have transaction; throw if one exists |
| `NESTED` | Savepoint within existing transaction |

```java
// REQUIRES_NEW example: audit logging should commit even if outer tx rolls back
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logAudit(AuditEntry entry) {
    auditRepository.save(entry);
    // Commits independently of the outer transaction
}

// NESTED example: partial rollback with savepoints
@Transactional
public void processItems(List<Item> items) {
    for (Item item : items) {
        try {
            processItem(item);  // @Transactional(propagation = NESTED)
        } catch (ItemProcessingException e) {
            // savepoint rolled back, outer transaction continues
            log.error("Item {} failed: {}", item.getId(), e.getMessage());
        }
    }
}
```

---

## Question 6: What is the N+1 query problem in Hibernate? How do you fix it?

**N+1 problem:** Fetching a list of N entities, then firing one query per entity to load a relationship.

```java
// Entity with lazy-loaded orders
@Entity
public class Customer {
    @OneToMany(fetch = FetchType.LAZY, mappedBy = "customer")
    private List<Order> orders;
}

// SERVICE CODE — triggers N+1
List<Customer> customers = customerRepository.findAll();  // 1 query: SELECT * FROM customers
customers.forEach(c -> {
    // For EACH customer: SELECT * FROM orders WHERE customer_id = ?
    System.out.println(c.getOrders().size());  // N queries!
    // 1 + N total queries for N customers
});
```

**Solutions:**

```java
// Solution 1: JOIN FETCH in JPQL
@Query("SELECT c FROM Customer c LEFT JOIN FETCH c.orders WHERE c.id IN :ids")
List<Customer> findWithOrders(@Param("ids") List<Long> ids);
// 1 query: SELECT customers.*, orders.* FROM customers LEFT JOIN orders

// Solution 2: EntityGraph
@EntityGraph(attributePaths = {"orders", "orders.items"})
List<Customer> findAll();

// Solution 3: @BatchSize — single batch query per relationship
@OneToMany
@BatchSize(size = 100)
private List<Order> orders;
// Loads orders for 100 customers in one query: IN (id1, id2, ..., id100)

// Solution 4: DTO projection with JOIN (most efficient)
@Query("SELECT new com.example.CustomerSummary(c.id, c.name, COUNT(o)) " +
       "FROM Customer c LEFT JOIN c.orders o GROUP BY c.id, c.name")
List<CustomerSummary> findCustomerSummaries();

// Detection with Hibernate stats
spring.jpa.properties.hibernate.generate_statistics=true
// Log: "HQL: X" — if X >> 1 for a single page load, you have N+1
```

---

## Question 7: Explain the difference between FetchType.LAZY and FetchType.EAGER.

```java
@Entity
public class Order {

    @ManyToOne(fetch = FetchType.EAGER)  // default for @ManyToOne, @OneToOne
    @JoinColumn(name = "customer_id")
    private Customer customer;  // loaded immediately with Order (JOIN)

    @OneToMany(fetch = FetchType.LAZY)  // default for @OneToMany, @ManyToMany
    private List<OrderItem> items;  // SQL proxy — only loaded when items.get(0) called
}
```

**EAGER pitfalls:**
- `@ManyToOne(EAGER)` on 3 levels of relationships → exponential queries
- Cannot avoid loading related entity even when not needed

**LAZY pitfalls:**
- `LazyInitializationException` when accessing lazy collection outside transaction:
```java
// WRONG — session closed after findById()
Order order = orderRepository.findById(id);
// transaction ends here
order.getItems().size();  // LazyInitializationException!

// FIX 1: use JOIN FETCH
// FIX 2: @Transactional on the calling method (keep session open)
// FIX 3: DTO projection — never load entity when you only need a few fields
```

**Best practice:**
- Default: `LAZY` for all relationships
- Load eagerly only at query level when you know you'll need the relationship
- Use DTO projections for read-only queries — skip entity loading entirely

---

## Question 8: What is the Spring Data JPA Repository hierarchy?

```
Repository<T, ID>  (marker interface)
    ↓
CrudRepository<T, ID>  (save, findById, findAll, delete, count)
    ↓
PagingAndSortingRepository<T, ID>  (findAll with Pageable and Sort)
    ↓
JpaRepository<T, ID>  (saveAll, flush, deleteAllInBatch, getOne)
    +
JpaSpecificationExecutor<T>  (findAll(Specification))
```

```java
// Extend JpaRepository for full JPA support
public interface OrderRepository extends JpaRepository<Order, Long>,
        JpaSpecificationExecutor<Order> {

    // Derived query method — Spring generates JPQL automatically
    List<Order> findByCustomerIdAndStatus(Long customerId, OrderStatus status);

    // With Pageable
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);

    // Custom JPQL
    @Query("SELECT o FROM Order o WHERE o.createdAt > :since AND o.totalAmount > :amount")
    List<Order> findHighValueOrdersSince(@Param("since") LocalDateTime since,
                                         @Param("amount") BigDecimal amount);

    // Native SQL
    @Query(value = "SELECT * FROM orders WHERE status = ?1 LIMIT ?2",
           nativeQuery = true)
    List<Order> findTopByStatusNative(String status, int limit);

    // Modifying query
    @Modifying
    @Transactional
    @Query("UPDATE Order o SET o.status = :status WHERE o.id IN :ids")
    int updateStatuses(@Param("ids") List<Long> ids, @Param("status") OrderStatus status);

    // Projection — only load required columns
    List<OrderSummary> findByCustomerId(Long customerId);
}

// Projection interface
public interface OrderSummary {
    Long getId();
    OrderStatus getStatus();
    BigDecimal getTotalAmount();
    @Value("#{target.customer.name}")  // SpEL
    String getCustomerName();
}
```

---

## Question 9: What is the Hibernate Session (EntityManager) lifecycle and the first-level cache?

**Session/EntityManager states:**
```
Transient → Persistent → Detached → Removed
                ↑
         first-level cache
```

- **Transient:** Object not associated with any session. Changes not tracked.
- **Persistent:** Object associated with a session. Changes tracked and flushed at commit.
- **Detached:** Was persistent, session closed. Changes NOT tracked.
- **Removed:** Scheduled for deletion at flush.

```java
@Transactional
public void updateOrder(Long orderId) {
    Order order = entityManager.find(Order.class, orderId);  // PERSISTENT
    order.setStatus(OrderStatus.CONFIRMED);
    // No explicit save() needed — dirty checking detects change, flushes on commit

    Order sameOrder = entityManager.find(Order.class, orderId);
    // Returns SAME INSTANCE from first-level cache — no second SQL query!
    assertSame(order, sameOrder);  // true
}

// Detached entity problem
Order detachedOrder;
try {
    EntityManager em = emFactory.createEntityManager();
    em.getTransaction().begin();
    detachedOrder = em.find(Order.class, 1L);
    em.getTransaction().commit();
    em.close();  // session closed
}
// Now detachedOrder is DETACHED

detachedOrder.setStatus(OrderStatus.SHIPPED);
// Change NOT tracked — need to merge

@Transactional
public void reattach(Order detached) {
    Order managed = entityManager.merge(detached);  // copies state back to managed entity
    // managed.setStatus() changes are now tracked
}
```

---

## Question 10: What is the second-level cache in Hibernate? How do you configure it?

**First-level cache:** Per-session (transaction). Automatic, always enabled.

**Second-level cache:** Shared across sessions (SessionFactory level). Optional, requires configuration.

```java
// Enable second-level cache
spring.jpa.properties.hibernate.cache.use_second_level_cache=true
spring.jpa.properties.hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
spring.jpa.properties.javax.cache.provider=org.ehcache.jsr107.EhcacheCachingProvider

// Annotate entities that should be cached
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // or READ_ONLY, NONSTRICT_READ_WRITE
public class ProductCategory {
    @Id private Long id;
    private String name;
    // Reference data — rarely changes, frequently accessed
}

// Query cache — cache JPQL query results
@Query("SELECT c FROM ProductCategory c ORDER BY c.name")
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<ProductCategory> findAllCategories();
```

**When to use second-level cache:**
- Reference/lookup data: product categories, country codes, currencies
- Data that changes infrequently (< once/hour)
- Data that is read much more often than written

**When NOT to use:**
- Frequently changing data — cache eviction overhead negates benefit
- Entity types with complex ownership (concurrent updates from multiple services)
- If you have Redis/Ehcache at the application level, second-level cache is redundant

---

## Question 11: Explain Spring Security's authentication flow.

```
Request
   ↓
UsernamePasswordAuthenticationFilter (or JWT filter)
   ↓
AuthenticationManager (ProviderManager)
   ↓
AuthenticationProvider (DaoAuthenticationProvider)
   ↓
UserDetailsService.loadUserByUsername()
   ↓
UserDetails (with granted authorities)
   ↓
PasswordEncoder.matches()
   ↓
Authenticated token → SecurityContextHolder
   ↓
Authorization checks proceed
```

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // enables @PreAuthorize, @PostAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginProcessingUrl("/api/auth/login")
                .successHandler(jwtSuccessHandler())
                .failureHandler(authFailureHandler())
            )
            .sessionManagement(sm -> sm
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .addFilterBefore(jwtAuthenticationFilter(),
                UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

---

## Question 12: What is Spring's transaction synchronization? How does @Transactional interact with JPA?

**Transaction synchronization:** Spring's mechanism for binding resources (JDBC connections, Hibernate sessions) to the current thread's transaction.

```java
// When @Transactional method is called:
// 1. TransactionSynchronizationManager.bindResource() stores connection in ThreadLocal
// 2. All JPA/JDBC operations in the same thread use the SAME connection
// 3. On commit/rollback, all bound resources are released

@Transactional
public void processOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();  // connection A
    // Spring stores connection A in ThreadLocal

    inventoryService.reserve(order);  // uses SAME connection A
    paymentService.charge(order);     // uses SAME connection A
    // All participate in the SAME transaction

    // Commit → all changes committed atomically
}

// Register callback on transaction outcome
@Transactional
public void sendEmailAfterCommit(Order order) {
    orderRepository.save(order);

    // Register callback: only send email if transaction commits
    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                emailService.sendOrderConfirmation(order.getId());
            }
        }
    );
}
```

---

## Question 13: What is Spring Boot auto-configuration? How do you create a custom auto-configuration?

**Auto-configuration:** Spring Boot automatically configures beans based on what's on the classpath.

```
@SpringBootApplication
    → @EnableAutoConfiguration
        → imports spring.factories / META-INF/spring/AutoConfiguration.imports
            → DataSourceAutoConfiguration (if JDBC on classpath)
            → JpaRepositoriesAutoConfiguration (if JPA on classpath)
            → SecurityAutoConfiguration (if Spring Security on classpath)
            → Each @Conditional annotation controls if config applies
```

```java
// Custom auto-configuration for an internal library
@AutoConfiguration
@ConditionalOnClass(SlackClient.class)  // only if SlackClient is on classpath
@ConditionalOnMissingBean(SlackNotificationService.class)  // user can override
@EnableConfigurationProperties(SlackProperties.class)
public class SlackAutoConfiguration {

    @Bean
    @ConditionalOnProperty(name = "slack.enabled", havingValue = "true")
    public SlackNotificationService slackNotificationService(SlackProperties props) {
        return new SlackNotificationService(
            new SlackClient(props.getToken()),
            props.getDefaultChannel()
        );
    }
}

@ConfigurationProperties(prefix = "slack")
public class SlackProperties {
    private boolean enabled = false;
    private String token;
    private String defaultChannel = "#general";
    // getters, setters...
}

// Register in META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
// com.example.slack.SlackAutoConfiguration
```

---

## Question 14: What is Spring's @Cacheable and how does it work?

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        RedisCacheManager cm = RedisCacheManager.builder(redisConnectionFactory())
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10))
                .serializeValuesWith(
                    RedisSerializationContext.SerializationPair.fromSerializer(
                        new GenericJackson2JsonRedisSerializer()
                    )
                )
            )
            .withCacheConfiguration("products",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofHours(1)))  // longer TTL for stable data
            .build();
        return cm;
    }
}

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#productId",
               condition = "#productId > 0",       // only cache valid IDs
               unless = "#result == null")          // don't cache null results
    public Product getProduct(Long productId) {
        return productRepository.findById(productId)
            .orElseThrow(() -> new ResourceNotFoundException("Product", productId));
    }

    @CachePut(value = "products", key = "#product.id")  // update cache, always execute method
    public Product updateProduct(Product product) {
        return productRepository.save(product);
    }

    @CacheEvict(value = "products", key = "#productId")  // remove from cache
    public void deleteProduct(Long productId) {
        productRepository.deleteById(productId);
    }

    @CacheEvict(value = "products", allEntries = true)  // clear entire cache
    @Scheduled(fixedRate = 3600_000)
    public void evictAll() {}  // periodic cache clear
}
```

**Pitfall:** `@Cacheable` uses Spring AOP — self-invocation bypass applies. Calling `this.getProduct()` from within the same class doesn't go through the caching proxy.

---

## Question 15: How does Hibernate's dirty checking work?

**Dirty checking:** Hibernate tracks the original state of managed entities. At flush time, it compares current state to original state and generates UPDATE statements only for changed entities.

```java
@Transactional
public void updateOrderStatus(Long orderId, OrderStatus newStatus) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    // Hibernate: snapshot = {id: 1, status: PENDING, amount: 100}

    order.setStatus(newStatus);
    // No explicit save() needed!
    // At flush (before commit), Hibernate detects status changed
    // Generates: UPDATE orders SET status = ? WHERE id = ?

    // If nothing changes:
    // Hibernate: no UPDATE generated — optimized
}

// Bypassing dirty checking for bulk updates
@Modifying
@Query("UPDATE Order o SET o.status = :status WHERE o.customerId = :customerId")
int bulkUpdateStatus(@Param("customerId") Long customerId,
                      @Param("status") OrderStatus status);
// Direct SQL — bypasses first and second level cache!
// Clear cache after bulk updates:
entityManager.clear();  // or @Modifying(clearAutomatically = true)
```

**Performance implication:** With 10,000 managed entities in a session, dirty checking compares 10,000 snapshots at flush time. **Mitigation:** Use `StatelessSession` for bulk operations, projections for read-only queries, and `@QueryHints` to mark read-only queries.

---

## Question 16: What is Spring's event-driven programming model? How does it differ from application logic?

```java
// Domain event: something happened in the business
public class OrderShippedEvent extends ApplicationEvent {
    private final Long orderId;
    private final String trackingNumber;

    public OrderShippedEvent(Object source, Long orderId, String trackingNumber) {
        super(source);
        this.orderId = orderId;
        this.trackingNumber = trackingNumber;
    }
}

// Publisher: focused on core business logic
@Service
public class ShippingService {
    private final ApplicationEventPublisher events;

    @Transactional
    public Shipment shipOrder(Long orderId, String trackingNumber) {
        Shipment shipment = createShipment(orderId, trackingNumber);
        shipmentRepository.save(shipment);
        events.publishEvent(new OrderShippedEvent(this, orderId, trackingNumber));
        return shipment;
        // ShippingService doesn't know about emails, inventory, analytics
    }
}

// Listeners: independently handle the event
@Component
public class CustomerNotificationListener {
    @EventListener
    @Async
    public void onShipped(OrderShippedEvent event) {
        emailService.sendShippingNotification(event.getOrderId(), event.getTrackingNumber());
    }
}

@Component
public class AnalyticsListener {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onShipped(OrderShippedEvent event) {
        analytics.record("order_shipped", event.getOrderId());
    }
}
```

**Benefits:**
- **Loose coupling** — `ShippingService` doesn't depend on email, analytics, etc.
- **Easy extension** — add new listener without changing `ShippingService`
- **Testability** — test `ShippingService` without mocking all downstream services

---

## Question 17: What is Spring Profiles? How do you use them for environment-specific config?

```java
// Profile-specific beans
@Configuration
@Profile("production")
public class ProductionDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return buildHikariPool(productionDbUrl, 20);
    }
}

@Configuration
@Profile({"development", "test"})
public class LocalDataSourceConfig {
    @Bean
    public DataSource dataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }
}

// Profile-specific components
@Service
@Profile("!production")  // all profiles except production
public class StubEmailService implements EmailService {
    @Override
    public void sendEmail(String to, String subject, String body) {
        log.info("STUB: Would send email to {} with subject {}", to, subject);
    }
}
```

```yaml
# application.yml — shared
spring:
  datasource:
    url: ${DB_URL:jdbc:h2:mem:testdb}  # default to H2 if DB_URL not set

# application-production.yml — activated with spring.profiles.active=production
spring:
  datasource:
    url: ${DB_URL}  # required in production
  jpa:
    show-sql: false

# application-development.yml
spring:
  jpa:
    show-sql: true
    format-sql: true
  h2:
    console:
      enabled: true
```

**Activate profile:**
- Env var: `SPRING_PROFILES_ACTIVE=production`
- VM arg: `-Dspring.profiles.active=production`
- Programmatically: `SpringApplication.setAdditionalProfiles("production")`

---

## Question 18: What is optimistic vs pessimistic locking in JPA?

**Optimistic locking:** Assumes conflicts are rare. Check at commit time — if another transaction modified the record since you read it, throw `OptimisticLockException`.

```java
@Entity
public class Account {
    @Id private Long id;
    private BigDecimal balance;

    @Version  // version column — incremented on each update
    private int version;
}

// Transaction A reads account (version=1)
// Transaction B reads account (version=1)
// Transaction B updates (version becomes 2)
// Transaction A tries to update — version mismatch!
// → OptimisticLockException thrown, transaction A must retry

@Transactional
@Retryable(value = OptimisticLockException.class, maxAttempts = 3)
public void debit(Long accountId, BigDecimal amount) {
    Account account = accountRepository.findById(accountId).orElseThrow();
    account.setBalance(account.getBalance().subtract(amount));
    accountRepository.save(account);  // throws OptimisticLockException if concurrent update
}
```

**Pessimistic locking:** Lock the row immediately when reading. Other transactions wait.

```java
// Pessimistic read — other readers allowed, writers blocked
@Lock(LockModeType.PESSIMISTIC_READ)
Optional<Account> findWithReadLockById(Long id);

// Pessimistic write — all other access blocked
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Optional<Account> findWithWriteLockById(@Param("id") Long id);
// Generates: SELECT ... FOR UPDATE (PostgreSQL)
```

**When to use:**
- **Optimistic:** Low conflict probability, read-heavy, web applications with short transactions
- **Pessimistic:** High conflict probability, financial operations where you must serialize

---

## Question 19: What is Flyway and how does it integrate with Spring Boot?

**Database migration tools:** Manage schema changes as versioned, ordered scripts.

```
db/migration/
├── V1__create_orders_table.sql
├── V2__add_customer_index.sql
├── V3__add_order_status_column.sql
└── R__refresh_order_summary_view.sql  (repeatable migration)
```

```sql
-- V1__create_orders_table.sql
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    total_amount NUMERIC(19,2) NOT NULL,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- V3__add_order_notes.sql
ALTER TABLE orders ADD COLUMN notes TEXT;
```

```yaml
# Spring Boot auto-runs Flyway on startup
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true  # for existing databases
    out-of-order: false        # enforce strict ordering
    validate-on-migrate: true  # checksum validation
```

**Flyway stores migration history in `flyway_schema_history` table. If a migration script changes after running, Flyway detects the checksum mismatch and fails to start — protecting against accidental changes.**

```java
// Test migrations automatically
@SpringBootTest
@AutoConfigureTestDatabase(replace = NONE)  // use real Postgres via Testcontainers
class MigrationTest {
    @Autowired DataSource dataSource;

    @Test
    void migrationsApplySuccessfully() {
        // Spring Boot auto-runs Flyway during test startup — test passes if no exception
    }

    @Test
    void ordersTableHasRequiredColumns() throws SQLException {
        try (Connection conn = dataSource.getConnection();
             ResultSet rs = conn.getMetaData().getColumns(null, null, "orders", null)) {
            Set<String> columns = new HashSet<>();
            while (rs.next()) columns.add(rs.getString("COLUMN_NAME"));
            assertThat(columns).contains("id", "customer_id", "status", "created_at");
        }
    }
}
```

---

## Question 20: What is Spring's ConditionalOn* family of annotations?

```java
// @ConditionalOnClass — only create bean if class is on classpath
@Bean
@ConditionalOnClass(name = "io.lettuce.core.RedisClient")
public CacheManager redisCacheManager() { ... }

// @ConditionalOnMissingBean — create only if no other bean of this type exists
@Bean
@ConditionalOnMissingBean(EmailService.class)
public EmailService defaultEmailService() {
    return new SmtpEmailService();  // user can override by defining their own EmailService
}

// @ConditionalOnProperty — create based on config property
@Bean
@ConditionalOnProperty(name = "feature.new-checkout.enabled", havingValue = "true")
public CheckoutService newCheckoutService() { ... }

// @ConditionalOnExpression — SpEL expression
@Bean
@ConditionalOnExpression("${app.payment.providers.stripe.enabled:false} "
    + "and '${app.environment}' == 'production'")
public StripePaymentProcessor stripeProcessor() { ... }

// @ConditionalOnWebApplication — only in web context
@Configuration
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
public class WebMvcSecurityConfig { ... }

// @Profile is syntactic sugar for @Conditional:
@Profile("production") // equivalent to @ConditionalOnProperty... with profile
```

**Use in auto-configuration to allow graceful degradation:**
```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
public class ConnectionPoolMetricsAutoConfiguration {
    @Bean
    @ConditionalOnBean(DataSource.class)
    @ConditionalOnMissingBean(HikariPoolMXBean.class)
    public PoolMetricsExporter poolMetricsExporter(DataSource ds) {
        return new PoolMetricsExporter(ds);
    }
}
```

