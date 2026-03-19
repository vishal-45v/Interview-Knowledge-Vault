# Chapter 05 — Advanced SQL: Scenario Questions

Real-world scenarios requiring mastery of window functions, recursive CTEs,
JSONB, and advanced PostgreSQL features.

---

1. You have a `transactions` table with `(account_id, transaction_date, amount)`.
   Write a single SQL query that returns, for each transaction: the transaction
   date, amount, the running total for that account up to that date, the
   30-day rolling average, and a flag indicating whether the amount is above
   the account's overall average. You cannot use a subquery-per-row approach.

2. Your `employees` table has a self-referencing `manager_id` column representing
   an organizational hierarchy 8 levels deep with 50,000 employees. Write a
   recursive CTE that returns every employee with their full management chain
   (all ancestors up to the CEO), the depth level, and a breadcrumb path
   string like "CEO > VP Engineering > Director > Manager > Alice". What
   safeguard must you add to prevent infinite recursion if the data has cycles?

3. You need to implement "find the top 3 products by revenue for each category,
   but only include categories that had at least $10,000 in total revenue."
   The result should show category, product name, revenue, and rank within
   category. How do you write this efficiently using window functions rather
   than correlated subqueries?

4. A marketing analytics query needs: for each user, the date of their first
   purchase, the date of their second purchase, the days between first and second
   purchase, and the amount of their largest purchase in their first 30 days.
   All from a single `purchases` table with `(user_id, purchase_date, amount)`.
   Write the SQL.

5. Your product catalog stores structured variant data in a JSONB column:
   `{"color": "red", "size": "L", "weight_kg": 1.5, "tags": ["sale", "new"]}`.
   Write queries to: (a) find all products with the tag "sale", (b) find all
   products where weight_kg > 1.0, (c) find the distinct set of all tags used
   across all products, (d) update the price field in the JSONB without
   overwriting other fields.

6. You are building a search feature for a blog platform. Posts have a `title`
   (TEXT) and `body` (TEXT). Users type a search phrase. Write the full-text
   search query using tsvector/tsquery. Include: the correct index to create,
   how to handle multi-word search, how to rank results by relevance, and
   how to highlight matching terms in the result.

7. You have a table `daily_metrics` with `(date, region, product, revenue,
   cost)`. Write a single query that produces a multi-dimensional report showing
   subtotals: total by (region, product), total by region, total by product,
   and a grand total. The output should have NULL in the grouping column where
   a subtotal aggregates across that dimension. How do you distinguish a NULL
   grouping from a genuine NULL value in the data?

8. An `orders` table receives thousands of inserts per second from an event
   stream. Duplicate order IDs can arrive (at-least-once delivery). If the order
   already exists, you want to update the status only if the new status is
   "later" in the lifecycle (pending → processing → shipped → delivered).
   Write the upsert using INSERT ON CONFLICT.

9. You need to create a materialized view that aggregates 500M rows of raw
   events into daily summaries per user. The view must be refreshable without
   downtime (reads must continue during refresh). You also need the refresh to
   happen automatically every hour. Describe the complete setup including the
   materialized view definition, the refresh strategy, and how you would automate
   it in PostgreSQL.

10. You have a table with an `event_data` JSONB column containing varied
    structures from different event sources. The data volume is 50M rows.
    Analytical queries filter on `event_data->>'event_type'` and on whether
    specific keys exist within nested objects. Currently all queries are doing
    sequential scans. Design the optimal indexing strategy and rewrite one
    representative slow query to use it.
