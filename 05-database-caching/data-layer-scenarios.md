# Data Layer Scenarios

---

## Scenario 1: N+1 in Production

**Symptom:** API endpoint takes 3 seconds with 200 orders. Each order triggers a separate DB query for customer info.

**Detection:** Enable Hibernate statistics, see 201 queries for a request that should be 1-2.

**Fix:** Add JOIN FETCH or @EntityGraph to load customers with orders in one query.

---

## Scenario 2: Connection Pool Exhaustion

**Symptom:** Requests timeout with "Unable to acquire JDBC Connection".

**Cause:** Long-running transactions (external API calls inside @Transactional), too many concurrent requests, pool too small.

**Fix:**
1. Move external API calls outside @Transactional
2. Increase pool size carefully (watch DB max connections)
3. Add connection timeout and retry

---

## Scenario 3: Database Migration Failure

**Symptom:** New deployment fails, app won't start. "Flyway migration failed: column already exists".

**Cause:** Migration ran partially on one instance. Other instances also tried to run the same migration.

**Fix:** Flyway uses a lock table (flyway_schema_history) to prevent concurrent migrations. One instance wins the lock. If migration failed, it must be repaired before proceeding. Fix the failing migration SQL and run `flyway repair` to reset the state.

---

## Scenario 4: Optimistic Lock Exception in High-Traffic

**Symptom:** 422 errors from OptimisticLockException on flash sale. Many users trying to buy the last item.

**Fix:** Catch OptimisticLockException and retry, or switch to pessimistic locking for high-contention scenarios. For inventory, use atomic DB operations:
```sql
UPDATE products SET stock = stock - 1 WHERE id = ? AND stock > 0
-- Rows updated = 1: success; 0: out of stock
```

---

## Scenario 5: Slow Full-Text Search

**Symptom:** Product search by name/description is slow on 5M products.

**Fix:** 
- PostgreSQL: Add GIN index with tsvector, use to_tsquery
- Elasticsearch: Dedicated search engine, faster for complex text queries

```sql
-- PostgreSQL full-text search
ALTER TABLE products ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (to_tsvector('english', name || ' ' || coalesce(description, ''))) STORED;
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
SELECT * FROM products WHERE search_vector @@ to_tsquery('english', 'laptop & portable');
```
