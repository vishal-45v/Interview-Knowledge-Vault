# Chapter 09: Time-Series Databases — Theory Questions

1. What are the defining characteristics of time-series data that make it different from
   general-purpose data? Why do purpose-built TSDBs outperform PostgreSQL for time-series
   workloads, and under what conditions does PostgreSQL remain competitive?

2. Explain InfluxDB's data model: measurements, tags, fields, and timestamps. What is the
   fundamental difference between a tag and a field, and what happens to query performance
   if you store a high-cardinality value as a tag instead of a field?

3. Describe InfluxDB's TSM (Time-Structured Merge Tree) engine. How does it differ from
   a standard LSM tree, and what optimizations does it make specifically for time-series
   data (column-encoded blocks, delta encoding, run-length encoding)?

4. What is the cardinality explosion problem in time-series databases? Give a concrete
   example of how an innocuous-seeming tag choice can cause InfluxDB to run out of memory,
   and explain the internal mechanism (series index) that makes cardinality expensive.

5. Explain InfluxDB retention policies and continuous queries (InfluxDB 1.x) / tasks
   (InfluxDB 2.x+). How are they used together to implement a downsampling and data
   tiering strategy?

6. Describe Prometheus's data model (metrics, labels, samples). What is the difference
   between a counter, gauge, histogram, and summary metric type, and when do you use each?

7. What is the difference between an instant vector and a range vector in PromQL? Give
   examples of when each is appropriate, and explain what `rate()` vs `irate()` compute.

8. Explain Prometheus's pull-based scrape model. What are its advantages and disadvantages
   compared to a push-based model (like StatsD or InfluxDB line protocol push)? How does
   `remote_write` change Prometheus's operational model?

9. Describe Prometheus's local TSDB storage format. What are TSDB blocks, how long do
   they cover, what is the WAL's role, and when are blocks compacted?

10. What is TimescaleDB's hypertable? How does automatic time-based partitioning into
    chunks work, and what is the advantage of chunk-level operations (compression, retention,
    reorder) over operating on the full table?

11. Explain TimescaleDB continuous aggregates. How do they differ from materialized views
    in standard PostgreSQL, and what is the "cagg refresh" mechanism?

12. What is ClickHouse's MergeTree engine? How does data merge work, what is the role of
    the primary key vs the order key, and how does ClickHouse handle late-arriving data?

13. Describe ClickHouse's ReplicatedMergeTree and its relationship to ZooKeeper/ClickHouse
    Keeper. What consistency model does it provide, and how are distributed queries
    executed across shards?

14. What is the difference between a TimescaleDB hypertable and a ClickHouse MergeTree
    table for the same time-series workload? Under what access patterns does each win?

15. Explain OpenTSDB's data model. How does it use HBase as a storage backend, and what
    is the "metric + tags → row key" translation? What are the limitations this imposes?

16. What are time-series anti-patterns? Describe at least four patterns that lead to poor
    performance or incorrect results in time-series databases.

17. Explain the "out-of-order write" problem in time-series databases. How does each TSDB
    (InfluxDB, Prometheus, TimescaleDB, ClickHouse) handle writes that arrive out of
    chronological order?

18. What is downsampling in a time-series context? Describe a concrete downsampling
    pipeline — from raw high-frequency data to multiple retention tiers — including the
    aggregation functions used at each tier.
