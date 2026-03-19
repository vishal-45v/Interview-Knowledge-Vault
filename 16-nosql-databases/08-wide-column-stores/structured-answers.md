# Chapter 08: Wide-Column Stores — Structured Answers

## Q1: Explain the LSM-Tree write and read path in detail, including write amplification.

**Answer:**

The Log-Structured Merge Tree is the storage engine beneath HBase, Cassandra, LevelDB,
and RocksDB. It trades increased write throughput for potentially increased read latency,
managed through compaction.

**Write path (fast path):**

```
User Write: PUT(key="user:1001", value={name: "Alice", age: 32})

Step 1: WAL (Write-Ahead Log) — sequential append to disk
  └── Provides durability: if server crashes after WAL write, replay recovers data
  └── WAL is per-RegionServer in HBase; per-node in Cassandra (CommitLog)

Step 2: MemTable — in-memory sorted structure (usually a red-black tree or skip list)
  └── Sorted by key for efficient merge
  └── Writes return to client HERE — no disk read required
  └── HBase MemStore; Cassandra MemTable

Step 3: MemTable Flush — when MemTable reaches threshold (e.g., 128MB)
  └── Sorted MemTable written to disk as an SSTable (L0 file)
  └── SSTable is immutable once written — never modified in place
  └── Multiple L0 files accumulate: SSTable-001, SSTable-002, ...
```

**Read path (potentially slow):**

```
User Read: GET(key="user:1001")

Step 1: Check MemTable — O(log n) in-memory lookup
  └── HIT: return immediately (best case)
  └── MISS: continue to disk

Step 2: Check Bloom Filter for each SSTable at each level
  └── Bloom filter says "NOT PRESENT" → skip this file entirely
  └── Bloom filter says "POSSIBLY PRESENT" → must check file

Step 3: For each file not filtered out, binary search the SSTable index
  └── SSTable index is loaded in memory (block index)
  └── Seek to the block containing the key
  └── Decompress and scan the block

Step 4: Return the most recent version (highest timestamp) found across all files

WORST CASE: MemTable miss + all Bloom filters say "possibly present"
  → Read amplification = N files read for 1 logical record
```

**Write amplification explained:**

```
Level 0:   4 SSTables (threshold: 4 files triggers L0→L1 compaction)
Level 1:   10 SSTables, max 10MB total (one 10MB file per 10MB budget)
Level 2:   100 SSTables, max 100MB total
Level 3:   1000 SSTables, max 1GB total
Level 4:   10000 SSTables, max 10GB total

Leveled Compaction Write Amplification:
When a key moves from L0 → L1:
  - Key is read from L0 file
  - Merged with the ENTIRE L1 file that overlaps in key range
  - Written to new L1 file
  WA at L0→L1: ~10x (rewrite 10MB L1 for each 1MB of new L0 data)

  Repeat for L1→L2: ~10x
  Total WA: ~10 + 10 + 10 + 10 = ~40x (each byte written 40 times)

Size-Tiered Compaction Write Amplification:
  - Compact similarly-sized SSTables together
  - WA ≈ log_base(multiplier)(total_data_size / memtable_size)
  - Lower WA but more files at each tier → higher read amplification
```

**Practical implications for HBase:**

```bash
# Monitor write amplification in HBase
# Via JMX metrics
curl http://regionserver:16030/jmx \
  | jq '.beans[] | select(.name | contains("RegionServer,sub=Server")) |
    {
      "bytesIn": .bytesInRemoteResults,
      "compactionBytesWritten": .compactionOutputFileBytes,
      "flushedBytes": .flushedCellsSize
    }'

# Tuning: increase memtable size to reduce L0→L1 compaction frequency
# hbase-site.xml
# <property>
#   <name>hbase.hregion.memstore.flush.size</name>
#   <value>268435456</value>  <!-- 256MB instead of default 128MB -->
# </property>

# Monitor SSTable file count (high count = high read amplification)
hbase shell
> count_regions 'mytable'  # High region count may indicate too many small SSTables
```

---

## Q2: Design a high-performance HBase row key for a time-series IoT sensor platform.

**Answer:**

**Requirements:**
- 500,000 devices, 10M readings/minute
- Query patterns: latest N readings per device, range by time, cross-device aggregation

**Anti-patterns to avoid:**

