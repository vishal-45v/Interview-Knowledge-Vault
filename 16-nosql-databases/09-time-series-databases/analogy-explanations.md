# Chapter 09: Time-Series Databases — Analogy Explanations

## Analogy 1: Cardinality Explosion — The Library's Infinite Card Catalog

**The Story:**
A library has a card catalog with one card per unique book. The catalog is kept entirely
in one small wooden cabinet — it must fit in the main foyer so every librarian can check it.
When books arrive, new cards are added. This works fine for 50,000 books. But one day, a
new librarian starts cataloging every COPY of every book with a unique card (including the
specific printing, binding, and purchase date). Suddenly the card catalog explodes to 50
million entries and can no longer fit in the foyer — it fills the entire building. No
librarian can navigate it, and checking whether a new book is in the catalog takes all day.

**Connection to the database:**
In InfluxDB, each unique combination of tags defines a "series." The series index is
kept in memory. A `request_id` tag (unique per request) creates one new series per
HTTP request. At 10M requests/hour, the series count grows by 10M/hour. The series
index cannot be evicted — it must stay in RAM to support fast writes and queries.

```python
# Cardinality monitoring script
from influxdb_client import InfluxDBClient

client = InfluxDBClient(url="http://localhost:8086", token="token", org="org")
query_api = client.query_api()

# Check total series count across all measurements
cardinality_query = '''
import "influxdata/influxdb/schema"
schema.measurementTagValues(bucket: "metrics", measurement: "http_requests", tag: "request_id")
|> count()
'''
# If this returns > 1M, you have a cardinality problem

# Identify high-cardinality tags before they become a problem
tag_cardinality_query = '''
import "influxdata/influxdb/schema"
schema.tagValues(bucket: "metrics", tag: "request_id")
|> count()
|> yield(name: "request_id_cardinality")
'''

# REMEDIATION: Remove the high-cardinality tag from new writes
# Change: tags: {service: "api", request_id: "abc123"}  ← BAD
# To:     tags: {service: "api"}, fields: {request_id: "abc123"}  ← GOOD
```

The rule of thumb: tags should have cardinality under 100,000 total unique values across
the entire bucket. Labels/tags work like the card catalog — they must fit in the foyer.
High-cardinality values belong in fields — they're like book content, stored in the stacks.

---

## Analogy 2: Prometheus Pull Model — The Doctor's Checkup vs the Patient Calling

**The Story:**
In a push-based health monitoring system, patients call the doctor every 10 seconds to
report their temperature. This floods the doctor's phone line. If a patient is fine,
those 8,640 calls per day are wasted. If a patient forgets to call (server is busy),
the doctor has no data — a silent failure.

Prometheus uses a pull model: the doctor (Prometheus) calls each patient (scrape target)
every 15 seconds. If a patient doesn't answer, Prometheus records a "scrape failed" metric
— the silence IS the signal. The doctor controls the schedule, not the patients.

**Connection to the database:**

```yaml
# prometheus.yml — Prometheus controls who is scraped and when
global:
  scrape_interval: 15s      # How often Prometheus calls each target
  evaluation_interval: 15s  # How often alerting rules are evaluated

scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with this annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: 'true'
      # Get the metrics port from the annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        target_label: __address__
        replacement: '${1}:${2}'

# PUSH equivalent (for comparison): Telegraf or StatsD agent
# Agent runs on each server and pushes metrics to InfluxDB
# Problem: if agent crashes, silence is indistinguishable from "server is fine"
# Prometheus solves this: scrape failure creates up=0 metric (explicit failure signal)
```

```promql
# Monitor scrape health — shows which targets are down
up{job="kubernetes-pods"} == 0
# Returns all pods that failed their last scrape

# Alert on scrape failure
- alert: TargetDown
  expr: up == 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Target {{ $labels.instance }} is down"
    description: "{{ $labels.job }}/{{ $labels.instance }} failed to scrape for 5 minutes"
```

