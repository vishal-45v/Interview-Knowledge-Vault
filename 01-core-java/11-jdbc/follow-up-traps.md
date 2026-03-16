# JDBC — Follow-Up Traps

## Trap 1: DriverManager.getConnection() vs DataSource — Which for Production?

**Trap**: A candidate says "both work the same way."

**Reality**: `DriverManager.getConnection()` creates a brand new physical connection on every call. It is acceptable for command-line tools, quick scripts, or unit tests, but it is entirely unsuitable for production web applications. Every HTTP request would open and close a TCP connection to the database — costing 20–200ms per request just in connection overhead.

`DataSource` backed by a connection pool (HikariCP, c3p0) is the only acceptable choice for production. Spring Boot auto-configures a HikariCP `DataSource` by default when you add a JDBC starter. You should never call `DriverManager.getConnection()` in production application code.

---

## Trap 2: What Happens If You Don't Close a Connection?

**Trap**: "The garbage collector will close it eventually."

**Reality**: If you are using a connection pool, the `Connection` object returned is a proxy wrapper around a physical connection. Calling `close()` on it does not destroy the physical connection — it returns the proxy to the pool. If you never call `close()`, the pool considers that connection permanently in-use. After enough such leaks, the pool is exhausted and `getConnection()` blocks until `connectionTimeout` is reached, then throws `SQLTimeoutException`.

Even if the garbage collector eventually finalizes the object, you cannot rely on GC timing, and most connection pool implementations do not trigger pool return on finalization. Always use try-with-resources:

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement pstmt = conn.prepareStatement(sql)) {
    // work here
} // conn.close() called automatically even if exception thrown
```

HikariCP's `leakDetectionThreshold` can log a stack trace if a connection is held longer than the configured threshold, helping you find the offending code.

---

## Trap 3: PreparedStatement SQL Injection Prevention — How Does It Actually Work?

**Trap**: "PreparedStatement escapes the quotes in the input."

**Reality**: Escaping is a secondary effect, not the mechanism. The actual protection comes from protocol-level separation of code and data. When you use `PreparedStatement`, the SQL template is sent to the database in one message and compiled into an execution plan. The parameter values are sent in a completely separate message (or as typed binary values) and are treated by the database engine as data — not as SQL syntax.

The database's parser has already finished parsing the SQL when it receives the parameters. It is impossible for a parameter to inject new SQL keywords because the parsing phase is complete. Compare:

```java
// Vulnerable — input becomes part of the SQL string before parsing
String sql = "SELECT * FROM users WHERE name = '" + userInput + "'";
stmt.executeQuery(sql);
// If userInput = "' OR '1'='1", the query becomes:
// SELECT * FROM users WHERE name = '' OR '1'='1'

// Safe — SQL parsed first, then parameter bound as typed data
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM users WHERE name = ?"
);
pstmt.setString(1, userInput);  // sent as a string datum, not SQL
```

---

## Trap 4: ResultSet Is Closed When the Statement Is Closed

**Trap**: Returning a `ResultSet` from a method after the `Statement` has been closed in a `finally` block.

**Reality**: Per the JDBC specification, closing a `Statement` automatically closes any open `ResultSet` objects it produced. If you close your `PreparedStatement` before the caller has finished iterating the `ResultSet`, every subsequent call to `rs.next()` or `rs.getString()` will throw `SQLException: ResultSet is closed`.

Common anti-pattern:

```java
// BUG: ResultSet returned after Statement is closed
public ResultSet findUser(long id) throws SQLException {
    try (PreparedStatement pstmt = conn.prepareStatement("SELECT * FROM users WHERE id=?")) {
        pstmt.setLong(1, id);
        return pstmt.executeQuery(); // ResultSet closed when pstmt closes!
    }
}
```

The fix is to either process the ResultSet inside the same try block, or map it to a domain object before returning.

---

## Trap 5: Connection Pool Exhaustion Symptoms

**Trap**: "The application is slow — must be a database query problem."

**Reality**: Pool exhaustion looks like a query performance problem but is actually a resource problem. Symptoms include:

- All threads blocking on `dataSource.getConnection()` at the same time
- `SQLTimeoutException: Connection is not available, request timed out after 30000ms`
- Thread dumps showing dozens of threads stuck in HikariCP's `getConnection()`
- The database server itself is idle (not busy processing queries)

Common causes: connection leaks (not closing connections), long-running transactions holding connections, pool size too small for the load, and slow queries that hold connections for extended periods.

Diagnosis: enable HikariCP metrics (`hikari.metrics`), check `HikariPoolMXBean.getActiveConnections()` vs `getTotalConnections()`, and enable `leakDetectionThreshold`.

---

## Trap 6: Auto-Commit Mode — Default Is True

**Trap**: "I don't need to call commit() — JDBC handles it."

**Reality**: JDBC connections default to `autoCommit = true`, which means every individual SQL statement is automatically wrapped in its own transaction and immediately committed. This is fine for single-statement operations but catastrophic for multi-step operations that must be atomic.

If you execute three related `UPDATE` statements without disabling auto-commit, and the second one succeeds but the third throws an exception, the first two updates are already permanently committed — you have a half-updated database with no way to roll back.

Always explicitly call `conn.setAutoCommit(false)` when performing multi-statement operations, and always pair it with explicit `commit()` and `rollback()` calls:

```java
conn.setAutoCommit(false);
try {
    debitAccount(conn, fromId, amount);
    creditAccount(conn, toId, amount);
    conn.commit();
} catch (SQLException e) {
    conn.rollback(); // both debit and credit undone
    throw e;
}
```

---

## Trap 7: Savepoints in JDBC Transactions

**Trap**: "You can only rollback the entire transaction."

**Reality**: JDBC supports `Savepoint` objects that let you mark intermediate points within a transaction and roll back only to that point, preserving earlier work in the same transaction.

```java
conn.setAutoCommit(false);
stmt.executeUpdate("INSERT INTO orders ...");           // step 1
Savepoint sp = conn.setSavepoint("after_order");
stmt.executeUpdate("INSERT INTO order_items ...");      // step 2
stmt.executeUpdate("UPDATE inventory ...");             // step 3

