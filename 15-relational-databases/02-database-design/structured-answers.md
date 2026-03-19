# Chapter 02 — Database Design: Structured Answers

Complete, interview-ready answers for the most important database design questions
at the senior engineer / database architect level.

---

## Q1: Walk me through normalization from 1NF to BCNF with a real example.

**Answer:**

Start with a denormalized "orders" table that captures everything in one place:

```sql
-- Unnormalized: order_id, customer_name, customer_city, product_ids (comma list),
-- product_names (comma list), quantities, product_prices
CREATE TABLE orders_raw (
    order_id       INT,
    customer_name  TEXT,
    customer_city  TEXT,
    product_ids    TEXT,  -- '101,102,103' -- violates 1NF
    product_names  TEXT,  -- 'Pen,Notebook,Bag'
    quantities     TEXT,  -- '2,1,1'
    prices         TEXT   -- '1.50,5.00,25.00'
);
```

**First Normal Form (1NF):** Every column must be atomic (one value per cell).
No repeating groups. Every row must be uniquely identifiable.

```sql
-- 1NF: expand product lists into rows, add PK
CREATE TABLE orders_1nf (
    order_id      INT,
    customer_name TEXT,
    customer_city TEXT,
    product_id    INT,
    product_name  TEXT,
    quantity      INT,
    price         NUMERIC,
    PRIMARY KEY (order_id, product_id)
);
-- Functional dependencies identified:
-- order_id → customer_name, customer_city
-- product_id → product_name, price
-- (order_id, product_id) → quantity
```

**Second Normal Form (2NF):** No partial dependencies on the composite PK.
`customer_name` depends only on `order_id` (partial). `product_name` and `price`
depend only on `product_id` (partial). Only `quantity` depends on the full
composite key.

```sql
-- 2NF: remove partial dependencies
CREATE TABLE customers (id INT PRIMARY KEY, name TEXT, city TEXT);
CREATE TABLE products  (id INT PRIMARY KEY, name TEXT, price NUMERIC);
CREATE TABLE orders    (order_id INT PRIMARY KEY, customer_id INT REFERENCES customers);
CREATE TABLE order_items (
    order_id   INT REFERENCES orders,
    product_id INT REFERENCES products,
    quantity   INT,
    PRIMARY KEY (order_id, product_id)
);
```

**Third Normal Form (3NF):** No transitive dependencies. If `customer_city`
depends on `customer_id` which depends on `order_id`, and `city` is really a
property of the customer (not the order), that's transitive. Already resolved
above by putting `city` on the customers table. A new example:

```sql
-- Suppose products has: product_id, category_id, category_name
-- category_name depends on category_id, not product_id → transitive dependency
-- Fix:
CREATE TABLE categories (id INT PRIMARY KEY, name TEXT);
ALTER TABLE products ADD COLUMN category_id INT REFERENCES categories;
ALTER TABLE products DROP COLUMN category_name;
```

**Boyce-Codd Normal Form (BCNF):** For every functional dependency X → Y, X must
be a superkey. BCNF is stricter than 3NF. Example of 3NF but not BCNF:

```sql
-- Teaching assignments: (student, course) → professor, (professor) → course
-- Two candidate keys: (student, course) and (student, professor)
-- Dependency: professor → course violates BCNF because professor is not a superkey

-- BCNF decomposition:
CREATE TABLE professor_courses (professor TEXT PRIMARY KEY, course TEXT);
CREATE TABLE student_professors (student TEXT, professor TEXT,
    PRIMARY KEY (student, professor));
-- Now every determinant is a superkey in its table
```

---

## Q2: Explain the star schema vs snowflake schema trade-off for analytics workloads.

**Answer:**

Both are dimensional modeling patterns for data warehouses and analytical schemas.
The difference is in how dimension tables are structured.

**Star Schema:** Dimension tables are flat and denormalized. Every attribute of
a dimension lives in one wide table. One join from fact table to any dimension.

```sql
-- Star schema
CREATE TABLE fact_sales (
    sale_id     BIGINT PRIMARY KEY,
    date_key    INT REFERENCES dim_date,
    product_key INT REFERENCES dim_product,
    customer_key INT REFERENCES dim_customer,
    store_key   INT REFERENCES dim_store,
    revenue     NUMERIC,
    quantity    INT
);

-- Wide, flat, denormalized dimension
CREATE TABLE dim_product (
    product_key      INT PRIMARY KEY,
    product_id       INT,
    product_name     TEXT,
    category_name    TEXT,   -- denormalized from categories table
    subcategory_name TEXT,   -- denormalized
    brand_name       TEXT,   -- denormalized
    supplier_name    TEXT    -- denormalized
);
```

