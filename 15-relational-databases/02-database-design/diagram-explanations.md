# Chapter 02 — Database Design: Diagram Explanations

Visual representations of ER modeling, normalization, schema patterns, and
constraint flows.

---

## Diagram 1: Entity-Relationship Cardinality Notation

```
  Crow's Foot Notation (most common in modern ERDs)

  Symbol meanings:
  ───────────────────────────────────────────────────────
  ──|     Exactly one (mandatory)
  ──o     Zero or one (optional)
  ──<     Many (one or more)
  ──o<    Zero or more (optional many)
  ──|<    One or more (mandatory many)

  Examples:
  ──────────────────────────────────────────────────────────────

  Customer ────|────────────o<──── Order
               1                  0..*
  (A customer has zero or more orders; each order has exactly one customer)

  Employee ────o────────────o──── Parking_Spot
               0..1              0..1
  (An employee optionally has one spot; a spot is optionally assigned to one employee)

  Student ─────|<───────────|<──── Course  (M:N resolved via junction table)
               1..*               1..*
         ↓ resolved as ↓
  Student ────|────── Enrollment ────|──── Course
              1             *            1
  (Each enrollment belongs to exactly one student and one course)

  ──────────────────────────────────────────────────────────────
  SQL representation of each cardinality:

  1:N  → FK on the "many" side: orders.customer_id → customers.id
  M:N  → Junction table: (student_id, course_id) in enrollments
  1:1  → FK with UNIQUE on the "one" side or merge into one table
```

---

## Diagram 2: Normalization Progression — The Orders Table

```
  UNNORMALIZED (0NF)
  ┌──────────┬──────────────┬───────────────────────────────────┐
  │ order_id │ customer     │ items                             │
  ├──────────┼──────────────┼───────────────────────────────────┤
  │ 1001     │ Alice/NYC    │ [Pen×2/$1.50, Notebook×1/$5.00]   │
  │ 1002     │ Bob/LA       │ [Bag×1/$25.00, Pen×3/$1.50]       │
  └──────────┴──────────────┴───────────────────────────────────┘
  Problems: repeating groups in items column, composite customer value

  ──► 1NF (atomic values, no repeating groups)
  ┌──────────┬───────────────┬──────────┬────────────┬───────┬────────┐
  │ order_id │ customer_name │ cust_city│ product_id │ qty   │ price  │
  ├──────────┼───────────────┼──────────┼────────────┼───────┼────────┤
  │ 1001     │ Alice         │ NYC      │ P01        │ 2     │ 1.50   │
  │ 1001     │ Alice         │ NYC      │ P02        │ 1     │ 5.00   │
  │ 1002     │ Bob           │ LA       │ P03        │ 1     │ 25.00  │
  │ 1002     │ Bob           │ LA       │ P01        │ 3     │ 1.50   │
  └──────────┴───────────────┴──────────┴────────────┴───────┴────────┘
  PK = (order_id, product_id)
  Partial dependencies: order_id → customer_name, cust_city
                        product_id → price

  ──► 2NF (remove partial dependencies)
  ┌──────────┬─────────────┐  ┌────────────┬────────┐  ┌──────────┬────────────┬─────┐
  │ order_id │ customer_id │  │ product_id │ price  │  │ order_id │ product_id │ qty │
  ├──────────┼─────────────┤  ├────────────┼────────┤  ├──────────┼────────────┼─────┤
  │ 1001     │ C01         │  │ P01        │ 1.50   │  │ 1001     │ P01        │ 2   │
  │ 1002     │ C02         │  │ P02        │ 5.00   │  │ 1001     │ P02        │ 1   │
  └──────────┴─────────────┘  │ P03        │ 25.00  │  │ 1002     │ P03        │ 1   │
  orders                      └────────────┴────────┘  │ 1002     │ P01        │ 3   │
                               products                 └──────────┴────────────┴─────┘
  ┌─────┬───────┬──────┐                                order_items
  │ id  │ name  │ city │
  ├─────┼───────┼──────┤
  │ C01 │ Alice │ NYC  │ Still has city — potential transitive dep if city→state
  │ C02 │ Bob   │ LA   │
  └─────┴───────┴──────┘
  customers

  ──► 3NF (remove transitive dependencies, e.g. city → state → region)
  customers: (id, name, zip_code)
  zip_codes:  (zip_code, city, state)  ← extracted
```

