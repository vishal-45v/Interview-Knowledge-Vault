# JDBC — Scenario Questions

> 14 real-world scenarios on JDBC operations, connection pooling, and database integration patterns.

---

## Scenario 1: Bulk Import with Batch Insert

**Situation:** Import 1 million product records from CSV to PostgreSQL efficiently.

```java
@Service
public class ProductBulkImporter {

    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private DataSource dataSource;

    public ImportResult importProducts(List<ProductCsvRecord> records) throws SQLException {
        long imported = 0;
        List<String> errors = new ArrayList<>();
        int batchSize = 1000;

        String sql = "INSERT INTO products (sku, name, price, category, active) " +
                     "VALUES (?, ?, ?, ?, ?) " +
                     "ON CONFLICT (sku) DO UPDATE SET " +  // Upsert
                     "name = EXCLUDED.name, price = EXCLUDED.price, " +
                     "category = EXCLUDED.category, updated_at = NOW()";

        try (Connection conn = dataSource.getConnection()) {
            conn.setAutoCommit(false);
            PreparedStatement ps = conn.prepareStatement(sql);
            int count = 0;

            for (ProductCsvRecord record : records) {
                try {
                    ps.setString(1, record.getSku());
                    ps.setString(2, record.getName());
                    ps.setBigDecimal(3, record.getPrice());
                    ps.setString(4, record.getCategory());
                    ps.setBoolean(5, record.isActive());
                    ps.addBatch();

                    if (++count % batchSize == 0) {
                        int[] results = ps.executeBatch();
                        imported += results.length;
                        conn.commit();
                        ps.clearBatch();
                        log.info("Imported {} products", imported);
                    }
                } catch (Exception e) {
                    errors.add("SKU " + record.getSku() + ": " + e.getMessage());
                }
            }

            // Final batch
            int[] results = ps.executeBatch();
            imported += results.length;
            conn.commit();
        }

        return new ImportResult(imported, errors);
    }
}
```

---

## Scenario 2: Connection Pool Exhaustion Diagnosis

**Situation:** Under high load, HikariCP throws "Connection is not available, request timed out after 5000ms." Diagnose and fix.

```java
// Step 1: Add metrics monitoring
@Component
public class HikariPoolMonitor {

    @Autowired private DataSource dataSource;

    @Scheduled(fixedRate = 5000)
    public void logPoolStats() {
        if (dataSource instanceof HikariDataSource hikari) {
            HikariPoolMXBean pool = hikari.getHikariPoolMXBean();
            log.info("Pool stats: active={}, idle={}, waiting={}, total={}",
                pool.getActiveConnections(),
                pool.getIdleConnections(),
                pool.getThreadsAwaitingConnection(),
                pool.getTotalConnections());
        }
    }
}

// Step 2: Enable Hikari logging in application.yml
// logging:
//   level:
//     com.zaxxer.hikari: DEBUG

// Step 3: Check what's holding connections in PostgreSQL
@Repository
public class PoolDiagnostics {
    public List<Map<String, Object>> getActiveConnections() {
        return jdbcTemplate.queryForList(
            "SELECT pid, state, wait_event, query_start, left(query, 100) as query " +
            "FROM pg_stat_activity " +
            "WHERE datname = current_database() AND state != 'idle' " +
            "ORDER BY query_start");
    }
}

// Common fixes:
// 1. Long @Transactional methods holding connection while making external API calls
// 2. Pool size too small: max-pool-size < (concurrent threads × connections per thread)
// 3. Slow queries holding connections
// 4. Connection leaks (not returned to pool)

// Fix for external API calls within transaction:
@Transactional
public Order createOrder(OrderRequest request) {
    // WRONG: makes external call while holding DB connection!
    inventoryService.reserve(request.getProductId()); // HTTP call → slow → connection held

    return orderRepository.save(new Order(request));
}

// Better: minimize transaction scope
public Order createOrder(OrderRequest request) {
    // Reserve inventory OUTSIDE transaction
    inventoryService.reserve(request.getProductId());

    // Short transaction for just the DB write
    return orderTransactionService.saveOrder(new Order(request));
}

@Transactional
public Order saveOrder(Order order) {
    return orderRepository.save(order);
}
```

