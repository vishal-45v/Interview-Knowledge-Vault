# Transactions & Propagation — Scenarios

> 22 scenarios covering @Transactional behavior, propagation levels, isolation, and common pitfalls.

---

## Scenario 1: Understanding REQUIRED Propagation

**Context:** Two services with @Transactional:
```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        paymentService.charge(order);  // REQUIRED by default
    }
}

@Service
public class PaymentService {
    @Transactional  // REQUIRED by default
    public void charge(Order order) {
        paymentRepository.save(order);
    }
}
```

**Question:** How many database transactions are created?

**Answer:**

**One transaction.** `REQUIRED` (the default) means: join the existing transaction if one is active; create a new one if none exists.

1. `placeOrder()` starts Transaction T1
2. `charge()` is called — sees T1 is active → joins T1 (same connection!)
3. If `charge()` throws → T1 is rolled back (both `orderRepository.save()` AND `paymentRepository.save()` roll back)
4. If `placeOrder()` commits → T1 commits both saves

```
T1 begins
├── orderRepository.save(order)      [T1]
└── paymentService.charge(order)
    └── paymentRepository.save(order) [T1 — same transaction!]
T1 commits
```

---

## Scenario 2: REQUIRES_NEW — Guaranteed Separate Transaction

**Context:** Audit logging should persist even if the main transaction rolls back:

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        auditService.logAction("ORDER_PLACED", order.getId());  // REQUIRES_NEW
        // what if this throws?
        paymentService.charge(order);  // might fail
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAction(String action, Long orderId) {
        auditRepository.save(new AuditEntry(action, orderId));
    }
}
```

**Question:** If `paymentService.charge()` throws, does the audit log persist?

**Answer:**

**Yes, the audit log persists.** `REQUIRES_NEW` suspends the caller's transaction and creates an independent one:

```
T1 begins (OrderService.placeOrder)
├── orderRepository.save(order)  [T1]
├── auditService.logAction(...)
│   T1 SUSPENDED
│   T2 begins (REQUIRES_NEW)
│   └── auditRepository.save(...)  [T2]
│   T2 commits  ← audit log committed independently!
│   T1 RESUMED
└── paymentService.charge(order)  ← throws RuntimeException
T1 ROLLS BACK  (orderRepository.save rolled back)
AuditEntry is already committed — not rolled back!
```

**Warning:** `REQUIRES_NEW` uses TWO database connections simultaneously. Under high concurrency, this can cause connection pool exhaustion. Use sparingly.

---

## Scenario 3: The Self-Invocation Trap

**Context:**
```java
@Service
public class OrderService {
    public void placeOrder(Order order) {
        saveOrder(order);     // calls own method
        notifyUser(order);
    }

    @Transactional
    public void saveOrder(Order order) {
        orderRepository.save(order);
    }
}
```

**Problem:** `saveOrder()` is `@Transactional` but the transaction never starts.

**Question:** Why and how to fix?

**Answer:**

Spring's `@Transactional` uses AOP (proxy-based). When you call `this.saveOrder()` within the same class, you bypass the proxy entirely — calling the raw method. The proxy's interceptor never runs, so no transaction is started.

```
Client → OrderServiceProxy.placeOrder()
         → [proxy interceptor checks @Transactional... not annotated]
         → OrderService.placeOrder() (real object)
            → this.saveOrder()  ← bypasses proxy entirely!
            → no transaction created!
```

**Fix 1 — Move to separate bean (best):**
```java
@Service
public class OrderSaver {
    @Transactional
    public void save(Order order) { orderRepository.save(order); }
}

@Service
public class OrderService {
    private final OrderSaver orderSaver;
    public void placeOrder(Order order) {
        orderSaver.save(order);  // calls via proxy → transaction works!
    }
}
```

**Fix 2 — Self-inject (hacky but works):**
```java
@Service
public class OrderService {
    @Autowired @Lazy OrderService self;  // inject the proxy

    public void placeOrder(Order order) {
        self.saveOrder(order);  // through the proxy
    }