```
ANTI-PATTERN 1: Sequential timestamp as row key prefix
"2024011512000000001_device-001"  ← All writes go to last region (hotspot)

ANTI-PATTERN 2: Random UUID as row key
"550e8400-e29b-41d4-a716-446655440000"  ← Writes distributed but impossible to range scan

ANTI-PATTERN 3: Device ID only
"device-001"  ← All data for same device in same region (cannot scale per device)
```

**Recommended schema:**

```java
import org.apache.hadoop.hbase.util.Bytes;
import java.security.MessageDigest;

public class IoTRowKey {

    /**
     * Row key format: SALT(1B) + DEVICE_ID(8B) + INVERTED_TIMESTAMP(8B)
     *
     * SALT: distributes writes across regions
     * DEVICE_ID: groups all readings for a device together (good for range scan)
     * INVERTED_TIMESTAMP: newest data first within each device
     */
    public static byte[] buildRowKey(String deviceId, long timestamp) {
        // 1-byte salt: hash(deviceId) % NUM_REGIONS
        byte salt = (byte)(deviceId.hashCode() % 16);  // 16 salt buckets

        // Device ID: padded to fixed length for lexicographic ordering
        byte[] deviceIdBytes = Bytes.toBytes(
            String.format("%-20s", deviceId).replace(' ', '0')
        );

        // Inverted timestamp: newest reads come first in range scan
        long invertedTs = Long.MAX_VALUE - timestamp;
        byte[] tsBytes = Bytes.toBytes(invertedTs);

        return Bytes.add(new byte[]{salt}, deviceIdBytes, tsBytes);
    }

    public static byte[] buildScanStart(String deviceId) {
        byte salt = (byte)(deviceId.hashCode() % 16);
        byte[] deviceIdBytes = Bytes.toBytes(String.format("%-20s", deviceId).replace(' ', '0'));
        return Bytes.add(new byte[]{salt}, deviceIdBytes, Bytes.toBytes(Long.MIN_VALUE));
        // Long.MIN_VALUE as inverted ts = furthest-future timestamp = scan start
    }

    public static byte[] buildScanStop(String deviceId) {
        byte salt = (byte)(deviceId.hashCode() % 16);
        byte[] deviceIdBytes = Bytes.toBytes(String.format("%-20s", deviceId).replace(' ', '0'));
        // Increment device ID byte to get scan stop (exclusive)
        deviceIdBytes[deviceIdBytes.length - 1]++;
        return Bytes.add(new byte[]{salt}, deviceIdBytes);
    }
}

// HBase table schema
HTableDescriptor tableDesc = new HTableDescriptor(TableName.valueOf("sensor_data"));

// Column family 1: raw readings (high write rate, short retention)
HColumnDescriptor rawCF = new HColumnDescriptor("r");
rawCF.setMaxVersions(1);                          // Only keep latest version
rawCF.setTimeToLive(7 * 24 * 3600);              // 7-day TTL
rawCF.setCompressionType(Compression.Algorithm.SNAPPY);
rawCF.setBloomFilterType(BloomType.ROW);
rawCF.setDataBlockEncoding(DataBlockEncoding.FAST_DIFF);  // Prefix compression
tableDesc.addFamily(rawCF);

// Column family 2: aggregated data (lower write rate, long retention)
HColumnDescriptor aggCF = new HColumnDescriptor("a");
aggCF.setMaxVersions(1);
aggCF.setTimeToLive(365 * 24 * 3600);            // 1-year TTL
aggCF.setCompressionType(Compression.Algorithm.GZ);  // Better ratio for older data
tableDesc.addFamily(aggCF);

// Pre-split into 16 regions (matching our 16 salt values)
byte[][] splits = new byte[15][];
for (int i = 1; i <= 15; i++) {
    splits[i-1] = new byte[]{(byte) i};
}
admin.createTable(tableDesc, splits);
```

**Query implementations:**

```java
// Query 1: Latest 100 readings for device-001
Scan scan = new Scan();
scan.setStartRow(IoTRowKey.buildScanStart("device-001"));
scan.setStopRow(IoTRowKey.buildScanStop("device-001"));
scan.addFamily(Bytes.toBytes("r"));
scan.setMaxResultSize(100);
scan.setCaching(100);
scan.setReversed(false);  // Forward scan = newest first (due to inverted timestamp)

// Query 2: Readings in time range 2024-01-15 to 2024-01-16
long startTs = parseDate("2024-01-15").getTime();
long endTs   = parseDate("2024-01-16").getTime();
// Inverted: high timestamp value = earlier time
scan.setStartRow(IoTRowKey.buildRowKey("device-001", endTs));    // End time = start of scan
scan.setStopRow(IoTRowKey.buildRowKey("device-001", startTs));   // Start time = end of scan
```

