# Spring & Hibernate Basics — Scenario Questions

---

## Scenario 1: Fix LazyInitializationException in a REST API

**Setup:** A REST endpoint returns a `CustomerDto` that includes a list of orders. In production, you're getting `LazyInitializationException: could not initialize proxy - no Session`.

```java
// PROBLEM CODE
@GetMapping("/customers/{id}")
public CustomerDto getCustomer(@PathVariable Long id) {
    Customer customer = customerRepository.findById(id).orElseThrow();
    // Transaction ends when findById() returns (Spring Data default: @Transactional on repo)
    return mapper.toDto(customer);  // mapper accesses customer.getOrders() → EXCEPTION
}

// Root cause: the JPA session closes after findById(). Mapping accesses lazy collection after that.

// SOLUTION 1: JOIN FETCH in repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    @Query("SELECT c FROM Customer c LEFT JOIN FETCH c.orders WHERE c.id = :id")
    Optional<Customer> findWithOrdersById(@Param("id") Long id);
}

@GetMapping("/customers/{id}")
public CustomerDto getCustomer(@PathVariable Long id) {
    Customer customer = customerRepository.findWithOrdersById(id).orElseThrow();
    return mapper.toDto(customer);  // orders already loaded
}

// SOLUTION 2: @EntityGraph
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    @EntityGraph(attributePaths = {"orders", "orders.items"})
    Optional<Customer> findById(Long id);
}

// SOLUTION 3: DTO projection — avoid loading entities when you only need read-only data
@Query("SELECT new com.example.CustomerDto(c.id, c.name, c.email, " +
       "COUNT(o), SUM(o.totalAmount)) " +
       "FROM Customer c LEFT JOIN c.orders o " +
       "WHERE c.id = :id GROUP BY c.id, c.name, c.email")
Optional<CustomerDto> findCustomerSummary(@Param("id") Long id);

// SOLUTION 4: @Transactional on the service method (session stays open through mapping)
@Service
public class CustomerService {
    @Transactional(readOnly = true)  // session stays open, lazy loading works
    public CustomerDto getCustomerWithOrders(Long id) {
        Customer customer = customerRepository.findById(id).orElseThrow();
        return mapper.toDto(customer);  // lazy loading works — session still open
    }
}
```

**Best practice:** Use DTO projections for read-only queries — never load entities just to map them.

---

## Scenario 2: Solving N+1 Query Problem in an Order Summary Report

**Setup:** A report endpoint is taking 30 seconds for 500 orders. Hibernate stats show 1501 queries.

```java
// PROBLEM: 1 + 500 (customers) + 500 (items) + 500 (shipping) = 1501 queries
@GetMapping("/reports/orders")
public List<OrderReportRow> getOrderReport() {
    List<Order> orders = orderRepository.findAll();  // 1 query
    return orders.stream()
        .map(o -> new OrderReportRow(
            o.getId(),
            o.getCustomer().getName(),    // N queries for customer
            o.getItems().size(),           // N queries for items
            o.getShipping().getAddress()  // N queries for shipping
        ))
        .toList();
}

// SOLUTION 1: JOIN FETCH (watch out for Cartesian product with multiple collections)
@Query("SELECT DISTINCT o FROM Order o " +
       "LEFT JOIN FETCH o.customer " +
       "LEFT JOIN FETCH o.shipping " +
       "WHERE o.createdAt > :since")
List<Order> findWithDetailsForReport(@Param("since") LocalDateTime since);

// Multiple collections → use @BatchSize to avoid Cartesian product
@Entity
public class Order {
    @ManyToOne(fetch = FetchType.EAGER)  // single JOIN for customer
    private Customer customer;

    @OneToMany
    @BatchSize(size = 100)  // loads items for 100 orders at once: IN (id1...id100)
    private List<OrderItem> items;
}

// SOLUTION 2 (BEST): DTO projection with a single query — no entity loading at all
@Query("""
    SELECT new com.example.OrderReportRow(
        o.id,
        c.name,
        COUNT(i),
        SUM(i.price * i.quantity),
        s.address
    )
    FROM Order o
    JOIN o.customer c
    LEFT JOIN o.items i
    LEFT JOIN o.shipping s
    WHERE o.createdAt > :since
    GROUP BY o.id, c.name, s.address
    """)
List<OrderReportRow> findReportRows(@Param("since") LocalDateTime since);
// 1 query, no entity loading, no session needed for serialization
```

