# JDBC — Structured Answers

## 1. What Is the Difference Between DriverManager and DataSource?

`DriverManager` is a static utility class in `java.sql` that creates a new physical database connection every time `getConnection()` is called. Each call performs a full TCP handshake, authenticates with the database server, and allocates server-side session resources. This is straightforward but expensive — typically 20–200ms per call.

`DataSource` is an interface in `javax.sql` that abstracts connection acquisition. Its most important implementation in production is a connection pool (e.g., HikariCP). The pool creates physical connections at startup and reuses them. `getConnection()` borrows an already-open connection from the pool in microseconds.

```java
// DriverManager — fine for scripts/tests, not for production
Connection conn = DriverManager.getConnection(
    "jdbc:postgresql://localhost:5432/mydb", "user", "password"
);

// DataSource with HikariCP — production standard
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
config.setUsername("user");
config.setPassword("password");
config.setMaximumPoolSize(10);
DataSource ds = new HikariDataSource(config);
Connection conn = ds.getConnection(); // borrowed from pool, returns in <1ms
```

Spring Boot auto-configures a HikariCP `DataSource` when `spring-boot-starter-data-jpa` or `spring-boot-starter-jdbc` is on the classpath — you never write this setup manually.

**When to use DriverManager**: CLI tools, database migration scripts run once, unit tests without a framework.
**When to use DataSource**: every production web application, every service handling concurrent requests.

---

## 2. How Does Connection Pooling Work? Why Use HikariCP?

Connection pooling maintains a cache of ready-to-use database connections. The pool manages the lifecycle: creating connections at startup, lending them to application threads, reclaiming them when the application calls `close()`, validating their health, and evicting stale ones.

**HikariCP internals that make it fast**:
- Uses a custom lock-free data structure called `ConcurrentBag` to hand out connections without `synchronized` blocks.
- Uses `AtomicReference` and compare-and-swap operations instead of locks.
- Bytecode manipulation via `JavassistProxyFactory` to generate ultra-thin proxy wrappers around `Connection`, `Statement`, and `ResultSet`.
- Aggressive connection validation using `isValid()` or a validation query only when the connection has been idle for a configurable time.

```java
// Spring Boot application.properties configuration
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=user
spring.datasource.password=secret
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
spring.datasource.hikari.leak-detection-threshold=60000
```

**Why HikariCP over alternatives (c3p0, DBCP2, Tomcat JDBC)**:
- Consistently fastest in benchmarks (orders-of-magnitude lower latency under contention).
- Smallest codebase (~130KB JAR), easiest to reason about.
- Best default settings — sensible out of the box without extensive tuning.
- Active maintenance and Spring Boot's default choice since Spring Boot 2.0.

---

## 3. Why Is PreparedStatement Preferred Over Statement?

There are three independent reasons — any one of them alone justifies the preference.

**Reason 1 — SQL Injection Prevention**

```java
// Statement — SQL injection vulnerability
String userInput = "' OR '1'='1"; // attacker-controlled input
String sql = "SELECT * FROM users WHERE email = '" + userInput + "'";
// Resulting SQL: SELECT * FROM users WHERE email = '' OR '1'='1'
// Returns ALL users — authentication bypass

// PreparedStatement — injection impossible
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM users WHERE email = ?"
);
pstmt.setString(1, userInput);
// DB receives email as a string datum, not SQL — injection has no effect
```

**Reason 2 — Performance for Repeated Execution**

The database parses and plans the query once when you call `prepareStatement()`. On repeated executions with different parameters, only the parameter values are sent — the parse and plan steps are skipped.

```java
PreparedStatement pstmt = conn.prepareStatement(
    "INSERT INTO events (user_id, event_type, ts) VALUES (?, ?, ?)"
);
for (Event e : events) {
    pstmt.setLong(1, e.getUserId());
    pstmt.setString(2, e.getType());
    pstmt.setTimestamp(3, e.getTimestamp());
    pstmt.executeUpdate(); // plan reused from cache
}
```

**Reason 3 — Correct Handling of Special Characters and Types**

`setString()`, `setTimestamp()`, `setDate()` handle quoting, escaping, and type conversion correctly. With string concatenation you must manually escape apostrophes, handle null values, and format dates correctly for each database dialect — a maintenance nightmare.

---

## 4. How Do You Handle Transactions in Plain JDBC?

JDBC transactions require three steps: disable auto-commit, execute your statements, then either commit or rollback.

```java
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false); // step 1: begin transaction
try {
    // step 2: execute statements
    PreparedStatement debit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance - ? WHERE id = ?"
    );
    debit.setBigDecimal(1, amount);
    debit.setLong(2, fromAccountId);
    int rows = debit.executeUpdate();
    if (rows != 1) throw new IllegalStateException("Account not found");

    PreparedStatement credit = conn.prepareStatement(
        "UPDATE accounts SET balance = balance + ? WHERE id = ?"
    );
    credit.setBigDecimal(1, amount);
    credit.setLong(2, toAccountId);
    credit.executeUpdate();

    conn.commit(); // step 3a: commit — both updates permanent
} catch (Exception e) {
    conn.rollback(); // step 3b: rollback — neither update persists
    throw e;
} finally {
    conn.setAutoCommit(true); // restore for pool reuse
    conn.close();             // return to pool
}
```