    @Transactional
    public void saveOrder(Order order) { ... }
}
```

**Fix 3 — Annotate the calling method:**
```java
@Service
public class OrderService {
    @Transactional  // annotate the outer method
    public void placeOrder(Order order) {
        saveOrder(order);  // same transaction — self-invocation OK when outer has @Transactional
        notifyUser(order);
    }

    public void saveOrder(Order order) { orderRepository.save(order); }
}
```

---

## Scenario 4: Transaction Rollback Rules

**Context:**
```java
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    try {
        paymentService.charge(order);
    } catch (PaymentException e) {
        log.warn("Payment failed: {}", e.getMessage());
        // Does the transaction roll back?
    }
}
```

**Question:** Does the transaction commit or roll back?

**Answer:**

The transaction **commits** (unless `PaymentException` extends `RuntimeException` and bubbles up uncaught).

Spring's default rollback rules:
- **Rolls back** on `RuntimeException` and `Error` (unchecked exceptions that propagate)
- **Does NOT roll back** on checked exceptions (`Exception`, `IOException`, etc.)
- **Does NOT roll back** if exception is caught and swallowed

In this scenario, `PaymentException` is caught → transaction commits → `orderRepository.save()` is committed even though payment failed!

**Fix options:**

```java
// Option 1 — Don't catch the exception (let it propagate):
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    paymentService.charge(order);  // throws RuntimeException → rollback
}

// Option 2 — Rollback explicitly after catching:
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    try {
        paymentService.charge(order);
    } catch (PaymentException e) {
        log.warn("Payment failed", e);
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();  // mark rollback
    }
}

// Option 3 — Specify rollback for checked exceptions:
@Transactional(rollbackFor = {PaymentException.class, IOException.class})
public void processOrder(Order order) { ... }

// Option 4 — Specify NO rollback for certain exceptions:
@Transactional(noRollbackFor = PaymentException.class)
public void processOrder(Order order) { ... }
```

---

## Scenario 5: Isolation Levels in Practice

**Context:** A flight booking system needs to prevent double-booking.

**Question:** Which isolation level should you use?

**Answer:**

```java
// READ_UNCOMMITTED — can read uncommitted changes (dirty reads):
// Not suitable — could see someone else's in-progress booking and think seat is taken

// READ_COMMITTED — can't read uncommitted, but phantom reads possible:
@Transactional(isolation = Isolation.READ_COMMITTED)
// Still vulnerable: two bookings could read "seat available" simultaneously before either commits

// REPEATABLE_READ — consistent reads within transaction:
@Transactional(isolation = Isolation.REPEATABLE_READ)
// Better, but still allows phantom rows in range queries

// SERIALIZABLE — fully serialized execution:
@Transactional(isolation = Isolation.SERIALIZABLE)
// Prevents all anomalies, but highest locking/performance cost

// BEST approach for booking — optimistic locking with @Version:
@Entity
public class Seat {
    @Id Long id;
    String seatNumber;
    boolean booked;
    @Version int version;  // optimistic lock version
}

@Transactional
public void bookSeat(Long seatId, Long userId) {
    Seat seat = seatRepo.findById(seatId).orElseThrow();
    if (seat.isBooked()) throw new SeatAlreadyBookedException();
    seat.setBooked(true);
    seatRepo.save(seat);
    // If another transaction modified this seat between our read and write:
    // @Version check fails → OptimisticLockException → retry
}
```

---

## Scenario 6: Read-Only Transaction Optimization

**Context:** A reporting service reads large amounts of data.

**Question:** What does `readOnly = true` do?

**Answer:**

```java
@Service
public class ReportService {
    @Transactional(readOnly = true)  // optimization hints
    public ReportData generateReport(DateRange range) {
        return reportRepository.findByDateRange(range.getStart(), range.getEnd());
    }
}
```

**What `readOnly = true` does:**
1. **Hibernate flush mode:** Set to `MANUAL` (never) — Hibernate won't check for dirty objects at transaction end (no dirty checking overhead)
2. **Query optimization hint:** JPA/Hibernate may skip creating snapshots for change detection
3. **Database optimization:** Some JDBC drivers/databases can optimize read-only transactions (e.g., MySQL master-slave routing — reads go to replica)
4. **Spring message:** Tells Spring this is a read-only operation — Spring may route to a read replica if configured

**What it does NOT do:**
- Does NOT prevent you from calling `save()` (but changes may not be flushed)
- Does NOT guarantee the transaction is rolled back instead of committed
- Is just a hint — behavior varies by JPA provider

---

## Scenario 7: NESTED Propagation

**Context:** Save order with a save point — partial rollback without affecting outer transaction.

**Answer:**

`NESTED` creates a savepoint within the outer transaction. If the nested transaction rolls back, it rolls back to the savepoint — outer transaction continues.

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);

        try {
            bonusService.applyBonus(order);  // NESTED — optional bonus
        } catch (Exception e) {
            log.warn("Bonus failed, continuing", e);
            // bonusService.applyBonus rolled back to savepoint
            // but order was already saved — outer transaction continues
        }

        notificationService.notify(order);
    }
}

@Service
public class BonusService {
    @Transactional(propagation = Propagation.NESTED)
    public void applyBonus(Order order) {
        bonusRepository.save(new Bonus(order));
        // If this throws: rolls back to savepoint, outer transaction intact
    }
}
```

