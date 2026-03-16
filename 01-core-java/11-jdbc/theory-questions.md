# JDBC — Theory Questions

> 18 theory questions on JDBC fundamentals, connection pooling, transactions, and Spring JDBC/JPA integration for Senior Java Backend Engineers.

---

**Q1. What are the key components of JDBC?**

```
JDBC Architecture:
Java Application
     ↓
DriverManager / DataSource        (connection management)
     ↓
JDBC Driver (vendor-specific)     (implements JDBC interfaces)
     ↓
Database Server
```

Core interfaces:
- `Connection` — database session; executes statements
- `Statement` — SQL execution (plain, no params)
- `PreparedStatement` — pre-compiled SQL with parameters (use this)
- `CallableStatement` — stored procedure calls
- `ResultSet` — query results (cursor-based)
- `DataSource` — preferred connection factory (from connection pool)

---

**Q2. Why should you always use PreparedStatement instead of Statement?**

```java
// WRONG — String concatenation = SQL injection vulnerability!
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);
// If email = "admin'--", this becomes:
// SELECT * FROM users WHERE email = 'admin'--' → strips password check!

// CORRECT — PreparedStatement with parameters
String sql = "SELECT * FROM users WHERE email = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, email);  // Automatically escaped
ResultSet rs = pstmt.executeQuery();

// PreparedStatement advantages:
// 1. SQL injection prevention — params are never interpreted as SQL
// 2. Performance — query plan cached after first parse
// 3. Type safety — setString(), setLong(), setTimestamp() etc.
// 4. Readable — parameters clearly separated from SQL structure
```

---

**Q3. How does JDBC transaction management work?**

```java
Connection conn = dataSource.getConnection();
try {
    conn.setAutoCommit(false);  // Begin transaction (default is autocommit = true)

    PreparedStatement debit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance - ? WHERE id = ?");
    debit.setBigDecimal(1, amount);
    debit.setLong(2, fromAccountId);
    debit.executeUpdate();

    PreparedStatement credit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?");
    credit.setBigDecimal(1, amount);
    credit.setLong(2, toAccountId);
    credit.executeUpdate();

    conn.commit();  // Commit both updates atomically

} catch (SQLException e) {
    conn.rollback();  // Rollback both updates
    throw new RuntimeException("Transfer failed", e);
} finally {
    conn.setAutoCommit(true);  // Restore default
    conn.close();              // Return to pool
}
```

---

**Q4. What are JDBC savepoints?**

```java
Savepoint savepoint = null;
try {
    conn.setAutoCommit(false);

    // Step 1: Create order
    createOrder(conn, order);

    savepoint = conn.setSavepoint("after_order_created");  // Mark savepoint

    // Step 2: Reserve inventory (may fail)
    reserveInventory(conn, order.getItems());

    conn.commit();

} catch (InsufficientInventoryException e) {
    // Roll back only to savepoint — keep the order creation
    conn.rollback(savepoint);
    // Mark order as "pending inventory" instead of failing completely
    markOrderPendingInventory(conn, order);
    conn.commit();
} catch (Exception e) {
    conn.rollback();  // Full rollback
    throw e;
}
```

---

**Q5. What is connection pooling and how does HikariCP work?**

Opening a database connection takes ~5-10ms (TCP handshake + TLS + authentication). A connection pool maintains a set of pre-opened connections, reusing them across requests.

HikariCP (default in Spring Boot) lifecycle:
```
App starts → min-idle connections created and tested
Request arrives → pool.getConnection() → returns idle connection
Request completes → conn.close() → returns to pool (NOT actually closed!)
No idle connections → wait up to connection-timeout for one to become available
Connection idle > idle-timeout → removed from pool
Connection age > max-lifetime → removed and replaced
```

```yaml
spring.datasource.hikari:
  maximum-pool-size: 10       # Max total connections
  minimum-idle: 5             # Always keep 5 warm
  connection-timeout: 5000    # Max wait for connection (ms)
  idle-timeout: 600000        # Remove idle after 10 min
  max-lifetime: 1800000       # Replace connection after 30 min
  keepalive-time: 30000       # Send keepalive query every 30s
  connection-test-query: "SELECT 1"  # Test before handing to app
```

---

**Q6. What is the difference between `execute()`, `executeQuery()`, and `executeUpdate()`?**