**Isolation levels** control what a transaction can see from concurrent transactions:

```java
conn.setTransactionIsolation(Connection.TRANSACTION_READ_COMMITTED);
// Options: READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, SERIALIZABLE
```

Most databases default to `READ_COMMITTED`. `SERIALIZABLE` prevents all anomalies but has the highest locking overhead.

---

## 5. What Is a Connection Leak and How Do You Detect It?

A connection leak occurs when application code acquires a connection from the pool but never returns it via `close()`. The most common cause is a missing `close()` call in an exception path:

```java
// LEAKING code — exception before close() leaves connection abandoned
Connection conn = dataSource.getConnection();
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.executeQuery(); // if this throws, conn is never closed
conn.close();         // never reached on exception
```

**Detection methods**:

1. **HikariCP `leakDetectionThreshold`**: Set this to the maximum expected connection hold time (e.g., 2000ms for a web request). HikariCP logs a warning with the full stack trace of the thread that borrowed the connection if it is not returned within that time.

```properties
spring.datasource.hikari.leak-detection-threshold=2000
```

2. **Pool metrics**: Monitor `hikari.connections.active` (Micrometer). If it climbs continuously and never returns to baseline after load stops, you have a leak.

3. **Thread dumps**: During an incident, take a thread dump. Leaked connections show no live thread referencing them — the borrowing thread has long since completed its HTTP request.

**Prevention**: Always use try-with-resources for `Connection`, `Statement`, and `ResultSet`:

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(sql);
     ResultSet rs = pstmt.executeQuery()) {
    while (rs.next()) {
        process(rs);
    }
} // all three closed automatically in reverse order, even on exception
```

---

## 6. How Do You Perform Batch Inserts Efficiently in JDBC?

Batch execution groups multiple parameter sets into a single database round-trip. The speedup is dramatic for bulk inserts because network latency (typically 1–5ms per round-trip) dominates total execution time when inserting thousands of rows individually.

```java
String sql = "INSERT INTO product_events (product_id, event_type, event_time) VALUES (?, ?, ?)";

conn.setAutoCommit(false);
try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
    int batchCount = 0;
    for (ProductEvent event : events) {
        pstmt.setLong(1, event.getProductId());
        pstmt.setString(2, event.getEventType());
        pstmt.setTimestamp(3, Timestamp.from(event.getTime()));
        pstmt.addBatch();                  // queue the parameter set

        if (++batchCount % 1000 == 0) {   // flush every 1000 rows
            int[] counts = pstmt.executeBatch();  // single network call
            conn.commit();
        }
    }
    pstmt.executeBatch(); // flush remainder
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
    throw e;
}
```

**Key practices**:
- Commit in chunks (1000–5000 rows) to avoid holding a giant transaction that locks tables or overflows the transaction log.
- Check the returned `int[]` from `executeBatch()` — each element is the row count for that statement. `Statement.SUCCESS_NO_INFO` (-2) means success but no count available.
- PostgreSQL supports `reWriteBatchedInserts=true` on the connection URL, which rewrites individual INSERTs into a single multi-row `INSERT INTO ... VALUES (...), (...), (...)` for even higher throughput.

```
# PostgreSQL batch rewrite:
jdbc:postgresql://localhost:5432/mydb?reWriteBatchedInserts=true
```

---

## 7. What Is the N-Tier JDBC Architecture?

The JDBC architecture cleanly separates an application into layers, each with a single responsibility:

**Presentation Tier**: HTTP controllers, REST endpoints, or UI. Knows nothing about SQL.

**Business Logic Tier**: Service classes implementing business rules. Orchestrates data access but does not write SQL directly.

**Data Access Tier (DAO/Repository)**: Classes that translate between domain objects and SQL. The only layer that uses JDBC or JPA.

**JDBC API**: Standard `java.sql` interfaces that the DAO layer calls. Acts as the contract between your code and the database driver.

**JDBC Driver**: Vendor-supplied JAR that implements the `java.sql` interfaces and communicates with the specific database over a network protocol.

**Database Server**: The actual database engine.

```java
// DAO pattern — isolates SQL from business logic
@Repository
public class OrderDao {
    private final DataSource dataSource;

    public Order findById(long id) {
        String sql = "SELECT id, customer_id, total FROM orders WHERE id = ?";
        try (Connection conn = dataSource.getConnection();
             PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setLong(1, id);
            try (ResultSet rs = pstmt.executeQuery()) {
                if (rs.next()) {
                    return mapRow(rs);
                }
            }
        } catch (SQLException e) {
            throw new DataAccessException("Failed to find order " + id, e);
        }
        return null;
    }

