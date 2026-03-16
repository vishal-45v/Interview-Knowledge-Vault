# Hibernate & JPA Internals — Scenarios

> 20+ scenarios covering entity lifecycle, dirty checking, L1/L2 cache, and Hibernate performance.

---

## Scenario 1: Understanding Entity States

**Problem:** Explain what happens to entity state when you call various JPA operations.

```java
@Service
@Transactional
public class ProductService {

    public void demonstrateEntityStates() {

        // 1. TRANSIENT: Object not associated with any EntityManager
        Product product = new Product("Widget", 9.99);
        // product has no DB row, no EntityManager knows about it

        // 2. PERSISTENT: Managed by EntityManager (inside TX)
        entityManager.persist(product);
        // Now in persistence context, INSERT will be generated on flush
        // Changes are automatically tracked (dirty checking)

        product.setPrice(10.99);  // EntityManager tracks this change!
        // No explicit save() needed — change will be persisted on flush

        // 3. DETACHED: Was managed, but EntityManager closed or entity evicted
        entityManager.detach(product);
        // OR transaction ended (if no OSIV)
        product.setPrice(11.99);  // Change NOT tracked! Will NOT be saved

        // 4. REMOVED: Marked for deletion
        Product toDelete = entityManager.find(Product.class, 1L);
        entityManager.remove(toDelete);
        // DELETE will be generated on flush

        // To re-attach a detached entity:
        Product mergedProduct = entityManager.merge(product);
        // Returns a NEW managed instance, original remains detached
    }
}
```

---

## Scenario 2: Dirty Checking — How Hibernate Detects Changes

**Problem:** You update an entity but don't call `save()`. Will changes be persisted?

```java
@Service
@Transactional
public class OrderService {

    public void updateOrderStatus(Long orderId, String newStatus) {
        // find() returns a MANAGED entity
        Order order = orderRepository.findById(orderId).orElseThrow();

        // NO explicit save() call — just modify the entity
        order.setStatus(newStatus);
        order.setUpdatedAt(Instant.now());

        // On transaction commit (end of @Transactional method):
        // Hibernate compares current state to snapshot taken at load time
        // Detects: status changed, updatedAt changed
        // Generates: UPDATE orders SET status=?, updated_at=? WHERE id=?
        // ← Automatically saved!
    }
}
```

**How dirty checking works:**
1. When entity is loaded, Hibernate stores a snapshot of its state
2. On flush, Hibernate compares current state to snapshot
3. For changed fields, Hibernate generates UPDATE SQL
4. All fields in the entity are included in the UPDATE by default (or use `@DynamicUpdate`)

```java
@Entity
@DynamicUpdate  // Only include changed fields in UPDATE
public class Order {
    // With @DynamicUpdate: UPDATE orders SET status=? WHERE id=?
    // Without: UPDATE orders SET status=?, name=?, price=?, ... (all fields)
}
```

---

## Scenario 3: First-Level Cache (Persistence Context)

**Problem:** You call `findById()` twice in the same transaction. Will there be two SQL queries?

```java
@Service
@Transactional
public class ProductService {

    public void demonstrateL1Cache() {
        // First call: SELECT * FROM products WHERE id=1
        Product p1 = productRepository.findById(1L).orElseThrow();

        // Second call: NO SQL! Returns from L1 cache (same instance)
        Product p2 = productRepository.findById(1L).orElseThrow();

        System.out.println(p1 == p2);  // true — SAME object reference!

        // L1 cache is scoped to the EntityManager (= transaction by default)
        // Cannot be disabled per query (unlike L2 cache)
    }
}

// PROBLEM: L1 cache can return stale data
@Service
@Transactional(readOnly = true)
public ProductService {

    public List<Product> getProducts() {
        List<Product> products = productRepository.findAll();

        // Another thread updates product #1 in DB

        Product p1 = productRepository.findById(1L).orElseThrow();
        // Returns the cached version (stale!) — not re-queried from DB

        return products;
    }
}

// Fix for stale L1 cache:
entityManager.refresh(product);  // Force re-query from DB
```

---

## Scenario 4: Second-Level Cache

**Problem:** You want to cache entities across transactions to avoid repeated DB queries for rarely-changing data.