---

## Scenario 3: @Transactional Self-Invocation Bug

**Setup:** A method decorated with `@Transactional` is calling another `@Transactional` method on `this`. The inner transaction is not being created.

```java
@Service
public class OrderService {

    @Transactional
    public void processOrder(Long orderId) {
        Order order = findById(orderId);
        // Calling this.sendConfirmation() — goes through THIS reference, NOT the proxy!
        sendConfirmation(order);  // @Transactional on this method is IGNORED
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendConfirmation(Order order) {
        // This expects a NEW transaction, but runs in processOrder's transaction
        // because self-invocation bypasses the AOP proxy
        confirmationRepository.save(new Confirmation(order));
    }
}

// FIXES:

// Fix 1: Extract to a separate Spring bean
@Service
public class ConfirmationService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendConfirmation(Order order) {
        confirmationRepository.save(new Confirmation(order));
    }
}

// Fix 2: Inject self-reference (hacky, avoid if possible)
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // injected proxy

    public void processOrder(Long orderId) {
        self.sendConfirmation(order);  // goes through proxy → AOP applies
    }
}

// Fix 3: Use AopContext.currentProxy() (requires @EnableAspectJAutoProxy(exposeProxy=true))
public void processOrder(Long orderId) {
    ((OrderService) AopContext.currentProxy()).sendConfirmation(order);
}
```

---

## Scenario 4: Optimistic Locking for Concurrent Order Updates

**Setup:** Multiple threads can update the same order simultaneously (e.g., two customer service agents updating order status). Implement safe concurrent updates.

```java
@Entity
public class Order {
    @Id @GeneratedValue private Long id;
    private OrderStatus status;

    @Version
    private int version;  // Hibernate manages this
}

// Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Spring Data automatically uses @Version for optimistic locking
}

// Service with retry
@Service
public class OrderService {

    @Retryable(
        value = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public Order updateStatus(Long orderId, OrderStatus newStatus, String updatedBy) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new ResourceNotFoundException("Order", orderId));

        if (!order.getAllowedTransitions().contains(newStatus)) {
            throw new InvalidStateTransitionException(order.getStatus(), newStatus);
        }

        order.setStatus(newStatus);
        order.setLastModifiedBy(updatedBy);
        // If another transaction modified this order since we read it:
        // Hibernate detects version mismatch → throws StaleObjectStateException
        // Spring translates to OptimisticLockingFailureException
        return orderRepository.save(order);
    }

    @Recover
    public Order handleOptimisticLockFailure(OptimisticLockingFailureException ex,
                                              Long orderId, OrderStatus status, String user) {
        log.error("Failed to update order {} after 3 attempts due to concurrent modification",
            orderId);
        throw new ConcurrentModificationException(
            "Order " + orderId + " was modified concurrently. Please retry.");
    }
}
```

---

## Scenario 5: Dynamic Multi-Tenant DataSource Routing

**Setup:** A SaaS application serves multiple tenants. Each tenant has their own database schema. Route queries to the correct schema based on the current tenant.

