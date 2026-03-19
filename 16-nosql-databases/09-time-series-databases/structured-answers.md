# Chapter 09: Time-Series Databases — Structured Answers

## Q1: Design a complete downsampling and data tiering strategy for a metrics platform at scale.

**Answer:**

**Requirements:**
- Raw data: 10-second resolution, 30 days retention
- Medium term: 1-minute averages, 1-year retention
- Long term: 1-hour aggregates, 5-year retention
- Query SLA: < 1 second for any query within 24 hours, < 10 seconds for 1-year range

**Using InfluxDB with Flux tasks:**

```javascript
// Tier 1: Raw data (10-second, 30 days)
// Written directly by scraping agents
// Retention policy on bucket: 30 days

// Tier 2: 1-minute downsampling task
option task = {
    name: "downsample_to_1min",
    every: 1m,
    offset: 30s  // Wait 30s for late-arriving data
}

from(bucket: "metrics_raw")
    |> range(start: -2m, stop: -1m)  // Process 1 minute of data with overlap
    |> filter(fn: (r) => r._measurement == "system_metrics")
    |> aggregateWindow(
        every: 1m,
        fn: (tables=<-, column) => tables
            |> reduce(
                fn: (r, accumulator) => ({
                    count: accumulator.count + 1,
                    sum:   accumulator.sum + r._value,
                    min:   if r._value < accumulator.min then r._value else accumulator.min,
                    max:   if r._value > accumulator.max then r._value else accumulator.max,
                }),
                identity: {count: 0, sum: 0.0, min: 1e10, max: -1e10}
            )
    )
    |> map(fn: (r) => ({r with
        _value: r.sum / float(v: r.count),
        _field: r._field + "_avg"
    }))
    |> to(bucket: "metrics_1min", org: "myorg")

// Also write min, max, count to the 1min bucket for later analysis
```

**Using TimescaleDB continuous aggregates (production-ready approach):**

```sql
-- Raw data hypertable
CREATE TABLE metrics (
    time        TIMESTAMPTZ NOT NULL,
    metric_name TEXT NOT NULL,
    host        TEXT NOT NULL,
    value       DOUBLE PRECISION NOT NULL
);
SELECT create_hypertable('metrics', 'time', chunk_time_interval => INTERVAL '1 day');

-- Create indexes for common access patterns
CREATE INDEX ON metrics (metric_name, host, time DESC);

-- Tier 2: 1-minute continuous aggregate
CREATE MATERIALIZED VIEW metrics_1min
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', time) AS bucket,
    metric_name,
    host,
    AVG(value)                                          AS avg,
    MIN(value)                                          AS min,
    MAX(value)                                          AS max,
    COUNT(*)                                            AS cnt,
    PERCENTILE_CONT(0.50) WITHIN GROUP (ORDER BY value) AS p50,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY value) AS p95,
    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY value) AS p99
FROM metrics
GROUP BY bucket, metric_name, host
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_1min',
    start_offset     => INTERVAL '3 minutes',
    end_offset       => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute'
);

-- Tier 3: 1-hour continuous aggregate built on top of 1-minute aggregate
CREATE MATERIALIZED VIEW metrics_1hr
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', bucket) AS bucket,
    metric_name,
    host,
    AVG(avg)                      AS avg,  -- avg of avgs is approximate
    MIN(min)                      AS min,  -- min of mins is exact
    MAX(max)                      AS max,  -- max of maxs is exact
    SUM(cnt)                      AS cnt   -- sum of counts is exact
    -- NOTE: p95 cannot be aggregated from p95s — store raw for this if needed
FROM metrics_1min
GROUP BY time_bucket('1 hour', bucket), metric_name, host
WITH NO DATA;

SELECT add_continuous_aggregate_policy('metrics_1hr',
    start_offset     => INTERVAL '3 hours',
    end_offset       => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Retention policies: automatic data lifecycle
SELECT add_retention_policy('metrics',    INTERVAL '30 days');  -- Drop raw after 30d
SELECT add_retention_policy('metrics_1min', INTERVAL '1 year'); -- Drop 1min after 1yr
-- metrics_1hr: no retention (keep 5 years, covered by disk budget)

-- Compression for older chunks (significant space savings)
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'metric_name, host',  -- Group related data
    timescaledb.compress_orderby   = 'time DESC'
);
SELECT add_compression_policy('metrics', INTERVAL '7 days');  -- Compress chunks > 7 days old
-- Typical compression ratio: 10-20x for time-series data
```

