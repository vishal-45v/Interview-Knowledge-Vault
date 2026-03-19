# Spring Transactions — Trap Questions

> Subtle transaction questions that expose gaps in understanding of Spring proxy mechanics, rollback rules, and JPA behavior.

---

## Trap 1: @Transactional on Interface vs Implementation

**Question:** Should you put `@Transactional` on the interface or the implementation class?

```java
// Option A: Interface
public interface UserService {
    @Transactional
    void createUser(User user);
}

// Option B: Implementation
@Service
public class UserServiceImpl implements UserService {
    @Transactional
    public void createUser(User user) { ... }
}
```

**Answer:** Always put `@Transactional` on the **implementation class**, not the interface.

When using CGLIB proxies (Spring Boot default), the proxy extends the implementation class directly. Annotations on interfaces are only picked up by JDK proxies (interface-based proxying). If you switch from JDK to CGLIB proxy, annotations on interfaces are silently ignored.

Additionally, `@Transactional` is part of the implementation concern (how the data is stored), not the contract (what the service does). It doesn't belong on the interface.

---

## Trap 2: Exception Caught Inside @Transactional Method

**Question:** Will this transaction be rolled back?

```java
@Transactional
public void processPayment(Payment payment) {
    try {
        chargeCard(payment);  // throws RuntimeException
    } catch (RuntimeException e) {
        log.error("Charge failed", e);
        // Don't re-throw — just log and continue
    }
    updatePaymentStatus(payment, FAILED);
}
```

**Answer:** No, it will **commit** — and `FAILED` status will be saved.

Spring only marks the transaction for rollback when an exception propagates out of the `@Transactional` method boundary. If you catch the exception internally, Spring's `TransactionInterceptor` never sees it and commits normally.

**Is this a problem?**

It depends. If you want to record the failure (`FAILED` status), this might be intentional. If you wanted to rollback the entire thing, you need to either:
1. Re-throw the exception
2. Call `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`

---

## Trap 3: REQUIRES_NEW and Connection Pool Exhaustion

**Question:** What happens with this code under load?

```java
@Transactional  // Opens Connection 1
public void processOrders(List<Order> orders) {
    for (Order order : orders) {
        orderItemService.saveItems(order);  // Opens Connection 2 (REQUIRES_NEW)
    }
}

@Transactional(propagation = REQUIRES_NEW)  // Needs a second connection
public void saveItems(Order order) {
    itemRepository.saveAll(order.getItems());
}
```

**Answer:** Connection pool exhaustion. `REQUIRES_NEW` **suspends** the current transaction and opens a **new DB connection** for the nested transaction. With 100 concurrent requests and `processOrders()` holding Connection 1 while calling `saveItems()` which needs Connection 2, you double the connections per thread.

If your pool has 10 connections and 10 threads each hold 1 connection and wait for a second:
- All 10 connections taken
- All 10 threads waiting for a second connection
- **Deadlock** — nobody gets a connection, all threads block forever

Fix: Avoid `REQUIRES_NEW` inside loops. Use batching or move to a different service boundary.

---

## Trap 4: The Checked Exception Rollback Surprise

**Question:** This method throws a checked exception. Will the transaction roll back?

```java
@Transactional
public void importFromCsv(MultipartFile file) throws IOException {
    List<User> users = parseCsv(file);
    userRepository.saveAll(users);

    if (users.isEmpty()) {
        throw new IOException("CSV file is empty");  // checked exception
    }
}
```

**Answer:** **No rollback** — the records will be saved even though an exception is thrown. Spring's default rollback rule only covers `RuntimeException` and `Error`. Checked exceptions cause a **commit** by default.

This surprises developers who expect "if an exception is thrown, the transaction rolls back." That's only true for unchecked exceptions.

**Fix:**
```java
@Transactional(rollbackFor = IOException.class)
public void importFromCsv(MultipartFile file) throws IOException { ... }
```

---

## Trap 5: @Transactional readOnly = true and Writes

**Question:** Will this throw an exception?

```java
@Transactional(readOnly = true)
public User createUser(User user) {
    return userRepository.save(user);  // INSERT statement
}
```

**Answer:** Usually **no exception** — it depends on the database and driver.

`readOnly = true` is a **hint**, not an enforcement. Hibernate uses it to skip dirty checking and snapshot caching. The database may or may not enforce it. PostgreSQL and MySQL typically allow writes in read-only transactions at the JDBC level.

However, if your application uses **read replica routing** (AbstractRoutingDataSource), this code would route to the read replica, and writes to a read replica **will fail** with an error.

**Lesson:** `readOnly = true` on a method that writes is a logical error even if it doesn't always throw.

---

## Trap 6: @Transactional and Thread Boundaries

**Question:** Does a transaction span multiple threads?

