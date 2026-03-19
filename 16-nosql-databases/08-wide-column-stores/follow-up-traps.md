# Chapter 08: Wide-Column Stores — Follow-Up Traps

## Trap 1: "HBase is just a distributed version of a relational table"

**What most people say:**
"HBase organizes data into rows and columns like SQL, but distributed across many servers."

**Correct answer:**
HBase is fundamentally different from a relational table in three ways:

1. **Sparse columns**: Each row can have completely different columns. A 1-billion-row
   HBase table can have different columns for every row with no null-value overhead.
   SQL tables have fixed column schemas — absent values are NULLs that still consume storage.

2. **Column families are physical storage units**: All columns in a column family are
   stored together in one HFile on disk. Different column families in the same table are
   stored in SEPARATE files. This means queries that only access one column family avoid
   reading data for other column families entirely.

3. **Multi-versioned data**: HBase stores multiple versions of each cell (configurable).
   The "value" at (row, column, timestamp) is a unique cell. SQL has no built-in versioning.

```java
// HBase multi-version get — retrieve last 3 versions of a cell
Get get = new Get(Bytes.toBytes("user:1001"));
get.addColumn(Bytes.toBytes("profile"), Bytes.toBytes("email"));
get.setMaxVersions(3);  // Get last 3 versions

Result result = table.get(get);
Cell[] cells = result.getColumnCells(
    Bytes.toBytes("profile"), Bytes.toBytes("email")
);
for (Cell cell : cells) {
    long timestamp = cell.getTimestamp();
    String value = Bytes.toString(CellUtil.cloneValue(cell));
    System.out.println("Version " + timestamp + ": " + value);
}
// Output:
// Version 1705349200000: bob.new@example.com  (latest)
// Version 1704649200000: bob.old@example.com
// Version 1703649200000: bob.original@example.com
```

## Trap 2: "LSM-tree writes are faster than B-tree writes because there's no random I/O"

**What most people say:**
"LSM trees batch writes in the memtable and flush sequentially — this is always faster."

**Correct answer:**
LSM write performance is more nuanced. While the INITIAL write to memtable + WAL is
faster than a B-tree page write (sequential vs random), the TOTAL cost of a write
includes compaction. Write amplification measures how many actual bytes are written
to disk per byte of user data:

```
Leveled compaction write amplification:
- Each byte of data is rewritten approximately 10-30x as it flows from L0 → L1 → L2 → L3
- For L levels with fanout factor F: Write Amplification ≈ L × F

For F=10, L=4: WA ≈ 40x
Every 1 byte of user data triggers 40 bytes of disk I/O across all compactions

Size-tiered compaction write amplification:
- WA ≈ log(total_size / memtable_size) × tier_multiplier
- Lower WA than leveled, but more read amplification
```

The B-tree comparison: a B-tree update writes the modified page (4-16KB) regardless
of how small the change (a 10-byte update rewrites the whole page). For random point
updates, B-tree WA is comparable to LSM. For sequential bulk inserts, LSM wins.
For mixed workloads, the optimal choice depends on the write:read ratio.

```bash
# Observe HBase compaction write amplification in real time
hbase shell
> list_regions 'mytable'
> compact 'mytable'     # Trigger manual compaction

# Check HBase metrics for compaction write amplification
curl -s 'http://regionserver:16030/jmx?qry=Hadoop:service=HBase,name=RegionServer,sub=Server' \
  | jq '.beans[0] | {
      "compactionOutputFileCount": .compactionOutputFileCount,
      "compactionInputFileCount": .compactionInputFileCount,
      "compactionInputSize": .compactionInputSize,
      "compactionOutputSize": .compactionOutputSize
    }'
# WA = compactionOutputSize / (original user data written)
```

## Trap 3: "Bloom filters eliminate false negatives"

**What most people say:**
"A Bloom filter tells you definitively if a key is NOT present, preventing unnecessary
disk reads."

**Correct answer:**
Bloom filters have NO FALSE NEGATIVES (if it says "not present," the key definitely isn't
there) but DO have FALSE POSITIVES (if it says "possibly present," the key might or
might not be there). This asymmetry is the opposite of what people often say.

