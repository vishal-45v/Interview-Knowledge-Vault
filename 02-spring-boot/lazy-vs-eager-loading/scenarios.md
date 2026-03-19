# Lazy vs Eager Loading — Scenarios

> 15+ real-world scenarios covering FetchType choices, LazyInitializationException, and OSIV.

---

## Scenario 1: LazyInitializationException — The Most Common Hibernate Bug

**Problem:** You load an `Order` in a service method annotated with `@Transactional`. After the method returns, the controller accesses the order's items and gets `LazyInitializationException`.

```java
// Entity
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    @OneToMany(fetch = FetchType.LAZY, mappedBy = "order")
    private List<OrderItem> items;  // Lazy by default for @OneToMany
}

// Service
@Service
public class OrderService {

    @Transactional  // TX ends when method returns
    public Order findOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
        // Order.items proxy created but NOT initialized
    }
}

// Controller
@GetMapping("/orders/{id}")
public OrderResponse getOrder(@PathVariable Long id) {
    Order order = orderService.findOrder(id);
    // TX is already closed here!

    int itemCount = order.getItems().size();
    // LazyInitializationException: could not initialize proxy - no Session
    return new OrderResponse(order);
}
```

**Root cause:** The persistence context (Session) is closed after `@Transactional` ends. Accessing the lazy collection outside the transaction tries to initialize it, but there's no active Session.

---

## Scenario 2: Fix 1 — Initialize Inside Transaction

```java
// Option A: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);

// Option B: @EntityGraph
@EntityGraph(attributePaths = {"items"})
Optional<Order> findById(Long id);

// Option C: Hibernate.initialize()
@Transactional
public Order findOrder(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    Hibernate.initialize(order.getItems());  // Force initialization
    return order;  // Returns with items already loaded
}
```

---

## Scenario 3: Fix 2 — Use DTOs (Best Practice)

```java
// Project to a DTO inside the transaction — avoids entity lifecycle issues entirely
@Service
public class OrderService {

    @Transactional(readOnly = true)
    public OrderDTO findOrder(Long id) {
        Order order = orderRepository.findById(id).orElseThrow();

        // Map to DTO inside TX — access any lazy collection here safely
        return new OrderDTO(
            order.getId(),
            order.getStatus(),
            order.getItems().stream()  // Safe: still in TX
                .map(item -> new OrderItemDTO(item.getProductName(), item.getQuantity()))
                .collect(Collectors.toList())
        );
        // Return DTO (plain object, no JPA proxy) — no lazy issues ✓
    }
}
```

---

## Scenario 4: Open Session in View (OSIV) — Convenience vs Performance

**What OSIV does:** Keeps the persistence context (Session) open for the entire HTTP request, including after the service layer returns. This allows controllers and view templates to access lazy collections.

```yaml
# Spring Boot default: OSIV enabled
spring:
  jpa:
    open-in-view: true  # Default — logs a WARNING
```

**Why OSIV is problematic:**
- Keeps a DB connection open for the entire HTTP request
- Includes time spent rendering views, serializing JSON, etc.
- Under high load, DB connection pool is exhausted
- Hides N+1 problems (lazy loading works, but issues masked)

**Recommended approach:**
```yaml
spring:
  jpa:
    open-in-view: false  # Recommended for production
```

```java
// With OSIV disabled, you MUST be explicit about what you load.
// This forces good design: load everything you need in the service layer.
@Service
@Transactional(readOnly = true)
public class OrderService {

    public OrderDTO getOrderForResponse(Long id) {
        // Explicitly load everything needed using JOIN FETCH or @EntityGraph
        Order order = orderRepository.findByIdWithItemsAndCustomer(id)
            .orElseThrow(() -> new ResourceNotFoundException("Order", id));
        return orderMapper.toDTO(order);
    }
}
```

---

## Scenario 5: FetchType.EAGER — When It Helps and Hurts

