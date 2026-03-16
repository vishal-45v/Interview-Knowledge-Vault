# Spring Transactions — Structured Answers

> Deep-dive reference answers for the most commonly asked @Transactional interview questions.

---

## Q1: What Does @Transactional Do Internally?

Spring wraps the annotated class in a CGLIB proxy. When a method is called through the proxy, Spring's `TransactionInterceptor` handles the transaction lifecycle:

```
Client ──→ [TransactionInterceptor Proxy] ──→ [Real Bean]

TransactionInterceptor.invoke():
  1. TransactionAttributeSource.getTransactionAttribute()  // read @Transactional metadata
  2. PlatformTransactionManager.getTransaction()           // BEGIN or join
  3. ──────────────────────────────────────────────────────
     |   Real method executes
     |   DB operations run on the active connection
  ──────────────────────────────────────────────────────────
  4. If exception thrown:
     → rollbackOn check → rollback or commit
  5. If no exception:
     → PlatformTransactionManager.commit()
```

### Key Configuration Parameters

```java
@Transactional(
    propagation = Propagation.REQUIRED,      // default
    isolation = Isolation.DEFAULT,            // default (DB default)
    readOnly = false,                         // default
    timeout = -1,                             // default (no timeout)
    rollbackFor = RuntimeException.class,     // default
    noRollbackFor = {},                       // default
    transactionManager = "transactionManager" // default
)
```

---

## Q2: All Propagation Types Explained

```
REQUIRED (default):
  Caller has TX:  ┌────TX────┐
                  │  Callee  │  ← joins existing TX
                  └──────────┘
  Caller no TX:   ┌────TX────┐
                  │  Callee  │  ← creates new TX
                  └──────────┘

REQUIRES_NEW:
  Caller has TX:  ┌────TX1───┐      ┌────TX2───┐
                  │  Caller  │ ───► │  Callee  │  ← new TX, TX1 suspended
                  └──────────┘      └──────────┘
  Callee TX commits/rolls back independently of TX1

NESTED:
  Caller has TX:  ┌────TX────────────────────┐
                  │  Caller   [SAVEPOINT]     │
                  │           ┌────────────┐  │
                  │           │   Callee   │  │  ← nested in outer TX
                  │           └────────────┘  │
                  │  (callee rollback → back   │
                  │   to savepoint, outer OK)  │
                  └──────────────────────────┘

SUPPORTS:
  Caller has TX:  Callee joins it
  Caller no TX:   Callee runs without TX

NOT_SUPPORTED:
  Always runs without TX (suspends caller's TX if present)

MANDATORY:
  Must have caller TX — throws IllegalTransactionStateException if none

NEVER:
  Must NOT have caller TX — throws if caller has TX
```

---

## Q3: The Self-Invocation Trap — 3 Solutions

### Problem

```java
@Service
public class NotificationService {

    @Transactional
    public void sendAll(List<User> users) {
        for (User user : users) {
            sendNotification(user);  // self-invocation — bypasses proxy!
        }
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void sendNotification(User user) {
        // Intended: each notification in its own TX
        // Actual: runs in same TX as sendAll() (or no TX if sendAll() had none)
        notificationLog.save(new NotificationLog(user));
    }
}
```

### Solution 1: Inject Self

```java
@Service
public class NotificationService {

    @Autowired
    private NotificationService self;  // Spring injects the proxy

    @Transactional
    public void sendAll(List<User> users) {
        for (User user : users) {
            self.sendNotification(user);  // goes through proxy ✓
        }
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void sendNotification(User user) {
        notificationLog.save(new NotificationLog(user));
    }
}
```

### Solution 2: Extract to Separate Service

```java
@Service
public class NotificationService {

    @Autowired
    private NotificationSender sender;  // separate class

    @Transactional
    public void sendAll(List<User> users) {
        for (User user : users) {
            sender.sendNotification(user);  // external call → goes through proxy ✓
        }
    }
}

@Service
public class NotificationSender {

    @Transactional(propagation = REQUIRES_NEW)
    public void sendNotification(User user) {
        notificationLog.save(new NotificationLog(user));
    }
}
```