```java
// 1. Enable second-level cache
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // or NONSTRICT_READ_WRITE, READ_ONLY
public class Country {

    @Id
    private String code;  // e.g., "US"
    private String name;  // e.g., "United States"
}

// 2. Configure cache provider in application.yml
// spring:
//   jpa:
//     properties:
//       hibernate.cache.use_second_level_cache: true
//       hibernate.cache.region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
//       javax.cache.provider: org.ehcache.jsr107.EhcacheCachingProvider

// 3. Usage — automatic once configured
@Service
@Transactional(readOnly = true)
public class OrderService {

    public String getCountryName(String code) {
        // TX 1: SELECT FROM countries → stores in L2 cache
        Country country = countryRepository.findById(code).orElseThrow();

        // TX 2 (different transaction): gets from L2 cache — NO DB query!
        Country sameCountry = countryRepository.findById(code).orElseThrow();

        return country.getName();
    }
}
```

---

## Scenario 5: merge() vs persist() — When to Use Each

```java
@Service
@Transactional
public class ProductService {

    // persist(): Use for NEW entities (no ID assigned by application)
    public Product createProduct(CreateProductRequest request) {
        Product product = new Product(request.getName(), request.getPrice());
        // product.getId() == null
        entityManager.persist(product);
        // After persist(): product is now MANAGED, has ID from DB sequence
        return product;
    }

    // merge(): Use for DETACHED entities (have an ID, came from outside TX)
    public Product updateProduct(ProductDTO dto) {
        // dto was loaded in a previous request — it's DETACHED
        Product detachedProduct = productMapper.toEntity(dto);
        // detachedProduct.getId() != null

        // merge(): looks up entity in DB, copies state, returns MANAGED copy
        Product managedProduct = entityManager.merge(detachedProduct);
        // detachedProduct is still DETACHED!
        // managedProduct is the new MANAGED version with updated state
        return managedProduct;
    }
}
```

---

## Scenario 6: @Version for Optimistic Locking

```java
@Entity
public class FlightSeat {

    @Id @GeneratedValue
    private Long id;

    private String seatNumber;
    private boolean booked;

    @Version  // Hibernate increments this on each update
    private Long version;
}

@Service
@Transactional
public class SeatBookingService {

    public Booking bookSeat(Long seatId, User user) {
        FlightSeat seat = seatRepository.findById(seatId).orElseThrow();

        if (seat.isBooked()) {
            throw new SeatAlreadyBookedException("Seat already booked: " + seat.getSeatNumber());
        }

        seat.setBooked(true);
        // Hibernate generates:
        // UPDATE flight_seats SET booked=true, version=2 WHERE id=? AND version=1
        // If another TX already updated (version changed to 2):
        //   UPDATE returns 0 rows → StaleObjectStateException → OptimisticLockException

        return bookingRepository.save(new Booking(seat, user));
    }
}

// Handle optimistic lock failure at controller or service layer
@Service
public class SeatBookingFacade {

    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    @Transactional
    public Booking bookSeat(Long seatId, User user) {
        return seatBookingService.bookSeat(seatId, user);
    }
}
```

---

## Scenario 7: Entity Graph for Performance

**Problem:** You need to load an `Order` with its `items` and `customer` in a single query without N+1:

```java
// Define entity graph
@NamedEntityGraph(
    name = "Order.withItemsAndCustomer",
    attributeNodes = {
        @NamedAttributeNode("items"),
        @NamedAttributeNode("customer")
    }
)
@Entity
public class Order { ... }

// Use in repository
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @EntityGraph("Order.withItemsAndCustomer")
    Optional<Order> findById(Long id);

    // Or define inline:
    @EntityGraph(attributePaths = {"items", "customer"})
    List<Order> findByStatus(OrderStatus status);
}

// Generates a JOIN query:
// SELECT o.*, i.*, c.* FROM orders o
// LEFT JOIN order_items i ON i.order_id = o.id
// LEFT JOIN customers c ON c.id = o.customer_id
// WHERE o.id = ?
```

---

## Scenario 8: JPQL vs Native Query