```java
Statement stmt = conn.createStatement();

// executeQuery() — for SELECT; returns ResultSet
ResultSet rs = stmt.executeQuery("SELECT * FROM orders");

// executeUpdate() — for INSERT, UPDATE, DELETE; returns row count
int rowsAffected = stmt.executeUpdate("UPDATE orders SET status = 'SHIPPED' WHERE id = 1");

// execute() — any SQL; returns true if first result is ResultSet
boolean hasResultSet = stmt.execute("SELECT * FROM orders");
if (hasResultSet) {
    ResultSet rs2 = stmt.getResultSet();
} else {
    int count = stmt.getUpdateCount();
}

// executeBatch() — execute multiple statements as batch
PreparedStatement ps = conn.prepareStatement("INSERT INTO orders VALUES (?, ?)");
for (Order order : orders) {
    ps.setLong(1, order.getId());
    ps.setString(2, order.getStatus());
    ps.addBatch();  // Queue up
}
int[] counts = ps.executeBatch();  // Execute all at once
```

---

**Q7. How do you read ResultSet data? What are common pitfalls?**

```java
ResultSet rs = pstmt.executeQuery();

while (rs.next()) {
    // Column by name (preferred — order-independent)
    Long id = rs.getLong("id");
    String status = rs.getString("status");
    Timestamp createdAt = rs.getTimestamp("created_at");
    BigDecimal total = rs.getBigDecimal("total");

    // Handle SQL NULL — rs.getLong() returns 0 for NULL, not null!
    Long nullableId = rs.getLong("nullable_col");
    if (rs.wasNull()) nullableId = null;  // Check AFTER reading

    // Or use getObject() to get null-safe Object
    Long nullable = (Long) rs.getObject("nullable_col");
}

// Don't forget to close ResultSet — use try-with-resources
try (PreparedStatement pstmt = conn.prepareStatement(sql);
     ResultSet rs2 = pstmt.executeQuery()) {
    while (rs2.next()) { ... }
}
```

---

**Q8. What is JDBC batch operations and when should you use it?**

```java
// Without batch — N database round trips
for (Order order : 10000Orders) {
    PreparedStatement ps = conn.prepareStatement("INSERT INTO orders ...");
    ps.setLong(1, order.getId());
    ps.executeUpdate();  // One round trip per insert!
}

// With batch — 1 round trip (or few with batching)
conn.setAutoCommit(false);
PreparedStatement ps = conn.prepareStatement("INSERT INTO orders (id, status, total) VALUES (?, ?, ?)");

int batchSize = 500;
int count = 0;

for (Order order : orders) {
    ps.setLong(1, order.getId());
    ps.setString(2, order.getStatus());
    ps.setBigDecimal(3, order.getTotal());
    ps.addBatch();

    if (++count % batchSize == 0) {
        ps.executeBatch();  // Send batch
        conn.commit();
        ps.clearBatch();
    }
}
ps.executeBatch();  // Remaining items
conn.commit();

// Spring JdbcTemplate batch
jdbcTemplate.batchUpdate(
    "INSERT INTO orders (id, status, total) VALUES (?, ?, ?)",
    orders,
    500,  // Batch size
    (ps, order) -> {
        ps.setLong(1, order.getId());
        ps.setString(2, order.getStatus());
        ps.setBigDecimal(3, order.getTotal());
    }
);
```

---

**Q9. How does Spring JdbcTemplate simplify JDBC?**

`JdbcTemplate` handles connection management, exception translation, and ResultSet iteration:

```java
@Repository
public class OrderRepository {

    @Autowired private JdbcTemplate jdbcTemplate;

    // Query for single value
    int count = jdbcTemplate.queryForObject(
        "SELECT COUNT(*) FROM orders WHERE status = ?",
        Integer.class, "PENDING");

    // Query for single row
    Order order = jdbcTemplate.queryForObject(
        "SELECT * FROM orders WHERE id = ?",
        orderRowMapper, orderId);

    // Query for list
    List<Order> orders = jdbcTemplate.query(
        "SELECT * FROM orders WHERE customer_id = ?",
        orderRowMapper, customerId);

    // Update
    int rows = jdbcTemplate.update(
        "UPDATE orders SET status = ? WHERE id = ?",
        "SHIPPED", orderId);

    // Streaming large results
    jdbcTemplate.query(
        "SELECT * FROM orders WHERE created_at > ?",
        rs -> {
            while (rs.next()) {
                process(orderRowMapper.mapRow(rs, 0));
            }
        },
        lastMonthTimestamp
    );
}

// RowMapper implementation
private final RowMapper<Order> orderRowMapper = (rs, rowNum) ->
    new Order(
        rs.getLong("id"),
        rs.getString("status"),
        rs.getBigDecimal("total"),
        rs.getTimestamp("created_at").toInstant()
    );
```

---

**Q10. What is Spring's `NamedParameterJdbcTemplate`?**

Uses named parameters instead of positional `?`, improving readability:

