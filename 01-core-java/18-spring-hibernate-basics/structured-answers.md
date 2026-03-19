# Spring & Hibernate Basics — Structured Answers

---

## Q1: What Is the Spring IoC Container and How Does It Manage Beans?

The IoC (Inversion of Control) container is the core of the Spring Framework. Instead of your code creating and managing dependencies (`new EmailService()`), you declare what you need and the container creates, configures, wires, and manages the lifecycle of all objects (beans).

```java
// Without IoC — tight coupling, hard to test
public class OrderService {
    private EmailService emailService = new EmailService();  // hardcoded
    private PaymentService paymentService = new PaymentService(new StripeClient());

    // To test, you can't swap implementations without changing the class
}

// With IoC — loose coupling, testable
@Service
public class OrderService {
    private final EmailService emailService;
    private final PaymentService paymentService;

    // Spring provides these via constructor injection
    public OrderService(EmailService emailService, PaymentService paymentService) {
        this.emailService   = emailService;
        this.paymentService = paymentService;
    }
}
```

**How the container manages beans:**
```java
// Two main ApplicationContext implementations:
// - AnnotationConfigApplicationContext (for Java config / component scanning)
// - ClassPathXmlApplicationContext (legacy XML config)

// Spring Boot auto-creates the context:
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(MyApp.class, args);

        // Manually retrieve a bean (usually you don't need to do this)
        OrderService svc = ctx.getBean(OrderService.class);
    }
}

// Beans are registered via:
@Component, @Service, @Repository, @Controller  // component scan
@Bean in @Configuration class                    // explicit factory method
@Import                                          // import another configuration
```

**Bean scopes:**
```java
@Component
@Scope("singleton")   // default — one instance per container
public class UserService { ... }

@Component
@Scope("prototype")   // new instance per injection / getBean() call
public class ShoppingCart { ... }

@Component
@Scope("request")     // new instance per HTTP request (web apps)
public class RequestContext { ... }

@Component
@Scope("session")     // new instance per HTTP session
public class UserSession { ... }
```

---

## Q2: What Is the Difference Between Constructor Injection and Field Injection? Which Is Better?

**Field injection:**
```java
@Service
public class OrderService {
    @Autowired
    private OrderRepository repository;  // injected by Spring reflection

    @Autowired
    private EmailService emailService;
}
```

Problems:
- Cannot declare `final` — object is mutable
- Hard to unit test without Spring context
- Hidden dependencies — no way to see them from the constructor
- Circular dependency problems detected late (at runtime)
- Spring can inject `null` if a bean is missing (misconfig) — NullPointerException at runtime

**Constructor injection (recommended):**
```java
@Service
public class OrderService {
    private final OrderRepository repository;    // immutable — safe to share
    private final EmailService emailService;

    // Spring auto-detects single constructors (no @Autowired needed in Spring 4.3+)
    public OrderService(OrderRepository repository, EmailService emailService) {
        this.repository   = Objects.requireNonNull(repository, "repository must not be null");
        this.emailService = Objects.requireNonNull(emailService, "emailService must not be null");
    }
}
```

Benefits:
- **Immutability** — final fields cannot be accidentally modified after construction
- **Testability** — instantiate with mock dependencies in unit tests, no Spring context needed
- **Fail-fast** — if a required bean is missing, ApplicationContext fails to start (not a NPE at runtime)
- **Visible dependencies** — constructor signature shows every dependency explicitly
- **No circular dependency hiding** — Spring fails at startup for circular constructor dependencies, forcing you to fix the design

```java
// Testing OrderService without Spring:
@Test
void placeOrderSendsEmail() {
    // Can instantiate directly — no Spring context needed
    EmailService mockEmail = mock(EmailService.class);
    OrderRepository mockRepo = mock(OrderRepository.class);
    OrderService service = new OrderService(mockRepo, mockEmail);

    service.placeOrder(new OrderRequest(...));

    verify(mockEmail).send(any());
}
```

**Lombok makes constructor injection effortless:**
```java
@Service
@RequiredArgsConstructor  // generates constructor for all final fields
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;
    private final PaymentService paymentService;
}
```

---

## Q3: How Does Spring AOP Work Under the Hood? What Is a Proxy?