```java
// EAGER: Always loaded, even when you don't need it
@ManyToOne(fetch = FetchType.EAGER)  // Default for @ManyToOne and @OneToOne
private Customer customer;

// Problem: Every time you load an Order, you ALWAYS get the Customer
// Even for: SELECT COUNT(*) FROM orders — still JOINs customer!

@OneToMany(fetch = FetchType.EAGER)  // NEVER do this for collections!
private List<OrderItem> items;
// Problem: Loading 1000 orders → 1000 customers + 1000 item lists loaded eagerly
// Cartesian product in SQL: orders × items → potentially millions of rows
```

**Rule of thumb:**
- `@ManyToOne` and `@OneToOne`: LAZY is preferred (default is EAGER but change it)
- `@OneToMany` and `@ManyToMany`: Always LAZY (default, keep it)
- Load associations explicitly when needed with JOIN FETCH or @EntityGraph

---

## Scenario 6: Selective Loading with Projections

**Problem:** Your order list page only needs order ID, status, and total price. Loading full entities with all associations is wasteful.

```java
// Interface-based projection — Spring Data JPA
public interface OrderSummary {
    Long getId();
    OrderStatus getStatus();
    BigDecimal getTotal();
    String getCustomerEmail();  // From @ManyToOne Customer (auto-joined)
}

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Returns projections — no entity, no lazy loading issues, no L1 cache overhead
    List<OrderSummary> findByStatus(OrderStatus status);
}

// Constructor-based projection (JPQL DTO projection)
@Query("SELECT new com.example.dto.OrderSummaryDTO(o.id, o.status, o.total) " +
       "FROM Order o WHERE o.status = :status")
List<OrderSummaryDTO> findOrderSummaries(@Param("status") OrderStatus status);
```

---

## Scenario 7: Batch Loading — @BatchSize

**Problem:** You have orders with lazy-loaded items. Accessing items triggers N+1 queries. You can't always use JOIN FETCH (sometimes causes Cartesian product).

```java
@Entity
public class Order {

    @OneToMany(fetch = FetchType.LAZY, mappedBy = "order")
    @BatchSize(size = 25)  // Load items in batches of 25 orders at a time
    private List<OrderItem> items;
}

// Without @BatchSize: 100 orders → 100 SELECT queries for items (N+1)
// With @BatchSize(25): 100 orders → 4 SELECT queries for items (batch of 25)
// SQL: WHERE order_id IN (1, 2, 3, ..., 25)
```

---

## Scenario 8: JPQL JOIN FETCH vs JOIN

```java
// JOIN (for filtering only — does NOT initialize collection)
@Query("SELECT o FROM Order o JOIN o.customer c WHERE c.email = :email")
List<Order> findByCustomerEmail(@Param("email") String email);
// customer is used for filtering but still LAZY — accessing o.getCustomer() outside TX throws exception

// JOIN FETCH (loads the association eagerly for this query only)
@Query("SELECT o FROM Order o JOIN FETCH o.customer c WHERE c.email = :email")
List<Order> findByCustomerEmailWithCustomer(@Param("email") String email);
// customer is initialized — can access outside TX ✓

// Multiple JOIN FETCH — be careful of Cartesian product
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items JOIN FETCH o.customer")
List<Order> findAllWithItemsAndCustomer();
// DISTINCT needed to avoid duplicate Order rows in result
```

---

## Scenario 9: Testing Lazy Loading

```java
@SpringBootTest
@Transactional  // Test runs in a transaction — lazy loading works
class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @Test
    void shouldLoadOrderWithItems() {
        // This test runs inside a TX, so lazy loading is available
        Order order = orderService.findById(testOrderId);
        assertThat(order.getItems()).hasSize(3);  // Works ✓
    }
}

// WARNING: @Transactional on test methods auto-rolls back
// This means your changes to DB don't persist between tests

// For testing OSIV=false behavior (outside TX):
@Test
void shouldThrowWhenAccessingLazyOutsideTransaction() {
    // Load in TX
    Order order = orderService.findById(testOrderId);
    // TX ended — order is detached

    // This should throw LazyInitializationException
    assertThatThrownBy(() -> order.getItems().size())
        .isInstanceOf(LazyInitializationException.class);
}
```