```java
// JPQL: Object-oriented queries on entities/fields
@Query("SELECT o FROM Order o WHERE o.customer.email = :email AND o.status = :status")
List<Order> findByCustomerEmailAndStatus(@Param("email") String email,
                                          @Param("status") OrderStatus status);

// Advantages: DB-agnostic, entity lifecycle management, type-safe
// Disadvantages: Cannot use DB-specific functions, limited for complex queries

// Native SQL: Database-specific queries
@Query(value = "SELECT * FROM orders o WHERE o.customer_id = :customerId " +
               "AND o.created_at > NOW() - INTERVAL '30 days' " +
               "ORDER BY o.total DESC",
       nativeQuery = true)
List<Order> findRecentOrdersNative(@Param("customerId") Long customerId);

// Advantages: Full SQL power, DB-specific optimizations
// Disadvantages: DB-specific, bypasses entity lifecycle, harder to test
```

---

## Scenario 9: Batch Operations with saveAll()

**Problem:** You need to import 100,000 products. Calling `save()` in a loop is extremely slow.

```java
// SLOW: 100,000 individual INSERT statements
for (ProductDTO dto : productDTOs) {
    productRepository.save(new Product(dto.getName(), dto.getPrice()));
}

// FAST: Batch inserts
@Service
@Transactional
public class ProductImportService {

    @PersistenceContext
    private EntityManager entityManager;

    public void importProducts(List<ProductDTO> dtos) {
        int batchSize = 100;

        for (int i = 0; i < dtos.size(); i++) {
            Product product = new Product(dtos.get(i).getName(), dtos.get(i).getPrice());
            entityManager.persist(product);

            if (i % batchSize == 0 && i > 0) {
                entityManager.flush();   // Send to DB
                entityManager.clear();  // Clear L1 cache (prevents OOM)
            }
        }
        entityManager.flush();  // Final flush
    }
}

// Configure batch inserts in application.yml:
// spring:
//   jpa:
//     properties:
//       hibernate.jdbc.batch_size: 100
//       hibernate.order_inserts: true
//       hibernate.order_updates: true
```

---

## Scenario 10: @Transient vs transient

```java
@Entity
public class Product {

    @Id @GeneratedValue
    private Long id;

    private String name;
    private BigDecimal price;

    @Transient  // Not persisted to DB column
    private BigDecimal priceWithTax;  // Calculated field

    // This field IS included in Java serialization but NOT in DB
    @Transient
    private Map<String, String> attributes = new HashMap<>();

    // Java transient keyword: excluded from Java serialization (NOT DB)
    // Hibernate ignores the Java transient keyword for persistence!
    private transient String tempCalculation;  // Still persisted to DB!
}
```

---

## Scenario 11: SessionFactory vs EntityManagerFactory

```java
// JPA-standard way (prefer this in Spring Boot):
@PersistenceContext
private EntityManager entityManager;

// Hibernate-native way (when you need Hibernate-specific features):
@Autowired
private SessionFactory sessionFactory;

// Open a Hibernate Session:
try (Session session = sessionFactory.openSession()) {
    Transaction tx = session.beginTransaction();
    Product product = session.get(Product.class, 1L);
    product.setPrice(new BigDecimal("15.99"));
    tx.commit();
}

// Hibernate-specific operations:
session.createNativeQuery("...", Product.class);
session.setDefaultReadOnly(true);
session.doWork(connection -> { ... });  // Access JDBC Connection directly
```

---

## Scenario 12: Soft Delete with @Where and @SQLDelete

```java
@Entity
@SQLDelete(sql = "UPDATE products SET deleted_at = NOW() WHERE id = ?")
@Where(clause = "deleted_at IS NULL")  // All queries exclude soft-deleted rows
public class Product {

    @Id @GeneratedValue
    private Long id;

    private String name;
    private BigDecimal price;

    @Column(name = "deleted_at")
    private Instant deletedAt;
}

// entityManager.remove(product) → runs SQL in @SQLDelete → sets deleted_at
// productRepository.findAll() → adds "WHERE deleted_at IS NULL" automatically
// productRepository.findById(deletedProductId) → returns empty (excluded by @Where)

// To find soft-deleted records, use native query or remove @Where with @FilterDef
@FilterDef(name = "includeDeleted")
@Filter(name = "includeDeleted", condition = "1=1")  // No filter
@Entity
public class Product { ... }
```