**Query routing across tiers (application-level):**

```python
from datetime import datetime, timedelta

def query_metrics(metric: str, host: str, start: datetime, end: datetime) -> list:
    now = datetime.utcnow()
    duration = end - start

    if end > now - timedelta(days=30):
        # Recent data: use raw table for exact values
        table = "metrics"
        resolution = "10 seconds"
    elif duration <= timedelta(days=365):
        # Medium-term: use 1-minute aggregates
        table = "metrics_1min"
        resolution = "1 minute"
    else:
        # Long-term: use hourly aggregates
        table = "metrics_1hr"
        resolution = "1 hour"

    query = f"""
        SELECT bucket as time, avg, min, max, p95
        FROM {table}
        WHERE metric_name = %s
          AND host = %s
          AND bucket BETWEEN %s AND %s
        ORDER BY bucket
    """
    return db.execute(query, (metric, host, start, end))
```

---

## Q2: Explain Prometheus's TSDB storage architecture and block lifecycle.

**Answer:**

```
PROMETHEUS TSDB STORAGE LAYOUT:
/prometheus/data/
├── 01BKGV7JBM69T2G1BGBGM6KB12/  ← Block: 2-hour window (default)
│   ├── chunks/
│   │   ├── 000001               ← Chunk data (XOR compressed float64 values)
│   │   ├── 000002
│   │   └── ...
│   ├── index                    ← Inverted index: label → series IDs + chunk offsets
│   ├── meta.json                ← Block metadata (time range, stats, compaction level)
│   └── tombstones               ← Soft deletes (deletions applied during compaction)
├── 01BKGTZQ1SYQJTR4PB43C8PD98/  ← Another 2-hour block
├── wal/                          ← Write-Ahead Log (current 2-hour window in progress)
│   ├── 00000001                  ← WAL segment (128MB default)
│   ├── 00000002
│   └── checkpoint.00000001       ← WAL checkpoint (periodic snapshot)
└── chunks_head/                  ← Head chunks (current active samples, in-memory)
```

**Block lifecycle:**

```
PHASE 1: Head block (in-memory, backed by WAL)
- Duration: 2 hours (configurable with --storage.tsdb.min-block-duration)
- All incoming scrapes written to WAL + in-memory head chunks
- WAL provides durability; head chunks provide fast query access
- After 2 hours, head block is sealed and persisted as a Block on disk
- WAL segments are checkpointed (redundant entries trimmed)

PHASE 2: Level-0 blocks (2-hour persisted blocks)
- Persisted to disk as immutable TSDB blocks
- Multiple level-0 blocks accumulate (one per 2-hour window)
- After ~10+ level-0 blocks exist, compaction triggers

PHASE 3: Compaction (background process)
- Compaction merges multiple blocks into larger blocks
- Level 0 (2h) × 5 → Level 1 (10h) block
- Level 1 (10h) × 5 → Level 2 (50h) block
- Level 2 (50h) × 5 → Level 3 (250h) block
- Benefits: fewer files to query, tombstones applied, deduplication
- Max block duration: --storage.tsdb.max-block-duration (default: 25% of retention)

PHASE 4: Retention enforcement
- Blocks older than retention period are deleted atomically (directory removal)
- Default retention: 15 days (--storage.tsdb.retention.time)
```

**WAL recovery process:**

```
On Prometheus restart:
1. Load all completed (sealed) blocks from disk into memory index
2. Replay WAL from last checkpoint:
   a. Read WAL segments after checkpoint
   b. Replay all samples into head block in-memory
   c. Rebuild head block state
3. Any WAL segment that cannot be replayed = samples lost
   (but all WAL-committed samples are replayed)
```

**PromQL query execution touching multiple blocks:**

```promql
# This query must read from 3 blocks (today's head + 2 sealed blocks)
http_requests_total{job="api"}[6h]

# Query execution:
# 1. Identify which blocks cover the [6h] range
# 2. For each block, use the inverted index to find matching series
# 3. For each series, find relevant chunks
# 4. Decompress chunks and apply XOR decoding to get float64 values
# 5. Merge results chronologically
# 6. Return a range vector (all values in the 6h window)
```

---

## Q3: Write a Prometheus alerting rule and recording rule for a production SLO.

**Answer:**

**SLO:** 99.9% of API requests return HTTP 200-399 within 500ms over any 30-day rolling window.