```java
// Positional — error-prone with many parameters
jdbcTemplate.update(
    "UPDATE orders SET status = ?, total = ?, notes = ? WHERE id = ? AND customer_id = ?",
    "SHIPPED", total, notes, orderId, customerId);

// Named parameters — clear and order-independent
NamedParameterJdbcTemplate namedTemplate = new NamedParameterJdbcTemplate(dataSource);

MapSqlParameterSource params = new MapSqlParameterSource()
    .addValue("status", "SHIPPED")
    .addValue("total", total)
    .addValue("notes", notes)
    .addValue("orderId", orderId)
    .addValue("customerId", customerId);

namedTemplate.update(
    "UPDATE orders SET status = :status, total = :total, notes = :notes " +
    "WHERE id = :orderId AND customer_id = :customerId",
    params);

// With a bean (SqlParameterSource)
namedTemplate.update(
    "INSERT INTO orders (id, status, total) VALUES (:id, :status, :total)",
    new BeanPropertySqlParameterSource(order));  // Uses getters/fields
```

---

**Q11. What are JDBC isolation levels?**

```java
// Set isolation level on connection
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);

// Levels:
// TRANSACTION_READ_UNCOMMITTED — sees uncommitted data (dirty reads possible)
// TRANSACTION_READ_COMMITTED  — only sees committed data (default in most DBs)
// TRANSACTION_REPEATABLE_READ — same read returns same data within transaction
// TRANSACTION_SERIALIZABLE    — full isolation, transactions fully sequential

// PostgreSQL default: READ COMMITTED
// MySQL InnoDB default: REPEATABLE READ
```

---

**Q12. How do you handle CLOB and BLOB data in JDBC?**

```java
// Writing BLOB
PreparedStatement ps = conn.prepareStatement(
    "INSERT INTO documents (id, content, thumbnail) VALUES (?, ?, ?)");
ps.setLong(1, documentId);

// Text (CLOB)
ps.setString(2, largeTextContent);  // Small strings
// Or for large:
Reader reader = new FileReader("document.txt");
ps.setCharacterStream(2, reader, fileLength);

// Binary (BLOB)
ps.setBinaryStream(3, imageInputStream, imageSize);
// Or: ps.setBytes(3, smallImageBytes);  // For small binary data

// Reading BLOB
ResultSet rs = stmt.executeQuery("SELECT thumbnail FROM documents WHERE id = ?");
if (rs.next()) {
    // Read as stream (memory efficient)
    try (InputStream is = rs.getBinaryStream("thumbnail")) {
        Files.copy(is, outputPath);
    }
    // Or as bytes (small blobs only)
    byte[] bytes = rs.getBytes("thumbnail");
}
```

---

**Q13. What are SQL exceptions and how do you handle them?**

```java
try {
    pstmt.executeUpdate();
} catch (SQLIntegrityConstraintViolationException e) {
    // Foreign key, unique, not null violation
    // Error code and SQLState are DB-specific:
    if (e.getSQLState().startsWith("23")) {
        throw new DuplicateKeyException("Unique constraint violated", e);
    }
} catch (SQLTimeoutException e) {
    throw new QueryTimeoutException("Query took too long", e);
} catch (SQLRecoverableException e) {
    // Connection issue — might succeed if retried
    throw new TransientDataAccessException("Recoverable SQL error", e);
} catch (SQLException e) {
    // e.getErrorCode() — DB vendor-specific code
    // e.getSQLState() — standard 5-char ANSI/ISO code
    log.error("SQL error: state={}, code={}, msg={}",
              e.getSQLState(), e.getErrorCode(), e.getMessage());
    throw new DataAccessException("SQL execution failed", e);
}

// Spring translates vendor-specific codes to DataAccessException hierarchy
// using SQLExceptionTranslator (SQLErrorCodeSQLExceptionTranslator)
```

---

**Q14. What is the `ResultSetMetaData` and when is it useful?**

```java
ResultSet rs = stmt.executeQuery("SELECT * FROM orders LIMIT 1");
ResultSetMetaData meta = rs.getMetaData();

int columnCount = meta.getColumnCount();
for (int i = 1; i <= columnCount; i++) {
    System.out.println("Column: " + meta.getColumnName(i));
    System.out.println("Type:   " + meta.getColumnTypeName(i));
    System.out.println("Nullable: " + meta.isNullable(i));
    System.out.println("Size:   " + meta.getColumnDisplaySize(i));
}

// Use case: generic CSV exporter
public String exportToCsv(String query) throws SQLException {
    StringBuilder csv = new StringBuilder();
    try (Statement stmt = conn.createStatement();
         ResultSet rs = stmt.executeQuery(query)) {

        ResultSetMetaData meta = rs.getMetaData();
        int cols = meta.getColumnCount();

        // Header row
        for (int i = 1; i <= cols; i++) {
            csv.append(meta.getColumnName(i));
            if (i < cols) csv.append(",");
        }
        csv.append("\n");

        // Data rows
        while (rs.next()) {
            for (int i = 1; i <= cols; i++) {
                csv.append(rs.getString(i));
                if (i < cols) csv.append(",");
            }
            csv.append("\n");
        }
    }
    return csv.toString();
}
```

