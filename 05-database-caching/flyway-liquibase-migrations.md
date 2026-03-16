# Flyway & Liquibase Database Migrations

---

## Flyway for Spring Boot

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true
    out-of-order: false
```

### Migration File Naming Convention

```
db/migration/
├── V1__create_users_table.sql
├── V2__create_orders_table.sql
├── V3__add_status_to_orders.sql
├── V4__create_products_table.sql
└── V4_1__add_product_index.sql   ← sub-version

Format: V{version}__{description}.sql
version: 1, 2, 3... or 1.1, 1.2...
```

### Migration SQL Examples

```sql
-- V3__add_status_to_orders.sql
ALTER TABLE orders ADD COLUMN status VARCHAR(50) NOT NULL DEFAULT 'PENDING';
CREATE INDEX idx_orders_status ON orders(status);

-- V5__add_full_text_search.sql
ALTER TABLE products ADD COLUMN search_vector tsvector;
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
UPDATE products SET search_vector = to_tsvector('english', name || ' ' || description);
```

---

## Flyway Best Practices

1. **Never modify existing migrations** — Flyway checksums are validated
2. **Always test migrations on production-size data** — ALTER on 100M rows takes time
3. **Use transactions for DDL when possible** (PostgreSQL supports transactional DDL)
4. **Separate index creation from table changes** — CREATE INDEX CONCURRENTLY doesn't hold lock
5. **Run migrations as separate step before deploying new code** (init container pattern)

---

## Safe Migration Strategies

```sql
-- SAFE: Add nullable column (backward compatible)
ALTER TABLE orders ADD COLUMN notes TEXT;

-- SAFE: Create index concurrently (no lock)
CREATE INDEX CONCURRENTLY idx_orders_notes ON orders(notes);

-- RISKY: Add NOT NULL column with default (locks table while updating all rows)
-- For large tables, do it in steps:
-- Step 1: Add nullable
ALTER TABLE orders ADD COLUMN priority INT;
-- Step 2: Backfill (can be done in batches)
UPDATE orders SET priority = 1 WHERE priority IS NULL;
-- Step 3: Add NOT NULL constraint
ALTER TABLE orders ALTER COLUMN priority SET NOT NULL;
```