---

## Scenario 3: Optimistic Locking with JDBC

**Situation:** Two users might update the same order simultaneously. Implement optimistic locking without JPA.

```java
@Repository
public class OrderRepository {

    @Autowired private JdbcTemplate jdbcTemplate;

    public Order findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT id, status, total, version FROM orders WHERE id = ?",
            (rs, rowNum) -> new Order(
                rs.getLong("id"),
                rs.getString("status"),
                rs.getBigDecimal("total"),
                rs.getInt("version")  // Version column for optimistic locking
            ),
            id);
    }

    public void update(Order order) {
        int rowsUpdated = jdbcTemplate.update(
            "UPDATE orders SET status = ?, total = ?, version = version + 1 " +
            "WHERE id = ? AND version = ?",  // Only update if version matches
            order.getStatus(), order.getTotal(),
            order.getId(), order.getVersion()  // Compare with current version
        );

        if (rowsUpdated == 0) {
            // Either order doesn't exist or was modified by another transaction
            throw new OptimisticLockException(
                "Order " + order.getId() + " was modified concurrently");
        }
    }
}

// Migration to add version column:
// ALTER TABLE orders ADD COLUMN version INT DEFAULT 0 NOT NULL;
```

---

## Scenario 4: Generic JDBC Repository

**Situation:** Build a reusable RowMapper that maps ResultSet columns to POJO fields by name.

```java
public class AutoRowMapper<T> implements RowMapper<T> {

    private final Class<T> targetClass;
    // Cache field lookup for performance
    private final Map<String, Field> fieldsByColumnName;

    public AutoRowMapper(Class<T> clazz) {
        this.targetClass = clazz;
        this.fieldsByColumnName = Arrays.stream(clazz.getDeclaredFields())
            .peek(f -> f.setAccessible(true))
            .collect(Collectors.toMap(
                f -> {
                    Column col = f.getAnnotation(Column.class);
                    return col != null ? col.name() : camelToSnake(f.getName());
                },
                f -> f
            ));
    }

    @Override
    public T mapRow(ResultSet rs, int rowNum) throws SQLException {
        try {
            T instance = targetClass.getDeclaredConstructor().newInstance();
            ResultSetMetaData meta = rs.getMetaData();

            for (int i = 1; i <= meta.getColumnCount(); i++) {
                String colName = meta.getColumnName(i).toLowerCase();
                Field field = fieldsByColumnName.get(colName);
                if (field != null) {
                    Object value = rs.getObject(i);
                    if (rs.wasNull()) value = null;
                    field.set(instance, convertValue(value, field.getType()));
                }
            }
            return instance;
        } catch (Exception e) {
            throw new SQLException("Failed to map row to " + targetClass.getSimpleName(), e);
        }
    }

    private static String camelToSnake(String camelCase) {
        return camelCase.replaceAll("([A-Z])", "_$1").toLowerCase();
    }

    private Object convertValue(Object value, Class<?> targetType) {
        if (value == null) return null;
        if (targetType.isInstance(value)) return value;
        if (targetType == Long.class && value instanceof Integer i) return i.longValue();
        if (targetType == Integer.class && value instanceof Long l) return l.intValue();
        return value;
    }
}

// Usage
RowMapper<Order> mapper = new AutoRowMapper<>(Order.class);
List<Order> orders = jdbcTemplate.query("SELECT * FROM orders", mapper);
```

---

## Scenario 5: Database Health Check