---

## Q3: Compare leveled compaction vs size-tiered compaction with concrete workload examples.

**Answer:**

**Leveled Compaction (default in Cassandra STCS → LCS, default in RocksDB):**

```
Structure:
L0: [SST-A][SST-B][SST-C][SST-D]  ← Unordered, may overlap in key space
L1: [0-99][100-199][200-299] ...    ← Sorted, NO key range overlap between files
L2: [0-9][10-19][20-29] ...         ← Sorted, NO key range overlap, 10x larger than L1

Key property: Each key exists in AT MOST ONE file at levels L1+
→ Point reads require checking: MemTable + Bloom filters + AT MOST 1 file per level
→ Read amplification = O(num_levels) ≈ 4-5 reads for most workloads
→ Write amplification ≈ 10-40x (keys rewritten at each level crossing)
```

**Size-Tiered Compaction (default Cassandra STCS, HBase default):**

```
Structure:
Tier 1: [1MB][1MB][1MB][1MB]         ← 4 ~1MB files → compact to 1 4MB file
Tier 2: [4MB][4MB][4MB][4MB]         ← 4 ~4MB files → compact to 1 16MB file
Tier 3: [16MB][16MB][16MB][16MB]     → compact to 1 64MB file
Tier 4: [64MB]...

Key property: Files at same tier have same size; trigger compaction when 4+ files exist
→ Write amplification ≈ log(N/L₀) ≈ 2-10x (lower than leveled!)
→ Read amplification: high (multiple files at each tier may contain the same key)
→ Space amplification: high (duplicate data across tiers during compaction)
```

**Choose leveled compaction when:**
```python
# Example: User profile reads — random point lookups dominate
# Leveled ensures O(num_levels) file reads per lookup

# Profile: 80% reads, 20% writes
# Read pattern: random point lookups (user_id → profile)
# Write pattern: updates to existing records
use_leveled_compaction = {
    "reason": "Read amplification matters most",
    "read_amp": 4,       # O(levels)
    "write_amp": 30,     # Acceptable for read-heavy workload
    "space_amp": 1.1,    # Good — no significant duplicate data
    "works_best_for": ["read-heavy", "random point lookups", "update-heavy"]
}
```

**Choose size-tiered compaction when:**
```python
# Example: append-only event log — write throughput is critical
# STCS maximizes write throughput by minimizing write amplification

# Profile: 99% writes (append-only), 1% reads (sequential scans)
# Write pattern: time-series events, never updated
# Read pattern: full table scans for analytics
use_size_tiered = {
    "reason": "Write throughput matters most",
    "write_amp": 5,      # Low — each byte rewritten fewer times
    "read_amp": 20,      # High — but scans are sequential (cached)
    "space_amp": 1.5,    # Higher — duplicates during compaction
    "works_best_for": ["write-heavy", "append-only", "sequential scans", "time-series"]
}
```

**Hybrid: TWCS (Time-Window Compaction Strategy) for time-series:**
```python
# Cassandra TWCS compacts data within the SAME time window together
# Data outside the current window is NEVER rewritten (zero WA for old data)
# Perfect for time-series where old data is immutable

# Configuration
CREATE TABLE sensor_readings (
    device_id UUID,
    time TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY (device_id, time)
) WITH compaction = {
    'class': 'TimeWindowCompactionStrategy',
    'compaction_window_unit': 'HOURS',
    'compaction_window_size': 1,    -- Compact within 1-hour windows
    'min_threshold': 4,
    'max_threshold': 32
} AND default_time_to_live = 2592000;  -- 30 days TTL
```

---

## Q4: Explain HBase's write path from client Put() through to durable storage.

**Answer:**