---

## Diagram 3: Star Schema vs Snowflake Schema

```
  STAR SCHEMA
  ─────────────────────────────────────────────────────────────────

            dim_date ──────────────────────────────────────┐
            (date_key, day, month, quarter, year, holiday) │
                                                           │
  dim_customer ─────── fact_sales ────── dim_product       │
  (customer_key,       (sale_id,         (product_key,     │
   name, city,    ──── date_key,    ───── product_id,      │
   segment,            customer_key,      name,            │
   country)            product_key,       category,   ◄────┘
                       store_key,         brand,
                       revenue,           price)
                       quantity)
                            │
                       dim_store
                       (store_key, name, city, region)

  Each dimension is ONE table — one join to access any attribute.
  Wide, denormalized. Best for BI tools and columnar engines.

  SNOWFLAKE SCHEMA
  ─────────────────────────────────────────────────────────────────

  dim_customer ─────── fact_sales ─────── dim_product
                            │                   │
                       dim_store           dim_category
                            │                   │
                       dim_region          dim_brand
                                                │
                                          dim_supplier

  Each dimension normalized into sub-dimensions.
  Fewer storage bytes; more joins per query.
  Typically 20-40% slower for full-scan analytics queries.
```

---

## Diagram 4: Foreign Key ON DELETE Behaviors

```
  Parent: customers (id=42, name='Alice')
  Child:  orders where customer_id = 42 → orders #101, #102, #103

  DELETE FROM customers WHERE id = 42;

  ┌─────────────────┬──────────────────────────────────────────────────┐
  │ FK behavior     │ Result                                           │
  ├─────────────────┼──────────────────────────────────────────────────┤
  │ ON DELETE       │ ERROR: cannot delete customer while orders       │
  │ RESTRICT        │ exist. Application must delete orders first.     │
  ├─────────────────┼──────────────────────────────────────────────────┤
  │ ON DELETE       │ Orders #101, #102, #103 are automatically        │
  │ CASCADE         │ deleted. Any grandchild rows cascade further.    │
  │                 │ DANGER: silent mass deletion.                    │
  ├─────────────────┼──────────────────────────────────────────────────┤
  │ ON DELETE       │ Orders remain; their customer_id is set to NULL. │
  │ SET NULL        │ Requires customer_id to be NULLable.             │
  │                 │ Orders become "orphaned" but still exist.        │
  ├─────────────────┼──────────────────────────────────────────────────┤
  │ ON DELETE       │ Like RESTRICT but deferred. Checked at end of   │
  │ NO ACTION       │ statement (or transaction if DEFERRABLE).        │
  ├─────────────────┼──────────────────────────────────────────────────┤
  │ ON DELETE       │ Sets customer_id to the column's DEFAULT value   │
  │ SET DEFAULT     │ (rarely used; default must be a valid FK value). │
  └─────────────────┴──────────────────────────────────────────────────┘

  CASCADE chain depth example:
  customers ──► orders ──► order_items ──► shipment_items ──► returns
  One DELETE customers: cascades 4 levels deep, potentially millions of rows
```

---

## Diagram 5: Temporal Table Design (Slowly Changing Dimensions)

