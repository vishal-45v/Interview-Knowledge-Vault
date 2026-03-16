# Spring Transactions — Architecture Notes

> Internal architecture of Spring transaction management, PlatformTransactionManager, and JPA integration.

---

## Transaction Management Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                  Spring Transaction Infrastructure                │
│                                                                  │
│  @Transactional                                                  │
│       │                                                          │
│       ▼                                                          │
│  TransactionInterceptor (AOP Advice)                             │
│       │                                                          │
│       ▼                                                          │
│  TransactionAttributeSource                                      │
│  (reads @Transactional metadata: propagation, isolation, etc.)   │
│       │                                                          │
│       ▼                                                          │
│  PlatformTransactionManager                                      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  DataSourceTransactionManager  ← JDBC / JdbcTemplate       │  │
│  │  JpaTransactionManager         ← JPA / Hibernate           │  │
│  │  JtaTransactionManager         ← Distributed (XA)          │  │
│  │  ReactiveTransactionManager    ← WebFlux / R2DBC           │  │
│  └────────────────────────────────────────────────────────────┘  │
│       │                                                          │
│       ▼                                                          │
│  TransactionSynchronizationManager                               │
│  (ThreadLocal storage for current TX resources)                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## TransactionSynchronizationManager — ThreadLocal Storage

Spring binds transaction resources (connections, sessions) to the current thread:

```java
// Spring stores these as ThreadLocal variables:
TransactionSynchronizationManager.bindResource(dataSource, connectionHolder);
// Current connection is stored per-thread

// This is how JPA EntityManager and JDBC Connection share the same transaction:
// 1. TransactionInterceptor opens TX → gets Connection from DataSource
// 2. Stores Connection in ThreadLocal (bound to DataSource key)
// 3. JdbcTemplate calls DataSourceUtils.getConnection(dataSource)
//    → Gets SAME Connection from ThreadLocal
// 4. JPA EntityManager also gets same Connection
// → Both use the same DB connection → same transaction ✓
```

```
Thread 1:
  ThreadLocal[dataSource] → ConnectionHolder { connection: conn1, txActive: true }

Thread 2:
  ThreadLocal[dataSource] → ConnectionHolder { connection: conn2, txActive: true }

Each thread has its own independent transaction ✓
```

---

## REQUIRED Propagation — Join or Create

```
Case 1: Caller has active transaction
─────────────────────────────────────
                ┌──────────────── TX ──────────────────┐
                │                                       │
  ServiceA      │  serviceA.methodA()                  │
  @Transactional│     │                                │
  (REQUIRED)    │     └──► serviceB.methodB()          │
                │          @Transactional(REQUIRED)     │
                │          ← Joins SAME transaction     │
                │                                       │
                │  If methodB() throws → whole TX fails │
                └───────────────────────────────────────┘

Case 2: No active transaction
─────────────────────────────
  serviceA.methodA() [no @Transactional]
       │
       └──► serviceB.methodB()
            @Transactional(REQUIRED)
            ← Creates NEW transaction
            ┌──── TX ────┐
            │ methodB()  │
            └────────────┘
```

---

## REQUIRES_NEW — Transaction Suspension

```
                ┌──────────── TX1 (suspended) ──────────────┐
                │                                            │
  ServiceA.methodA() ──┐                                    │
  @Transactional       │ calls serviceB.methodB()            │
  (REQUIRED)           │                                    │
                        └──► ┌──── TX2 ────────────────┐    │
                             │  serviceB.methodB()       │   │
                             │  @Transactional           │   │
                             │  (REQUIRES_NEW)           │   │
                             │                           │   │
                             │  TX2 commits or rolls back│   │
                             │  independently of TX1     │   │
                             └───────────────────────────┘   │
                │                                            │
                │  TX1 resumes                               │
                └────────────────────────────────────────────┘

Key: TX1 is SUSPENDED during TX2 execution.
TX2 uses a DIFFERENT database connection.
TX2 result does NOT affect TX1.
```

---

## NESTED — Savepoints

```
NESTED propagation uses JDBC Savepoints (if DataSource supports it):

                ┌──────────── TX1 ───────────────────────────┐
                │                                             │
  ServiceA.doWork()                                          │
  @Transactional(REQUIRED)                                   │
       │                                                     │
       │  INSERT order_header                                │
       │                                                     │
       │  ─── SAVEPOINT sp1 ──────────────────────┐         │
       │  │                                        │         │
       │  │  serviceB.saveOrderItems()             │         │
       │  │  @Transactional(NESTED)                │         │
       │  │                                        │         │
       │  │  INSERT order_items (batch)            │         │
       │  │                                        │         │
       │  │  [Exception thrown]                    │         │
       │  │  ↓ ROLLBACK TO sp1 (NOT full TX)       │         │
       │  └────────────────────────────────────────┘         │
       │                                                     │
       │  order_header INSERT still intact ✓                 │
       │  order_items rolled back to savepoint ✓             │
       │  Can retry or take alternate path                   │
       │                                                     │
       └─────────────────────────────────────────────────────┘

REQUIRES_NEW vs NESTED:
- REQUIRES_NEW: Completely independent TX (different connection)
- NESTED: Same TX, partial rollback to savepoint (same connection)
```

