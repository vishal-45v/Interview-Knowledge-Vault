# SQL Indexing & Query Optimization

> Production database performance patterns for Java backend engineers.

---

## Index Types and When to Use Them

### B-Tree Index (Default)

```sql
-- Most common — good for equality, range, ORDER BY
CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Composite index — column order matters!
-- Use leftmost prefix rule:
CREATE INDEX idx_orders_status_created ON orders(status, created_at);
-- Supports: WHERE status = 'PENDING'
-- Supports: WHERE status = 'PENDING' AND created_at > '2024-01-01'
-- Does NOT support: WHERE created_at > '2024-01-01' (skips leftmost column)
```

### Partial Index (PostgreSQL)

```sql
-- Index only the rows you actually query
-- Much smaller → faster scans, less I/O
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'PENDING';
-- Only indexes pending orders — much smaller than full index on large table!

-- Query using this index:
SELECT * FROM orders
WHERE status = 'PENDING' AND created_at > NOW() - INTERVAL '7 days';
```

### Covering Index (Include columns)

```sql
-- Index contains all columns needed by the query
-- Query satisfied entirely from index (no heap lookup)
CREATE INDEX idx_products_category_name_price ON products(category)
INCLUDE (name, price);  -- PostgreSQL syntax

-- This query uses only the index (no table access):
SELECT name, price FROM products WHERE category = 'Electronics';
```

---

## Query Optimization

### EXPLAIN ANALYZE

```sql
-- Always use EXPLAIN ANALYZE to understand query plans
EXPLAIN ANALYZE
SELECT o.id, o.total, c.email
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'PENDING'
AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC;

-- Read the output:
-- Seq Scan → BAD for large tables (no index used)
-- Index Scan → GOOD (uses index)
-- Index Only Scan → BEST (index covers all needed columns)
-- Nested Loop → OK for small result sets
-- Hash Join → Good for medium result sets
-- Merge Join → Good when both sides are sorted

-- cost=0.00..8.27 rows=1 width=32
--   actual time=0.052..0.053 rows=1 loops=1
```

---

## The N+1 Problem in SQL

```java
// Spring Data JPA — N+1 problem
@Query("SELECT o FROM Order o WHERE o.status = 'PENDING'")
List<Order> findPendingOrders();
// + N queries for each order.getCustomer() access

// Fix: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = 'PENDING'")
List<Order> findPendingOrdersWithCustomer();

// Generated SQL (single query):
// SELECT o.*, c.* FROM orders o
// INNER JOIN customers c ON c.id = o.customer_id
// WHERE o.status = 'PENDING'
```

---

## Connection Pool Tuning (HikariCP)

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10        # Rule of thumb: (CPU cores * 2) + effective spindle count
      minimum-idle: 5              # Keep warm connections
      connection-timeout: 30000   # Wait up to 30s for a connection
      idle-timeout: 600000        # Remove idle connections after 10 min
      max-lifetime: 1800000       # Rotate connections every 30 min
      validation-timeout: 5000    # Validate connection timeout
      connection-test-query: "SELECT 1"
```

**Pool sizing formula (PostgreSQL/HikariCP):**
- For CPU-bound: `connections = (CPU_cores × 2) + effective_spindle_count`
- For I/O-bound: Can go higher, but watch for context switching overhead
- For typical web app (8 cores): pool size of 10-20 is often optimal

---

## Common Slow Query Patterns

```sql
-- 1. Full table scan due to function on indexed column
-- BAD: index not used because of YEAR() function
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- GOOD: range condition allows index use
SELECT * FROM orders
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- 2. LIKE with leading wildcard
-- BAD: can't use B-tree index
SELECT * FROM users WHERE email LIKE '%@gmail.com';

-- GOOD: fixed prefix uses index
SELECT * FROM users WHERE email LIKE 'john%';

-- 3. Implicit type conversion
-- BAD: phone_number is VARCHAR, passing INTEGER prevents index use
SELECT * FROM users WHERE phone_number = 5551234567;

-- GOOD: matching types
SELECT * FROM users WHERE phone_number = '5551234567';

-- 4. SELECT *
-- BAD: fetches all columns including large BLOBs
SELECT * FROM products WHERE category = 'Electronics';

-- GOOD: fetch only needed columns
SELECT id, name, price FROM products WHERE category = 'Electronics';
```

---

## Pagination Performance

```sql
-- BAD: OFFSET becomes slow on large tables
-- OFFSET 100000 means the DB scans and discards 100,000 rows
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000;

-- GOOD: Keyset/Cursor pagination using indexed column
-- "Give me 20 rows after ID 100000"
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;

-- Spring Data with JPA:
-- List<Order> findByIdGreaterThanOrderByIdAsc(Long afterId, Pageable pageable)
-- Page<Order> = bad for high offset pages
-- Slice<Order> = better (doesn't count total)
```

---

## Database Indexes in Practice

```java
// Spring Boot entity with indexes
@Entity
@Table(
    name = "orders",
    indexes = {
        @Index(name = "idx_orders_customer_id", columnList = "customer_id"),
        @Index(name = "idx_orders_status_created", columnList = "status, created_at"),
        @Index(name = "idx_orders_created_at", columnList = "created_at")
    }
)
public class Order {

    @Id @GeneratedValue
    private Long id;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @Column(name = "customer_id", nullable = false)
    private Long customerId;

    @Column(name = "created_at", nullable = false)
    private Instant createdAt;
}
```