**Snowflake Schema:** Dimension tables are normalized. Categories are in their
own table, brands in their own table, etc.

```sql
-- Snowflake schema
CREATE TABLE dim_product_sf (
    product_key  INT PRIMARY KEY,
    product_name TEXT,
    category_key INT REFERENCES dim_category,
    brand_key    INT REFERENCES dim_brand
);
CREATE TABLE dim_category (category_key INT PRIMARY KEY, name TEXT, parent_category_key INT);
CREATE TABLE dim_brand    (brand_key INT PRIMARY KEY, name TEXT, supplier_key INT);
```

**When to choose Star:**
- Query performance is the top priority.
- ETL writes are less frequent than reads.
- Business users write their own queries — fewer joins means less confusion.
- Column-store engines (Redshift, BigQuery) benefit from fewer joins.

**When to choose Snowflake:**
- Storage is constrained and dimension tables are very large.
- Dimension attributes change frequently (normalization reduces update scope).
- The ETL pipeline already has normalized staging tables.

**Performance reality:** Most modern columnar analytics engines favor star schemas
because they can prune columns efficiently from wide tables, and the cost of an
extra join with a dictionary-encoded column is non-trivial at billion-row scale.

---

## Q3: Design a complete audit trail system. What are the trade-offs of each approach?

**Answer:**

There are four main patterns, each with different trade-offs:

**Pattern 1: Audit Columns on Every Table**
```sql
ALTER TABLE orders ADD COLUMN created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW();
ALTER TABLE orders ADD COLUMN updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW();
ALTER TABLE orders ADD COLUMN created_by  TEXT NOT NULL DEFAULT current_user;
ALTER TABLE orders ADD COLUMN updated_by  TEXT NOT NULL DEFAULT current_user;

-- Enforce updated_at via trigger (don't trust application code)
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER orders_updated_at
BEFORE UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```
Tells you WHEN and WHO but not WHAT changed. Sufficient for basic compliance.

**Pattern 2: Separate Audit Table (Shadow Table)**
```sql
CREATE TABLE orders_audit (
    audit_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    audit_time  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    audit_user  TEXT NOT NULL DEFAULT current_user,
    audit_op    CHAR(1) NOT NULL CHECK (audit_op IN ('I','U','D')),
    order_id    INT,
    old_status  TEXT,
    new_status  TEXT,
    old_amount  NUMERIC,
    new_amount  NUMERIC
    -- ... all audited columns
);

CREATE OR REPLACE FUNCTION audit_orders()
RETURNS TRIGGER AS $$
BEGIN
    IF (TG_OP = 'UPDATE') THEN
        INSERT INTO orders_audit (audit_op, order_id, old_status, new_status, old_amount, new_amount)
        VALUES ('U', OLD.id, OLD.status, NEW.status, OLD.amount, NEW.amount);
    ELSIF (TG_OP = 'INSERT') THEN
        INSERT INTO orders_audit (audit_op, order_id, new_status, new_amount)
        VALUES ('I', NEW.id, NEW.status, NEW.amount);
    ELSIF (TG_OP = 'DELETE') THEN
        INSERT INTO orders_audit (audit_op, order_id, old_status, old_amount)
        VALUES ('D', OLD.id, OLD.status, OLD.amount);
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```
Captures WHAT changed. Schema changes to main table require parallel audit table
changes — high maintenance burden.

**Pattern 3: Generic JSONB Audit Log**
```sql
CREATE TABLE audit_log (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    table_name  TEXT NOT NULL,
    row_id      TEXT NOT NULL,
    op          CHAR(1) NOT NULL,
    changed_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    changed_by  TEXT,
    old_data    JSONB,
    new_data    JSONB,
    diff        JSONB  -- computed as new_data - old_data keys where values differ
);
```
Most flexible — one table for all audit data. Schema changes to source tables
are automatically captured. Queries for specific fields require JSONB operators
which are slower than typed columns. Best for compliance where you mostly need
"what was the state at time T" rather than "find all rows where status changed."