**Situation:** Implement a comprehensive database health check for Spring Boot Actuator.

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired private DataSource dataSource;
    @Autowired private JdbcTemplate jdbcTemplate;

    @Override
    public Health health() {
        try {
            // 1. Check connectivity with timeout
            try (Connection conn = dataSource.getConnection()) {
                if (!conn.isValid(3)) {  // 3 second timeout
                    return Health.down()
                        .withDetail("error", "Connection is not valid")
                        .build();
                }
            }

            // 2. Check query execution
            long start = System.nanoTime();
            Integer result = jdbcTemplate.queryForObject("SELECT 1", Integer.class);
            long queryTime = (System.nanoTime() - start) / 1_000_000;

            // 3. Check pool metrics
            Map<String, Object> poolInfo = new LinkedHashMap<>();
            if (dataSource instanceof HikariDataSource hds) {
                HikariPoolMXBean pool = hds.getHikariPoolMXBean();
                poolInfo.put("active", pool.getActiveConnections());
                poolInfo.put("idle", pool.getIdleConnections());
                poolInfo.put("waiting", pool.getThreadsAwaitingConnection());
                poolInfo.put("total", pool.getTotalConnections());
                poolInfo.put("max", hds.getMaximumPoolSize());

                if (pool.getThreadsAwaitingConnection() > 5) {
                    return Health.down()
                        .withDetail("error", "Connection pool under pressure")
                        .withDetails(poolInfo)
                        .build();
                }
            }

            return Health.up()
                .withDetail("queryTime", queryTime + "ms")
                .withDetails(poolInfo)
                .build();

        } catch (Exception e) {
            return Health.down(e)
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

---

## Scenario 6: Streaming Large Result Set

**Situation:** Export 5 million orders to CSV. Each order has relationships to load. Do it without OOM.

```java
@Service
public class OrderExportService {

    @Autowired private JdbcTemplate jdbcTemplate;

    // Stream results to OutputStream — never loads all into memory
    public void exportToCsv(OutputStream outputStream,
                             Instant fromDate, Instant toDate) throws IOException {

        try (PrintWriter writer = new PrintWriter(
                new BufferedWriter(new OutputStreamWriter(outputStream, StandardCharsets.UTF_8)))) {

            // Header
            writer.println("order_id,customer_email,total,status,created_at,item_count");

            // Stream from DB in chunks using fetch size
            jdbcTemplate.query(
                "SELECT o.id, c.email, o.total, o.status, o.created_at, " +
                "       COUNT(oi.id) as item_count " +
                "FROM orders o " +
                "JOIN customers c ON c.id = o.customer_id " +
                "LEFT JOIN order_items oi ON oi.order_id = o.id " +
                "WHERE o.created_at BETWEEN ? AND ? " +
                "GROUP BY o.id, c.email, o.total, o.status, o.created_at " +
                "ORDER BY o.id",
                new Object[]{Timestamp.from(fromDate), Timestamp.from(toDate)},
                pstmt -> pstmt.setFetchSize(1000),  // Fetch 1000 rows at a time
                rs -> {
                    try {
                        writer.printf("%s,%s,%.2f,%s,%s,%d%n",
                            rs.getLong("id"),
                            escapeCsv(rs.getString("email")),
                            rs.getDouble("total"),
                            rs.getString("status"),
                            rs.getTimestamp("created_at").toInstant(),
                            rs.getInt("item_count"));

                        if (writer.checkError()) throw new IOException("Writer error");
                    } catch (IOException e) {
                        throw new RuntimeException(e);
                    }
                }
            );
        }
    }

    private String escapeCsv(String value) {
        if (value == null) return "";
        if (value.contains(",") || value.contains("\"") || value.contains("\n")) {
            return "\"" + value.replace("\"", "\"\"") + "\"";
        }
        return value;
    }
}
```

---

## Scenario 7: Database Migration with JDBC

**Situation:** Implement a lightweight migration runner that tracks applied migrations.

```java
@Component
public class DatabaseMigrationRunner {

    @Autowired private JdbcTemplate jdbcTemplate;

    @PostConstruct
    public void runMigrations() throws IOException {
        ensureMigrationTable();
        Set<String> applied = getAppliedMigrations();

        List<Path> migrations = findMigrationFiles();
        for (Path migrationFile : migrations) {
            String filename = migrationFile.getFileName().toString();
            if (!applied.contains(filename)) {
                applyMigration(migrationFile, filename);
            }
        }
    }

    private void ensureMigrationTable() {
        jdbcTemplate.execute("""
            CREATE TABLE IF NOT EXISTS schema_migrations (
                filename VARCHAR(255) PRIMARY KEY,
                applied_at TIMESTAMP NOT NULL DEFAULT NOW(),
                checksum VARCHAR(64) NOT NULL
            )""");
    }

    private Set<String> getAppliedMigrations() {
        return new HashSet<>(
            jdbcTemplate.queryForList("SELECT filename FROM schema_migrations", String.class));
    }

    @Transactional
    public void applyMigration(Path file, String filename) throws IOException {
        String sql = Files.readString(file, StandardCharsets.UTF_8);
        String checksum = computeChecksum(sql);

        log.info("Applying migration: {}", filename);
        jdbcTemplate.execute(sql);

        jdbcTemplate.update(
            "INSERT INTO schema_migrations (filename, checksum) VALUES (?, ?)",
            filename, checksum);

        log.info("Applied migration: {}", filename);
    }

    private List<Path> findMigrationFiles() throws IOException {
        return Files.list(Path.of("classpath:db/migrations"))
            .filter(p -> p.toString().endsWith(".sql"))
            .sorted()  // Lexicographic order = timestamp order (V001_, V002_...)
            .collect(Collectors.toList());
    }
}
```

---

## Scenario 8: Tenant-Aware Data Source

**Situation:** Multi-tenant SaaS app — each tenant has a separate database schema. Route connections based on tenant context.

```java
// AbstractRoutingDataSource routes to different datasources based on lookup key
@Component
public class TenantRoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        // Read tenant ID from ThreadLocal (set in request filter)
        return TenantContext.getCurrentTenantId();
    }
}

// Tenant context (ThreadLocal-based)
public class TenantContext {
    private static final ThreadLocal<String> currentTenant = new ThreadLocal<>();

    public static void setCurrentTenant(String tenantId) {
        currentTenant.set(tenantId);
    }

    public static String getCurrentTenantId() {
        return currentTenant.get();
    }

    public static void clear() {
        currentTenant.remove();
    }
}

// Filter to set tenant from JWT or subdomain
@Component
@Order(1)
public class TenantFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String tenantId = extractTenantId(request);
        TenantContext.setCurrentTenant(tenantId);
        try {
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();  // CRITICAL: prevent leaking to next request
        }
    }

    private String extractTenantId(HttpServletRequest request) {
        // Extract from subdomain: tenant1.app.com → "tenant1"
        String host = request.getServerName();
        return host.split("\\.")[0];
        // Or from JWT claim, header, etc.
    }
}
```

---

## Scenario 9: Custom ResultSet Processor for Analytics

**Situation:** Execute dynamic analytics queries and return results as a generic data structure.

```java
@Service
public class AnalyticsQueryService {

    @Autowired private JdbcTemplate jdbcTemplate;

    // Execute any SELECT and return as list of maps
    public QueryResult executeAnalyticsQuery(AnalyticsQuery query) {
        // Validate query (must be SELECT only)
        String sql = query.getSql().trim().toLowerCase();
        if (!sql.startsWith("select")) {
            throw new SecurityException("Only SELECT queries allowed in analytics");
        }

        List<Map<String, Object>> rows = new ArrayList<>();
        List<String> columns = new ArrayList<>();

        jdbcTemplate.query(
            query.getSql(),
            query.getParams().toArray(),
            pstmt -> pstmt.setFetchSize(10000),
            rs -> {
                if (columns.isEmpty()) {
                    ResultSetMetaData meta = rs.getMetaData();
                    for (int i = 1; i <= meta.getColumnCount(); i++) {
                        columns.add(meta.getColumnLabel(i));
                    }
                }

                Map<String, Object> row = new LinkedHashMap<>();
                for (String col : columns) {
                    Object val = rs.getObject(col);
                    // Normalize types for JSON serialization
                    if (val instanceof Timestamp ts) val = ts.toInstant();
                    if (val instanceof java.sql.Date d) val = d.toLocalDate();
                    row.put(col, rs.wasNull() ? null : val);
                }
                rows.add(row);
            }
        );

        return new QueryResult(columns, rows);
    }
}
```

---

## Scenario 10: JDBC with Stored Procedures

**Situation:** Call a stored procedure that calculates order totals and returns multiple result sets.

```java
@Repository
public class OrderStoredProcRepository {

    @Autowired private DataSource dataSource;

    public OrderSummary calculateOrderSummary(Long orderId) throws SQLException {
        try (Connection conn = dataSource.getConnection();
             CallableStatement cs = conn.prepareCall(
                "{CALL calculate_order_summary(?, ?, ?, ?)}")) {

            cs.setLong(1, orderId);           // IN parameter
            cs.registerOutParameter(2, Types.DECIMAL);   // OUT: subtotal
            cs.registerOutParameter(3, Types.DECIMAL);   // OUT: tax
            cs.registerOutParameter(4, Types.DECIMAL);   // OUT: total

            cs.execute();

            return new OrderSummary(
                cs.getBigDecimal(2),  // subtotal
                cs.getBigDecimal(3),  // tax
                cs.getBigDecimal(4)   // total
            );
        }
    }

    // Spring SimpleJdbcCall — cleaner for stored procedures
    public OrderSummary calculateViaSimpleJdbcCall(Long orderId) {
        SimpleJdbcCall call = new SimpleJdbcCall(dataSource)
            .withProcedureName("calculate_order_summary")
            .declareParameters(
                new SqlParameter("orderId", Types.BIGINT),
                new SqlOutParameter("subtotal", Types.DECIMAL),
                new SqlOutParameter("tax", Types.DECIMAL),
                new SqlOutParameter("total", Types.DECIMAL)
            );

        SqlParameterSource in = new MapSqlParameterSource("orderId", orderId);
        Map<String, Object> out = call.execute(in);

        return new OrderSummary(
            (BigDecimal) out.get("subtotal"),
            (BigDecimal) out.get("tax"),
            (BigDecimal) out.get("total")
        );
    }
}
```

---

## Scenario 11: Connection Validation and Retry

**Situation:** Implement a data access layer that retries on transient connection failures.

```java
@Service
public class ResilientJdbcService {

    @Autowired private JdbcTemplate jdbcTemplate;

    private static final int MAX_RETRIES = 3;
    private static final long RETRY_DELAY_MS = 1000;

    public <T> T executeWithRetry(Supplier<T> operation) {
        int attempt = 0;
        Exception lastException = null;

        while (attempt < MAX_RETRIES) {
            try {
                return operation.get();
            } catch (DataAccessResourceFailureException e) {
                // Transient: connection refused, network error
                lastException = e;
                attempt++;
                log.warn("DB operation failed (attempt {}/{}): {}",
                          attempt, MAX_RETRIES, e.getMessage());

                if (attempt < MAX_RETRIES) {
                    try { Thread.sleep(RETRY_DELAY_MS * attempt); }
                    catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException("Interrupted during retry", ie);
                    }
                }
            } catch (DataIntegrityViolationException e) {
                throw e;  // Not transient — don't retry
            }
        }

        throw new ServiceUnavailableException(
            "DB operation failed after " + MAX_RETRIES + " attempts", lastException);
    }

    public Order findOrder(Long id) {
        return executeWithRetry(() ->
            jdbcTemplate.queryForObject(
                "SELECT * FROM orders WHERE id = ?",
                orderRowMapper, id)
        );
    }
}
```

---

## Scenario 12: Pagination with Total Count

**Situation:** Implement server-side pagination that returns both the page data and the total count.

```java
@Repository
public class OrderPaginationRepository {

    @Autowired private NamedParameterJdbcTemplate namedTemplate;

    public Page<Order> findOrders(OrderSearchCriteria criteria, Pageable pageable) {
        MapSqlParameterSource params = new MapSqlParameterSource();

        // Build dynamic WHERE clause
        StringBuilder where = new StringBuilder("WHERE 1=1");
        if (criteria.getStatus() != null) {
            where.append(" AND status = :status");
            params.addValue("status", criteria.getStatus());
        }
        if (criteria.getCustomerId() != null) {
            where.append(" AND customer_id = :customerId");
            params.addValue("customerId", criteria.getCustomerId());
        }
        if (criteria.getFromDate() != null) {
            where.append(" AND created_at >= :fromDate");
            params.addValue("fromDate", criteria.getFromDate());
        }

        // Count query (no LIMIT/OFFSET/ORDER BY)
        long total = namedTemplate.queryForObject(
            "SELECT COUNT(*) FROM orders " + where,
            params, Long.class);

        // Data query with pagination
        String orderBy = "ORDER BY " + sanitizeOrderBy(pageable.getSort());
        params.addValue("limit", pageable.getPageSize());
        params.addValue("offset", pageable.getOffset());

        List<Order> orders = namedTemplate.query(
            "SELECT * FROM orders " + where + " " + orderBy + " LIMIT :limit OFFSET :offset",
            params, orderRowMapper);

        return new PageImpl<>(orders, pageable, total);
    }

    // SECURITY: whitelist allowed sort columns to prevent SQL injection
    private static final Set<String> ALLOWED_SORT_COLUMNS = Set.of(
        "id", "created_at", "total", "status", "customer_id");

    private String sanitizeOrderBy(Sort sort) {
        if (sort.isUnsorted()) return "id ASC";
        return sort.stream()
            .filter(order -> ALLOWED_SORT_COLUMNS.contains(order.getProperty()))
            .map(order -> order.getProperty() + " " + order.getDirection())
            .collect(Collectors.joining(", "));
    }
}
```

---

## Scenario 13: Cross-Database Transaction with XA

**Situation:** You need a transaction spanning PostgreSQL and MySQL (two different databases).

```java
// XA (2-phase commit) with Atomikos
@Configuration
public class XaTransactionConfig {

    @Bean
    public XADataSource postgresXaDataSource() {
        PGXADataSource ds = new PGXADataSource();
        ds.setUrl("jdbc:postgresql://postgres:5432/orders");
        ds.setUser("app");
        ds.setPassword(password);
        return ds;
    }

    @Bean
    public XADataSource mysqlXaDataSource() {
        MysqlXADataSource ds = new MysqlXADataSource();
        ds.setUrl("jdbc:mysql://mysql:3306/inventory");
        ds.setUser("app");
        ds.setPassword(password);
        return ds;
    }

    @Bean
    public UserTransactionManager atomikosTransactionManager() throws SystemException {
        UserTransactionManager utm = new UserTransactionManager();
        utm.setForceShutdown(false);
        return utm;
    }

    @Bean
    public JtaTransactionManager transactionManager() throws SystemException {
        return new JtaTransactionManager(atomikosTransactionManager());
    }
}

// Usage — @Transactional spans both databases!
@Transactional  // Uses JTA transaction manager
public void processOrder(Order order) {
    orderRepository.save(order);             // Writes to PostgreSQL
    inventoryRepository.reserve(order);      // Writes to MySQL
    // Both committed or both rolled back
}
```

---

## Scenario 14: SQL Performance Monitoring

**Situation:** Log slow SQL queries and track statistics for performance tuning.

```java
// Custom DataSource proxy that times queries
public class MonitoringDataSource implements DataSource {

    private final DataSource delegate;
    private final long slowQueryThresholdMs;

    @Override
    public Connection getConnection() throws SQLException {
        return new MonitoringConnection(delegate.getConnection(), slowQueryThresholdMs);
    }

    // Proxy Connection that wraps PreparedStatement
    private static class MonitoringConnection implements Connection {
        @Override
        public PreparedStatement prepareStatement(String sql) throws SQLException {
            return new MonitoringPreparedStatement(
                delegate.prepareStatement(sql), sql, slowQueryThresholdMs);
        }
    }

    // Proxy PreparedStatement that times execution
    private static class MonitoringPreparedStatement implements PreparedStatement {
        @Override
        public ResultSet executeQuery() throws SQLException {
            long start = System.nanoTime();
            try {
                ResultSet rs = delegate.executeQuery();
                long elapsedMs = (System.nanoTime() - start) / 1_000_000;
                recordQueryTime(sql, elapsedMs);
                return rs;
            } catch (SQLException e) {
                recordQueryError(sql);
                throw e;
            }
        }

        private void recordQueryTime(String sql, long elapsedMs) {
            metrics.timer("jdbc.query.duration").record(elapsedMs, TimeUnit.MILLISECONDS);
            if (elapsedMs > slowQueryThresholdMs) {
                log.warn("SLOW QUERY ({}ms): {}", elapsedMs, sql);
            }
        }
    }
}
```