### Solution 3: AopContext.currentProxy()

```java
@Service
public class NotificationService {

    @Transactional
    public void sendAll(List<User> users) {
        NotificationService proxy = (NotificationService) AopContext.currentProxy();
        for (User user : users) {
            proxy.sendNotification(user);  // goes through proxy ✓
        }
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void sendNotification(User user) {
        notificationLog.save(new NotificationLog(user));
    }
}

// Requires: @EnableAspectJAutoProxy(exposeProxy = true)
```

---

## Q4: Rollback Rules — What Triggers Rollback?

### Default Behavior

| Exception Type | Default Behavior |
|---------------|-----------------|
| `RuntimeException` (unchecked) | Rollback |
| `Error` | Rollback |
| `Exception` (checked) | **NO rollback** (commit!) |

```java
@Service
public class PaymentService {

    @Transactional
    public void processPayment(Payment payment) throws IOException {
        // If IOException thrown → COMMITS (checked exception!)
        // If RuntimeException thrown → ROLLS BACK
        // This is COUNTER-INTUITIVE for most developers
    }
}
```

### Customizing Rollback

```java
// Rollback on specific checked exception
@Transactional(rollbackFor = {IOException.class, SQLException.class})
public void processPayment(Payment payment) throws IOException { ... }

// Rollback on any Exception
@Transactional(rollbackFor = Exception.class)
public void processPayment(Payment payment) throws IOException { ... }

// Don't rollback on specific runtime exception
@Transactional(noRollbackFor = OptimisticLockException.class)
public void processPayment(Payment payment) {
    // OptimisticLockException won't cause rollback — handle it manually
}
```

### Programmatic Rollback

```java
@Transactional
public void processPayment(Payment payment) {
    try {
        chargeCard(payment);
    } catch (Exception e) {
        // Mark for rollback without throwing
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        log.error("Payment failed, rolling back", e);
    }
}
```

---

## Q5: Isolation Levels and Concurrency Problems

```
Isolation Level     | Dirty Read | Non-Repeatable Read | Phantom Read
--------------------|------------|---------------------|-------------
READ_UNCOMMITTED    |     ✓      |         ✓           |      ✓
READ_COMMITTED      |     ✗      |         ✓           |      ✓
REPEATABLE_READ     |     ✗      |         ✗           |      ✓
SERIALIZABLE        |     ✗      |         ✗           |      ✗

Default for most DBs: READ_COMMITTED (PostgreSQL, Oracle, SQL Server)
MySQL InnoDB default: REPEATABLE_READ
```

### Concurrency Problems Explained

```java
// DIRTY READ: Read uncommitted data from another transaction
// TX1: UPDATE accounts SET balance = balance - 100 WHERE id = 1
// TX2: SELECT balance FROM accounts WHERE id = 1 → sees -100 before TX1 commits
// TX1: ROLLBACK → but TX2 already read the dirty value!

// NON-REPEATABLE READ: Same query returns different data
// TX1: SELECT price FROM products WHERE id = 5 → price = 100
// TX2: UPDATE products SET price = 200 WHERE id = 5; COMMIT
// TX1: SELECT price FROM products WHERE id = 5 → price = 200 ← different!

// PHANTOM READ: Query returns different rows
// TX1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING' → 10
// TX2: INSERT INTO orders (status) VALUES ('PENDING'); COMMIT
// TX1: SELECT COUNT(*) FROM orders WHERE status = 'PENDING' → 11 ← phantom!
```

### Choosing Isolation Level

