# Chapter 09: Time-Series Databases — Scenario Questions

1. Your InfluxDB cluster is experiencing OOM crashes every 72 hours. Investigation shows
   the series cardinality has exploded from 50,000 to 12 million series in 3 weeks. A
   developer recently added a `request_id` tag to all HTTP request metrics. The cardinality
   counter in the InfluxDB UI confirms each request_id creates a new series. The system
   generates 10M requests/hour. Walk through the root cause, immediate remediation, and
   the correct data model change.

2. Your team runs Prometheus to monitor a Kubernetes cluster with 500 pods, each exposing
   200 metrics. Total unique time series: 100,000. Queries during incident response take
   10-15 seconds. The on-call engineer can't debug fast enough. You need to get query
   latency under 2 seconds for the 50 most common PromQL queries. How do you approach
   this with recording rules, and how do you decide which queries to pre-compute?

3. Your TimescaleDB hypertable for IoT sensor data is 2TB. The database server has 64GB
   RAM and reads are increasingly served from disk. A TimescaleDB consultant suggests
   enabling native compression on chunks older than 7 days. The team is nervous about
   the impact on ongoing writes and analytical queries. Walk through the trade-offs,
   the compression mechanics, and the operational procedure for enabling compression
   without downtime.

4. Your ClickHouse cluster (5 shards, 2 replicas each) processes 50 million log events
   per hour. A distributed query scanning 30 days of data takes 45 seconds. Product
   wants it under 5 seconds. Investigation shows only 1 of 5 shards is being read for
   most queries because the distribution key puts all data for the most-queried service
   ("payment-service") on shard 1. Diagnose the skew, explain the ClickHouse query
   execution model, and propose a redistribution strategy.

5. Your Prometheus remote_write pipeline sends metrics to a long-term storage backend
   (Thanos). During a network partition lasting 4 hours, Prometheus buffered 4 hours of
   metrics in its WAL. When connectivity restored, it tried to replay 4 hours of samples.
   The remote_write queue built up, causing metric ingestion to lag by 6 hours. New
   alerts are firing 6 hours late. How does Prometheus handle WAL replay, what are the
   limits, and how do you design for this failure mode?

6. You are designing a multi-tenant time-series platform for 500 enterprise customers
   using InfluxDB OSS. Each customer generates up to 10,000 series with 10-second
   resolution, 1-year retention. Total: 5 billion series. A single InfluxDB instance
   cannot handle 5 billion series (cardinality limit). How do you architect a sharded
   multi-tenant InfluxDB deployment, and how do you route writes and queries to the
   correct shard?

7. Your monitoring system uses Grafana + Prometheus + AlertManager. An alert "PodCrashLoop"
   fires, the on-call engineer investigates, but by the time they look at the dashboard,
   the pod has restarted successfully and metrics look normal. The root cause is lost
   because Prometheus only keeps 15 days of data. How do you redesign the monitoring
   architecture to preserve pre-incident metrics for post-mortems, ensure alerts don't
   flap, and maintain sub-minute alert detection latency?

8. Your IoT platform receives sensor readings from 1 million devices, each sending data
   every 5 seconds (200K readings/second peak). You currently use InfluxDB but it cannot
   keep up with the write load. You are considering migrating to ClickHouse or TimescaleDB.
   Write a concrete evaluation plan: what benchmarks you would run, what data model each
   requires, what you would measure (write latency, compression ratio, query latency for
   your specific access patterns), and your final decision criteria.

9. A product analytics team queries your TimescaleDB hypertable for daily active users
   (DAU) across a 2-year period. The query scans 730 partitioned chunks and takes 90
   seconds. The same query in ClickHouse (on replicated data) takes 3 seconds because
   of columnar storage. The team wants to migrate to ClickHouse. Make the case for and
   against the migration, including operational complexity, SQL compatibility, exact query
   translation, and the total engineering cost.

10. Your InfluxDB 3.x deployment uses Flux tasks for downsampling. A task that computes
    hourly averages from 1-minute raw data is running 45 minutes behind schedule —
    it processes data at 1x realtime but is designed to run every 30 minutes. As the
    task falls further behind, future runs also slow down because they process more data.
    This is the "downsampling debt spiral." How do you break out of it without losing
    data, and how do you redesign the downsampling architecture to be self-healing?