**NESTED vs REQUIRES_NEW:**
- `NESTED`: Still part of the outer transaction (same connection, savepoint)
- `REQUIRES_NEW`: Completely independent transaction (new connection)
- `NESTED` requires JDBC savepoint support — not all databases support it

---

## Scenario 8: Transaction in Async Methods

**Context:**
```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        asyncEmailService.sendConfirmation(order);  // @Async
    }
}

@Service
public class AsyncEmailService {
    @Async
    @Transactional
    public CompletableFuture<Void> sendConfirmation(Order order) {
        emailLogRepository.save(new EmailLog(order.getId(), "CONFIRMATION_SENT"));
        return CompletableFuture.completedFuture(null);
    }
}
```

**Question:** Does `sendConfirmation()` participate in the caller's transaction?

**Answer:**

**No.** `@Async` methods run in a separate thread. Transactions are thread-bound. The async method starts in a new thread with NO transaction context — it will create its own transaction (REQUIRED = new transaction since none exists in this thread).

So:
- `orderRepository.save()` → runs in the caller's transaction T1
- `emailLogRepository.save()` → runs in a NEW transaction T2 (different thread)
- If T1 rolls back, T2 is already committed (email log stays!)
- If email throwing happens after order saved → email log persists, order may or may not

**Design consideration:** For async operations that need data from the current transaction, ensure the transaction commits BEFORE triggering the async operation, or pass necessary data (not entities) to the async method.

---

## Scenario 9: Transaction Boundaries with @EventListener

**Context:**
```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderPlacedEvent(order));
        // Does the listener participate in this transaction?
    }
}

@Component
public class InventoryUpdater {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        inventoryRepository.decrementStock(event.getOrder());
    }
}
```

**Question:** Is the event listener part of the transaction?

**Answer:**

By default, `@EventListener` fires synchronously within the publisher's transaction. So `inventoryRepository.decrementStock()` IS part of `placeOrder()`'s transaction.

To run AFTER commit (when you need the data to be persisted first):
```java
@Component
public class NotificationSender {
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        // Runs after transaction commits — data is persisted
        emailService.sendOrderConfirmation(event.getOrder());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onOrderFailed(OrderPlacedEvent event) {
        // Runs if transaction rolled back
        alertOpsTeam(event.getOrder());
    }
}
```

---

## Scenario 10: Transaction Propagation — SUPPORTS and NOT_SUPPORTED

**Answer:**

```java
// SUPPORTS — use transaction if one exists, no transaction if not:
@Transactional(propagation = Propagation.SUPPORTS)
public Order findOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
    // Can read outside a transaction (no locking)
    // If called within a transaction, uses it (consistent read)
}

// NOT_SUPPORTED — suspend transaction if one exists, run without:
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void sendEmailNotification(Order order) {
    // Email sending should not be in a DB transaction!
    // Suspends caller's transaction for this method's execution
    emailClient.send(order.getUserEmail(), buildEmailBody(order));
    // After method returns, caller's transaction resumes
}

// NEVER — throw exception if transaction exists:
@Transactional(propagation = Propagation.NEVER)
public void bulkImport(List<Order> orders) {
    // This method intentionally should not run in a transaction
    // (manages its own batch commits)
    throw new IllegalTransactionStateException() if in a transaction
}

// MANDATORY — must have transaction, else throw:
@Transactional(propagation = Propagation.MANDATORY)
public void validateAndSave(Order order) {
    // Must be called from within a transaction — enforces this
    orderRepository.save(order);
}
```