```yaml
# prometheus/rules/api_slo.yml

groups:
  - name: api_slo_recording_rules
    interval: 1m
    rules:
      # Pre-compute request rate by status class (burn down recording rules)
      # This is evaluated every minute and stored as a new metric — queries use THIS
      - record: job:http_requests:rate5m
        expr: |
          sum by (job, status_class) (
            label_replace(
              rate(http_request_duration_seconds_count[5m]),
              "status_class",
              "$1xx",
              "status",
              "([0-9]).*"
            )
          )

      # Pre-compute success rate (fast bucket for SLO calculation)
      - record: job:http_requests_success:rate5m
        expr: |
          sum by (job) (
            rate(http_request_duration_seconds_bucket{le="0.5"}[5m])
          )
          /
          sum by (job) (
            rate(http_request_duration_seconds_count[5m])
          )

      # 30-day SLO window success rate
      - record: job:http_requests_success:rate30d
        expr: |
          sum by (job) (
            rate(http_request_duration_seconds_bucket{le="0.5"}[30d])
          )
          /
          sum by (job) (
            rate(http_request_duration_seconds_count[30d])
          )

  - name: api_slo_alerts
    rules:
      # Multi-window, multi-burn-rate alerting (Google SRE Workbook approach)
      # Alert fires if error budget is burning too fast in short AND long window

      # CRITICAL: 14.4x burn rate (consuming monthly budget in 2 hours)
      - alert: APIErrorBudgetBurnCritical
        expr: |
          (
            job:http_requests_success:rate5m{job="api-gateway"} < 0.9856
            and
            job:http_requests_success:rate1h{job="api-gateway"} < 0.9856
          )
        for: 1m
        labels:
          severity: critical
          slo: "api-latency-availability"
        annotations:
          summary: "API error budget burning at 14.4x rate"
          description: |
            API {{ $labels.job }} success rate {{ $value | humanizePercentage }}
            is burning error budget at 14.4x the monthly rate.
            At this rate, monthly budget exhausted in 2 hours.
          runbook_url: "https://wiki.example.com/runbooks/api-error-budget"
          dashboard_url: "https://grafana.example.com/d/api-slo"

      # WARNING: 6x burn rate (consuming monthly budget in 5 hours)
      - alert: APIErrorBudgetBurnWarning
        expr: |
          (
            job:http_requests_success:rate30m{job="api-gateway"} < 0.994
            and
            job:http_requests_success:rate6h{job="api-gateway"} < 0.994
          )
        for: 10m
        labels:
          severity: warning
          slo: "api-latency-availability"
        annotations:
          summary: "API error budget burning at 6x rate"
          description: |
            Monthly SLO budget at risk. Current 30m success rate: {{ $value | humanizePercentage }}

      # SLO BREACH: Actual 30-day window below 99.9%
      - alert: APISLOBreach
        expr: |
          job:http_requests_success:rate30d{job="api-gateway"} < 0.999
        for: 0m  # Instant alert — SLO breach is always critical
        labels:
          severity: critical
          slo: "api-latency-availability"
        annotations:
          summary: "API 30-day SLO breached"
          description: |
            30-day rolling success rate {{ $value | humanizePercentage }} < 99.9% SLO.
            Remaining error budget for month: {{ printf "%.2f" (($value - 0.999) * 100) }}%
```

**Grafana SLO dashboard query:**

```promql
# Current SLO attainment (last 30 days)
job:http_requests_success:rate30d{job="api-gateway"}

# Error budget remaining (fraction of 0.1% monthly budget remaining)
(job:http_requests_success:rate30d{job="api-gateway"} - 0.999) / (1 - 0.999)

# Error budget burn rate (how fast we're consuming budget)
(1 - job:http_requests_success:rate1h{job="api-gateway"}) /
(1 - job:http_requests_success:rate30d{job="api-gateway"})
```

---

## Q4: Compare InfluxDB, TimescaleDB, and ClickHouse for a high-cardinality IoT workload.

**Answer:**

**Workload:** 1M devices × 5 metrics × every 10 seconds = 500,000 writes/second, 5 billion
series total, query patterns: per-device range, cross-device aggregation, anomaly detection.

**InfluxDB v3 (Cloud or OSS):**

