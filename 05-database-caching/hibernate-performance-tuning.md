# Hibernate Performance Tuning

---

## Identifying Performance Issues

```yaml
# Enable SQL logging (dev only — verbose!)
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE  # Log parameter values
spring:
  jpa:
    properties:
      hibernate.generate_statistics: true
      hibernate.session.events.log.LOG_QUERIES_SLOWER_THAN_MS: 100
```

---

## Key Hibernate Performance Settings

```yaml
spring:
  jpa:
    properties:
      # Batch inserts/updates (critical for bulk operations)
      hibernate.jdbc.batch_size: 100
      hibernate.order_inserts: true
      hibernate.order_updates: true
      hibernate.batch_versioned_data: true
      
      # Second-level cache
      hibernate.cache.use_second_level_cache: true
      hibernate.cache.use_query_cache: true
      
      # Connection pool (HikariCP)
      hibernate.connection.provider_class: com.zaxxer.hikari.hibernate.HikariConnectionProvider
```

---

## Batch Insert Performance

```java
// Without batch: 10,000 individual INSERTs → slow
for (Product p : products) {
    productRepository.save(p);
}

// With batch (flush+clear pattern):
@Transactional
public void bulkInsert(List<Product> products) {
    for (int i = 0; i < products.size(); i++) {
        entityManager.persist(products.get(i));
        if (i % 100 == 0 && i > 0) {
            entityManager.flush();   // Send batch to DB
            entityManager.clear();  // Clear L1 cache to prevent OOM
        }
    }
    entityManager.flush();
}
// Result: 10,000 records → 100 batched INSERT statements
```

---

## Read-Only Transactions

```java
// Hibernate disables dirty checking for readOnly=true transactions
// Saves: snapshot comparison on every entity during flush
// Significant perf improvement for read-heavy endpoints

@Transactional(readOnly = true)
public Page<ProductDTO> searchProducts(ProductFilter filter, Pageable pageable) {
    return productRepository.search(filter, pageable)
        .map(productMapper::toDTO);
}
```

---

## @QueryHints for Advanced Optimization

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Hint: Use read replica for this query
    @QueryHints(value = {
        @QueryHint(name = HINT_PASS_DISTINCT_THROUGH, value = "false"),
        @QueryHint(name = "org.hibernate.readOnly", value = "true"),
        @QueryHint(name = "jakarta.persistence.query.timeout", value = "5000") // 5s timeout
    })
    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
    List<Order> findByStatusWithItems(@Param("status") OrderStatus status);
}
```
