# Database Transactions & Isolation Levels

---

## ACID Properties

**Atomicity:** All operations in a transaction succeed or all fail. No partial commits.

**Consistency:** Transaction brings the database from one valid state to another. Constraints never violated.

**Isolation:** Concurrent transactions appear to execute serially (to a degree specified by isolation level).

**Durability:** Committed data survives crashes. Wal (Write-Ahead Log) ensures this.

---

## Isolation Levels and Their Problems

```
                    Dirty   Non-Repeatable  Phantom
Isolation Level     Reads   Reads           Reads
READ UNCOMMITTED     ✓       ✓               ✓      (lowest isolation)
READ COMMITTED       ✗       ✓               ✓      (PostgreSQL default)
REPEATABLE READ      ✗       ✗               ✓      (MySQL InnoDB default)
SERIALIZABLE         ✗       ✗               ✗      (highest isolation)

✓ = problem CAN occur
✗ = problem prevented
```

---

## Practical Isolation Level Selection

```java
// READ COMMITTED — for most operations (default)
@Transactional(isolation = Isolation.READ_COMMITTED)
public void processPayment(Payment payment) { ... }

// REPEATABLE READ — when re-reading data within TX is important
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Report generateAuditReport(LocalDate date) {
    // First read and second read of same data returns same values
    List<Transaction> txns = transactionRepo.findByDate(date);
    BigDecimal total = calculateTotal(txns);
    // Another call here will return the same total (no non-repeatable read)
    return new Report(txns, total);
}

// SERIALIZABLE — for high-stakes operations (seat booking, inventory)
@Transactional(isolation = Isolation.SERIALIZABLE)
public Booking bookLastSeat(Flight flight) {
    int available = seatRepo.countAvailable(flight);
    if (available == 0) throw new NoSeatsException();
    return seatRepo.reserve(flight);
    // SERIALIZABLE prevents phantom read — no other TX can insert a booking
}
```

---

## Optimistic vs Pessimistic Locking

**Optimistic Locking (prefer for low-contention):**
```java
@Entity
public class Product {
    @Version
    private Long version;  // Hibernate increments on each update
}

// If two threads update simultaneously:
// T1: READ version=5, UPDATE version=6 → success
// T2: READ version=5, UPDATE version=6 → OptimisticLockException (already 6)
```

**Pessimistic Locking (for high-contention, financial):**
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)  // SELECT FOR UPDATE
Optional<Account> findByIdForUpdate(Long id);

// T1 acquires lock, T2 waits
// No lost updates possible
```

---

## Deadlock Detection

```
T1: LOCK table_a row 1
    → wants LOCK table_b row 1

T2: LOCK table_b row 1
    → wants LOCK table_a row 1

DEADLOCK! DB detects cycle → kills one transaction (victim)
Victim: TransactionRolledBackException / DeadlockLoserDataAccessException

Prevention:
1. Always lock resources in same order (A before B)
2. Use single transaction for related operations
3. Keep transactions short
4. Use optimistic locking instead
```
