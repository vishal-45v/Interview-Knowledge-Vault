# N+1 Problem — Scenarios

> 15+ real-world scenarios covering N+1 detection, JOIN FETCH, @EntityGraph, and @BatchSize solutions.

---

## Scenario 1: Classic N+1 in a Service

**Problem:** Loading orders with their items triggers an N+1 problem.

```java
@Service
@Transactional(readOnly = true)
public class OrderReportService {

    public List<OrderSummary> generateReport() {
        List<Order> orders = orderRepository.findAll();  // 1 query: SELECT * FROM orders

        return orders.stream()
            .map(order -> {
                // N queries — one per order!
                int itemCount = order.getItems().size();  // SELECT * FROM order_items WHERE order_id=?
                BigDecimal total = order.getItems().stream()
                    .map(OrderItem::getPrice)
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

                return new OrderSummary(order.getId(), itemCount, total);
            })
            .collect(Collectors.toList());
    }
    // With 1000 orders: 1 + 1000 = 1001 queries!
}
```

**Detecting the N+1:**
- Enable SQL logging: `spring.jpa.show-sql=true`
- Use P6Spy or DataSource proxy to count queries per request
- Hibernate Statistics: `hibernate.generate_statistics=true`

```java
// Log query count per request with datasource-proxy:
@Bean
public DataSource dataSource(DataSourceProperties properties) {
    return ProxyDataSourceBuilder
        .create(properties.initializeDataSourceBuilder().build())
        .logQueryBySlf4j(SLF4JLogLevel.DEBUG)
        .countQuery()
        .build();
}
```

---

## Scenario 2: Fix 1 — JOIN FETCH in JPQL

```java
// Fix the N+1 with a single JOIN FETCH query
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items")
    List<Order> findAllWithItems();
}

// SQL generated:
// SELECT DISTINCT o.*, i.* FROM orders o
// INNER JOIN order_items i ON i.order_id = o.id
// ← All data in ONE query!

@Service
@Transactional(readOnly = true)
public class OrderReportService {

    public List<OrderSummary> generateReport() {
        List<Order> orders = orderRepository.findAllWithItems();  // 1 query!

        return orders.stream()
            .map(order -> {
                int itemCount = order.getItems().size();  // From L1 cache — no SQL!
                BigDecimal total = ...;
                return new OrderSummary(order.getId(), itemCount, total);
            })
            .collect(Collectors.toList());
    }
}
```

---

## Scenario 3: Fix 2 — @EntityGraph

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Applies entity graph to standard findAll()
    @Override
    @EntityGraph(attributePaths = {"items"})
    List<Order> findAll();

    // Or define a named entity graph on the entity:
    // @NamedEntityGraph(name="Order.withItems", attributeNodes=@NamedAttributeNode("items"))
    @EntityGraph("Order.withItems")
    List<Order> findByStatus(OrderStatus status);

    // Load multiple associations:
    @EntityGraph(attributePaths = {"items", "items.product", "customer"})
    Optional<Order> findById(Long id);
}
```

---

## Scenario 4: Fix 3 — @BatchSize (Best for Paginated Lists)

```java
@Entity
public class Order {

    @OneToMany(fetch = FetchType.LAZY, mappedBy = "order")
    @BatchSize(size = 50)  // Load items in batches of 50
    private List<OrderItem> items;

    @ManyToOne(fetch = FetchType.LAZY)
    @BatchSize(size = 50)  // Also works on @ManyToOne (for lists of orders with same customer)
    private Customer customer;
}

// Without @BatchSize (N+1):
// 100 orders → 100 SELECT WHERE order_id=?

// With @BatchSize(50):
// 100 orders → 2 SELECT WHERE order_id IN (50 IDs each) = 2 queries
```

**When to use @BatchSize vs JOIN FETCH:**

| Scenario | Use |
|----------|-----|
| Always need the association | JOIN FETCH / @EntityGraph |
| Pagination (Page<Order>) | @BatchSize (JOIN FETCH breaks pagination) |
| Large collections | @BatchSize |
| Multiple separate queries OK | @BatchSize |

---

## Scenario 5: N+1 in @ManyToOne (Author-Books)

```java
// Load all books — each book has a lazy Author
@Service
public class BookService {

    public List<BookDTO> getAllBooks() {
        List<Book> books = bookRepository.findAll();  // 1 query

        return books.stream()
            .map(book -> new BookDTO(
                book.getTitle(),
                book.getAuthor().getName()  // N queries! One per book
            ))
            .collect(Collectors.toList());
    }
}