```
CLIENT PUT() REQUEST FLOW:
═══════════════════════════════════════════════════════════════════════════

Put put = new Put(Bytes.toBytes("user:1001"));
put.addColumn("profile", "email", "alice@example.com");
table.put(put);

STEP 1: Client → ZooKeeper (one-time bootstrap)
  ZooKeeper holds: /hbase/meta-region-server = RegionServer-2:16020
  Client reads this ONCE, caches for session duration

STEP 2: Client → hbase:meta table (on RegionServer-2)
  META table maps: "user:1001" → RegionServer-4 hosts region "user_table,,1234567890"
  Client caches this mapping for future requests

STEP 3: Client → RegionServer-4 (direct RPC, bypasses HMaster)
  HMaster is NOT in the write path — it only handles admin operations

ON REGIONSERVER-4:
STEP 4: WAL (Write-Ahead Log) write — SYNCHRONOUS by default
  WAL entry: {sequence_id: 1234, region: "user_table,,1234567890",
               action: PUT, row: "user:1001", family: "profile",
               qualifier: "email", value: "alice@example.com", timestamp: now}

  Durability options:
  - SYNC_WAL: fsync WAL to disk (slowest, safest — default)
  - FSYNC_WAL: fsync WAL + parent directory inode (paranoid mode)
  - ASYNC_WAL: write to OS buffer, no fsync (fast, data loss on OS crash)
  - SKIP_WAL: no WAL write at all (fastest, cannot recover from crash)

STEP 5: MemStore write — IN MEMORY
  MemStore is a sorted ConcurrentSkipListMap in the JVM heap
  Write returns to CLIENT here — data is durable in WAL, visible in MemStore

STEP 6: MemStore flush (asynchronous background process)
  Triggers when:
  - MemStore size > hbase.hregion.memstore.flush.size (128MB default)
  - Total MemStore across all regions > hbase.regionserver.global.memstore.size
  - Flush operation:
    a. Create new "snapshot" of MemStore
    b. Write snapshot to HFile in HDFS (sequential write)
    c. Update HFile reference in region's StoreFile list
    d. WAL segments up to this point can be archived/deleted

STEP 7: Compaction (background)
  Minor compaction: merge small HFiles within a store
  Major compaction: merge ALL HFiles in a store, remove tombstones and expired data

DATA LOSS SCENARIOS AND SAFEGUARDS:
┌────────────────────┬───────────────┬─────────────────────────────────────┐
│ Failure Point      │ WAL written?  │ Recovery                            │
├────────────────────┼───────────────┼─────────────────────────────────────┤
│ During WAL write   │ No            │ Write never acknowledged to client  │
│ After WAL, before  │ Yes           │ WAL replay on RegionServer restart  │
│ MemStore           │               │                                     │
│ During MemStore    │ Yes           │ WAL replay — MemStore is ephemeral  │
│ flush to HFile     │               │                                     │
│ HDFS block         │ Yes           │ HDFS 3x replication handles it      │
│ failure            │               │                                     │
│ RegionServer JVM   │ Yes           │ HMaster detects via ZK session,     │
│ crash              │               │ reassigns regions, replays WAL      │
└────────────────────┴───────────────┴─────────────────────────────────────┘
```

```java
// Tuning durability vs performance
Put put = new Put(Bytes.toBytes("user:1001"));
put.setDurability(Durability.ASYNC_WAL);  // 3-5x faster, tiny data loss window
// Use for: metrics, counters, data that can be reconstructed
// Never use for: financial transactions, user data

// Bulk load (bypasses WAL entirely for maximum throughput)
// Use for: initial data loads, batch ETL
// Tool: ImportTsv or custom HFile writer → BulkLoad API
```

---

## Q5: Design an HBase schema for a multi-tenant audit log system with compliance requirements.

**Answer:**

**Requirements:**
- Multi-tenant (20 clients), each with their own data isolation
- Compliance: 7-year retention, immutable records, WORM semantics
- Query: by tenant+timerange, by user, by action type
- Scale: 10K events/second globally, 1TB/year

**Schema design:**