---

## Scenario 11: Long-Running Transactions

**Context:** A service imports 100,000 records in a single transaction. After 30 minutes, it fails with a database timeout.

**Question:** How to fix?

**Answer:**

Long-running transactions hold database locks, consume connection pool resources, and are vulnerable to timeouts and failures.

```java
// Bad — single transaction for huge batch:
@Transactional
public void importOrders(List<Order> orders) {  // 100K orders!
    orders.forEach(orderRepository::save);  // 30+ minute transaction
}

// Fix — batch processing with separate transactions per batch:
@Service
public class OrderImporter {
    @Autowired OrderImporter self;  // self-proxy for @Transactional

    public void importOrders(List<Order> orders) {
        Lists.partition(orders, 100)  // Guava partition
            .forEach(batch -> self.importBatch(batch));
    }

    @Transactional  // new transaction per 100-order batch
    public void importBatch(List<Order> batch) {
        batch.forEach(orderRepository::save);
        // Commits and releases connection every 100 orders
    }
}

// Even better — Spring Batch for large batch jobs
@Bean
public Job importOrdersJob(Step step) {
    return jobBuilder.get("importOrders").start(step).build();
}

@Bean
public Step step(ItemReader<Order> reader, ItemWriter<Order> writer) {
    return stepBuilder.get("importStep")
        .<Order, Order>chunk(100)  // 100 items per transaction chunk
        .reader(reader).writer(writer)
        .build();
}
```

---

## Scenario 12: Optimistic vs Pessimistic Locking

**Context:** Account balance update must be consistent under concurrent requests.

**Answer:**

```java
// Optimistic locking — detect conflict on commit:
@Entity
public class Account {
    @Id Long id;
    BigDecimal balance;
    @Version int version;  // incremented on every update
}

@Transactional
public void transfer(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepo.findById(fromId).orElseThrow();
    Account to = accountRepo.findById(toId).orElseThrow();

    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));

    // If another transaction modified either account between our read and write:
    // @Version mismatch → OptimisticLockException → must retry
}

// Pessimistic locking — lock rows at read time:
@Transactional
public void transferPessimistic(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepo.findByIdWithLock(fromId);  // SELECT FOR UPDATE
    Account to = accountRepo.findByIdWithLock(toId);      // SELECT FOR UPDATE
    // Nobody else can modify these rows until this transaction ends
    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));
}

// Repository:
public interface AccountRepository extends JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Optional<Account> findByIdWithLock(@Param("id") Long id);
}
```

**When to use each:**
- **Optimistic:** Low conflict scenarios, high read throughput, short transactions
- **Pessimistic:** High conflict scenarios, long transactions, critical financial operations

---

## Scenario 13: Transaction Timeout

**Context:** Set a maximum duration for a transaction.

**Answer:**

```java
@Transactional(timeout = 30)  // 30 seconds
public void processLargeOrder(Order order) {
    // If this method takes > 30 seconds:
    // TransactionTimedOutException is thrown (typically before commit)
}

// Programmatic timeout check:
@Transactional
public void processBatch(List<Order> orders) {
    TransactionStatus status = TransactionAspectSupport.currentTransactionStatus();
    for (Order order : orders) {
        if (status.isRollbackOnly()) break;  // transaction marked for rollback
        processOrder(order);
    }
}
```

---

## Scenario 14: Database vs Application-Level Transaction

**Context:** When do you need @Transactional vs handling in the database?

**Answer:**