The pull model's key advantage: the scraping schedule is centrally controlled, so you can
tell exactly what data you have and what's missing. In a push model, a quiet server and a
healthy server look identical until you investigate.

---

## Analogy 3: TimescaleDB Chunks — The Filing Cabinet with Labeled Drawers

**The Story:**
A law firm stores 10 years of case documents. Everything is in one giant room with no
organization — finding documents from last month means searching through 10 years of paper.
A smart paralegal introduces labeled drawers: one drawer per month. Now "find all documents
from January 2024" means opening exactly ONE drawer. Deleting old documents (7-year
retention) means removing entire drawer at once instead of individual papers.

TimescaleDB chunks are the drawers. The hypertable is the filing room. Each chunk covers
a specific time range. Queries with time predicates only open the relevant drawers.
Retention deletes entire drawer-sized chunks atomically (DROP TABLE, not DELETE).

**Connection to the database:**

```sql
-- Check chunk sizes and time ranges
SELECT
    chunk_schema,
    chunk_name,
    range_start::date,
    range_end::date,
    pg_size_pretty(total_bytes) AS size,
    pg_size_pretty(heap_bytes) AS heap_size,
    pg_size_pretty(index_bytes) AS index_size,
    compression_status
FROM timescaledb_information.chunks
WHERE hypertable_name = 'sensor_readings'
ORDER BY range_start DESC LIMIT 10;

-- Output shows:
-- _hyper_1_100_chunk  2024-01-15  2024-01-16  1.2 GB  (compressed: 120 MB)
-- _hyper_1_99_chunk   2024-01-14  2024-01-15  1.1 GB  (compressed: 110 MB)

-- This query opens ONLY the chunk covering 2024-01-15:
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*)
FROM sensor_readings
WHERE time BETWEEN '2024-01-15' AND '2024-01-16';
-- Shows: Custom Scan (ChunkAppend) → Seq Scan on _hyper_1_100_chunk
-- Chunks for other days: NOT IN PLAN (excluded at query time)

-- Compression ratio per chunk
SELECT
    chunk_name,
    before_compression_total_bytes / after_compression_total_bytes AS compression_ratio
FROM chunk_compression_stats('sensor_readings')
WHERE after_compression_total_bytes IS NOT NULL;
-- Typical: 10-20x compression ratio
```

The chunk architecture enables "partition-wise" operations that PostgreSQL cannot do on
a flat table: compress each chunk individually with optimal settings (recent chunks
uncompressed for fast writes, old chunks heavily compressed for storage efficiency).

---

## Analogy 4: PromQL Range Vectors — The Time-Lapse Camera

**The Story:**
You're monitoring plant growth. A photographer points a camera at a plant and takes a
photo every 15 minutes (scrape interval). To measure growth RATE, you compare the plant's
height in photos from the last 5 minutes (instant comparison) vs photos from the last
2 hours (trend analysis). The 5-minute comparison shows current speed. The 2-hour
comparison shows the average speed, filtering out brief spurts or pauses.

In PromQL, `metric[5m]` is the "last 5 minutes of photos" — a range vector. Functions
like `rate()` and `increase()` analyze that time window.

**Connection to the database:**

```promql
# INSTANT VECTOR: a single value at this moment
http_requests_total{job="api"}
# Returns: {job="api", instance="server-1"} → 1,542,891 (current count)

# RANGE VECTOR: all samples in the last 5 minutes
http_requests_total{job="api"}[5m]
# Returns: {job="api", instance="server-1"} → [(T-4m, 1542800), (T-3m, 1542830), ...]
# Cannot use range vectors in alerts directly — must apply a function

# RATE: average per-second rate over the range window
rate(http_requests_total{job="api"}[5m])
# Returns: ~0.5 (if ~150 requests occurred in 5 minutes: 150/300s = 0.5/s)
# rate() handles counter resets automatically

# INCREASE: total increase over the range (rate * duration)
increase(http_requests_total{job="api"}[5m])
# Returns: ~150 (total requests in 5 minutes)

# The "5m" window is the LOOKBACK WINDOW, not the averaging window
# If there are no samples in the last 5m, result is empty (no data, not 0)
# This is why "no data" alerts are different from "value is 0" alerts

# Visualization: instant vector for dashboards
# rate(requests[1m]) — smooth but responds to changes quickly
# rate(requests[15m]) — very smooth, good for trends
# irate(requests[5m]) — spiky, shows instantaneous rate between last 2 scrapes
```

