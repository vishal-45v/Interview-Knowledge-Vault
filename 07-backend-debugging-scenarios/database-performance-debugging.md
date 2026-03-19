# Database Performance Debugging

---

## PostgreSQL Query Diagnostics

```sql
-- Find slow queries (pg_stat_statements extension)
SELECT query,
       calls,
       total_exec_time / 1000 as total_seconds,
       mean_exec_time / 1000 as mean_seconds,
       rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- Find tables without indexes on foreign keys
SELECT c.conrelid::regclass AS table,
       a.attname AS column
FROM pg_constraint c
JOIN pg_attribute a ON a.attrelid = c.conrelid AND a.attnum = ANY(c.conkey)
WHERE c.contype = 'f'
AND NOT EXISTS (
    SELECT FROM pg_index i
    WHERE i.indrelid = c.conrelid
    AND a.attnum = ANY(i.indkey)
);

-- Check index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Indexes never used!
ORDER BY schemaname, tablename;
```

---

## HikariCP Connection Pool Diagnostics

```yaml
logging:
  level:
    com.zaxxer.hikari: DEBUG
    com.zaxxer.hikari.HikariConfig: DEBUG
```

```java
// Access HikariCP metrics
HikariPoolMXBean pool = ((HikariDataSource)dataSource).getHikariPoolMXBean();
int active = pool.getActiveConnections();
int idle = pool.getIdleConnections();
int waiting = pool.getThreadsAwaitingConnection();
int total = pool.getTotalConnections();

// Alert if waiting > 0 for sustained period
if (waiting > 0) {
    log.warn("Connection pool under pressure: {} threads waiting", waiting);
}
```

---

## Diagnosing Connection Pool Exhaustion

```
Symptoms:
  - "Unable to acquire JDBC Connection" errors
  - Requests timing out
  - hikaricp_connections_pending > 0

Causes:
  1. Pool too small (max connections too low)
  2. Long-running transactions holding connections
  3. Slow queries
  4. External API calls inside @Transactional

Diagnosis:
  SELECT pid, usename, wait_event_type, wait_event, state, query
  FROM pg_stat_activity
  WHERE state != 'idle'
  ORDER BY query_start;

  Look for: Long-running queries, transactions open > 30s

Fix:
  1. Short-term: increase pool size (hikari.maximum-pool-size)
  2. Long-term: Move external API calls outside @Transactional
  3. Add timeouts: hikari.connection-timeout=30000
```

---

## Common DB Performance Anti-Patterns

```java
// 1. SELECT * in JPA
@Query("SELECT p FROM Product p")  // Loads all fields including BLOBs
// Fix:
@Query("SELECT new ProductSummary(p.id, p.name, p.price) FROM Product p")

// 2. COUNT before pagination (slow on large tables)
Page<Order> orders = orderRepository.findAll(pageable);  // Runs COUNT(*)
// Fix: Use Slice<Order> if total count not needed
Slice<Order> orders = orderRepository.findAll(pageable);  // No COUNT query

// 3. saveAll() with auto-generated IDs
@GeneratedValue(strategy = IDENTITY)  // Prevents batching! Each INSERT is separate
// Fix: Use SEQUENCE strategy for batch inserts
@GeneratedValue(strategy = SEQUENCE)

// 4. Bidirectional relationship not managed correctly
order.addItem(item);  // Only updates item.order, not order.items list
// Must call both sides:
item.setOrder(order);
order.getItems().add(item);
```
