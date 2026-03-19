# Spring & Hibernate Basics — Follow-Up Traps

---

## Trap 1: @Transactional on Private Methods — Why It Doesn't Work

```java
@Service
public class OrderService {

    @Transactional  // THIS IS IGNORED
    private void saveOrderInternal(Order order) {
        orderRepository.save(order);
    }

    public void placeOrder(OrderRequest req) {
        Order order = new Order(req);
        saveOrderInternal(order);  // no transaction applied!
    }
}
```

**Why?** Spring AOP creates a subclass proxy (CGLIB) or interface-based proxy (JDK). The proxy overrides your methods to add transaction behavior. **You cannot override a private method** in a subclass — it's not part of the public contract. Therefore, Spring's proxy cannot intercept the call.

Spring will NOT warn you at startup. The annotation silently has no effect.

**Fix:** make the method `public` (or at least `protected` for CGLIB). Or move it to a separate `@Service` bean and inject it.

---

## Trap 2: Self-Invocation With @Transactional — The Same Trap

Even if the method is public, calling it from the same class bypasses the proxy:

```java
@Service
public class OrderService {

    public void placeOrder(OrderRequest req) {
        // 'this' refers to the real OrderService, NOT the Spring proxy
        this.saveWithNewTransaction(req);  // proxy is bypassed!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveWithNewTransaction(OrderRequest req) {
        // The REQUIRES_NEW transaction is NOT started!
        // This runs in whatever transaction is already active (or none)
    }
}
```

The client calls `orderServiceProxy.placeOrder()`. Inside `placeOrder()`, `this.saveWithNewTransaction()` calls directly on the real object — the proxy is not involved.

**Fixes:**
```java
// Fix 1: Inject the proxy via @Autowired self-injection
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // Spring injects the proxy here

    public void placeOrder(OrderRequest req) {
        self.saveWithNewTransaction(req);  // goes through proxy!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void saveWithNewTransaction(OrderRequest req) { ... }
}

// Fix 2: Extract to a separate bean
@Service
public class OrderPersistenceService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void save(OrderRequest req) { ... }
}

@Service
public class OrderService {
    @Autowired
    private OrderPersistenceService persistenceService;

    public void placeOrder(OrderRequest req) {
        persistenceService.save(req);  // different bean = proxy is used
    }
}
```

---

## Trap 3: @Autowired on Final Fields Without Constructor Injection — Issues

```java
// PROBLEMATIC
@Service
public class OrderService {
    @Autowired
    private final OrderRepository repository;  // field injection + final = compile error!
}
// 'repository' must be initialized in declaration or constructor — can't be injected by field
```

Field injection using `@Autowired` requires the field to be non-final. Final fields can only be set in constructors.

```java
// CORRECT — constructor injection with final field
@Service
@RequiredArgsConstructor  // Lombok generates constructor for all final fields
public class OrderService {
    private final OrderRepository repository;  // @Autowired not needed with Lombok
    private final EmailService emailService;
}
```

Without Lombok:
```java
@Service
public class OrderService {
    private final OrderRepository repository;
    private final EmailService emailService;

    public OrderService(OrderRepository repository, EmailService emailService) {
        this.repository = repository;
        this.emailService = emailService;
    }
}
```

Spring auto-detects single constructors — no `@Autowired` needed.

---

## Trap 4: Lazy-Loaded Entity After Session Closes — LazyInitializationException

```java
@Entity
public class Order {
    @OneToMany(fetch = FetchType.LAZY)  // default for @OneToMany
    private List<OrderItem> items;
}

// Service method — Session opens and closes here
@Service
public class OrderService {
    @Transactional
    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
        // Session closes when @Transactional method returns
        // 'items' collection is not loaded yet (lazy)
    }
}

// Controller
@GetMapping("/orders/{id}")
public OrderResponse getOrder(@PathVariable Long id) {
    Order order = orderService.getOrder(id);
    // Session is CLOSED here
    order.getItems().size();  // org.hibernate.LazyInitializationException!
    // "could not initialize proxy - no Session"
}
```

**Fixes:**
```java
// Fix 1: Eagerly fetch in the query when you know you need items
@Query("SELECT o FROM Order o LEFT JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);

// Fix 2: Use @Transactional in the controller (anti-pattern but sometimes needed)
// Opens session for entire request

// Fix 3: Use DTOs — map inside the @Transactional boundary
@Transactional(readOnly = true)
public OrderDTO getOrderDTO(Long id) {
    Order order = repo.findById(id).orElseThrow();
    return new OrderDTO(order.getId(), order.getItems().stream()
        .map(item -> new ItemDTO(item.getId(), item.getPrice()))
        .collect(Collectors.toList()));
    // All lazy loading done BEFORE Session closes
}

// Fix 4: @EntityGraph — declare which associations to fetch
@EntityGraph(attributePaths = {"items", "items.product"})
Optional<Order> findById(Long id);
```

