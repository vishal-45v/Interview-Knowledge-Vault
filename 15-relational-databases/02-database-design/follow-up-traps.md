# Chapter 02 — Database Design: Follow-Up Traps

Tricky follow-up questions that reveal whether a candidate truly understands
normalization theory, constraint semantics, and real-world design trade-offs.

---

## Trap 1: "A table with a single-column primary key is always at least in 2NF."

**What most people say:** Correct — partial dependencies require a composite key,
so a single-column PK eliminates the possibility.

**Correct answer:** True, but this is often misunderstood as meaning single-column
PK tables are "well-designed." They can still violate 3NF via transitive
dependencies. A table with (employee_id, department_id, department_name) has a
single PK but violates 3NF because department_name depends on department_id,
not on employee_id.

```sql
-- Violates 3NF despite single-column PK
CREATE TABLE employees (
    employee_id   INT PRIMARY KEY,
    department_id INT,
    department_name TEXT,   -- transitive: employee_id → department_id → department_name
    salary        NUMERIC
);
-- Anomaly: if the department is renamed, you update thousands of rows
-- Fix: extract department to its own table
```

---

## Trap 2: "ON DELETE CASCADE is the safest option because it keeps data consistent."

**What most people say:** Yes, CASCADE prevents orphaned records automatically.

**Correct answer:** CASCADE is convenient but can be catastrophically destructive.
Deleting a parent row silently deletes all children, grandchildren, and deeper
descendants. In a schema with deep cascade chains, deleting one customer can
cascade to orders → order_items → shipments → returns — potentially deleting
millions of rows with no warning.

```sql
-- This single DELETE can remove thousands of rows across 5 tables silently:
DELETE FROM customers WHERE id = 42;
-- Cascades to: orders → order_items → shipments → returns → invoices

-- Safer for most business entities:
ON DELETE RESTRICT -- prevents deletion if children exist, forces explicit cleanup

-- Appropriate for CASCADE: ownership relationships (e.g., deleting a draft post
-- should delete its draft images — they have no independent existence)
```

The safest default for business-critical entities is RESTRICT. Use CASCADE only
when the child has no independent existence outside the parent.

---

## Trap 3: "UNIQUE constraints can't have NULL values because NULL violates uniqueness."

**What most people say:** Correct — NULL breaks uniqueness, so UNIQUE columns
must be NOT NULL.

**Correct answer:** Wrong. In standard SQL (and PostgreSQL), NULL is considered
distinct from every other value including other NULLs, so multiple NULLs are
allowed in a UNIQUE column.

```sql
CREATE TABLE users (
    id    SERIAL PRIMARY KEY,
    email TEXT UNIQUE  -- NOT NULL not required
);

INSERT INTO users (email) VALUES (NULL);
INSERT INTO users (email) VALUES (NULL);  -- Also succeeds! Two NULLs are allowed.
INSERT INTO users (email) VALUES ('a@b.com');
INSERT INTO users (email) VALUES ('a@b.com');  -- FAILS: duplicate non-null value
```

Note: MySQL treats this differently from PostgreSQL in some versions. This is a
portability trap. If you want to allow at most one NULL, you need a partial index:

```sql
CREATE UNIQUE INDEX one_null_email ON users (email) WHERE email IS NOT NULL;
-- Now: multiple NULLs still allowed (they're just not indexed as unique)
-- But the above doesn't actually limit NULLs, it just ignores them in the index
```

---

## Trap 4: "Denormalization always means worse data integrity."

**What most people say:** Yes, denormalization introduces redundancy which causes
update anomalies and inconsistency.

**Correct answer:** Partially true, but denormalization with careful write-side
discipline can maintain integrity. The key insight: in many systems, derived or
cached data that is "denormalized" is only written by one controlled path (a
trigger, an event handler, a materialized view refresh), making inconsistency
highly unlikely in practice.

```sql
-- Denormalized: order_total stored on orders table
ALTER TABLE orders ADD COLUMN order_total NUMERIC;

-- Maintain consistency via trigger (not application code)
CREATE OR REPLACE FUNCTION update_order_total()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE orders SET order_total = (
        SELECT SUM(price * quantity) FROM order_items WHERE order_id = NEW.order_id
    ) WHERE id = NEW.order_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER recalc_order_total
AFTER INSERT OR UPDATE OR DELETE ON order_items
FOR EACH ROW EXECUTE FUNCTION update_order_total();
```

The legitimate cases for denormalization: read-heavy reporting columns, columns
used in indexes for performance, columns that make partitioning possible, and
analytics aggregations that are prohibitively expensive to compute live.

---

## Trap 5: "Auto-increment integer IDs are better than UUIDs for primary keys."

**What most people say:** Yes, integers are smaller and faster for joins and index
lookups.

**Correct answer:** It depends on the use case. Integers (BIGINT GENERATED ALWAYS
AS IDENTITY) are smaller (8 bytes vs 16 bytes), produce sequential inserts that
are friendly to B-tree indexes (no page splits at random locations), and are
easier to read in debugging.

UUIDs have critical advantages: they can be generated client-side without a
database round-trip (enabling optimistic inserts), they do not expose row counts
or insertion rate (security benefit), and they are globally unique across shards
and databases without coordination.