AOP (Aspect-Oriented Programming) allows you to add behavior (logging, transactions, security) to existing code without modifying that code. Spring AOP uses **proxies** to achieve this.

**How proxy creation works:**

At application startup, Spring's `BeanPostProcessor` (specifically `AnnotationAwareAspectJAutoProxyCreator`) inspects every bean. If a bean matches any advice (e.g., has `@Transactional`, or matches a pointcut expression), Spring wraps it in a proxy.

```java
// You write:
@Service
public class OrderService {
    @Transactional
    public Order placeOrder(OrderRequest req) {
        return orderRepository.save(new Order(req));
    }
}

// Spring creates at runtime (conceptually):
public class OrderServiceProxy extends OrderService {
    private final TransactionManager txManager;
    private final OrderService realOrderService;

    @Override
    public Order placeOrder(OrderRequest req) {
        TransactionStatus tx = txManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Order result = realOrderService.placeOrder(req);  // call real method
            txManager.commit(tx);
            return result;
        } catch (RuntimeException e) {
            txManager.rollback(tx);
            throw e;
        }
    }
}

// Spring injects this proxy everywhere, not the real OrderService
```

**JDK Dynamic Proxy vs CGLIB:**
```java
// JDK proxy — requires an interface
public interface IOrderService { Order placeOrder(OrderRequest req); }

@Service
public class OrderService implements IOrderService { ... }
// Spring creates: Proxy.newProxyInstance(classLoader, new Class[]{IOrderService.class}, handler)

// CGLIB proxy — subclasses the concrete class (no interface needed)
@Service
public class ProductService { /* no interface */
    @Transactional
    public void save(Product p) { ... }
}
// Spring CGLIB subclasses ProductService and overrides save() to add TX
// Limitation: cannot proxy final classes or final methods
```

**Custom AOP aspect:**
```java
@Aspect
@Component
public class ExecutionTimeAspect {

    // Pointcut: all methods in any @Service class
    @Around("@within(org.springframework.stereotype.Service)")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return pjp.proceed();  // call the real method
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            log.info("{}.{}() took {}ms",
                pjp.getSignature().getDeclaringTypeName(),
                pjp.getSignature().getName(),
                elapsed);
        }
    }
}
```

---

## Q4: Explain Hibernate Entity States With a Real-World Example

```java
@Entity
@Table(name = "products")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private BigDecimal price;
    // getters/setters
}

@Service
public class ProductService {

    @Autowired
    private EntityManager em;

    public void demonstrateStates() {
        // --- TRANSIENT ---
        Product p = new Product();
        p.setName("Laptop");
        p.setPrice(new BigDecimal("999.99"));
        // p has no id, not tracked by Hibernate, not in DB
        System.out.println("State: TRANSIENT — id=" + p.getId()); // null

        // --- TRANSIENT -> PERSISTENT ---
        em.persist(p);  // Hibernate assigns ID, starts tracking
        // SQL: INSERT INTO products (name, price) VALUES ('Laptop', 999.99)
        System.out.println("State: PERSISTENT — id=" + p.getId()); // 1

        // While PERSISTENT: Hibernate dirty-checks this object
        p.setPrice(new BigDecimal("899.99"));
        // No SQL yet, but Hibernate notes the change

        em.flush(); // or commit — Hibernate issues:
        // SQL: UPDATE products SET price=899.99 WHERE id=1

        // --- PERSISTENT -> DETACHED ---
        em.detach(p);  // or em.close(), or transaction ends
        p.setName("Gaming Laptop");
        // Change is NOT tracked — no SQL will be generated for this change

        // --- DETACHED -> PERSISTENT (via merge) ---
        Product managed = em.merge(p);
        // Hibernate copies state from detached 'p' into a managed copy
        // SQL: SELECT * FROM products WHERE id=1 (to check if exists)
        // Then: UPDATE products SET name='Gaming Laptop' WHERE id=1

        // --- PERSISTENT -> REMOVED ---
        em.remove(managed);
        // SQL: DELETE FROM products WHERE id=1 (on flush)

        // After transaction commit: entity is effectively gone from DB
    }
}
```