```java
// Application-level @Transactional:
@Transactional
public void transferFunds(TransferRequest req) {
    accountService.debit(req.fromAccount(), req.amount());
    accountService.credit(req.toAccount(), req.amount());
    // Spring handles BEGIN/COMMIT/ROLLBACK
}

// Database-level stored procedure:
@Repository
public class AccountRepository {
    @PersistenceContext EntityManager em;

    public void transferFunds(TransferRequest req) {
        // Call stored procedure that handles transaction internally
        em.createStoredProcedureQuery("transfer_funds")
          .registerStoredProcedureParameter("from_id", Long.class, ParameterMode.IN)
          .registerStoredProcedureParameter("to_id", Long.class, ParameterMode.IN)
          .registerStoredProcedureParameter("amount", BigDecimal.class, ParameterMode.IN)
          .setParameter("from_id", req.fromAccount())
          .setParameter("to_id", req.toAccount())
          .setParameter("amount", req.amount())
          .execute();
    }
}

// Choose based on:
// Application-level: Business logic, cross-service operations, portability
// Database-level: Performance-critical, data integrity, legacy systems
```

---

## Scenario 15: Checking Transaction Status Programmatically

**Answer:**

```java
@Service
public class OrderService {
    @Transactional
    public void processOrder(Order order) {
        // Check if we're in a transaction:
        boolean inTransaction = TransactionSynchronizationManager.isActualTransactionActive();

        // Get current transaction info:
        String transactionName = TransactionSynchronizationManager.getCurrentTransactionName();

        // Register callback for after commit:
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
            @Override
            public void afterCommit() {
                // Runs after transaction commits — guaranteed data is persisted
                eventBus.publish(new OrderProcessedEvent(order.getId()));
            }

            @Override
            public void afterCompletion(int status) {
                if (status == STATUS_ROLLED_BACK) {
                    compensate(order);
                }
            }
        });

        orderRepository.save(order);
    }
}
```

---

## Scenario 16: @Transactional on Private Methods

**Context:**
```java
@Service
public class OrderService {
    @Transactional  // Does this work?
    private void saveOrderInternal(Order order) {
        orderRepository.save(order);
    }
}
```

**Answer:**

**No, it does NOT work.** Spring AOP proxies can only intercept public methods. `@Transactional` on private methods is silently ignored — no transaction is created.

The same is true for `protected` and package-private methods with the default JDK proxy. (AspectJ compile-time weaving can intercept any method, but Spring Boot uses JDK/CGLIB proxy by default.)

**Symptoms:** No transaction created, changes committed per statement (auto-commit), rollback doesn't work.

**Fix:** Make the method public, or move it to a separate bean.

---

## Scenario 17: Transaction Propagation Diagram

**Answer:**

```
Propagation summary:
┌─────────────────┬────────────────────────────────────────────────────┐
│ Propagation     │ Behavior                                            │
├─────────────────┼────────────────────────────────────────────────────┤
│ REQUIRED        │ Join existing or create new (DEFAULT)               │
│ REQUIRES_NEW    │ Suspend existing, always create new                 │
│ NESTED          │ Create savepoint within existing (or create new)    │
│ SUPPORTS        │ Join existing or run without transaction            │
│ NOT_SUPPORTED   │ Suspend existing, run without transaction           │
│ NEVER           │ Run without transaction; throw if one exists        │
│ MANDATORY       │ Join existing; throw if none exists                 │
└─────────────────┴────────────────────────────────────────────────────┘
```

---

## Scenario 18: JPA and Transaction Interaction

**Context:** Querying data inside and outside a transaction.

**Answer:**

```java
// INSIDE a transaction — Hibernate first-level cache is active:
@Transactional
public void processOrder(Long orderId) {
    Order order1 = orderRepo.findById(orderId).orElseThrow();
    // ... some logic
    Order order2 = orderRepo.findById(orderId).orElseThrow();
    // order1 == order2 (same Java object! — first-level cache hit, no SQL)
    System.out.println(order1 == order2);  // true
}

// OUTSIDE a transaction — each call returns a new object:
public void readOrder(Long orderId) {
    Order order1 = orderRepo.findById(orderId).orElseThrow();
    Order order2 = orderRepo.findById(orderId).orElseThrow();
    System.out.println(order1 == order2);  // false — separate objects, 2 SQL queries
}
```