```java
// Thread-local tenant context
public class TenantContext {
    private static final ThreadLocal<String> currentTenant = new ThreadLocal<>();

    public static void setTenant(String tenantId) { currentTenant.set(tenantId); }
    public static String getTenant() { return currentTenant.get(); }
    public static void clear() { currentTenant.remove(); }
}

// Routing data source
@Component
public class TenantRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        String tenant = TenantContext.getTenant();
        if (tenant == null) throw new IllegalStateException("No tenant in context");
        return tenant;
    }
}

// Configuration
@Configuration
public class MultiTenantDataSourceConfig {

    @Bean
    @Primary
    public DataSource dataSource(TenantRepository tenantRepo) {
        TenantRoutingDataSource router = new TenantRoutingDataSource();

        // Load all tenant data sources
        Map<Object, Object> dataSources = new HashMap<>();
        tenantRepo.findAll().forEach(tenant -> {
            dataSources.put(tenant.getId(), buildDataSource(tenant));
        });

        router.setTargetDataSources(dataSources);
        router.setDefaultTargetDataSource(dataSources.values().iterator().next());
        return router;
    }
}

// Filter to set tenant from JWT
@Component
public class TenantFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, ...) {
        String tenantId = extractTenantFromJwt(req);
        TenantContext.setTenant(tenantId);
        try {
            chain.doFilter(req, response);
        } finally {
            TenantContext.clear();  // CRITICAL: prevent leak to next request
        }
    }
}
```

---

## Scenario 6: Custom Spring Boot Auto-Configuration for Internal Library

**Setup:** Your team creates an internal audit-logging library. Clients should get it auto-configured with minimal setup.

```java
// Library: audit-logging-spring-boot-starter

// Properties
@ConfigurationProperties(prefix = "audit")
@Validated
public class AuditProperties {
    @NotBlank
    private String applicationName;
    private boolean enabled = true;
    private String outputFormat = "JSON";
    private Duration retentionPeriod = Duration.ofDays(90);
    // getters, setters...
}

// Auto-configuration
@AutoConfiguration
@ConditionalOnClass(AuditLogger.class)
@ConditionalOnProperty(name = "audit.enabled", havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public AuditLogger auditLogger(AuditProperties props) {
        return new AuditLogger(props.getApplicationName(), props.getOutputFormat());
    }

    @Bean
    @ConditionalOnBean(AuditLogger.class)
    @ConditionalOnMissingBean(AuditAspect.class)
    public AuditAspect auditAspect(AuditLogger logger) {
        return new AuditAspect(logger);
    }
}

// src/main/resources/META-INF/spring/
// org.springframework.boot.autoconfigure.AutoConfiguration.imports:
// com.example.audit.AuditAutoConfiguration
```

**Client configuration (just add dependency):**
```yaml
# application.yml
audit:
  application-name: order-service
  output-format: JSON
```

---

## Scenario 7: Bulk Operations with Hibernate

**Setup:** Import 100,000 records from a CSV file into the database. Naive approach causes OOM or takes too long.

```java
@Service
@Transactional
public class BulkImportService {

    @PersistenceContext
    private EntityManager em;

    // NAIVE approach — OOM for 100K records (all in first-level cache)
    public void importNaive(List<ProductCsv> records) {
        records.forEach(csv -> em.persist(mapper.toEntity(csv)));
    }

    // CORRECT: batch + periodic flush + clear
    public void importBatch(List<ProductCsv> records) {
        int batchSize = 500;
        for (int i = 0; i < records.size(); i++) {
            Product product = mapper.toEntity(records.get(i));
            em.persist(product);

            if ((i + 1) % batchSize == 0) {
                em.flush();   // write to DB
                em.clear();   // evict from first-level cache — prevent OOM
            }
        }
        em.flush();  // flush remainder
    }

    // EVEN BETTER: StatelessSession — no first-level cache, no dirty checking
    public void importStateless(List<ProductCsv> records) {
        SessionFactory sf = em.getEntityManagerFactory().unwrap(SessionFactory.class);
        try (StatelessSession session = sf.openStatelessSession()) {
            Transaction tx = session.beginTransaction();
            try {
                int batchSize = 1000;
                for (int i = 0; i < records.size(); i++) {
                    Product product = mapper.toEntity(records.get(i));
                    session.insert(product);  // no first-level cache overhead
                    if ((i + 1) % batchSize == 0) {
                        session.getTransaction().commit();
                        session.beginTransaction();
                    }
                }
                tx.commit();
            } catch (Exception e) {
                tx.rollback();
                throw e;
            }
        }
    }
}
```

