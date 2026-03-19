# Chapter 02 — Database Design: Scenario Questions

Real-world design scenarios that test normalization instincts, constraint knowledge,
and schema evolution skills.

---

1. You are designing a schema for a hospital system. Patients can have multiple
   diagnoses. Doctors can treat multiple patients. A single patient visit can
   involve multiple doctors and result in multiple diagnoses. Some diagnoses
   require follow-up visits that reference the original visit. Design the core
   schema (tables, primary keys, foreign keys, cardinality) and identify any
   M:N relationships and how you resolve them.

2. Your company's `orders` table was created three years ago and has this schema:
   `(order_id, customer_name, customer_email, customer_city, product_name,
   product_category, product_price, quantity, order_date)`. It has 80 million rows
   and is causing significant storage and update anomaly problems. Walk through
   the normalization process step by step to bring it to 3NF, naming every
   functional dependency you identify.

3. A fintech startup asks you to review their `transactions` table. It has a
   `status` column (VARCHAR) with values like 'pending', 'completed', 'failed'.
   They have no CHECK constraint on it. Their application code enforces the
   valid values. Last month, a bug introduced 'Completed' (capital C) for 2,000
   rows. How do you fix the data, prevent recurrence, and handle the migration
   without downtime on a table with 500M rows?

4. You are designing a multi-tenant SaaS platform. Each tenant has users, and
   users belong to teams. You need to decide between: (a) one database per tenant,
   (b) one schema per tenant in one database, (c) shared tables with a tenant_id
   column. Walk through the trade-offs and explain which you would choose for a
   system expecting 5,000 tenants with an average of 200 users each.

5. Your product team wants to implement "user profiles" where the set of profile
   fields is different for each user type (job seekers have a resume, employers
   have a company description, freelancers have a portfolio URL). Three engineers
   propose three different schemas: (a) one table with many nullable columns,
   (b) a separate table per user type, (c) an entity-attribute-value (EAV) table.
   Analyze the trade-offs of each approach.

6. You need to store a product catalog where products can have a variable number
   of attributes (color, size, material, weight, voltage, etc.) and each product
   type has different attributes. A junior developer suggests a JSON column for
   the attributes. A senior developer suggests a proper attribute table. How do
   you decide, and what factors would make you choose each option?

7. An e-commerce platform has `products`, `variants` (size/color combinations),
   and `inventory` tables. Currently, inventory is tracked at the product level.
   The business wants to track inventory at the variant level, but 30 other tables
   have foreign keys pointing to `products.id` assuming product-level inventory.
   How do you migrate the schema without breaking existing functionality or
   requiring a big-bang deployment?

8. Your team is building an event sourcing system where you want to keep a
   complete history of all changes to every entity. A colleague suggests adding
   `valid_from` and `valid_to` columns to every table. Another suggests a
   separate `events` table storing JSON diffs. A third suggests using PostgreSQL's
   temporal tables. Compare these three approaches on: storage efficiency, query
   complexity for current state, query complexity for historical state, and
   schema evolution difficulty.

9. You have been asked to design the schema for a permissions system where users
   can be assigned roles, roles have permissions, and permissions apply to
   specific resources (not globally). A user can have different roles in different
   contexts (e.g., admin in team A, viewer in team B). Design the schema and
   explain how you would query "does user X have permission P on resource R?"

10. A startup's database was designed with no foreign key constraints ("for
    performance"). Three years later, the data has significant orphaned records:
    orders with no customer, order items with no order, etc. You have been asked
    to add foreign key constraints. Outline a safe migration plan that handles
    the existing dirty data, adds the constraints, and prevents new orphaned
    records without a maintenance window.
