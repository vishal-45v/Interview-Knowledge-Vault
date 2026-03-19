# Hibernate & JPA Internals — Trap Questions

---

## Trap 1: equals/hashCode on JPA Entities

**Question:** Why can JPA entities have equals/hashCode issues?

**Answer:** JPA entities have a special lifecycle that breaks standard equals/hashCode contracts:
- Transient entity: id = null
- Persistent entity: id = 123
- Are they equal? Should be if they represent the same data.

Problem with id-based equals: A transient entity (id=null) added to a HashSet, then persisted (now has id), will be in the wrong bucket in the set.

Best practice: Use a business key (natural key) for equals/hashCode, NOT the generated ID. Or use UUID ids assigned in Java before persist.

---

## Trap 2: LazyInitializationException After Transaction Ends

**Question:** Why does this throw LazyInitializationException?

```java
@Service
public class OrderService {

    @Transactional
    public Order findOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
        // TX ends here — entity becomes detached
    }
}

// Controller:
Order order = orderService.findOrder(1L);
order.getItems().size();  // LazyInitializationException!
```

**Answer:** The lazy-loaded collection is not initialized before the transaction ends. The entity becomes detached, and accessing a lazy collection on a detached entity throws LazyInitializationException.

Fixes: Initialize in transaction (JOIN FETCH, @EntityGraph), use DTOs to return data outside the transaction, or enable OSIV (Open Session in View) — but OSIV has performance implications.

---

## Trap 3: merge() Doesn't Update — It Copies

**Question:** After calling merge(), why isn't the original entity managed?

**Answer:** merge() returns a NEW managed instance. The object you passed in remains DETACHED. Always use the returned value from merge():

```java
Product detached = getFromCache();
Product managed = entityManager.merge(detached);  // Use this one!
// detached is still not tracked
managed.setPrice(10.99);  // This change will be persisted
```

---

## Trap 4: CascadeType.ALL + orphanRemoval with Bidirectional

**Question:** This deletes data unexpectedly — why?

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items;
```

**Answer:** orphanRemoval = true means any OrderItem removed from the items list will be deleted from the DB, even if the OrderItem object still exists elsewhere. In a bidirectional relationship, if you only clear the list (items.clear()) without removing via order.getItems().remove(item), Hibernate deletes all items.

Be careful with orphanRemoval in entities that participate in multiple relationships.

---

## Trap 5: @Transactional readOnly and Dirty Checking

**Question:** Does @Transactional(readOnly = true) disable dirty checking?

**Answer:** Yes. With readOnly = true, Hibernate disables dirty checking (flush mode is set to MANUAL or NEVER). Even if you modify a managed entity, changes won't be flushed on transaction commit.

This is the main performance benefit of readOnly — especially for bulk queries loading many entities, skipping dirty checking saves significant time.
