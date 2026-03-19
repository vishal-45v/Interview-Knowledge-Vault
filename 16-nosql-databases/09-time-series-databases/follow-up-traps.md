# Chapter 09: Time-Series Databases — Follow-Up Traps

## Trap 1: "Tags and fields are interchangeable in InfluxDB — just use tags for everything"

**What most people say:**
"Tags make data queryable, so put all your metadata in tags for maximum flexibility."

**Correct answer:**
Tags are INDEXED in InfluxDB's in-memory series index. Every unique tag value combination
creates a new series. Fields are NOT indexed — they're stored as values within a series.
Using a high-cardinality value as a tag causes cardinality explosion:

```
SCENARIO: HTTP request metrics

BAD MODEL (request_id as tag):
measurement: http_requests
tags: service=auth, endpoint=/login, request_id=abc123  ← UNIQUE PER REQUEST
      └── Creates ONE NEW SERIES per request
      └── 10M requests/day = 10M new series/day
      └── InfluxDB holds ALL series keys in memory
      └── 10M series × 256 bytes each ≈ 2.5GB RAM per day
      └── After 1 week: 70M series = 17.5GB RAM → OOM CRASH

GOOD MODEL (request_id as field):
measurement: http_requests
tags: service=auth, endpoint=/login, status=200    ← LOW CARDINALITY
fields: duration_ms=142.3, request_id="abc123"    ← HIGH CARDINALITY VALUES
        └── Only ~20 unique series (service × endpoint × status)
        └── request_id stored per data point, not in series index
        └── Tradeoff: CANNOT query by request_id (no index)
        └── If you need to query by request_id → use a document store instead
```

```python
# Checking cardinality in InfluxDB
from influxdb_client import InfluxDBClient

client = InfluxDBClient(url="http://localhost:8086", token="my-token", org="myorg")
query_api = client.query_api()

# Show measurements with cardinality > 100K
query = '''
import "influxdata/influxdb/schema"

schema.measurementCardinality(bucket: "metrics")
|> filter(fn: (r) => r._value > 100000)
|> sort(columns: ["_value"], desc: true)
'''
result = query_api.query(query=query)
for table in result:
    for record in table.records:
        print(f"Measurement: {record['_measurement']}, Cardinality: {record['_value']}")
```

## Trap 2: "rate() and irate() are equivalent in Prometheus — just use rate()"

**What most people say:**
"Both compute per-second rate from a counter. rate() is safer because it uses more data
points."

**Correct answer:**
They compute fundamentally different things:
- `rate(counter[5m])`: Average per-second rate over the ENTIRE 5-minute window
- `irate(counter[5m])`: Instantaneous rate based on the LAST TWO data points in the window

```promql
# rate() — smoothed average (good for dashboards, trend visualization)
rate(http_requests_total{job="api"}[5m])
# Returns: (value_at_end - value_at_start) / 5*60
# If requests spiked at minute 3 then dropped, this shows the average

# irate() — instantaneous rate (good for detecting spikes in real-time)
irate(http_requests_total{job="api"}[5m])
# Returns: (last_value - second_to_last_value) / (time_delta between them)
# Captures the spike but is much noisier

# The 5m window in irate() is NOT the averaging window —
# it's the lookback window to find the last 2 data points.
# If there's a scrape gap > 5m, irate() returns "no data" (empty)
# Use rate() for recording rules and alert evaluation
# Use irate() only for short-window "right now" dashboards
```