---

## Scenario 19: Distributed Transactions

**Context:** Updating a database AND publishing a Kafka message atomically.

**Question:** How do you ensure both happen or neither happens?

**Answer:**

True distributed transactions (2PC) are complex and often avoided. Better patterns:

```java
// Option 1 — Transactional Outbox Pattern:
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);

    // Save message to outbox in SAME transaction:
    outboxRepository.save(new OutboxMessage(
        "order-events",
        order.getId().toString(),
        objectMapper.writeValueAsString(new OrderPlacedEvent(order))
    ));
    // Both saved or both rolled back — atomically

    // Separate background job reads outbox and publishes to Kafka:
    // If Kafka publish fails, outbox entry persists → retry
}

// Option 2 — @TransactionalEventListener with AFTER_COMMIT:
@Transactional
public void placeOrder(Order order) {
    orderRepository.save(order);
    eventPublisher.publishEvent(new OrderPlacedEvent(order));
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderPlaced(OrderPlacedEvent event) {
    kafkaTemplate.send("order-events", event);
    // If Kafka fails AFTER DB commit → message is lost
    // Use outbox pattern for true at-least-once delivery
}
```

---

## Scenario 20: Transaction in Spring Batch

**Context:** How does Spring Batch manage transactions?

**Answer:**

```java
@Bean
public Step processingStep(ItemReader<Order> reader,
                            ItemProcessor<Order, ProcessedOrder> processor,
                            ItemWriter<ProcessedOrder> writer) {
    return stepBuilderFactory.get("processOrders")
        .<Order, ProcessedOrder>chunk(100)  // chunk = transaction unit
        .reader(reader)
        .processor(processor)
        .writer(writer)
        .transactionManager(transactionManager)
        .build();
}

// Transaction behavior per chunk:
// 1. Read 100 items (no transaction)
// 2. Process all 100 items (no transaction)
// 3. Begin transaction
// 4. Write 100 processed items
// 5. Commit transaction (or rollback if writer fails)
// 6. Mark 100 items as "committed" in JobRepository
// 7. Repeat for next chunk

// Restart from last committed chunk if job fails
```

---

## Scenario 21: Two-Phase Locking

**Context:** Explain the issue and how Spring transactions relate to database-level locking.

**Answer:**

Database locking is managed by the database engine. Spring `@Transactional` controls WHEN the transaction starts and ends, which determines HOW LONG locks are held.

```java
// Short-lived transaction — locks held for minimal time:
@Transactional
public void updateUserProfile(Long userId, ProfileUpdateRequest request) {
    User user = userRepo.findById(userId).orElseThrow();
    user.setProfile(request);
    // Transaction commits immediately after method returns
    // Database lock on user row held only for this brief time
}

// Long transaction — locks held for extended time (avoid!):
@Transactional
public void reportGeneration() {
    List<Order> orders = orderRepo.findAll();  // shared lock on all orders
    // ... expensive processing for 10 minutes ...
    // All order rows locked for 10 minutes! Other transactions waiting
}
```

---

## Scenario 22: Testing Transactions

**Context:** Write tests that verify transactional behavior.

**Answer:**

```java
@SpringBootTest
@Transactional  // Each test runs in a transaction, rolled back after
class OrderServiceTransactionTest {

    @Autowired OrderService orderService;
    @Autowired OrderRepository orderRepository;

    @Test
    void testOrderSavedAndRolledBackOnPaymentFailure() {
        Order order = new Order(/* ... */);

        // The outer @Transactional rolls back after test
        assertThrows(PaymentException.class, () -> orderService.placeOrder(order));
        // What was saved (if anything) is rolled back by @Transactional on the test
    }

    @Test
    @Commit  // Override @Transactional rollback for this test
    void testOrderPersisted() {
        orderService.placeOrder(order);
        // Not rolled back — persisted to DB (useful for verifying persistence)
    }
}

// Testing REQUIRES_NEW — need to NOT roll back in test:
@Test
void testAuditLogPersistedDespiteFailure() {
    // Use @Sql to set up data, test in non-transactional context
    // Verify audit log exists after main transaction rolled back
}
```