```
  Tracking product price history over time

  APPROACH 1: valid_from / valid_to columns (SCD Type 2)
  ┌────────────┬──────────────┬───────┬─────────────────────┬─────────────────────┐
  │ product_id │ product_name │ price │ valid_from          │ valid_to            │
  ├────────────┼──────────────┼───────┼─────────────────────┼─────────────────────┤
  │ 101        │ Widget Pro   │ 19.99 │ 2022-01-01 00:00:00 │ 2023-06-01 00:00:00 │
  │ 101        │ Widget Pro   │ 24.99 │ 2023-06-01 00:00:00 │ 2024-01-01 00:00:00 │
  │ 101        │ Widget Pro   │ 29.99 │ 2024-01-01 00:00:00 │ NULL ← current      │
  └────────────┴──────────────┴───────┴─────────────────────┴─────────────────────┘

  Query: "What was the price on 2023-09-15?"
  SELECT price FROM product_history
  WHERE product_id = 101
    AND valid_from <= '2023-09-15'
    AND (valid_to > '2023-09-15' OR valid_to IS NULL);
  → 24.99

  Query: "Current price"
  SELECT price FROM product_history WHERE product_id = 101 AND valid_to IS NULL;
  → 29.99

  Required index:
  CREATE INDEX ON product_history (product_id, valid_from, valid_to);

  APPROACH 2: PostgreSQL tstzrange (uses GiST index for overlap queries)
  ┌────────────┬───────┬────────────────────────────────────────────────┐
  │ product_id │ price │ valid_during (tstzrange)                       │
  ├────────────┼───────┼────────────────────────────────────────────────┤
  │ 101        │ 19.99 │ [2022-01-01, 2023-06-01)                       │
  │ 101        │ 24.99 │ [2023-06-01, 2024-01-01)                       │
  │ 101        │ 29.99 │ [2024-01-01, infinity)                         │
  └────────────┴───────┴────────────────────────────────────────────────┘

  Exclusion constraint prevents overlapping ranges:
  ALTER TABLE product_history ADD CONSTRAINT no_overlap
  EXCLUDE USING gist (product_id WITH =, valid_during WITH &&);
```

---

## Diagram 6: Multi-Tenant Schema Design Options

```
  Option A: Database per Tenant
  ┌─────────────────────────────────────────────────────────────────┐
  │ db_tenant_1  │  db_tenant_2  │  db_tenant_3  │ ... db_tenant_N  │
  │ ┌──────────┐ │ ┌──────────┐  │ ┌──────────┐  │                  │
  │ │  users   │ │ │  users   │  │ │  users   │  │   5000 databases │
  │ │  orders  │ │ │  orders  │  │ │  orders  │  │   for 5000       │
  │ └──────────┘ │ └──────────┘  │ └──────────┘  │   tenants        │
  └─────────────────────────────────────────────────────────────────┘
  Pros: Perfect isolation, easy backup per tenant, custom schema per tenant
  Cons: Connection pool explosion, DB overhead × N, cross-tenant reports hard

  Option B: Schema per Tenant (PostgreSQL schemas)
  ┌─────────────────────────────────────────────────────────────────┐
  │                      single_database                            │
  │  tenant_1.users  │  tenant_2.users  │  tenant_3.users           │
  │  tenant_1.orders │  tenant_2.orders │  tenant_3.orders          │
  └─────────────────────────────────────────────────────────────────┘
  Pros: Shared connection pool, still isolated, cross-tenant SQL possible
  Cons: Schema migration must run for every tenant (N × migration time)
        PostgreSQL catalog bloat at high N (>1000 schemas)

  Option C: Shared Tables with tenant_id
  ┌──────────────────────────────────────────────────────────────┐
  │                      single_database                         │
  │  ┌───────────┬──────────────────────────────────────────┐    │
  │  │ users     │ tenant_id │ id  │ name  │ email          │    │
  │  ├───────────┼──────────────────────────────────────────┤    │
  │  │           │ t1        │ 101 │ Alice │ a@t1.com       │    │
  │  │           │ t1        │ 102 │ Bob   │ b@t1.com       │    │
  │  │           │ t2        │ 103 │ Carol │ c@t2.com       │    │
  │  └───────────┴──────────────────────────────────────────┘    │
  └──────────────────────────────────────────────────────────────┘
  Pros: Simple, one migration, shared resources efficiently
  Cons: tenant_id on every table and every query (one missing = data leak)
        Cross-tenant data leakage risk

  Recommended mitigation for Option C:
  ALTER TABLE users ENABLE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON users
      USING (tenant_id = current_setting('app.tenant_id')::INT);
  -- Now queries automatically filtered; impossible to forget WHERE tenant_id
```
