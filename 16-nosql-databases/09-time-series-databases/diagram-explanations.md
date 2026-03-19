# Chapter 09: Time-Series Databases — Diagram Explanations

## Diagram 1: InfluxDB Data Model — Measurement, Tags, Fields, Series

```
INFLUXDB DATA MODEL HIERARCHY
═══════════════════════════════════════════════════════════════════════════

MEASUREMENT: "cpu_usage"  (like a SQL table name)
│
├── SERIES 1: tags = {host="server-01", region="us-east", datacenter="dc1"}
│   │                └─────── SERIES KEY (indexed, in-memory) ──────────┘
│   └── DATA POINTS (time-ordered):
│       [T=1705000000, value=45.2, iowait=0.3, user=40.1, system=4.8]
│       [T=1705000010, value=47.8, iowait=0.4, user=42.0, system=5.4]
│       [T=1705000020, value=51.2, iowait=2.1, user=44.5, system=4.6]
│                        └── FIELDS (not indexed, stored in time blocks) ─┘
│
├── SERIES 2: tags = {host="server-02", region="us-east", datacenter="dc1"}
│   └── DATA POINTS: [T=1705000000, value=32.1, ...]
│
└── SERIES N: tags = {host="server-N", region="eu-west", datacenter="dc2"}
    └── DATA POINTS: [T=1705000000, value=67.4, ...]

CARDINALITY = number of unique series = N

LOW CARDINALITY (good):
  tags: {service, environment, region}
  3 services × 3 environments × 5 regions = 45 unique series
  All 45 series keys fit in a few KB of memory

HIGH CARDINALITY (dangerous):
  tags: {user_id, session_id, request_id}
  1M users × 5 sessions × 100K requests = 500 BILLION potential series
  Series index would require terabytes of RAM → instant OOM

TAG vs FIELD DECISION TREE:
  "Will I filter/group by this value in WHERE or GROUP BY clauses?"
    YES → Consider it a tag (IF cardinality is low, < 100K unique values)
    NO  → Use it as a field

  "Is the cardinality of this value bounded and manageable?"
    YES (e.g., http_method: GET/POST/PUT/DELETE = 4 values) → tag
    NO  (e.g., user_id = millions of unique values)          → field

LINE PROTOCOL (wire format for writes):
measurement,tag1=val1,tag2=val2 field1=val1,field2=val2 timestamp_nanoseconds
cpu_usage,host=server-01,region=us-east value=45.2,iowait=0.3 1705000000000000000
│          │                             │                      │
│          └─ tags (indexed, in-memory)  └─ fields (data)       └─ unix nanoseconds
└─ measurement
```

**Explanation:**
The separation between tags and fields is the most important design decision in InfluxDB.
Tags form the series key — every unique combination of tag values creates a separate in-memory
series entry. Fields are the measurements themselves — they're stored in compressed, time-ordered
blocks within the series and are NOT indexed. Mixing high-cardinality data into tags is the
single most common cause of InfluxDB performance degradation and OOM crashes.

---

## Diagram 2: Prometheus Pull Architecture and Data Flow

```
PROMETHEUS ECOSYSTEM ARCHITECTURE
═══════════════════════════════════════════════════════════════════════════

                   DISCOVERY                    SCRAPE
┌──────────────────────────────────────────────────────────────────────┐
│  Target Discovery Sources:          Prometheus Server                │
│  ┌──────────────┐                  ┌───────────────────────────────┐ │
│  │ Kubernetes   │──── pods ──────►│    Scrape Scheduler           │ │
│  │ API Server   │    services     │    (every 15s per target)      │ │
│  └──────────────┘                  │           │                   │ │
│  ┌──────────────┐                  │    HTTP GET /metrics          │ │
│  │ Consul/      │──── services ──►│           ▼                   │ │
│  │ etcd         │                  │    ┌─────────────┐            │ │
│  └──────────────┘                  │    │ Time-Series │            │ │
│  ┌──────────────┐                  │    │   Database  │            │ │
│  │ Static config│──── hosts ─────►│    │  (TSDB)     │            │ │
│  └──────────────┘                  │    └─────────────┘            │ │
│                                    │           │                   │ │
│                                    │    ┌─────────────┐            │ │
│                                    │    │  WAL (write │            │ │
│                                    │    │  ahead log) │            │ │
│                                    │    └─────────────┘            │ │
│                                    └───────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
         │ /metrics endpoint (HTTP)
         │ Text format:
         │ # HELP http_requests_total Total HTTP requests
         │ # TYPE http_requests_total counter
         │ http_requests_total{method="GET",status="200"} 1542891
         │ http_requests_total{method="POST",status="201"} 89432

TARGET ENDPOINTS (expose /metrics):
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────────┐
│ Application│  │   Node     │  │  Database  │  │  Pushgateway   │
│  (direct)  │  │  Exporter  │  │  Exporter  │  │ (batch jobs)   │
│ /metrics   │  │ /metrics   │  │ /metrics   │  │ /metrics       │
└────────────┘  └────────────┘  └────────────┘  └────────────────┘

LONG-TERM STORAGE (remote_write):
Prometheus ──remote_write──► Thanos Sidecar ──► Object Store (S3)
                                    │
                                    └──► Thanos Store   ──► Thanos Querier ──► Grafana
                          ← Historical data ──► Same query interface ──►
```

