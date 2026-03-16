# N+1 Problem — Trap Questions

---

## Trap 1: EAGER Loading Doesn't Solve N+1 for Collections

**Question:** I changed @OneToMany to EAGER. Why do I still have performance issues?

**Answer:** EAGER on @OneToMany just makes the N+1 happen invisibly at load time instead of when you access the collection. Hibernate still executes N queries — just automatically when the parent entity is loaded. This is often worse because it always loads data even when you don't need it.

The correct fix is JOIN FETCH or @EntityGraph, not changing to EAGER.

---

## Trap 2: Spring Data findAll() with @EntityGraph Uses LEFT JOIN

**Question:** Your @EntityGraph findAll() returns fewer orders than expected. Why?

**Answer:** @EntityGraph uses LEFT OUTER JOIN by default. If you use FETCH with an INNER JOIN via JOIN FETCH in JPQL, orders without items are excluded. @EntityGraph should return all orders (with or without items).

Check if your JPQL uses JOIN vs LEFT JOIN FETCH — the difference matters for entities that might have empty collections.

---

## Trap 3: @BatchSize on @ManyToOne vs @OneToMany

**Question:** @BatchSize works for @OneToMany but does it work for @ManyToOne?

**Answer:** Yes, but the batching behavior is different. For @ManyToOne, Hibernate batches multiple entities into a single IN query when loading the same type of associated entity. For example, loading 100 books each with the same Author type — @BatchSize(50) on author produces 2 queries instead of 100.

---

## Trap 4: COUNT Query and JOIN FETCH

**Question:** Using Spring Data pagination with a @Query that has JOIN FETCH — why is the count query wrong?

**Answer:** Spring Data Pagination requires a separate count query. When your main query has JOIN FETCH, the auto-derived count query joins unnecessarily. Provide an explicit countQuery:

```java
@Query(value = "SELECT o FROM Order o JOIN FETCH o.items",
       countQuery = "SELECT COUNT(o) FROM Order o")
Page<Order> findAllWithItems(Pageable pageable);
```

Without countQuery, the count includes duplicate rows from the JOIN, returning wrong total counts.