```java
@Transactional
public void processOrders() {
    List<Order> orders = orderRepository.findAll();

    orders.parallelStream()  // Spawns multiple threads!
          .forEach(order -> {
              // Are these in the same transaction?
              orderService.process(order);
          });
}
```

**Answer:** **No.** Spring transactions are bound to the current thread via `ThreadLocal`. The parallel stream creates worker threads that don't inherit the transaction context. Each thread either:
1. Has no transaction (if `process()` is not `@Transactional`)
2. Starts its own transaction (if `process()` is `@Transactional(REQUIRED)`)

None of the worker threads participate in the outer transaction.

**Practical rule:** Never use `parallelStream()` or create `Thread`/`CompletableFuture` inside a `@Transactional` method and expect them to share the transaction.

---

## Trap 7: Entity State After Transaction Rollback

**Question:** After the transaction rolls back, what state is `order` in?

```java
@Test
@Transactional
void testOrderCreation() {
    Order order = new Order();
    orderRepository.save(order);  // INSERT (within test TX)

    // Simulate external condition
    throw new RuntimeException("Test failure");
    // TX rolls back — order row deleted from DB
}

// After the test, is order.getId() still populated?
```

**Answer:** Yes, `order.getId()` is still populated (set by Hibernate during the `save()` call). The in-memory Java object is not affected by the transaction rollback — only the database state is reverted. The `id` field remains set with the value that was generated, even though the corresponding row no longer exists in the database.

This can cause confusion: you have an object with an ID that doesn't exist in the DB.

---

## Trap 8: NESTED vs REQUIRES_NEW — Wrong Choice

**Question:** You want an audit log that always saves even if the main operation fails. Which propagation do you use?

```java
@Transactional
public void deleteProduct(Long productId) {
    productRepository.deleteById(productId);
    auditService.log("Deleted product " + productId);
}

@Transactional(propagation = ???)
public void log(String message) {
    auditRepository.save(new AuditLog(message));
}
```

**Answer:** Use `REQUIRES_NEW`, **not** `NESTED`.

- `NESTED` uses a savepoint within the outer transaction. If the outer transaction rolls back (even after the nested commits), the **entire outer TX rolls back**, including the savepoint. The audit log is lost.
- `REQUIRES_NEW` creates a completely independent transaction. The audit log commits regardless of what happens to the outer transaction.

```java
@Transactional(propagation = REQUIRES_NEW)  // ← Independent TX
public void log(String message) {
    auditRepository.save(new AuditLog(message));
    // Commits even if outer TX rolls back
}
```

---

## Trap 9: @Transactional on @Scheduled Method

**Question:** Will this work correctly?

```java
@Component
public class CleanupJob {

    @Scheduled(fixedDelay = 60000)
    @Transactional
    public void cleanupOldRecords() {
        recordRepository.deleteOlderThan(LocalDate.now().minusDays(30));
    }
}
```

**Answer:** Yes, this works. `@Scheduled` invokes the method on a Spring-managed bean, and since `CleanupJob` is proxied by CGLIB, the `@Transactional` annotation is respected.

However, one subtle issue: `@Scheduled` and `@Transactional` both work through Spring AOP proxies. The Spring scheduler calls the proxy, which intercepts `@Transactional`, which starts a transaction, then delegates to the real method.

**Caution:** If `CleanupJob` is registered as a plain bean without Spring managing it (e.g., in a `@Configuration` without component scanning), `@Transactional` won't work.

---

## Trap 10: Long-Running Transaction and Lock Contention

**Question:** Your code performs an external API call inside a transaction. What's the problem?

```java
@Transactional
public Order placeOrder(OrderRequest request) {
    Order order = orderRepository.save(new Order(request));  // Locks the row

    // External API call — takes 2-5 seconds!
    PaymentResult result = paymentGateway.charge(request.getPaymentDetails());

    order.setPaymentStatus(result.getStatus());
    return orderRepository.save(order);  // Releases lock on commit
}
```

**Answer:** The database row is locked for the entire duration including the external API call (2-5 seconds). Other transactions trying to read or write this order will **block** or **timeout**.

At scale, this means high thread contention, connection pool exhaustion, and cascading failures.

**Fix:** Minimize transaction scope around only the DB operations:

```java
@Service
public class OrderService {

    // NO @Transactional here
    public Order placeOrder(OrderRequest request) {
        Order order = createOrderTransactionally(request);  // Short TX 1
        PaymentResult result = paymentGateway.charge(request.getPaymentDetails());  // Outside TX
        return updatePaymentStatusTransactionally(order.getId(), result);  // Short TX 2
    }

    @Transactional
    private Order createOrderTransactionally(OrderRequest request) {
        return orderRepository.save(new Order(request));
    }

    @Transactional
    private Order updatePaymentStatusTransactionally(Long orderId, PaymentResult result) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.setPaymentStatus(result.getStatus());
        return orderRepository.save(order);
    }
}
```