**Pattern 4: Temporal Tables (System-Versioned)**
```sql
-- PostgreSQL does not have native temporal tables (SQL:2011 standard)
-- but you can simulate with a history table and range type:
CREATE TABLE orders_history (LIKE orders);
ALTER TABLE orders_history ADD COLUMN valid_from TIMESTAMPTZ NOT NULL;
ALTER TABLE orders_history ADD COLUMN valid_to   TIMESTAMPTZ;
-- Valid_to = NULL means current row
-- Store a copy of the row on every UPDATE via trigger
```
Enables time-travel queries ("what did this order look like on 2024-03-15?").
Storage cost doubles or more. Most appropriate for entities with regulatory
requirement to reconstruct state at any point in time.

---

## Q4: When and how should you use CHECK constraints vs triggers vs application code?

**Answer:**

These three mechanisms sit at different levels of the defense-in-depth stack and
serve different purposes.

**CHECK Constraints — Use for:**
- Simple, single-row, single-column invariants
- Type-like restrictions (valid enum values, positive numbers, non-empty strings)
- Always prefer over application code for these cases

```sql
ALTER TABLE orders ADD CONSTRAINT valid_status
    CHECK (status IN ('pending', 'processing', 'shipped', 'delivered', 'cancelled'));

ALTER TABLE products ADD CONSTRAINT positive_price
    CHECK (price > 0);

ALTER TABLE events ADD CONSTRAINT valid_time_range
    CHECK (end_time > start_time);
```

**Triggers — Use for:**
- Cross-row or cross-table validation (CHECK cannot do this)
- Derived column maintenance (updated_at, denormalized sums)
- Audit logging
- Cascading business logic that must be atomic with the triggering change

```sql
-- Trigger for cross-row constraint: no overlapping bookings
-- CHECK cannot reference other rows, so trigger is necessary
CREATE FUNCTION check_booking_overlap() RETURNS TRIGGER AS $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM bookings
        WHERE room_id = NEW.room_id
          AND tsrange(check_in, check_out) && tsrange(NEW.check_in, NEW.check_out)
          AND id != NEW.id
    ) THEN
        RAISE EXCEPTION 'Booking overlap detected for room %', NEW.room_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Application Code — Use for:**
- User-friendly error messages (database exceptions are cryptic to end users)
- Business rules that involve external state (credit limits, inventory from
  another system, real-time pricing APIs)
- Rules that change frequently without schema migration
- Rules that differ by tenant or context

**The rule of thumb:** Use the deepest layer possible for each constraint. A
constraint at the database level is enforced regardless of which application,
script, or direct SQL connection touches the data. Application-only validation
is bypassed by every other access path.

---

## Q5: How do surrogate keys and natural keys affect schema design at scale?

**Answer:**

**Natural Key:** A key formed from real-world data attributes (email, SSN, ISBN,
ticker symbol). The key has business meaning.

**Surrogate Key:** An artificial key with no business meaning (auto-increment
integer, UUID, ULID). Generated by the system, not the business.

```sql
-- Natural key approach
CREATE TABLE users (
    email    TEXT PRIMARY KEY,
    name     TEXT,
    ...
);
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    user_email  TEXT REFERENCES users(email),  -- FK references natural key
    ...
);

-- Surrogate key approach
CREATE TABLE users (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email    TEXT UNIQUE NOT NULL,
    name     TEXT,
    ...
);
CREATE TABLE orders (
    id       BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    user_id  BIGINT REFERENCES users(id),  -- FK references surrogate key
    ...
);
```

**Natural key problems at scale:**
1. Natural keys can change. If a user changes their email, you must update every
   FK reference across every child table — potentially millions of rows.
2. Long natural keys (email = up to 254 chars) as FK columns in child tables
   waste significant storage compared to 8-byte BIGINT.
3. Composite natural keys (e.g., (country_code, tax_id)) must be carried into
   all child tables as composite FKs — greatly complicating joins.
4. Natural "uniqueness" is rarely absolute — SSNs can be reused, emails can be
   changed, ISBNs are not always unique.

**Surrogate key problems:**
1. Lose the ability to use the PK as a meaningful filter without joining back
   to the entity table.
2. Can expose row counts (integer) or timing (UUID v1) to clients.
3. UUIDs (v4) cause random B-tree page splits that fragment indexes.

**Pragmatic recommendation:**
- Always use a surrogate key as the primary key (BIGINT IDENTITY or UUID v7).
- Always add a UNIQUE constraint on the natural key(s) for data integrity.
- Use the surrogate key for all FK references.
- Expose the natural key in your API; use the surrogate key internally.