**Explanation:**
Prometheus's pull model centralizes control of the scrape schedule in the Prometheus server.
Each target exposes a `/metrics` HTTP endpoint in the Prometheus text exposition format or
OpenMetrics format. Prometheus fetches these on a configurable interval (15s default).
The `up{job="...", instance="..."}` metric is automatically created — value 1 if scrape
succeeded, 0 if it failed — making monitoring "silence" explicit rather than ambiguous.

---

## Diagram 3: TimescaleDB Hypertable Chunk Architecture

```
TIMESCALEDB HYPERTABLE: sensor_readings
═══════════════════════════════════════════════════════════════════════════

Logical view (what you query):
┌─────────────────────────────────────────────────────────────────────┐
│  SELECT * FROM sensor_readings WHERE time > NOW() - INTERVAL '7d'   │
│  (looks like a normal PostgreSQL table)                             │
└─────────────────────────────────────────────────────────────────────┘

Physical storage (actual implementation):
┌────────────────────────────────────────────────────────────────────┐
│  Hypertable: sensor_readings                                        │
│  Partition dimensions: time (interval: 1 day)                      │
│                        device_id (16 space partitions)             │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │ Chunk           │  │ Chunk           │  │ Chunk           │    │
│  │_hyper_1_100     │  │_hyper_1_99      │  │_hyper_1_50      │    │
│  │Time: 2024-01-15 │  │Time: 2024-01-14 │  │Time: 2023-12-26 │    │
│  │Device: 0-1/16   │  │Device: 0-1/16   │  │Device: 0-1/16   │    │
│  │Size: 1.2 GB     │  │Size: 1.1 GB     │  │Size: 110 MB     │    │
│  │Status: ACTIVE   │  │Status: ACTIVE   │  │Status: COMPRESSED│   │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐                          │
│  │ Chunk           │  │ Chunk           │                          │
│  │_hyper_1_100     │  │_hyper_1_99      │                          │
│  │Time: 2024-01-15 │  │Time: 2024-01-14 │                          │
│  │Device: 1-2/16   │  │Device: 1-2/16   │  ... 16 space partitions │
│  └─────────────────┘  └─────────────────┘                          │
└────────────────────────────────────────────────────────────────────┘

CHUNK EXCLUSION FOR RANGE QUERY:
SELECT avg(temperature)
FROM sensor_readings
WHERE time > '2024-01-14' AND time < '2024-01-16'   ← 2-day range
  AND device_id = 'device-001'

PLAN: Only opens chunks _hyper_1_99 and _hyper_1_100 for device-001 partition
      Skips: ALL other 98 chunks (chunk exclusion at query planning time)

COMPRESSION COMPARISON:
┌──────────────────────────────────┬──────────────────────────────────┐
│ Uncompressed chunk (recent)      │ Compressed chunk (>7 days old)   │
│ Size: 1.2 GB                     │ Size: 60-120 MB (10-20x ratio)   │
│ Read: direct PostgreSQL pages    │ Read: decompress batch on access  │
│ Write: normal INSERT/UPDATE      │ Write: NOT DIRECTLY WRITABLE      │
│ Index: standard B-tree           │ Index: metadata only + bloom      │
└──────────────────────────────────┴──────────────────────────────────┘

COMPRESSION ALGORITHM (columnar within chunk):
Time column:    Delta-delta encoding (successive differences → small integers)
  Raw:    [1705000000, 1705000010, 1705000020, 1705000030, ...]
  Delta:  [10, 10, 10, 10, ...]  → run-length encoded: (10, repeat=8640) = 4 bytes

Value column:   LZ4/ZSTD compression on delta-encoded float values
device_id:      Dictionary encoding (repeated strings → integer IDs)
```