    private Order mapRow(ResultSet rs) throws SQLException {
        Order o = new Order();
        o.setId(rs.getLong("id"));
        o.setCustomerId(rs.getLong("customer_id"));
        o.setTotal(rs.getBigDecimal("total"));
        return o;
    }
}
```

---

## 8. How Do You Call a Stored Procedure Using JDBC?

`CallableStatement` extends `PreparedStatement` and supports the JDBC stored procedure escape syntax `{call procedure_name(?, ?)}`.

```java
// Stored procedure: PROCEDURE get_user_stats(IN user_id BIGINT, OUT order_count INT, OUT total_spent DECIMAL)
String call = "{call get_user_stats(?, ?, ?)}";
try (CallableStatement cstmt = conn.prepareCall(call)) {
    // Register input parameter
    cstmt.setLong(1, userId);

    // Register output parameters — must declare SQL type
    cstmt.registerOutParameter(2, Types.INTEGER);
    cstmt.registerOutParameter(3, Types.DECIMAL);

    cstmt.execute();

    // Read output parameters after execution
    int orderCount = cstmt.getInt(2);
    BigDecimal totalSpent = cstmt.getBigDecimal(3);
    System.out.printf("User %d: %d orders, $%s spent%n", userId, orderCount, totalSpent);
}

// Procedure returning a ResultSet (common in PostgreSQL/MySQL):
try (CallableStatement cstmt = conn.prepareCall("{call find_active_users(?)}")) {
    cstmt.setString(1, "PREMIUM");
    boolean hasResultSet = cstmt.execute();
    if (hasResultSet) {
        try (ResultSet rs = cstmt.getResultSet()) {
            while (rs.next()) {
                processUser(rs);
            }
        }
    }
}
```

For functions (which return a value rather than using OUT parameters):

```java
// Function: FUNCTION calculate_tax(price DECIMAL) RETURNS DECIMAL
String call = "{ ? = call calculate_tax(?) }";
try (CallableStatement cstmt = conn.prepareCall(call)) {
    cstmt.registerOutParameter(1, Types.DECIMAL);
    cstmt.setBigDecimal(2, price);
    cstmt.execute();
    BigDecimal tax = cstmt.getBigDecimal(1);
}
```

---

## 9. What Are the Best Practices for ResultSet Processing?

**1. Use try-with-resources for all three JDBC objects**

```java
try (Connection conn = ds.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(sql);
     ResultSet rs = pstmt.executeQuery()) {
    while (rs.next()) {
        // process
    }
}
```

**2. Prefer column names over column indexes**

```java
// Fragile — breaks silently if column order changes in SELECT
String name = rs.getString(1);

// Robust — self-documenting and order-independent
String name = rs.getString("customer_name");
```

**3. Use rs.wasNull() for nullable columns**

```java
BigDecimal discount = rs.getBigDecimal("discount");
if (!rs.wasNull()) {
    applyDiscount(discount);
}
// Alternatively for primitives:
int age = rs.getInt("age");
if (rs.wasNull()) age = -1; // sentinel for null
```

**4. Control fetch size for large result sets**

```java
pstmt.setFetchSize(500); // avoid loading all rows into memory at once
```

**5. Map to domain objects immediately, close ResultSet promptly**

```java
List<User> users = new ArrayList<>();
try (ResultSet rs = pstmt.executeQuery()) {
    while (rs.next()) {
        users.add(new User(rs.getLong("id"), rs.getString("name")));
    }
}
// rs closed — now work with domain objects, not an open cursor
return users;
```

**6. Never hold a ResultSet open across a transaction boundary or return it from a method**. The ResultSet depends on an open Statement, which depends on an open Connection. Returning it leaks all three.

---

## 10. How Does HikariCP Decide the Optimal Pool Size?

HikariCP does not auto-tune pool size — that is the developer's responsibility. But HikariCP's documentation provides the formula:

```
maxPoolSize = (core_count * 2) + effective_spindle_count
```

For a 4-core machine with SSD storage (1 effective spindle):
`maxPoolSize = (4 * 2) + 1 = 9`

**The reasoning**: A database CPU can execute N queries simultaneously where N = number of cores. Every additional connection above that point adds to a queue on the OS scheduler, increasing context switching overhead. Threads waiting for I/O release the CPU, so a small multiplier (2x) accounts for threads blocked on disk or network I/O. SSD storage is so fast it barely counts as a spindle.

**In practice for a Spring Boot application**:

```java
// Typical production configuration
HikariConfig config = new HikariConfig();
config.setMaximumPoolSize(10);          // slightly above formula for safety margin
config.setMinimumIdle(5);              // keep 5 connections warm at all times
config.setConnectionTimeout(30_000);   // throw after 30s waiting for connection
config.setIdleTimeout(600_000);        // evict idle connections after 10 min
config.setMaxLifetime(1_800_000);      // recycle connections after 30 min (beat DB firewall timeouts)
config.setKeepaliveTime(60_000);       // keepalive ping every 60s for idle connections
config.setLeakDetectionThreshold(4_000); // warn if connection held > 4s
```

**Anti-pattern**: setting `maximumPoolSize=100` thinking more = faster. With a 4-core database, 100 connections means 90+ connections are always queuing, adding overhead with no throughput benefit. Benchmarks consistently show that fewer, properly-sized pools outperform oversized ones.