The open-session-in-view pattern (keeping Session open for the entire HTTP request) "fixes" this but is an anti-pattern — it leaks database connections into the view layer and hides N+1 problems.

---

## Trap 5: save() vs persist() vs merge() — What's the Difference?

```java
// JPA EntityManager:

// persist(entity) — makes a TRANSIENT entity PERSISTENT
// - Entity must be new (no ID, or not yet managed)
// - Must be called within a transaction
// - The entity object you pass in becomes managed
// - Returns void
entityManager.persist(newOrder);  // newOrder is now managed

// merge(entity) — copies state from DETACHED entity into a PERSISTENT copy
// - Works whether entity exists in DB or not (insert or update)
// - Returns a NEW managed entity (the original detached entity remains detached!)
// - Copies all field values to the managed copy
Order managedOrder = entityManager.merge(detachedOrder);  // managedOrder is managed, detachedOrder is not

// Spring Data JpaRepository:

// save(entity) — smart wrapper: calls persist for new, merge for existing
// - If entity has no ID (or ID is null): calls persist
// - If entity has an ID: calls merge
// The magic is in SimpleJpaRepository.save():
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);
    }
}
// "isNew" checks if ID is null, or if entity implements Persistable and isNew() returns true

// saveAndFlush(entity) — save() + immediately flush to DB (don't wait for commit)
// Useful when you need the DB to validate constraints immediately

// Classic Hibernate Session (not JPA):
// session.save(obj)        -> persist, returns generated ID
// session.update(obj)      -> update detached entity (assumes it exists)
// session.saveOrUpdate(obj)-> save if new, update if exists
// session.merge(obj)       -> JPA merge equivalent
```

**Key trap:** after `merge()`, the object you passed in is NOT managed. Always use the returned object:
```java
Order detached = getDetachedOrder();
entityManager.merge(detached);
detached.setStatus(SHIPPED);  // NOT tracked — wrong!

Order managed = entityManager.merge(detached);  // use the return value
managed.setStatus(SHIPPED);   // TRACKED — correct!
```

---

## Trap 6: Hibernate Dirty Checking — When Does It Flush?

Hibernate tracks all persistent entities. Before certain operations, it **flushes** — compares current state with the snapshot taken at load time and generates UPDATEs for changed fields.

**When does flush happen?**
1. Explicitly: `entityManager.flush()` or `session.flush()`
2. Before a query that might be affected by pending changes (FlushMode.AUTO — default)
3. At transaction commit

```java
@Transactional
public void updateOrder(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    // Hibernate snapshots: {status=PENDING, amount=100}

    order.setStatus(OrderStatus.PAID);
    // No UPDATE yet — just marks entity as dirty in memory

    List<Order> orders = orderRepository.findAll();
    // Hibernate auto-flushes BEFORE this query because the pending change
    // might affect the query results!
    // SQL: UPDATE orders SET status='PAID' WHERE id=?
    // SQL: SELECT * FROM orders
    // This is FlushMode.AUTO behavior
}
// At commit: Hibernate checks all managed entities for remaining dirty state
// Issues any remaining UPDATEs
```

**readOnly = true hint:**
```java
@Transactional(readOnly = true)
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
    // readOnly=true:
    // 1. Hibernate skips dirty checking (no need to track changes)
    // 2. No flush at commit
    // 3. Hibernate can optimize memory (some versions don't snapshot the entity)
    // Small performance improvement for read-only operations
}
```

---

## Trap 7: @Transactional readOnly=true — Does It Really Improve Performance?

**Yes, but modestly and indirectly:**

1. **Hibernate dirty checking skip:** Hibernate won't track or compare entity state for changes — saves CPU and memory for large result sets.
2. **No flush on commit:** no pre-commit flush = faster transaction close.
3. **JDBC hint:** Spring passes `connection.setReadOnly(true)` to the JDBC driver. Some drivers/databases optimize for this (MySQL: may route to read replica; Oracle: skips undo log generation).
4. **Spring Data JPA caching:** in some configurations, `@Transactional(readOnly=true)` signals that no second-level cache invalidation is needed.

**What it does NOT do:**
- Does NOT prevent you from calling `save()` or modifying entities — you'll just get an error at flush time
- Does NOT guarantee routing to a database read replica by itself (need `AbstractRoutingDataSource` for that)
- Does NOT improve performance dramatically for simple queries