```sql
-- UUID v4 (random) — causes index fragmentation due to random inserts
CREATE TABLE events (id UUID DEFAULT gen_random_uuid() PRIMARY KEY, ...);

-- UUID v7 (time-ordered) — monotonically increasing, B-tree friendly
-- Available in PostgreSQL 17 via gen_random_uuid() or pg_uuidv7 extension
-- Best of both worlds: globally unique + sequential

-- BIGINT serial — fast, compact, but leaks information
CREATE TABLE orders (id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY, ...);
```

For distributed systems or microservices sharing IDs across services, UUID v7
is usually the best choice. For single-database OLTP, BIGINT IDENTITY wins on
performance.

---

## Trap 6: "A star schema is just denormalization — it's bad practice for OLTP."

**What most people say:** Yes, normalized schemas are correct and star schemas
are a necessary evil for analytics.

**Correct answer:** Star schemas are purpose-built for analytical query patterns,
not a shortcut. OLTP and OLAP have fundamentally different access patterns:

OLTP: Many concurrent point queries, high writes, row-level access, needs ACID.
OLAP: Few complex queries, low writes, column-level access, needs scan performance.

The star schema's wide, denormalized dimension tables allow the query engine to
avoid many joins, which is critical for queries scanning billions of fact rows.
A normalized snowflake schema forces the query engine to join the product dimension
to the category dimension to the brand dimension for every row scanned — massively
more expensive for full-table analytics.

```sql
-- Star schema: one join from fact to dimension (fast for analytics)
SELECT
    d.year, d.quarter,
    p.category,
    SUM(f.revenue) AS total_revenue
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
GROUP BY d.year, d.quarter, p.category;

-- If normalized (snowflake): dim_product → dim_category → dim_brand
-- Three joins per row scanned instead of one
```

---

## Trap 7: "Soft deletes are straightforward — just add a deleted_at column."

**What most people say:** Yes, just filter WHERE deleted_at IS NULL in all queries.

**Correct answer:** Soft deletes have serious cascading engineering problems:

1. Every query must include `WHERE deleted_at IS NULL` — one missing filter causes
   "ghost data" to appear. This is extremely error-prone at scale.
2. UNIQUE constraints no longer work correctly:
```sql
-- Can't have two active rows with same email, but can have one active + one deleted
-- Standard UNIQUE constraint prevents this entirely
CREATE UNIQUE INDEX active_users_email ON users (email) WHERE deleted_at IS NULL;
-- Partial unique index is the fix
```
3. Foreign keys pointing to soft-deleted rows are not caught by the database — the
   row still physically exists, so FK constraints succeed even though the logical
   entity is "deleted."
4. Index bloat: soft-deleted rows remain in all indexes, inflating them without
   providing benefit to live-data queries.
5. The `deleted_at IS NULL` condition on a large table has low selectivity if many
   rows are deleted, making the partial index approach on `deleted_at IS NULL`
   much smaller than a full index.

Better alternatives for auditability: append-only event log, temporal tables,
or physical delete + separate archive table.

---

## Trap 8: "CHECK constraints enforce business rules at the database level, which
is always better than application-level validation."

**What most people say:** Yes, database constraints are more reliable than
application code.

**Correct answer:** CHECK constraints are valuable but have significant limitations:
1. They cannot reference other rows or other tables (no cross-row validation).
2. They cannot call user-defined functions that access tables (in standard SQL;
   PostgreSQL allows it but it can cause subtle issues under MVCC).
3. They run on every INSERT and UPDATE, which has a small but real cost.
4. Schema changes to CHECK constraints require ALTER TABLE, which may lock the
   table.

```sql
-- Works: single-row constraint
ALTER TABLE orders ADD CONSTRAINT positive_amount CHECK (amount > 0);

-- Does NOT work: cross-row constraint (can't check other rows)
-- "Ensure no two overlapping reservations for same room" → CANNOT be a CHECK
-- This requires a trigger or an exclusion constraint (PostgreSQL-specific):
ALTER TABLE reservations
ADD CONSTRAINT no_overlap
EXCLUDE USING gist (room_id WITH =, period WITH &&);
-- Uses the PostgreSQL range type and exclusion operator
```

Exclusion constraints are PostgreSQL's answer to cross-row integrity rules that
CHECK cannot handle.

---

## Trap 9: "ON DELETE RESTRICT and ON DELETE NO ACTION are the same thing."

**What most people say:** Yes, both prevent deletion when child rows exist.

**Correct answer:** The timing of constraint checking differs:

- **RESTRICT:** Checks the constraint immediately, before the end of the statement.
- **NO ACTION:** Checks the constraint at the end of the statement (or at transaction
  end if the constraint is DEFERRABLE).

This matters for complex UPDATE or DELETE statements that modify both parent and
child rows in one statement.

```sql
-- DEFERRABLE constraint: checked at COMMIT time, not at each statement
ALTER TABLE order_items ADD CONSTRAINT fk_order
    FOREIGN KEY (order_id) REFERENCES orders(id)
    ON DELETE NO ACTION
    DEFERRABLE INITIALLY DEFERRED;

-- Now you can temporarily violate the constraint within a transaction
BEGIN;
DELETE FROM orders WHERE id = 100;  -- Would fail with RESTRICT
INSERT INTO orders (id) VALUES (100);  -- Recreate it
-- At COMMIT time: FK is satisfied, no violation
COMMIT;
```

RESTRICT is the safe default. DEFERRABLE NO ACTION is for complex data migration
or circular foreign key scenarios.
