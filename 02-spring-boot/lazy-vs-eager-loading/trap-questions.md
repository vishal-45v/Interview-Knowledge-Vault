# Lazy vs Eager Loading — Trap Questions

---

## Trap 1: @OneToOne Lazy Loading Doesn't Work

**Question:** You set fetch = LAZY on a @OneToOne relationship, but Hibernate still loads it eagerly. Why?

**Answer:** Hibernate requires nullable @OneToOne associations to be EAGER by default because it needs to know whether the proxy should be null. Lazy loading only works reliably for @OneToOne when:
1. The association is declared on the owning side (has the FK column)
2. The column is NOT NULL (Hibernate can assume it exists)
3. You're using bytecode enhancement (advanced setup)

In most cases, @OneToOne(fetch=LAZY) silently falls back to EAGER. Use @ManyToOne instead where possible, or redesign to use @OneToMany.

---

## Trap 2: JOIN FETCH with Pagination

**Question:** Why does Hibernate log a warning when using JOIN FETCH with pagination?

```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items")
Page<Order> findAllWithItems(Pageable pageable);  // Warning logged!
```

**Answer:** "HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory!"

Hibernate cannot apply LIMIT/OFFSET at the SQL level when JOIN FETCH creates duplicate rows. Instead, it loads ALL matching rows into memory and then paginates in Java — which can cause OutOfMemoryError on large datasets.

Fix: Use a two-query approach — first paginate orders, then load items for those orders with a separate query.

---

## Trap 3: Eager Collections Cause Cartesian Product

**Question:** Why does this query return duplicate orders?

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.tags")
List<Order> findAll();
// Returns order #1 three times if it has 3 items and 2 tags → 3×2=6 rows
```

**Answer:** Joining two collections creates a Cartesian product. Order with 3 items and 2 tags produces 6 result rows in SQL (3 × 2). Without DISTINCT, Hibernate returns 6 duplicate Order objects.

Fix: Add DISTINCT, or use @BatchSize for one of the collections, or use separate queries.

---

## Trap 4: @Transactional Test and Lazy Loading

**Question:** Lazy loading works in your test, but fails in production. Why?

**Answer:** Tests annotated with @Transactional run the entire test method inside a transaction. This means lazy loading works throughout the test. In production, the transaction ends after the service method returns, making lazy loading fail in the controller/serialization layer.

Fix: Design service methods to return DTOs (not entities) or ensure all needed associations are loaded inside the service transaction.