```java
// TABLE: audit_events
// Row key: TENANT_HASH(2B) + TENANT_ID(10B) + INVERTED_TIMESTAMP(8B) + EVENT_UUID(16B)
// Column families:
//   "m" (metadata): event_type, user_id, ip_address, status  [frequently queried]
//   "d" (details): request_body, response_body, diff         [large, rarely queried]

// WHY SPLIT INTO TWO COLUMN FAMILIES:
// - "m" is small (< 1KB per event), accessed on every query → keep in BlockCache
// - "d" can be large (up to 1MB per event), accessed rarely → avoid polluting BlockCache
// - Separate CFs = separate HFiles = different compression/cache settings per CF

HTableDescriptor tableDesc = new HTableDescriptor(TableName.valueOf("audit_events"));

HColumnDescriptor metaCF = new HColumnDescriptor("m");
metaCF.setMaxVersions(1);                          // WORM: write once
metaCF.setTimeToLive(7 * 365 * 24 * 3600);       // 7-year TTL (2,592,000 seconds approx)
metaCF.setCompressionType(Compression.Algorithm.SNAPPY);
metaCF.setBlocksize(32768);                        // 32KB blocks: good for small rows
metaCF.setBlockCacheEnabled(true);                 // Keep in BlockCache
metaCF.setBloomFilterType(BloomType.ROW);
tableDesc.addFamily(metaCF);

HColumnDescriptor detailCF = new HColumnDescriptor("d");
detailCF.setMaxVersions(1);
detailCF.setTimeToLive(7 * 365 * 24 * 3600);
detailCF.setCompressionType(Compression.Algorithm.GZ);   // Better ratio for large payloads
detailCF.setBlocksize(262144);                           // 256KB blocks: good for large values
detailCF.setBlockCacheEnabled(false);                    // Don't pollute BlockCache
tableDesc.addFamily(detailCF);

// Pre-split: 1 region per tenant per quarter = 20 tenants * 4 quarters = 80 splits
// (over-provision initially; HBase splits further as needed)
```

```java
// Row key construction
public static byte[] buildAuditRowKey(String tenantId, long eventTimestamp, String eventUUID) {
    // 2-byte tenant hash for distribution across regions
    short tenantHash = (short) (Math.abs(tenantId.hashCode()) % Short.MAX_VALUE);

    // Pad tenant ID to fixed length for key consistency
    byte[] tenantBytes = Bytes.toBytes(String.format("%10s", tenantId).replace(' ', '_'));

    // Inverted timestamp: latest events first in range scan
    long invertedTs = Long.MAX_VALUE - eventTimestamp;

    // UUID ensures uniqueness for same-millisecond events
    byte[] uuidBytes = Bytes.toBytes(eventUUID.replace("-", ""));

    return Bytes.add(
        Bytes.toBytes(tenantHash),
        tenantBytes,
        Bytes.toBytes(invertedTs),
        uuidBytes
    );
}

// Write event (immutable — no update allowed by application contract)
public void writeAuditEvent(AuditEvent event) {
    byte[] rowKey = buildAuditRowKey(
        event.getTenantId(),
        event.getTimestamp(),
        event.getEventId()
    );

    Put put = new Put(rowKey);
    // Metadata CF
    put.addColumn("m", "type",    Bytes.toBytes(event.getEventType()));
    put.addColumn("m", "user_id", Bytes.toBytes(event.getUserId()));
    put.addColumn("m", "ip",      Bytes.toBytes(event.getIpAddress()));
    put.addColumn("m", "status",  Bytes.toBytes(event.getStatus()));
    put.addColumn("m", "ts",      Bytes.toBytes(event.getTimestamp()));
    // Details CF (only populated if details exist)
    if (event.getDetails() != null) {
        put.addColumn("d", "payload", Bytes.toBytes(event.getDetails()));
    }
    put.setDurability(Durability.SYNC_WAL);  // ALWAYS sync WAL for audit data

    table.put(put);
}

// Query: all events for tenant "acme" in the last 24 hours
public List<AuditEvent> queryByTenantAndTimeRange(
        String tenantId, long startMs, long endMs) {
    byte[] scanStart = buildAuditRowKey(tenantId, endMs, "000000000000000000000000000000");
    byte[] scanStop  = buildAuditRowKey(tenantId, startMs, "ffffffffffffffffffffffffffffffff");

    Scan scan = new Scan();
    scan.setStartRow(scanStart);
    scan.setStopRow(scanStop);
    scan.addFamily(Bytes.toBytes("m"));  // Only metadata CF (fast)
    scan.setCaching(1000);
    scan.setBatch(100);

    ResultScanner scanner = table.getScanner(scan);
    List<AuditEvent> events = new ArrayList<>();
    for (Result result : scanner) {
        events.add(parseAuditEvent(result));
    }
    return events;
}
```