```java
// Flight seat booking — prevent double booking
@Transactional(isolation = Isolation.SERIALIZABLE)
public Booking bookSeat(Flight flight, int seatNumber, User user) {
    // Ensures no phantom reads — only one booking per seat
    if (seatBookingRepository.existsBySeatAndFlight(seatNumber, flight)) {
        throw new SeatAlreadyBookedException();
    }
    return seatBookingRepository.save(new Booking(flight, seatNumber, user));
}

// Read-heavy reporting — dirty reads acceptable for performance
@Transactional(isolation = Isolation.READ_UNCOMMITTED, readOnly = true)
public DashboardStats getDashboardStats() {
    // Approximate counts are fine for dashboards
    return statsRepository.getStats();
}
```

---

## Q6: readOnly = true — What Does It Actually Do?

`@Transactional(readOnly = true)` is a hint to the framework and the database. It does NOT prevent writes — it hints:

**Hibernate optimizations:**
- Skips dirty checking (no need to track entity changes)
- Skips snapshot caching (saves memory)
- Can use read replicas if routing is configured

**Database optimizations:**
- MySQL/PostgreSQL may optimize the query plan
- Lock acquisition may be reduced

**Routing to read replica:**

```java
// AbstractRoutingDataSource can route based on transaction type
public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        boolean isReadOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
        return isReadOnly ? "replica" : "primary";
    }
}

// Usage
@Transactional(readOnly = true)
public List<Product> getProducts() {
    // Routed to READ REPLICA ✓
    return productRepository.findAll();
}
```

---

## Q7: @Transactional with @Async — Common Mistake

```java
@Service
public class ReportService {

    @Async
    @Transactional  // PROBLEM: different threads = different transactions
    public void generateReport(Long reportId) {
        // Runs in async thread pool thread
        // Original transaction (from caller) is in a different thread
        // This method starts its own new transaction — which is usually correct
        // BUT the caller's transaction is NOT visible to this method
        Report report = reportRepository.findById(reportId).orElseThrow();
        // If caller saved the report in same TX that hasn't committed yet,
        // this @Async method may not see it!
    }
}
```

**Fix:** Use `@TransactionalEventListener(phase = AFTER_COMMIT)` to ensure data is committed before async processing:

```java
@Service
public class ReportService {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Transactional
    public void requestReport(Long reportId) {
        reportRepository.save(new Report(reportId, PENDING));
        // Publish event — actual processing happens AFTER this TX commits
        eventPublisher.publishEvent(new ReportRequestedEvent(reportId));
    }
}

@Component
public class ReportProcessor {

    @TransactionalEventListener(phase = AFTER_COMMIT)
    @Async  // Runs in async thread pool AFTER commit
    public void processReport(ReportRequestedEvent event) {
        // Report is now visible in DB (TX committed) ✓
        generateReport(event.getReportId());
    }
}
```

---

## Q8: Optimistic vs Pessimistic Locking

### Optimistic Locking (@Version)

```java
@Entity
public class Product {

    @Id @GeneratedValue
    private Long id;

    private int stockQuantity;

    @Version  // Hibernate manages this
    private Long version;
}

@Transactional
public void decreaseStock(Long productId, int quantity) {
    Product product = productRepository.findById(productId).orElseThrow();

    if (product.getStockQuantity() < quantity) {
        throw new InsufficientStockException();
    }

    product.setStockQuantity(product.getStockQuantity() - quantity);
    // Hibernate generates:
    // UPDATE products SET stock_quantity = ?, version = version + 1
    // WHERE id = ? AND version = ?  ← if version changed, throws OptimisticLockException
}
```

### Pessimistic Locking (@Lock)

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)  // SELECT ... FOR UPDATE
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}

@Transactional
public void decreaseStock(Long productId, int quantity) {
    Product product = productRepository.findByIdForUpdate(productId).orElseThrow();
    // Row is LOCKED until this transaction ends
    // Other threads wait for the lock
    product.setStockQuantity(product.getStockQuantity() - quantity);
}
```

### When to Use Each

| Scenario | Locking Type |
|----------|-------------|
| Low contention, mostly reads | Optimistic |
| High contention, frequent conflicts | Pessimistic |
| Inventory management, seat booking | Pessimistic |
| User profile updates | Optimistic |
| Financial transactions | Pessimistic |
| Blog post editing | Optimistic |
