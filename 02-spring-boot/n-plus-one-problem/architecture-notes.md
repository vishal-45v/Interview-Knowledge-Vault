# N+1 Problem — Architecture Notes

---

## How Hibernate Executes Lazy Collections

```
// Order entity with lazy items list
Order order = session.find(Order.class, 1L);
// SQL: SELECT * FROM orders WHERE id=1
// items = Hibernate$PersistentBag (proxy) — not yet loaded

order.getItems()  ← triggers initialization
// Proxy checks: am I initialized? NO
// Proxy: session.initializeCollection(this, ...)
// SQL: SELECT * FROM order_items WHERE order_id=1
// items now populated
```

---

## @BatchSize Loading Flow

```
// 3 orders loaded, each with @BatchSize(2) on items
orderRepository.findAll()  // SELECT * FROM orders → [O1, O2, O3]

O1.getItems() triggered:
  → Check: other uninitialized orders? O2, O3
  → Batch: SELECT * FROM order_items WHERE order_id IN (1, 2)
  → O1 and O2 items loaded

O3.getItems() triggered:
  → Check: other uninitialized orders? None (batch exhausted)
  → Single: SELECT * FROM order_items WHERE order_id IN (3)

Total: 3 orders + 2 batch queries = 3 queries (vs 4 without batching)
With @BatchSize(50): any number of orders up to 50 → just 2 queries
```

---

## JOIN FETCH with Multiple Collections

```java
// FORBIDDEN: Cannot JOIN FETCH two collections in same query
@Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.payments")
// → MultipleBagFetchException: cannot simultaneously fetch multiple bags

// Fix 1: Use Set instead of List for one collection
@OneToMany
private Set<OrderItem> items;  // Set allowed

// Fix 2: Two separate queries with BatchSize
// Query 1: orders with items
// Query 2: orders (same) with payments — Hibernate de-dupes via L1 cache

// Fix 3: Load sequentially
List<Order> orders = orderRepository.findAllWithItems();
orders.forEach(o -> Hibernate.initialize(o.getPayments()));
```