---

## Analogy 5: ClickHouse MergeTree Parts — The Sorting Hat in a Filing Office

**The Story:**
New paper documents arrive at a filing office continuously. Rather than immediately
filing each document in its final alphabetical location (too slow during busy hours),
a clerk creates a fresh folder for each batch of documents that arrives, sorted
internally. Throughout the day, background workers merge smaller folders into larger,
more completely sorted folders. By end of day, there are a few large, well-sorted folders
instead of thousands of tiny ones. The documents are always readable from any folder;
the merging just makes future reading faster.

**Connection to the database:**

```sql
-- Each INSERT creates a new "part" (like a fresh folder)
INSERT INTO http_logs VALUES
    ('2024-01-15', now(), 'api-service', 200, 142.3, '/users/profile');
-- Creates: part "20240115_1_1_0" (date range, block range, level)

-- Background merge: parts are merged into larger sorted parts
-- Part 20240115_1_1_0 + Part 20240115_2_2_0 → Part 20240115_1_2_1 (level 1)
-- Part 20240115_1_2_1 + Part 20240115_3_4_1 → Part 20240115_1_4_2 (level 2)

-- See current parts in a table
SELECT
    partition,
    name,
    rows,
    bytes_on_disk,
    modification_time,
    level
FROM system.parts
WHERE table = 'http_logs' AND active = 1
ORDER BY modification_time DESC;

-- Parts at level 0 = recently inserted, not yet merged
-- High number of parts = inserts are too small (batch your inserts!)
-- Rule of thumb: insert batches of > 1000 rows, let ClickHouse merge to few parts

-- Force a manual merge (useful for testing or admin operations)
OPTIMIZE TABLE http_logs PARTITION '202401' FINAL;
-- FINAL keyword: merge until only one part remains per partition
```

ClickHouse's MergeTree design means reads are always consistent (all parts are visible),
writes are always fast (just create a new part), and the background merge process
optimizes storage and read performance over time without blocking operations.

---

## Analogy 6: Downsampling Tiers — Wine Aging and Storage

**The Story:**
A winery keeps different vintages of wine under different conditions. This year's
new wine is kept in active barrels — you can taste it any time, add to it, track its
daily progress. Wine from 5 years ago is bottled and stored in a climate-controlled
cellar — you can drink it any time but can no longer add to it. Wine from 20 years ago
is in a special vault — archived, sealed, you read about it in the catalog. Each tier
has different cost, accessibility, and detail level.