// If step 3 fails, we can rollback to sp, preserving step 1:
conn.rollback(sp);         // undoes steps 2 and 3 only
conn.commit();             // commits step 1 only
```

Not all databases support savepoints equally — check your driver's `DatabaseMetaData.supportsSavepoints()`.

---

## Trap 8: BLOB/CLOB Handling in JDBC

**Trap**: "I'll just use `getString()` / `getBytes()` for large data."

**Reality**: For truly large binary or text data, `getString()` and `getBytes()` load the entire value into JVM heap memory at once. For a 500MB file stored as a BLOB, this causes an `OutOfMemoryError`.

The correct approach uses streaming:

```java
// Reading a BLOB as a stream (avoids loading into memory all at once)
Blob blob = rs.getBlob("file_data");
try (InputStream in = blob.getBinaryStream()) {
    Files.copy(in, targetPath);
}

// Writing a BLOB using a stream
try (InputStream in = Files.newInputStream(sourcePath)) {
    pstmt.setBinaryStream(1, in, fileSize);
}

// For CLOB (large text):
Clob clob = rs.getClob("document_text");
try (Reader reader = clob.getCharacterStream()) {
    // read incrementally
}
```

Also: some drivers require you to free BLOB/CLOB resources explicitly with `blob.free()` to release server-side resources.

---

## Trap 9: DatabaseMetaData Use Cases

**Trap**: "DatabaseMetaData is just for framework developers, not application code."

**Reality**: `DatabaseMetaData` provides programmatic introspection of the database at runtime. Practical uses in application code:

```java
DatabaseMetaData meta = conn.getMetaData();

// Check if a table exists before creating it (migration tool):
ResultSet tables = meta.getTables(null, null, "USERS", new String[]{"TABLE"});
boolean exists = tables.next();

// Check database product and version (for dialect-specific SQL):
String product = meta.getDatabaseProductName();   // "PostgreSQL"
String version = meta.getDatabaseProductVersion(); // "14.5"

// Check feature support:
boolean supportsBatch = meta.supportsBatchUpdates();
boolean supportsSavepoints = meta.supportsSavepoints();

// Inspect columns of a table:
ResultSet cols = meta.getColumns(null, null, "ORDERS", null);
while (cols.next()) {
    System.out.println(cols.getString("COLUMN_NAME") + " : " + cols.getString("TYPE_NAME"));
}
```

---

## Trap 10: HikariCP Pool Sizing Formula

**Trap**: "More connections = better performance."

**Reality**: Adding more connections beyond the saturation point makes performance worse, not better. The HikariCP documentation (based on work by Brett Wooldridge) recommends:

```
pool size = (number of CPU cores) × 2 + number of effective spindle disks
```

For a typical server with 4 CPU cores and SSD storage (treated as 1 spindle):
`pool size = 4 × 2 + 1 = 9`

The reasoning: the database server can only execute as many queries in parallel as it has CPU cores. Additional connections beyond that point are simply queuing on the OS scheduler, adding context-switch overhead. For I/O-bound workloads (spinning disks), slightly more connections help because threads spend time waiting for disk, but the formula accounts for that.

For a production Spring Boot app with a 4-core database host:
```properties
spring.datasource.hikari.maximum-pool-size=10
spring.datasource.hikari.minimum-idle=5
```

This is far less than the 100 or 200 that developers often set instinctively.

---

## Trap 11: Statement.RETURN_GENERATED_KEYS

**Trap**: "After an INSERT, I query the table again to get the generated ID."

**Reality**: Querying the table after an insert wastes a round-trip and can return the wrong row in concurrent environments. JDBC provides `Statement.RETURN_GENERATED_KEYS` to retrieve the auto-generated primary key from the same `executeUpdate()` call:

```java
PreparedStatement pstmt = conn.prepareStatement(
    "INSERT INTO users (name, email) VALUES (?, ?)",
    Statement.RETURN_GENERATED_KEYS   // tell driver to return generated keys
);
pstmt.setString(1, "Alice");
pstmt.setString(2, "alice@example.com");
pstmt.executeUpdate();

try (ResultSet keys = pstmt.getGeneratedKeys()) {
    if (keys.next()) {
        long generatedId = keys.getLong(1);
    }
}
```

---

## Trap 12: setFetchSize() and Memory Usage

**Trap**: "JDBC loads one row at a time from the database."

**Reality**: By default, many JDBC drivers fetch all result rows into client memory in a single network operation. For a query returning 1 million rows, this means 1 million rows are loaded into the JVM heap before `rs.next()` is even called once.

`setFetchSize(n)` tells the driver to fetch rows in batches of n from the server:

```java
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM large_table",
    ResultSet.TYPE_FORWARD_ONLY,
    ResultSet.CONCUR_READ_ONLY
);
pstmt.setFetchSize(1000);  // fetch 1000 rows at a time from server
ResultSet rs = pstmt.executeQuery();

// PostgreSQL requires this for true streaming (cursor-based):
conn.setAutoCommit(false);
pstmt.setFetchSize(100);
```

PostgreSQL ignores `setFetchSize()` unless auto-commit is `false` because it uses server-side cursors for streaming, and cursors require an open transaction.