**Common real-world scenario with Spring Data:**
```java
@Transactional
public void updateProductPrice(Long id, BigDecimal newPrice) {
    Product p = productRepository.findById(id).orElseThrow();
    // p is PERSISTENT (tracked by current Hibernate Session)

    p.setPrice(newPrice);
    // No explicit save() call needed!
    // Hibernate dirty checks at transaction commit and issues UPDATE automatically
}
// @Transactional ends -> Session flushes -> UPDATE SQL -> commit -> Session closes
// p is now DETACHED
```

---

## Q5: What Is the First-Level Cache in Hibernate and How Does It Work?

The first-level cache (also called the Session cache or persistence context) is a mandatory, session-scoped cache built into every Hibernate Session. It stores every entity you load or save within a transaction, keyed by `(EntityClass, primaryKey)`.

```java
@Service
public class UserService {

    @Transactional
    public void demonstrateFirstLevelCache() {
        // Load user — SQL issued, entity stored in cache
        User user1 = userRepository.findById(1L).orElseThrow();
        // SQL: SELECT * FROM users WHERE id = 1

        // Load same user again — NO SQL! Returned from cache
        User user2 = userRepository.findById(1L).orElseThrow();
        // No SQL issued

        System.out.println(user1 == user2);  // true — exact same Java object

        // Modifying user1 also affects user2 (same object reference!)
        user1.setName("Alice Updated");
        System.out.println(user2.getName());  // "Alice Updated"
    }

    // Cross-transaction — each @Transactional method gets a new Session
    @Transactional
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();
        // SQL issued — new Session, empty cache
    }
}
```