// Fix: JOIN FETCH for @ManyToOne
@Query("SELECT b FROM Book b JOIN FETCH b.author")
List<Book> findAllWithAuthor();

// Fix: DTO Projection (avoids entity lifecycle entirely)
@Query("SELECT new com.example.dto.BookDTO(b.title, a.name) " +
       "FROM Book b JOIN b.author a")
List<BookDTO> findAllAsDTO();
```

---

## Scenario 6: N+1 with Spring Data Projections

```java
// Interface projection can still trigger N+1!
public interface OrderProjection {
    Long getId();
    String getStatus();
    List<OrderItemProjection> getItems();  // This might cause N+1!
}

// Safer: Use JPQL DTO constructor or flat projection
public interface FlatOrderProjection {
    Long getOrderId();
    String getOrderStatus();
    String getItemName();     // From JOIN — flat row per item
    BigDecimal getItemPrice();
}

@Query("SELECT o.id as orderId, o.status as orderStatus, " +
       "i.name as itemName, i.price as itemPrice " +
       "FROM Order o JOIN o.items i WHERE o.id = :id")
List<FlatOrderProjection> findOrderWithItemsFlat(@Param("id") Long id);
```

---

## Scenario 7: Detecting N+1 with Hibernate Statistics

```java
// application.yml
// spring.jpa.properties.hibernate.generate_statistics: true
// logging.level.org.hibernate.stat: DEBUG

@Test
void shouldNotHaveNPlusOneIssue() {
    SessionFactory sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
    sessionFactory.getStatistics().setStatisticsEnabled(true);
    sessionFactory.getStatistics().clear();

    // Trigger the operation being tested
    orderService.generateReport();

    Statistics stats = sessionFactory.getStatistics();
    long queryCount = stats.getQueryExecutionCount();

    // Assert that only a few queries were executed (not N+1)
    assertThat(queryCount).isLessThanOrEqualTo(3);
}
```

---

## Scenario 8: N+1 in Spring Data's @Query with Nested Objects

```java
// This seemingly simple @Query can cause N+1
@Query("SELECT o FROM Order o WHERE o.createdAt > :since")
List<Order> findRecentOrders(@Param("since") LocalDate since);

// If Order has @ManyToOne Customer (EAGER) — each order triggers customer SELECT
// If Order has @OneToMany items (LAZY) — items triggered when accessed

// Better: Projection query returns only what you need
@Query("SELECT new com.example.dto.OrderSummaryDTO(" +
       "o.id, o.status, o.total, c.firstName, c.lastName) " +
       "FROM Order o JOIN o.customer c " +
       "WHERE o.createdAt > :since")
List<OrderSummaryDTO> findRecentOrderSummaries(@Param("since") LocalDate since);
// Single JOIN query, returns DTOs directly — no N+1 possible
```

---

## Scenario 9: N+1 in Recursive/Self-Referential Relationships

```java
@Entity
public class Category {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @OneToMany(mappedBy = "parent", fetch = FetchType.LAZY)
    @BatchSize(size = 20)  // Critical for tree structures!
    private List<Category> children;

    @ManyToOne(fetch = FetchType.LAZY)
    private Category parent;
}

// Loading category tree without @BatchSize:
// Load root → 1 query
// Load root's children → 1 query
// Load each child's children → N queries
// → Exponential N+1 for deep trees!

// With @BatchSize:
// Load root → 1 query
// Load root's children → 1 query (batch by parent IDs)
// Load grandchildren → 1 query (batch by parent IDs)
```

---

## Scenario 10: GraphQL N+1 with DataLoader Pattern

**Problem:** GraphQL queries like `{ orders { id items { name } } }` cause N+1 when naively implemented.

```java
// GraphQL resolver — N+1 problem
@QueryMapping
public List<Order> orders() {
    return orderRepository.findAll();  // 1 query
}

@SchemaMapping
public List<OrderItem> items(Order order) {
    return order.getItems();  // N queries — one per order!
}

// Fix: DataLoader batching (Spring for GraphQL)
@SchemaMapping
public CompletableFuture<List<OrderItem>> items(Order order, DataLoader<Long, List<OrderItem>> loader) {
    return loader.load(order.getId());
    // Spring for GraphQL batches these requests:
    // Instead of N queries: 1 query WHERE order_id IN (all order IDs)
}
```