Time-series downsampling is wine aging for metrics data. Raw data (this year's wine)
is expensive to store but maximally detailed. Downsampled hourly averages (5-year-old
bottles) are cheaper and sufficient for trend analysis. Archive summaries (20-year vault)
are almost free but only tell you the broad strokes.

**Connection to the database:**

```
DOWNSAMPLING TIER ARCHITECTURE:

Tier 0: Raw (Active barrels)
├── Resolution: 10 seconds
├── Retention: 30 days
├── Storage: 50GB/day × 30 = 1.5TB
├── Query: point lookups, recent anomaly detection
└── Cost: HIGH (hot storage, SSD)

Tier 1: 1-minute aggregates (Bottled wine)
├── Resolution: 1 minute
├── Retention: 1 year
├── Storage: 1GB/day × 365 = 365GB
├── Columns: avg, min, max, p95, count per window
└── Cost: MEDIUM (warm storage, spinning disk)

Tier 2: 1-hour aggregates (Cold cellar)
├── Resolution: 1 hour
├── Retention: 5 years
├── Storage: 20MB/day × 1825 = 36GB
├── Columns: avg, min, max, count (p95 derived from tier 1 if kept)
└── Cost: LOW (cold storage, object store)
```

```sql
-- Query routing: pick the right tier automatically
CREATE OR REPLACE FUNCTION query_metrics(
    p_metric TEXT,
    p_start  TIMESTAMPTZ,
    p_end    TIMESTAMPTZ
) RETURNS TABLE(bucket TIMESTAMPTZ, value DOUBLE PRECISION) AS $$
DECLARE
    v_duration INTERVAL := p_end - p_start;
    v_now TIMESTAMPTZ := NOW();
BEGIN
    IF p_end > v_now - INTERVAL '30 days' AND v_duration <= INTERVAL '7 days' THEN
        -- Recent, short range: use raw data
        RETURN QUERY
        SELECT time_bucket('1 minute', time), AVG(value)
        FROM metrics_raw
        WHERE metric_name = p_metric AND time BETWEEN p_start AND p_end
        GROUP BY 1 ORDER BY 1;

    ELSIF v_duration <= INTERVAL '90 days' THEN
        -- Medium range: use 1-minute aggregates
        RETURN QUERY
        SELECT bucket, avg FROM metrics_1min
        WHERE metric_name = p_metric AND bucket BETWEEN p_start AND p_end
        ORDER BY bucket;

    ELSE
        -- Long range: use hourly aggregates
        RETURN QUERY
        SELECT bucket, avg FROM metrics_1hr
        WHERE metric_name = p_metric AND bucket BETWEEN p_start AND p_end
        ORDER BY bucket;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

---

## Analogy 7: Prometheus Histograms — The Highway Speed Camera

**The Story:**
A highway speed camera doesn't record the exact speed of every car. Instead, it counts
how many cars passed in speed brackets: "0-30mph: 5 cars, 30-60mph: 200 cars, 60-80mph:
1,500 cars, 80-100mph: 300 cars, 100+mph: 3 cars." From these buckets, you can estimate
that 99% of cars drove under 95mph (the 100+mph bucket has very few cars). The exact
99th percentile — you'd need to know each individual speed — is approximated from the
bucket boundaries.

This is exactly how Prometheus histograms work. The "le" label stands for "less than
or equal" — the bucket's upper bound. Percentile estimation (histogram_quantile) uses
linear interpolation within buckets.

**Connection to the database:**

```python
# Application instrumentation (Python with prometheus_client)
from prometheus_client import Histogram

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration in seconds',
    ['method', 'endpoint', 'status'],
    buckets=[
        0.005, 0.010, 0.025,   # Sub-10ms (fast path)
        0.050, 0.100,           # 10-100ms (normal)
        0.250, 0.500,           # 100-500ms (slow but ok)
        1.0, 2.5, 5.0, 10.0   # >1s (problematic)
    ]
)

@app.route('/users/<user_id>')
def get_user(user_id):
    with REQUEST_LATENCY.labels(
        method='GET',
        endpoint='/users/{id}',
        status='200'
    ).time():
        return fetch_user(user_id)

# This generates metrics like:
# http_request_duration_seconds_bucket{le="0.05"} 8432
# http_request_duration_seconds_bucket{le="0.1"}  9801
# http_request_duration_seconds_bucket{le="0.25"} 9998
# http_request_duration_seconds_bucket{le="+Inf"} 10000
# http_request_duration_seconds_count 10000
# http_request_duration_seconds_sum   523.4
```

```promql
# P99 latency — interpolated from buckets
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket{endpoint="/users/{id}"}[5m]))
)
# Returns approximately 0.23 seconds
# Accuracy: ±half-bucket-width around the true P99
# If true P99 is 0.23, and bucket boundaries are 0.1 and 0.25, error is ±0.075

# For better P99 accuracy at the SLO boundary (500ms):
# Add more buckets in the 250-750ms range:
# buckets = [..., 0.25, 0.3, 0.4, 0.5, 0.6, 0.75, 1.0, ...]
# Now the bucket width around 500ms is 100ms → error ±50ms
```