**Hibernate batch insert configuration:**
```yaml
spring.jpa.properties.hibernate.jdbc.batch_size: 500
spring.jpa.properties.hibernate.order_inserts: true
spring.jpa.properties.hibernate.order_updates: true
spring.jpa.properties.hibernate.batch_versioned_data: true
```

---

## Scenario 8: Testing Spring Services with Mockito

**Setup:** Write a comprehensive unit test for `OrderService.placeOrder()` that mocks the repository, payment service, and event publisher.

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock private OrderRepository orderRepository;
    @Mock private PaymentService paymentService;
    @Mock private ApplicationEventPublisher eventPublisher;
    @Mock private InventoryService inventoryService;

    @InjectMocks
    private OrderService orderService;

    @Captor
    private ArgumentCaptor<Order> orderCaptor;

    @Captor
    private ArgumentCaptor<ApplicationEvent> eventCaptor;

    @Test
    void placeOrder_shouldCreateOrderAndPublishEvent() {
        // Arrange
        CreateOrderRequest request = CreateOrderRequest.builder()
            .customerId(42L)
            .item("LAPTOP-001", 1, new BigDecimal("1500.00"))
            .build();

        when(inventoryService.checkAvailability(any())).thenReturn(true);
        when(paymentService.authorize(any())).thenReturn(PaymentAuth.success("AUTH-123"));
        when(orderRepository.save(any(Order.class))).thenAnswer(invocation -> {
            Order order = invocation.getArgument(0);
            order.setId(1L);  // simulate DB-generated ID
            return order;
        });

        // Act
        Order result = orderService.placeOrder(request);

        // Assert
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getStatus()).isEqualTo(OrderStatus.CONFIRMED);

        // Verify order persisted with correct data
        verify(orderRepository).save(orderCaptor.capture());
        Order savedOrder = orderCaptor.getValue();
        assertThat(savedOrder.getCustomerId()).isEqualTo(42L);
        assertThat(savedOrder.getTotalAmount()).isEqualByComparingTo("1500.00");

        // Verify event published
        verify(eventPublisher).publishEvent(eventCaptor.capture());
        assertThat(eventCaptor.getValue()).isInstanceOf(OrderPlacedEvent.class);
        OrderPlacedEvent event = (OrderPlacedEvent) eventCaptor.getValue();
        assertThat(event.getOrderId()).isEqualTo(1L);
    }

    @Test
    void placeOrder_shouldThrowWhenInventoryUnavailable() {
        // Arrange
        when(inventoryService.checkAvailability(any())).thenReturn(false);

        // Act + Assert
        assertThatThrownBy(() -> orderService.placeOrder(validRequest()))
            .isInstanceOf(InsufficientInventoryException.class)
            .hasMessageContaining("LAPTOP-001");

        // Verify no payment attempt made when inventory is unavailable
        verifyNoInteractions(paymentService);
        verifyNoInteractions(orderRepository);
    }
}
```

---

## Scenario 9: Spring Data Specification for Advanced Search

**Setup:** Build a product search endpoint with 10+ optional filters. Use Spring Data Specifications to build type-safe, composable queries.

```java
// Specification factory
@Component
public class ProductSpecifications {

    public static Specification<Product> hasCategory(String category) {
        return (root, query, cb) ->
            category == null ? cb.conjunction()  // always true
                : cb.equal(root.get("category"), category);
    }

    public static Specification<Product> hasPriceBetween(BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            Predicate pred = cb.conjunction();
            if (min != null) pred = cb.and(pred, cb.ge(root.get("price"), min));
            if (max != null) pred = cb.and(pred, cb.le(root.get("price"), max));
            return pred;
        };
    }

    public static Specification<Product> isInStock() {
        return (root, query, cb) -> cb.gt(root.get("stockQuantity"), 0);
    }

    public static Specification<Product> hasTag(String tag) {
        return (root, query, cb) -> {
            Join<Product, Tag> tags = root.join("tags", JoinType.INNER);
            return cb.equal(tags.get("name"), tag);
        };
    }

    public static Specification<Product> nameContains(String keyword) {
        return (root, query, cb) ->
            keyword == null ? cb.conjunction()
                : cb.like(cb.lower(root.get("name")), "%" + keyword.toLowerCase() + "%");
    }
}