```python
from influxdb_client_3 import InfluxDBClient3
import pandas as pd

client = InfluxDBClient3(
    host="https://eu-central-1-1.aws.cloud2.influxdata.com",
    token="my-token",
    database="iot_metrics"
)

# Write using line protocol (compact, high-throughput)
line_protocol = "\n".join([
    f"sensor_reading,device_id={did},sensor_type={stype} value={val} {ts}"
    for did, stype, val, ts in readings_batch
])
client.write(record=line_protocol)

# Query using SQL (InfluxDB v3 supports SQL natively)
result = client.query("""
    SELECT
        device_id,
        DATE_TRUNC('hour', time) as hour,
        AVG(value) as avg_temp,
        MAX(value) as max_temp
    FROM sensor_reading
    WHERE sensor_type = 'temperature'
      AND time >= NOW() - INTERVAL '7 days'
    GROUP BY device_id, hour
    ORDER BY device_id, hour
""")
df = result.to_pandas()

# InfluxDB v3 verdict for this workload:
# ✓ Purpose-built for time-series, excellent write throughput
# ✓ InfluxDB v3 uses Apache Arrow + Parquet — no cardinality explosion
# ✗ Higher cost for managed cloud at this scale
# ✗ Less flexible for ad-hoc SQL queries vs TimescaleDB/ClickHouse
```

**TimescaleDB:**

```sql
-- Schema
CREATE TABLE iot_readings (
    time       TIMESTAMPTZ NOT NULL,
    device_id  TEXT NOT NULL,
    sensor     TEXT NOT NULL,
    value      DOUBLE PRECISION NOT NULL
);
SELECT create_hypertable('iot_readings', 'time',
    chunk_time_interval => INTERVAL '1 day',
    partitioning_column => 'device_id',
    number_partitions   => 16  -- Space partitioning for writes distribution
);

-- Compression: crucial for 1M device IoT workload
ALTER TABLE iot_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id, sensor',
    timescaledb.compress_orderby   = 'time DESC'
);
SELECT add_compression_policy('iot_readings', INTERVAL '3 days');

-- TimescaleDB verdict:
-- ✓ Full SQL: JOINs with device metadata tables, subqueries, CTEs
-- ✓ PostgreSQL ecosystem: pgvector, PostGIS, pg_partman
-- ✓ 10-20x compression on time-series
-- ✗ Single-node write scaling limited by PostgreSQL WAL throughput (~200K rows/sec)
-- ✗ Requires PostgreSQL operational expertise
```

**ClickHouse:**

```sql
-- Schema for IoT (MergeTree partitioned by day)
CREATE TABLE iot_readings ON CLUSTER 'iot_cluster' (
    date       Date        MATERIALIZED toDate(time),
    time       DateTime64(3),
    device_id  LowCardinality(String),  -- Dictionary-encoded: very efficient for high-repeat values
    sensor     LowCardinality(String),
    value      Float32
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/iot_readings', '{replica}')
PARTITION BY toYYYYMM(date)
ORDER BY (device_id, sensor, time)  -- Sorted for efficient per-device range scans
SETTINGS index_granularity = 8192;

-- Distributed table for transparent sharding
CREATE TABLE iot_readings_dist ON CLUSTER 'iot_cluster'
AS iot_readings
ENGINE = Distributed('iot_cluster', 'default', 'iot_readings',
    sipHash64(device_id)  -- Route writes to shards by device_id hash
);

-- ClickHouse verdict:
-- ✓ Fastest for analytical aggregations (columnar, SIMD-vectorized)
-- ✓ Handles 500K writes/sec on a single node (async INSERT batching)
-- ✓ Excellent compression (LZ4 or ZSTD) + columnar layout
-- ✗ No ACID transactions; duplicates possible during insert failures
-- ✗ JOIN performance worse than PostgreSQL for complex relational queries
-- ✗ "Eventually consistent" deduplication with ReplacingMergeTree

-- WINNER for this workload: ClickHouse
-- Reason: dominant workload is analytical (cross-device aggregations, anomaly detection)
-- Not: relational joins or transactional updates
```

**Decision matrix:**

```
                    InfluxDB v3  TimescaleDB  ClickHouse
Write throughput:      ★★★★☆      ★★★☆☆       ★★★★★
Analytical queries:    ★★★☆☆      ★★★★☆       ★★★★★
SQL compatibility:     ★★★☆☆      ★★★★★       ★★★★☆
Cardinality handling:  ★★★★☆      ★★★★★       ★★★★★
Operational simplicity:★★★★☆     ★★★☆☆       ★★★☆☆
Join support:          ★★☆☆☆      ★★★★★       ★★★★☆
Compression:           ★★★★☆      ★★★★★       ★★★★★
```