The false positive rate depends on filter size and number of hash functions:
- p = (1 - e^(-kn/m))^k
- where k = hash functions, n = items inserted, m = filter size in bits

```java
// HBase Bloom filter configuration per column family
HColumnDescriptor columnFamily = new HColumnDescriptor("data");
// NONE: no bloom filter
// ROW: bloom filter on row key (good for row-level point lookups)
// ROWCOL: bloom filter on row+column (better for column-specific lookups)
columnFamily.setBloomFilterType(BloomType.ROW);

// False positive rate tuning (affects HFile size)
// Default 1% false positive rate — 9.6 bits per key
// 0.1% false positive rate — 14.4 bits per key
// Trade-off: lower FPR = larger HFile = more memory for filter

// A Bloom filter saying "key IS PRESENT" → must still check disk (might be FP)
// A Bloom filter saying "key NOT PRESENT" → skip this HFile, 100% certain
// For 10 HFiles per region, Bloom filter reduces disk reads from 10 to ~0.1 per FP
```

The common interview mistake: confusing "Bloom filter says NOT present" (guaranteed true)
with "Bloom filter says PRESENT" (might be false positive). HBase Bloom filters eliminate
the need to scan HFiles that definitely DON'T contain the requested key.

## Trap 4: "A major compaction always improves read performance"

**What most people say:**
"Major compaction merges all HFiles into one, so reads only need to check one file.
Always run major compaction to optimize performance."

**Correct answer:**
Major compaction is a double-edged sword:

Benefits:
- Reduces HFile count from N to 1 per column family per region → fewer disk seeks
- Removes tombstones (deleted cells) that accumulate and slow reads
- Removes expired versions and TTL-expired data

Costs and risks:
- Extremely I/O intensive: reads ALL data in ALL HFiles, writes ALL data back as one file
- On a 10TB region, major compaction reads and writes ~10TB → saturates HDFS network
- Competing with real-time reads and writes
- After major compaction, the single large HFile must be COMPLETELY rewritten for every
  future minor compaction — temporarily increases write amplification

```bash
# Check when major compaction last ran on a region
hbase shell
> list_regions 'mytable'
# Look at "LAST_COMPACTION" timestamp in the output

# Disable automatic major compaction (let you control timing)
hbase> alter 'mytable', MAJOR_COMPACTION_PERIOD => '0'

# Schedule manual major compaction during off-hours
# On the HBase shell or via Admin API:
hbase> major_compact 'mytable,row_start_key,region_name'

# Better: use rolling major compaction (one region at a time)
# to avoid saturating cluster I/O
```

In production, major compaction should be:
1. Disabled for automatic scheduling
2. Scheduled manually during off-peak hours
3. Done one region at a time (not all regions simultaneously)
4. Monitored with circuit breakers to abort if cluster health degrades

## Trap 5: "Row key design only matters for write distribution"

**What most people say:**
"You salt the row key to distribute writes. Once writes are distributed, you're fine."

**Correct answer:**
Row key design affects BOTH write distribution AND read performance, and these often
conflict. Optimizing for writes can destroy read performance:

```
SCENARIO: Event log with device_id + timestamp as row key

Option A: Sequential timestamp prefix (bad for writes, good for time-range reads)
Row key: "20240115_120000_device-001"
         └── Sequential keys → ALL writes go to the LAST region (hotspot!)
         └── Time-range scans work perfectly: SCAN start='2024*', stop='2025*'

Option B: Salted/Reversed timestamp (good for writes, bad for range reads)
Row key: "3_device-001_20240115_120000"  (salt = hash(device_id) % 4)
         └── Writes distributed across 4 regions (no hotspot)
         └── Time-range scan requires 4 SEPARATE scans (one per salt bucket),
             then client-side merge → complex application code

Option C: Device-prefix + reversed timestamp (balanced)
Row key: "device-001_9999999999_20240115"  (inverted timestamp: long.MAX - ts)
         └── Writes for same device go to same region (slight hotspot per device)
         └── Latest readings for a device: SCAN with startRow='device-001_9999999999'
             → Returns newest-first (because inverted timestamp)
         └── Cross-device reads require scatter-gather (scan all regions)
```