```java
// Good practice: mark all service methods that only read data
@Transactional(readOnly = true)
public Page<OrderDTO> getOrders(Pageable pageable) {
    return orderRepository.findAll(pageable).map(OrderMapper::toDTO);
}

@Transactional  // default: readOnly=false — for write operations
public Order placeOrder(OrderRequest req) {
    return orderRepository.save(new Order(req));
}
```

---

## Trap 8: Prototype Bean Injected Into Singleton Bean — The Gotcha

```java
@Component
@Scope("prototype")
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    // Intended to be per-user
}

@Service  // Singleton — created once
public class CartService {
    @Autowired
    private ShoppingCart cart;  // Injected ONCE at CartService creation

    public void addItem(Item item) {
        cart.addItem(item);  // ALL users share the same ShoppingCart object!
    }
}
```

The `ShoppingCart` is a prototype, but since it's injected into a singleton, Spring creates it exactly once — when the `CartService` singleton is created. The prototype nature is lost.

**Fix: use ApplicationContext.getBean() at runtime or @Lookup:**
```java
@Service
public abstract class CartService {

    @Lookup  // Spring overrides this at runtime to always return a new prototype
    public abstract ShoppingCart getCart();

    public void addItem(Item item) {
        ShoppingCart cart = getCart();  // new instance each call
        cart.addItem(item);
    }
}

// Or with ObjectProvider (modern approach):
@Service
public class CartService {
    @Autowired
    private ObjectProvider<ShoppingCart> cartProvider;

    public void addItem(Item item) {
        ShoppingCart cart = cartProvider.getObject();  // new prototype each call
        cart.addItem(item);
    }
}
```

---

## Trap 9: @PostConstruct and @PreDestroy — When Exactly Are They Called?

```java
@Component
public class CacheService {

    @PostConstruct
    public void init() {
        // Called AFTER:
        //   1. Constructor
        //   2. Dependency injection (@Autowired fields set)
        // Called BEFORE:
        //   1. Bean is put into the application context
        //   2. Any client can use this bean
        // Safe to use injected dependencies here!
        loadCacheFromDatabase();
    }

    @PreDestroy
    public void cleanup() {
        // Called when ApplicationContext is closing (shutdown)
        // Order of destruction is reverse of initialization
        // NOT called if the JVM is killed with kill -9 or System.exit()
        // Use a shutdown hook for guaranteed cleanup
        cacheStore.clear();
    }
}
```

**Common mistake:** accessing other beans in constructor:
```java
@Component
public class BadService {
    @Autowired
    private DependencyService dep;

    public BadService() {
        dep.doSomething();  // NullPointerException! @Autowired not set yet
    }

    @PostConstruct
    public void init() {
        dep.doSomething();  // CORRECT — DI is complete at @PostConstruct
    }
}
```

---

## Trap 10: Circular Dependency in Spring — How to Detect and Fix?

```java
// Circular dependency with constructor injection — fails at startup:
@Service
public class ServiceA {
    public ServiceA(ServiceB b) { ... }
}

@Service
public class ServiceB {
    public ServiceB(ServiceA a) { ... }
}
// Error: The dependencies of some of the beans in the application context
//        form a cycle: serviceA -> serviceB -> serviceA
```

Spring can detect constructor injection cycles at startup — this is actually good: fail fast.

**With field injection, Spring resolves the cycle (but it's a code smell):**
```java
@Service
public class ServiceA {
    @Autowired ServiceB b;  // Spring breaks the cycle using a partial proxy
}
@Service
public class ServiceB {
    @Autowired ServiceA a;
}
// This works but is a design problem
```

**How to fix circular dependencies:**
```java
// Fix 1: Introduce a third shared service that both depend on
@Service
public class SharedLogicService { ... }

@Service
public class ServiceA {
    public ServiceA(SharedLogicService shared) { ... }
}
@Service
public class ServiceB {
    public ServiceB(SharedLogicService shared) { ... }
}

// Fix 2: Use @Lazy on one injection point
@Service
public class ServiceA {
    public ServiceA(@Lazy ServiceB b) { ... }  // ServiceB proxy created lazily
}

// Fix 3: Use ApplicationEventPublisher for loose coupling
// ServiceA publishes an event; ServiceB listens — no direct dependency
@Service
public class ServiceA {
    @Autowired ApplicationEventPublisher publisher;

    public void doSomething() {
        publisher.publishEvent(new SomethingHappenedEvent(this));
    }
}

@Service
public class ServiceB {
    @EventListener
    public void handleEvent(SomethingHappenedEvent event) { ... }
}
```

Circular dependencies always indicate a design problem. Before using `@Lazy` as a band-aid, consider whether the two services should be merged, or whether a third abstraction should extract the shared logic.