**The hidden trap:** Counter resets. Both rate() and irate() handle counter resets
(when a counter goes from a high value to zero due to process restart) correctly by
detecting the decrease and treating it as a reset. BUT if your counter decrements
(which shouldn't happen for a counter, but does happen with incorrect instrumentation),
both will misinterpret it as a reset.

## Trap 3: "Prometheus pulls metrics — it doesn't miss any"

**What most people say:**
"Prometheus scrapes on a schedule, so as long as services are up, it never loses metrics."

**Correct answer:**
Prometheus has multiple mechanisms for metric loss:
1. **Staleness after 5 minutes**: If a series isn't scraped for 5m, Prometheus injects a
   "stale marker" — queries treat it as no data after that point.
2. **Short-lived jobs**: Processes that complete in < 1 scrape interval (e.g., a batch
   job running for 10 seconds when scrape interval is 15 seconds) are never scraped.
3. **WAL capacity**: Prometheus buffers undelivered samples in WAL. Default WAL size is
   determined by `--storage.tsdb.wal-segment-size`. If remote_write is down for >2 hours,
   WAL replay may time out.

```yaml
# For short-lived jobs, use Prometheus Pushgateway
# Batch job pushes metrics BEFORE exiting; Pushgateway holds them for scraping

# prometheus.yml
scrape_configs:
  - job_name: 'pushgateway'
    static_configs:
      - targets: ['pushgateway:9091']
    honor_labels: true  # Preserve labels set by the pushing job

# batch_job.sh
#!/bin/bash
# Wrap batch metrics in a push
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/backup_job
# HELP backup_duration_seconds Duration of backup job
# TYPE backup_duration_seconds gauge
backup_duration_seconds{database="users"} 142.3
# HELP backup_rows_processed Total rows processed
# TYPE backup_rows_processed counter
backup_rows_processed{database="users"} 5432198
EOF
```

Pushgateway is NOT meant for high-frequency metrics — it's specifically for batch jobs.
Using Pushgateway for application metrics defeats Prometheus's pull model and creates
a staleness problem (old values persist until overwritten).

## Trap 4: "TimescaleDB is just PostgreSQL with time-series features added on top"

**What most people say:**
"TimescaleDB is a PostgreSQL extension — any PostgreSQL knowledge applies directly."

**Correct answer:**
TimescaleDB IS a PostgreSQL extension, but hypertables change the storage model
fundamentally in ways that break several PostgreSQL assumptions:

1. **VACUUM/AUTOVACUUM**: Regular PostgreSQL autovacuum is largely useless for
   time-series data (time-series data is append-only — no updates to vacuum). TimescaleDB
   uses chunk-level compression instead of VACUUM for space reclamation.

2. **Index behavior**: A TimescaleDB hypertable index is actually one index per chunk.
   Creating an index on a 2TB hypertable creates hundreds of small indexes. The index
   creation query plan is very different from a 2TB flat table.

3. **EXPLAIN output**: A hypertable query shows a `Custom Scan (ChunkAppend)` node —
   not a standard `Seq Scan`. The optimizer makes chunk-exclusion decisions at runtime
   based on the WHERE clause time filter.

```sql
-- This query LOOKS like a normal PostgreSQL query
SELECT time_bucket('1 hour', time) AS hour, avg(temperature)
FROM sensor_readings
WHERE time > NOW() - INTERVAL '7 days'
  AND device_id = 'device-001'
GROUP BY hour ORDER BY hour;

-- EXPLAIN ANALYZE shows the TimescaleDB-specific execution
EXPLAIN ANALYZE SELECT ...;
-- Output includes:
-- Custom Scan (ChunkAppend) on sensor_readings
--   -> Seq Scan on _hyper_1_42_chunk (ONLY chunks in last 7 days)
--   -> Seq Scan on _hyper_1_41_chunk
--   -> ...
-- Chunks NOT in the time range are excluded at query planning time (constraint exclusion)
-- This is the "chunk exclusion" feature — NOT standard PostgreSQL partition pruning
```

The trap: migrating a large PostgreSQL time-series table to TimescaleDB requires careful
chunk size selection. Too large chunks = slow compression/decompression, poor cache
locality. Too small chunks = too many open file handles, high overhead per query.
The rule of thumb: chunks should fit in ~25% of available RAM.

## Trap 5: "ClickHouse primary keys work like SQL primary keys"

**What most people say:**
"ClickHouse has primary keys for uniqueness constraints and fast lookups."

**Correct answer:**
ClickHouse primary keys do NOT enforce uniqueness. They define the SORT ORDER of data
in each "part" and create a sparse index (one entry per ~8192 rows). They are COMPLETELY
different from SQL primary keys.

```sql
-- ClickHouse MergeTree primary key
CREATE TABLE http_logs (
    date Date,
    timestamp DateTime,
    service LowCardinality(String),
    status UInt16,
    duration_ms Float32,
    url String
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)        -- Physical partitioning by month
ORDER BY (service, date, timestamp) -- Primary key = sort order = sparse index
-- 'url' and 'duration_ms' have NO index — always full scan within a granule

-- This query is FAST (uses primary key index to skip granules):
SELECT count(), avg(duration_ms)
FROM http_logs
WHERE service = 'payment'       -- Primary key prefix → index skip
  AND date = '2024-01-15'       -- Primary key second column

-- This query is SLOW (no index on url, but ClickHouse scans fast anyway):
SELECT * FROM http_logs
WHERE url LIKE '%/admin%'       -- Full scan of matching partitions

-- Duplicate rows ARE allowed and common:
-- Multiple inserts of the same data create duplicates
-- Deduplication happens lazily during MergeTree background merges
-- Not guaranteed to be deduplicated at query time!
-- Use ReplacingMergeTree or AggregatingMergeTree for deduplication semantics
```

```sql
-- For point lookups, use a skip index (minmax, bloom_filter, or ngrambf)
ALTER TABLE http_logs
ADD INDEX url_bloom_filter url TYPE bloom_filter(0.01) GRANULARITY 1;
-- Creates a per-granule bloom filter for the url column
-- Filters out granules that definitely don't contain the URL
-- Does NOT guarantee uniqueness
```

## Trap 6: "Prometheus histograms give you exact percentiles"

**What most people say:**
"A histogram_quantile(0.99, ...) gives me the exact 99th percentile latency."

**Correct answer:**
Prometheus histograms give you APPROXIMATIONS, not exact percentiles. Histograms
pre-aggregate into fixed buckets. The histogram_quantile() function linearly interpolates
between bucket boundaries — it assumes uniform distribution within each bucket.

```promql
# Typical HTTP latency histogram definition in application code:
# histogram_buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
# All requests in the 0.1-0.25 second bucket are treated as uniformly distributed

histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{job="api"}[5m])
)
# This might show 0.23 seconds, but real p99 could be 0.24 or 0.22
# Accuracy depends on bucket density in the region of interest

# For more accurate high percentiles: add more buckets in the long tail
# histogram_buckets: [0.1, 0.15, 0.2, 0.25, 0.3, 0.4, 0.5, 0.75, 1.0, 1.5, 2.0, 5.0]
#                         └── Finer buckets in the 100-500ms range for SLO monitoring
```

Summary metrics (in client libraries) compute exact quantiles ON THE CLIENT SIDE but
cannot be aggregated across instances — a summary from 100 instances cannot be combined.
Histograms can be aggregated (sum(rate(bucket[5m]))) across instances at the cost of
approximation. For p99 SLO monitoring, histograms with fine buckets in the SLO range
are the correct approach.

## Trap 7: "More Prometheus instances = more metric retention"

**What most people say:**
"If I run 3 Prometheus instances, I get 3x the retention and redundancy."

**Correct answer:**
Vanilla Prometheus has NO built-in clustering or data federation. Multiple instances
are independent; they don't share or replicate data. The standard approach:

```
WRONG ASSUMPTION:
Prometheus-1 ──► 15 days data
Prometheus-2 ──► 15 days data (SAME data, NOT 30 days)
Prometheus-3 ──► 15 days data (SAME data, NOT 45 days)

CORRECT ARCHITECTURE for long-term retention:

Prometheus-1 (scrapes pods 1-250) ──────────────────► Thanos Sidecar
Prometheus-2 (scrapes pods 251-500) ────────────────► Thanos Sidecar
                                                             │
                                              ┌──────────────┘
                                              ▼
                                      Thanos Compact (deduplicates)
                                              │
                                              ▼
                                      Object Storage (S3/GCS)
                                      (years of retention, low cost)

Client queries → Thanos Querier (federates across all Prometheus instances + object store)
```

Running multiple Prometheus instances scrapin the same targets (HA pairs) creates
data redundancy, but Thanos/Cortex/Mimir must be used to deduplicate and expose a
unified query interface.

## Trap 8: "Downsampling averages are sufficient for all analytics"

**What most people say:**
"For old data, store hourly averages. The average is good enough."

**Correct answer:**
Averaging during downsampling DESTROYS statistical information permanently:

```
Original 1-minute data (60 points): peak=98%, average=45%, p99=87%

Downsampled to 1 hour — AVERAGE ONLY: 45%
  ← Peak of 98% is invisible!
  ← SLO breach at minute 47 (value = 98%) looks fine in hourly average

CORRECT DOWNSAMPLING: Store MULTIPLE aggregates per window
```

```sql
-- TimescaleDB continuous aggregate: store min, max, avg, count, percentile
CREATE MATERIALIZED VIEW sensor_hourly_stats
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    device_id,
    AVG(value)                              AS avg_value,
    MIN(value)                              AS min_value,
    MAX(value)                              AS max_value,
    COUNT(*)                                AS sample_count,
    PERCENTILE_CONT(0.95) WITHIN GROUP
        (ORDER BY value)                    AS p95_value,
    STDDEV(value)                           AS stddev_value,
    LAST(value, time)                       AS last_value  -- Most recent reading
FROM sensor_readings
GROUP BY bucket, device_id
WITH NO DATA;

-- Refresh policy
SELECT add_continuous_aggregate_policy('sensor_hourly_stats',
    start_offset => INTERVAL '3 hours',   -- Backfill up to 3 hours ago
    end_offset   => INTERVAL '1 hour',    -- Don't include last 1 hour (may be incomplete)
    schedule_interval => INTERVAL '1 hour'
);
```

Storing min, max, p95, count, and stddev allows reconstruction of many useful statistics
even after raw data is dropped. An average alone tells you very little about the shape
of the original distribution.
