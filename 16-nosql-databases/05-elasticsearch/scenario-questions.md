# Chapter 05 — Elasticsearch: Scenario Questions

Production Elasticsearch scenarios. Answer out loud before reviewing.

---

1. **The relevance scoring disaster**
   Your e-commerce search returns irrelevant results. A search for "apple" returns Apple
   MacBook Pro as result #1, but the customer wanted apples (the fruit). Your product
   catalogue has 5 million products. The word "apple" appears in 500,000 product descriptions
   (brand mentions) but only 2,000 products are actual apples. Explain why the BM25 scoring
   is behaving this way, and design a query strategy using `function_score`, boosting, and
   category signals to improve relevance without hand-curating every query.

2. **The mapping explosion**
   Your application dynamically adds user-defined metadata to documents. After 6 months,
   Elasticsearch throws `limit of total fields [1000] exceeded` errors on new index operations.
   The cluster master nodes are consuming excessive CPU. Explain what caused the mapping
   explosion, what the 1000-field limit protects against, and redesign the data model using
   `flattened` field type or nested key-value pairs to handle dynamic attributes without
   mapping explosion.

3. **The hot shard problem**
   Your logging cluster ingests 500,000 log events per second. All logs are time-stamped and
   indexed into daily indices. The current shard routing strategy assigns all writes for a given
   day to the same primary shard (the default routing by `_id` hash). Monitoring shows shard 3
   on `logs-2024-01-15` is at 100% disk write throughput while 4 other shards in that index
   are at 20%. Identify why this is happening, explain custom routing, and design a write
   distribution strategy for time-series log indices.

4. **The painfully slow aggregation**
   Your analytics dashboard runs a `terms` aggregation on a `text` field `description` to get
   the top 10 product descriptions by frequency. The query takes 45 seconds and the cluster
   CPU spikes to 95%. A senior engineer says "you need to enable fielddata" but another says
   "never enable fielddata on text." Explain why aggregating on a `text` field is problematic,
   what `fielddata` is, why it is dangerous, and how you would correctly solve this requirement
   without enabling fielddata on the `text` field.

5. **The reindex zero-downtime migration**
   Your product catalogue index has an incorrect mapping: `product_id` is mapped as `text`
   when it should be `keyword` (you cannot change a field's type in-place in Elasticsearch).
   The index has 10 million documents. You need to fix this without any downtime — searches
   must keep working throughout the migration. Walk through the complete procedure using
   aliases, the Reindex API, and index templates.

6. **The cluster red status**
   Your Elasticsearch cluster goes RED at 3am. Monitoring shows `unassigned_shards: 47`.
   The cluster has 6 data nodes and you have `number_of_replicas: 1`. Two data nodes were
   terminated simultaneously by an autoscaling event. Explain what caused the RED status,
   what the difference between YELLOW and RED is, and provide the step-by-step recovery
   procedure. What would you change in the cluster configuration to make it more resilient
   to simultaneous node loss?

7. **The bulk indexing performance**
   Your team is bulk-loading 1 billion historical documents into Elasticsearch for a
   migration. The default indexing speed is 5,000 documents/second, which means the
   migration would take 55 hours. You have a 24-hour window. Design a bulk indexing
   optimisation strategy: what Elasticsearch settings would you change for the migration
   window, what should you change back afterward, how do you tune the bulk request size,
   and how do you parallelise across shards?

8. **The nested query performance regression**
   Your document model stores order items as nested objects within orders. A query that
   searches for "orders containing a product with name 'MacBook Pro' AND quantity > 2"
   takes 8 seconds on 50 million documents. After adding more replicas, the query is
   still slow. Explain why nested queries are expensive, why adding replicas does not help,
   and propose an alternative document model (parent-child join type or denormalisation)
   that improves this query's performance.

9. **The ILM transition stuck**
   Your ILM policy should move indices from hot to warm (2 days), warm to cold (30 days),
   and delete at 90 days. An index `logs-2024-01-01` has been stuck in the warm phase for
   45 days. The ILM status shows `WAITING`. What conditions can cause an ILM transition
   to get stuck, how do you diagnose the issue using the ILM explain API, and how do you
   manually force the transition?

10. **The search template and A/B testing**
    Your team needs to run A/B tests on relevance ranking. Version A uses BM25 with default
    settings; Version B uses a custom function_score that boosts recent products. You need
    to serve both ranking strategies to 50% of users each, measure click-through rate, and
    switch to the winner without downtime. Design the Elasticsearch infrastructure for this
    using search templates and index aliases, and explain how you would handle the case
    where the two ranking strategies return results from different index versions.