**Explanation:**
TimescaleDB's chunk architecture turns a 2TB table into thousands of small, independently
manageable tables (chunks). The query planner's "constraint exclusion" feature eliminates
all chunks outside the query's time range at planning time — the database never touches
those chunks' data files. Compression is applied per-chunk after the chunk exits the
"recent writes" window, allowing append-heavy recent data to use standard row storage
while archiving older data with 10-20x compression ratios.

---

## Diagram 4: ClickHouse MergeTree — Parts, Granules, and Sparse Index

```
CLICKHOUSE MERGETREE STORAGE STRUCTURE
═══════════════════════════════════════════════════════════════════════════

TABLE: http_logs (ORDER BY (service, date, timestamp))
Data is SORTED by (service, date, timestamp) — this IS the sparse index key

PART: 20240115_1_5_2 (partition=2024-01, blocks 1-5, merge level 2)
│
├── data.bin (columnar data, compressed)
│   ┌───────────────────────────────────────────────────────────────┐
│   │ GRANULE 0 (8,192 rows): rows 0-8191                           │
│   │   service col: [api,api,api,...,api]         (all "api")       │
│   │   date col:    [2024-01-15,...,2024-01-15]   (all Jan 15)      │
│   │   timestamp:   [10:00:00, 10:00:01, ...]                       │
│   │   status col:  [200, 200, 404, 200, ...]                       │
│   │   duration col:[142.3, 89.1, 201.5, ...]                       │
│   ├───────────────────────────────────────────────────────────────┤
│   │ GRANULE 1 (8,192 rows): rows 8192-16383                        │
│   │   service col: [api,...,payment,payment,...]  (mixed services)  │
│   │   ...                                                           │
│   └───────────────────────────────────────────────────────────────┘
│
├── primary.idx (sparse index — ONE entry PER GRANULE)
│   Entry 0:  {service="api",     date=2024-01-15, timestamp=10:00:00}
│   Entry 1:  {service="api",     date=2024-01-15, timestamp=10:01:22}
│   Entry 2:  {service="payment", date=2024-01-15, timestamp=10:02:44}
│   ...
│   ← Covers 1 billion rows with ~120K index entries (8KB each)
│   ← Fits in RAM easily even for very large tables
│
└── [mark files, checksums, metadata]

QUERY EXECUTION:
SELECT count(), avg(duration_ms)
FROM http_logs
WHERE service = 'api' AND date = '2024-01-15'

Step 1: Use sparse index to find granules WHERE service starts at 'api'
        Skip all granules where primary.idx shows service > 'api' (e.g., 'payment')
        → Reduces from 1M granules to ~50,000 granules for 'api'

Step 2: For matching granules, read ONLY the 'duration_ms' and 'status' columns
        (other 8 columns are never read from disk → columnar advantage)

Step 3: Apply SIMD vectorized aggregation on the column data

SKIP INDEX (secondary index for non-primary-key columns):
CREATE TABLE http_logs ... INDEX status_idx status TYPE minmax GRANULARITY 4;
  → For each group of 4 granules, stores (min_status, max_status)
  → Query WHERE status = 404 skips granule groups where max_status < 404 or min_status > 404
  → Not as powerful as primary key but helps for low-cardinality column filters
```

**Explanation:**
ClickHouse's sparse index is the key to its analytical query performance. The primary key
defines the physical sort order AND the sparse index. With one index entry per 8,192 rows
(one "granule"), the entire index for a billion-row table fits in a few hundred MB of RAM.
Columnar storage means aggregation queries read only the relevant columns — a query on
duration_ms and service never touches the url column, even though both are in the same table.
This combination of sparse indexing plus columnar I/O gives ClickHouse its characteristic
10-100x query speedup over row-oriented databases for analytical workloads.