**Important: N+1 query problem (first-level cache doesn't prevent this):**
```java
@Transactional
public void processOrders() {
    List<Order> orders = orderRepository.findAll();   // 1 SQL for orders
    for (Order order : orders) {
        // LAZY LOADING — triggers 1 SQL per order for items!
        order.getItems().forEach(item -> process(item));
    }
    // Total: 1 + N queries where N = number of orders
}

// Fix: JOIN FETCH
@Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.items")
List<Order> findAllWithItems();  // 1 SQL with JOIN
```

---

## Q6: What Is Dirty Checking in Hibernate?

Dirty checking is Hibernate's mechanism to automatically detect which managed entities have changed since they were loaded, and to generate SQL `UPDATE` statements only for the changed entities and fields.

```java
@Transactional
public void updateUser(Long id, String newEmail, String newName) {
    User user = userRepository.findById(id).orElseThrow();
    // Hibernate takes a "snapshot": {name="Alice", email="alice@old.com", age=30}

    user.setEmail(newEmail);  // snapshot: email changed
    user.setName(newName);    // snapshot: name changed
    // user.setAge(30); — age unchanged

    // No explicit save() call needed
    // At flush/commit, Hibernate generates:
    // UPDATE users SET name=?, email=? WHERE id=?
    // (Only changed fields — age is not in the UPDATE)
}
```

**How dirty checking works internally:**
1. On `findById()`, Hibernate deep-copies the entity's field values into a `snapshot` array.
2. At flush time, Hibernate compares the current field values against the snapshot.
3. If any field differs, it's "dirty" and a SQL `UPDATE` is generated.
4. Hibernate uses the full entity's column set in the UPDATE statement by default (or only dirty columns with `@DynamicUpdate`).

```java
// @DynamicUpdate — generate UPDATE SQL with only changed columns
@Entity
@DynamicUpdate
public class Product {
    @Id private Long id;
    private String name;
    private BigDecimal price;
    private int stockQuantity;
}

// Without @DynamicUpdate:
// UPDATE products SET name=?, price=?, stock_quantity=? WHERE id=?  (all columns)

// With @DynamicUpdate (only price changed):
// UPDATE products SET price=? WHERE id=?  (only changed column)
// Reduces lock contention in concurrent scenarios
```

---

## Q7: What Is the Difference Between JPA save(), persist(), merge(), and saveOrUpdate()?

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> { }

// Spring Data JPA: save() — smart method
@Service
public class OrderService {

    public void createNewOrder(OrderRequest req) {
        Order newOrder = new Order(req);
        // newOrder.id is null (new entity)

        Order saved = orderRepository.save(newOrder);
        // Internally: entityManager.persist(newOrder) because isNew() == true
        // After persist: newOrder.id is assigned
        // SQL: INSERT INTO orders (...)
    }

    public void updateExistingOrder(Long id, OrderStatus status) {
        Order existing = orderRepository.findById(id).orElseThrow();
        // existing is already PERSISTENT — changes auto-tracked
        // No explicit save needed if still in same @Transactional method

        existing.setStatus(status);
        // Hibernate dirty checking handles the UPDATE at commit

        // But if you detach and re-attach:
        orderRepository.save(existing);  // calls entityManager.merge() because id != null
        // SQL: UPDATE orders SET status=? WHERE id=?
    }

    public void upsertOrder(Order order) {
        orderRepository.save(order);
        // If order.id == null: INSERT (persist)
        // If order.id != null: merge (copy state into managed instance -> UPDATE)
    }
}

// Pure JPA EntityManager API:
public void jpaApiExamples() {
    // persist — only for new entities, must be in transaction
    em.persist(new Order());                    // INSERT on flush

    // merge — works for new and detached, returns managed copy
    Order managed = em.merge(detachedOrder);    // INSERT or UPDATE
    // IMPORTANT: use 'managed', not 'detachedOrder'

    // detach
    em.detach(managedOrder);                    // stops tracking

    // remove — must be called on managed entity
    Order toDelete = em.find(Order.class, id);
    em.remove(toDelete);                        // DELETE on flush
}
```

**Legacy Hibernate Session API (pre-JPA):**
- `session.save()` → inserts, returns generated id
- `session.update()` → updates detached entity (throws if entity with same id already in session)
- `session.saveOrUpdate()` → insert if new, update if existing
- `session.merge()` → JPA-equivalent merge

---

## Q8: How Does Spring @Transactional Propagation REQUIRED vs REQUIRES_NEW Work?

```java
@Service
public class AuditService {

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void log(String action, Long userId) {
        auditLogRepository.save(new AuditLog(action, userId, Instant.now()));
        // Always runs in its own transaction
        // Commits independently of the calling transaction
    }
}

@Service
public class OrderService {
    @Autowired private AuditService auditService;
    @Autowired private OrderRepository orderRepository;

    @Transactional  // REQUIRED — default
    public Order placeOrder(OrderRequest req) {
        Order order = orderRepository.save(new Order(req));

        auditService.log("ORDER_PLACED", req.getUserId());
        // REQUIRES_NEW: auditService.log runs in TX2
        // TX1 is suspended during this call
        // TX2 commits when log() returns
        // TX1 resumes

        if (req.getAmount().compareTo(MAX_AMOUNT) > 0) {
            throw new RuntimeException("Amount exceeds limit");
        }
        return order;
    }
}

// What happens when placeOrder throws after auditService.log():
// TX2 (audit log) is already committed — audit record IS saved
// TX1 (order insert) is rolled back — order is NOT saved
// This is the correct behavior for audit logs
```

**Other propagation types:**

```java
MANDATORY   — must run within existing TX; throws if no TX
SUPPORTS    — joins if TX exists, non-transactional if not
NOT_SUPPORTED — runs non-transactionally, suspends existing TX
NEVER       — must NOT have a TX; throws if TX exists
NESTED      — creates a savepoint within existing TX (partial rollback)
```

```java
// NESTED: rollback to savepoint without rolling back entire TX
@Transactional(propagation = Propagation.REQUIRED)
public void processOrder(Order order) {
    mainRepository.save(order);

    try {
        optionalEnrichmentService.enrich(order);  // NESTED
    } catch (Exception e) {
        // Rolls back ONLY the nested enrichment, not the main order save
        log.warn("Enrichment failed, continuing without it");
    }
    // Main order is still saved
}
```

---

## Q9: What Is the Open-Session-in-View Anti-Pattern?

Open-Session-in-View (OSIV) keeps the Hibernate Session (and therefore a database connection) open for the **entire duration of an HTTP request**, including the view rendering phase.

**Spring Boot enables OSIV by default:**
```yaml
# application.properties
spring.jpa.open-in-view=true  # DEFAULT — produces a warning in newer Spring Boot
```

**What it enables:**
```java
@Controller
public class OrderController {
    @GetMapping("/orders/{id}")
    public String getOrder(@PathVariable Long id, Model model) {
        Order order = orderService.getOrder(id);
        // @Transactional in service already ended
        // But OSIV keeps Session open for the view!
        model.addAttribute("order", order);
        return "order-detail";
    }
}
// In the Thymeleaf template:
// th:each="item : ${order.items}"  <- lazy loading triggered HERE, outside service!
// This only works because OSIV kept the Session open
```

**Why it's an anti-pattern:**
1. **Holds DB connections during view rendering** — view rendering (template processing, JSON serialization) can be slow. During this time, a DB connection is held, reducing pool availability.
2. **Hides N+1 problems** — lazy loading in the view layer silently issues SQL, which is invisible and uncontrolled.
3. **Leaks persistence concern into view layer** — the view shouldn't trigger DB queries.
4. **Makes applications non-cloud-friendly** — in microservices with many requests, holding connections longer can exhaust the connection pool.

**Fix: disable OSIV and always fetch eagerly within the @Transactional boundary:**
```yaml
spring.jpa.open-in-view=false
```

```java
@Transactional(readOnly = true)
public OrderDTO getOrder(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    // Explicitly load what you need INSIDE the transaction
    return new OrderDTO(
        order.getId(),
        order.getStatus(),
        order.getItems().stream().map(ItemDTO::new).toList()  // lazy loaded here — OK
    );
    // Transaction ends; Session closes; DTO returned to controller — no Session needed
}
```

---

## Q10: How Do You Configure HikariCP Connection Pool in Spring Boot?

HikariCP is the default connection pool in Spring Boot 2+. It's the fastest Java connection pool available.

```yaml
# application.yml — comprehensive HikariCP configuration
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      # Pool sizing
      maximum-pool-size: 10          # Max connections in pool
      minimum-idle: 5                # Min idle connections maintained
      connection-timeout: 30000      # Wait time for a connection (ms) — 30s
      idle-timeout: 600000           # Remove idle connections after 10 min
      max-lifetime: 1800000          # Replace connections after 30 min (< DB timeout!)
      keepalive-time: 300000         # Send keepalive every 5 min

      # Validation
      connection-test-query: SELECT 1  # For JDBC4-compliant drivers (optional)
      validation-timeout: 5000         # Timeout for connection validation

      # Naming for monitoring
      pool-name: HikariCP-OrdersDB

      # PostgreSQL-specific
      data-source-properties:
        cachePrepStmts: true
        prepStmtCacheSize: 250
        prepStmtCacheSqlLimit: 2048
```

**How to size the connection pool:**
```
Formula (general guideline):
  pool_size = (num_cores * 2) + effective_spindle_count

For a service with:
  - 4 CPU cores, SSD storage (1 effective spindle):
  - pool_size = (4 * 2) + 1 = 9 ≈ 10

Too many connections is a common mistake:
  - Each DB connection consumes memory on the DB server
  - PostgreSQL: ~5-10MB per connection
  - 100 connections = 500MB-1GB used just for connections
  - DB servers handle concurrency internally; more connections don't mean more throughput
```

**Monitoring HikariCP:**
```java
@Bean
public HikariDataSource dataSource() {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
    config.setUsername("user");
    config.setPassword("pass");
    config.setMaximumPoolSize(10);
    config.setMetricRegistry(meterRegistry);  // Micrometer integration
    return new HikariDataSource(config);
}

// Metrics exposed via /actuator/metrics:
// hikaricp.connections.active
// hikaricp.connections.idle
// hikaricp.connections.pending  <- non-zero = pool exhaustion
// hikaricp.connections.timeout.total <- count connection timeouts
```

**Connection pool exhaustion — how to detect and fix:**
```java
// Symptom: HikariPool-1 - Connection is not available, request timed out after 30000ms

// Common causes:
// 1. Long-running @Transactional methods holding connections
// 2. Uncommitted transactions (missing @Transactional, manual connection not closed)
// 3. OSIV keeping connections open during view rendering
// 4. Pool size too small for the load

// Detection: monitor hikaricp.connections.pending metric
// If consistently > 0: pool is exhausted

// Fix:
// - Increase maximum-pool-size (up to DB limit)
// - Reduce connection hold time (shorter transactions)
// - Disable OSIV
// - Use @Transactional(readOnly=true) for reads (some optimizations)
// - Ensure all connections are released (try-with-resources for manual JDBC)
```