---

## JPA and Transaction Lifecycle

```
@Transactional method begins
         │
         ▼
  EntityManager created and bound to thread
  (or existing EM from TX context reused)
         │
         ▼
  ┌──────────────────────────────────────────┐
  │  First-Level Cache (Persistence Context) │
  │                                          │
  │  entityManager.find(User.class, 1L)      │
  │  → SQL: SELECT * FROM users WHERE id=1   │
  │  → Entity stored in L1 cache             │
  │                                          │
  │  entityManager.find(User.class, 1L)      │
  │  → No SQL! Returns cached entity ✓       │
  │                                          │
  │  entity.setName("New Name")              │
  │  → Change tracked by dirty checking      │
  └──────────────────────────────────────────┘
         │
         ▼
  @Transactional method ends
         │
         ▼
  EntityManager.flush()
  → Hibernate compares entity state to snapshot
  → Generates UPDATE for changed fields
  → Sends SQL to DB (still in transaction)
         │
         ▼
  TransactionManager.commit()
  → DB commits
         │
         ▼
  EntityManager.close()
  → L1 cache cleared
  → Entities become DETACHED
```

---

## Outbox Pattern for Distributed Transactions

```
Problem: How to atomically write to DB and publish to message queue?

NAIVE (WRONG):
  @Transactional
  void placeOrder(Order order) {
      orderRepository.save(order);          // TX1: DB write
      messageQueue.publish(orderCreated);   // TX2: Queue publish — different TX!
  }
  // If DB commits but queue publish fails → inconsistency!
  // If queue publish succeeds but DB commits → inconsistency!

OUTBOX PATTERN (CORRECT):
  @Transactional
  void placeOrder(Order order) {
      orderRepository.save(order);          // ─┐
      outboxRepository.save(new OutboxEvent  │  │ Same transaction!
          ("ORDER_CREATED", order.toJson())); // ─┘
  }

  // Separate process (Outbox Relay):
  @Scheduled(fixedDelay = 1000)
  void relayOutboxEvents() {
      List<OutboxEvent> events = outboxRepository.findUnpublished();
      for (OutboxEvent event : events) {
          messageQueue.publish(event);       // Retry if fails
          event.markPublished();
          outboxRepository.save(event);
      }
  }

Database:
┌──────────────────┐    ┌──────────────────────┐
│    orders        │    │    outbox_events       │
│ id, status, ...  │    │ id, type, payload,     │
│                  │    │ published, created_at   │
└──────────────────┘    └──────────────────────┘
         └──────── Both written in same TX ───────┘
```

---

## Transaction Timeout

```java
// Transaction timeout:
@Transactional(timeout = 30)  // 30 seconds
public void processLargeDataSet(List<Record> records) {
    // If this takes more than 30 seconds → TransactionTimedOutException
    for (Record record : records) {
        processRecord(record);
    }
}

// Timeout behavior:
// - When timeout expires, transaction is marked as rollback-only
// - Next DB operation throws TransactionTimedOutException
// - Transaction is rolled back

// Note: timeout starts when the transaction begins (when proxy intercepts)
// NOT when the first DB operation occurs
```

---

## Testing Transactions

```
Test Transaction Strategies:

@Transactional on test class:
  ┌─────────────────────────────────┐
  │  @Test                          │
  │  @Transactional ← applied here │
  │  void testSavesUser() {         │
  │      service.save(user);        │  ← DB operation in TX
  │      verify user in DB...       │
  │  }                              │
  │  ← Transaction AUTOMATICALLY    │
  │    ROLLED BACK after test ✓     │
  └─────────────────────────────────┘
  Benefit: DB cleaned up after each test
  Risk: @Async and @TransactionalEventListener
        may not see uncommitted data

@Commit — keep changes:
  @Test
  @Transactional
  @Commit  // Don't rollback — keep changes in DB
  void testPersistence() { ... }

@Rollback(false) — same as @Commit

Testing REQUIRES_NEW:
  @Transactional  // outer TX
  @Test
  void testRequiresNew() {
      // call service that uses REQUIRES_NEW
      // REQUIRES_NEW creates a new TX inside the outer TX
      // Both TXs visible in the test scope
  }
```