---

**Q15. How do you stream large query results?**

```java
// Problem: SELECT * FROM orders (10M rows) → OutOfMemoryError!

// Solution 1: JDBC Fetch Size (server-side cursor)
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM orders WHERE status = ?");
pstmt.setFetchSize(500);  // Fetch 500 rows at a time from DB
pstmt.setString(1, "PENDING");
ResultSet rs = pstmt.executeQuery();
// rs.next() fetches next batch from DB when current batch exhausted

// Solution 2: Spring JdbcTemplate with RowCallbackHandler
jdbcTemplate.query(
    "SELECT * FROM orders",
    pstmt -> pstmt.setFetchSize(500),  // Statement customizer
    (rs) -> {
        // Called once per row — no collection of all rows
        processRow(mapRow(rs));
    }
);

// Solution 3: LIMIT/OFFSET pagination
int offset = 0, limit = 1000;
while (true) {
    List<Order> batch = jdbcTemplate.query(
        "SELECT * FROM orders ORDER BY id LIMIT ? OFFSET ?",
        orderRowMapper, limit, offset);
    if (batch.isEmpty()) break;
    processBatch(batch);
    offset += limit;
}
// Better: use keyset pagination (WHERE id > lastId ORDER BY id LIMIT 1000)
```

---

**Q16. What is Spring Data JPA vs Spring JDBC?**

| Feature | Spring JDBC (JdbcTemplate) | Spring Data JPA (Hibernate) |
|---------|---------------------------|------------------------------|
| Abstraction | Low — writes SQL manually | High — generates SQL |
| Control | Full SQL control | Limited; use @Query for custom |
| Performance | Predictable | Hidden N+1, extra queries |
| Learning curve | Low (knows SQL) | Higher (JPQL, entity states) |
| Mapping | Manual RowMapper | Automatic via @Entity |
| Transactions | JdbcTemplate manages | @Transactional + EntityManager |
| Best for | Complex queries, reporting, bulk | CRUD-heavy domain model |

**Use JDBC when:**
- Performance-critical queries
- Complex SQL (window functions, CTEs, UNION)
- Bulk operations
- Read-only reports

**Use JPA when:**
- Rich domain model with relationships
- Standard CRUD operations
- Object-graph persistence (cascade)

---

**Q17. How do database connection pools detect stale connections?**

Connections can go stale if:
- Database server closed idle connection
- Network device dropped the connection
- Firewall timeout (common in cloud environments)

HikariCP stale connection detection:
```yaml
spring.datasource.hikari:
  connection-test-query: "SELECT 1"  # Validation query
  keepalive-time: 30000              # Send keepalive every 30s to prevent firewall drops
  max-lifetime: 1800000              # Replace connection every 30min (before DB closes it)
  # connection-init-sql: "SET timezone='UTC'"  # Init SQL on new connection
```

Pool validation strategies:
1. **Keep-alive**: send a heartbeat query periodically
2. **Validate on borrow**: test connection before returning to app (adds latency)
3. **Validate on return**: test connection on return to pool
4. **Max-lifetime**: replace connections before they go stale

---

**Q18. What is Spring's `@Transactional` integration with JDBC?**

```java
// Spring Transaction Manager for JDBC:
@Bean
public PlatformTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}

// Spring stores the connection in ThreadLocal (TransactionSynchronizationManager)
// JdbcTemplate retrieves the connection from ThreadLocal when in a transaction

@Service
public class OrderService {

    @Autowired private JdbcTemplate jdbcTemplate;

    @Transactional
    public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
        // Spring opens a transaction, stores connection in ThreadLocal
        jdbcTemplate.update(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?",
            amount, fromId);
        // SAME connection used — JdbcTemplate uses DataSourceUtils.getConnection()
        jdbcTemplate.update(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            amount, toId);
        // Spring commits on method exit, rolls back on RuntimeException
    }
}
```

Spring's `DataSourceUtils.getConnection(dataSource)`:
- If in `@Transactional` context: returns the transactional connection from ThreadLocal
- If not: returns a new connection from the pool