// Controller
@GetMapping("/products")
public Page<ProductResponse> search(
        @RequestParam(required = false) String category,
        @RequestParam(required = false) BigDecimal minPrice,
        @RequestParam(required = false) BigDecimal maxPrice,
        @RequestParam(required = false) String keyword,
        @RequestParam(defaultValue = "false") boolean inStockOnly,
        @RequestParam(required = false) String tag,
        @PageableDefault(size = 20) Pageable pageable) {

    Specification<Product> spec = Specification.where(
        ProductSpecifications.hasCategory(category))
        .and(ProductSpecifications.hasPriceBetween(minPrice, maxPrice))
        .and(ProductSpecifications.nameContains(keyword));

    if (inStockOnly) spec = spec.and(ProductSpecifications.isInStock());
    if (tag != null) spec = spec.and(ProductSpecifications.hasTag(tag));

    return productRepository.findAll(spec, pageable).map(mapper::toResponse);
}
```

---

## Scenario 10: Handling Cascading Transactions Correctly

**Setup:** `OrderService.placeOrder()` creates an order and calls `AuditService.log()`. The audit log must be committed even if the order transaction rolls back. The audit service must NOT be called if the order isn't placed.

```java
// WRONG: both in same transaction — audit rolls back with order
@Transactional
public Order placeOrder(CreateOrderRequest request) {
    Order order = createOrder(request);
    auditService.log("ORDER_PLACED", order.getId());  // same transaction!
    if (someCondition) throw new RuntimeException("rollback!");
    // Audit entry also rolled back!
}

// WRONG: AuditService with REQUIRES_NEW — called unconditionally
// (but order might not actually be placed due to validation before save)

// CORRECT: TransactionalEventListener — fires AFTER commit
@Transactional
public Order placeOrder(CreateOrderRequest request) {
    Order order = createOrder(request);
    orderRepository.save(order);
    // Publish event — will fire after commit
    eventPublisher.publishEvent(new OrderPlacedEvent(this, order));
    return order;
}

@Component
public class OrderAuditListener {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Runs AFTER the order transaction commits successfully
        // Has its own transaction (REQUIRED by default)
        auditService.log("ORDER_PLACED", event.getOrderId());
    }
}

// Verify in tests
@Test
void auditLoggedAfterSuccessfulOrder() {
    orderService.placeOrder(validRequest());
    verify(auditService).log("ORDER_PLACED", anyLong());
}

@Test
void noAuditIfOrderFails() {
    when(inventoryService.reserve(any())).thenThrow(new RuntimeException("inventory error"));
    assertThatThrownBy(() -> orderService.placeOrder(validRequest()));
    verifyNoInteractions(auditService);  // TransactionalEventListener not fired on rollback
}
```

---

## Follow-Up Questions

1. **Hibernate vs MyBatis:** When would you choose MyBatis over JPA/Hibernate?
   MyBatis: complex SQL, stored procedures, full control over queries. Hibernate: domain-rich model, automatic dirty checking, second-level cache, less boilerplate for CRUD.

2. **Spring Boot startup order:** How do you ensure Bean A initializes before Bean B?
   Use `@DependsOn("beanA")`, constructor injection (natural order), or `ApplicationRunner`/`CommandLineRunner` with `@Order` for post-startup initialization.

3. **Hibernate show-sql in production:** Should you enable `hibernate.show_sql` in production?
   No. It writes to stdout and affects performance. Use P6Spy or datasource-proxy for production SQL logging with parameter values and execution timing.

4. **@Transactional on private methods:** Does it work?
   No. Spring AOP cannot intercept private methods. `@Transactional` on private methods is silently ignored. Only works on public methods of Spring-managed beans.

5. **EntityManager thread safety:** Is EntityManager thread-safe?
   No. Each thread should have its own EntityManager. Spring's `@PersistenceContext` injects a thread-safe proxy that delegates to the current transaction's EntityManager.