```java
// Inverted timestamp row key — most recent data first
String deviceId = "device-001";
long timestamp = System.currentTimeMillis();
long invertedTs = Long.MAX_VALUE - timestamp;

byte[] rowKey = Bytes.add(
    Bytes.toBytes(deviceId),
    Bytes.toBytes("_"),
    Bytes.toBytes(String.format("%020d", invertedTs))
);
// Row key: "device-001_09223372036804775807"
// Newest reading has the LOWEST inverted timestamp → first in scan
```

## Trap 6: "HBase coprocessors are safe to use in multi-tenant clusters"

**What most people say:**
"Coprocessors are like stored procedures — a great way to push compute to the data."

**Correct answer:**
HBase coprocessors run IN THE REGIONSERVER JVM — a buggy coprocessor can crash the
entire RegionServer, taking down all tenants on that server. They are not sandboxed.

Risks:
1. Infinite loop or OOM in coprocessor → RegionServer JVM crash → data unavailable
2. Coprocessor that modifies global state → corrupts data for all tables
3. Security: coprocessors execute with full RegionServer privileges
4. Debugging: coprocessor stack traces mix with RegionServer logs

```java
// Observer coprocessor example — triggers on every Put
// A bug in this code affects ALL tables on the RegionServer
public class AuditCoprocessor extends BaseRegionObserver {
    @Override
    public void postPut(ObserverContext<RegionCoprocessorEnvironment> ctx,
                        Put put, WALEdit edit, Durability durability) {
        // THIS RUNS IN THE REGIONSERVER JVM
        // If this throws an exception, it propagates to the client as RPC error
        // If this loops infinitely, the RegionServer hangs

        // Safe pattern: wrap in try-catch, log errors, never block
        try {
            String row = Bytes.toString(put.getRow());
            auditLog.info("PUT to row: " + row);
        } catch (Exception e) {
            // CRITICAL: never let coprocessor exceptions bubble up
            LOG.error("Audit coprocessor failed (non-fatal)", e);
        }
    }
}
```

The safer alternative for multi-tenant clusters: use HBase replication to stream
WAL edits to a dedicated audit/analytics cluster. Or use Kafka Connect with HBase
as a source to stream changes without touching the RegionServer execution path.

## Trap 7: "Google Bigtable and HBase are interchangeable — same API, same model"

**What most people say:**
"HBase is Apache's implementation of the Bigtable paper, so they're basically the same."

**Correct answer:**
HBase was inspired by the Bigtable paper but diverges significantly in production:

| Dimension          | Google Bigtable             | Apache HBase                    |
|--------------------|-----------------------------|---------------------------------|
| Storage backend    | Colossus (GFS2)             | HDFS (Hadoop)                   |
| Consensus/coord.   | Chubby (Paxos)              | ZooKeeper (ZAB)                 |
| Replication        | Synchronous within zone     | Asynchronous replication        |
| Transactions       | Single-row atomic           | Single-row atomic               |
| Multi-row ACID     | Spanner (separate product)  | Omid/Tephra (separate library)  |
| Managed service    | Fully managed by Google     | Self-managed (EMR/Databricks)   |
| Tablet splits      | Automatic by Bigtable master| Automatic by HMaster            |
| Client API         | Bigtable client libraries   | HBase Java API (similar model)  |
| Performance        | 10ms p99 (SLA guaranteed)   | Highly variable (depends on HW) |

```python
# Google Bigtable client (google-cloud-bigtable)
from google.cloud import bigtable
from google.cloud.bigtable import column_family, row_filters

client = bigtable.Client(project="my-project", admin=True)
instance = client.instance("my-instance")
table = instance.table("user-events")

# Read row with filter on column family and version limit
row_filter = row_filters.RowFilterChain(filters=[
    row_filters.FamilyNameRegexFilter("activity"),
    row_filters.CellsColumnLimitFilter(5)  # Last 5 versions
])
row = table.read_row(b"user:1001", filter_=row_filter)
for cell in row.cells.get("activity", {}).get("login", []):
    print(f"Login at {cell.timestamp}: {cell.value.decode()}")
```

The biggest operational difference: Bigtable is serverless from the user's perspective —
you never manage tablet servers, ZooKeeper, or HDFS. HBase requires managing all three,
plus YARN (if on HDP/CDH), plus tuning JVM GC for RegionServers.
