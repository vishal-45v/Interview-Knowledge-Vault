# Chapter 08: Wide-Column Stores — Analogy Explanations

## Analogy 1: LSM Tree Levels — The Post Office Sorting System

**The Story:**
A busy post office receives thousands of letters per hour. Sorting each letter into its
final bin immediately would be too slow — the sorter would spend all their time running
to distant bins. Instead: letters arrive on a conveyor belt (MemTable), pile into a
staging area (L0), then batches are periodically sorted into regional bins (L1), and
those bins are sorted into city bins (L2), and eventually into final delivery routes (L3).
The deeper the sort, the more organized the bin, but sorting takes more time.

**Connection to the database:**
The staging conveyor (MemTable) is in-memory and fast. Flushing to L0 is like moving the
pile to a staging bin — sorted locally but potentially overlapping other piles. Compaction
is the sorting operation that merges overlapping piles into organized, non-overlapping bins
at deeper levels. The deeper a key goes, the less write amplification remains ahead of it.

```
MemTable (RAM):     Incoming writes, sorted by key
                    Flushed when full → L0 SSTable
      │
      ▼
L0 (disk):         4-8 SSTables, may overlap in key space
                   Compacted to L1 when count exceeds threshold
      │
      ▼
L1 (disk):         ~10 SSTables, NO overlap, total size ~10MB
                   Each key appears in exactly ONE L1 file
      │
      ▼
L2 (disk):         ~100 SSTables, NO overlap, total size ~100MB
      │
      ▼
L3 (disk):         ~1000 SSTables, NO overlap, total size ~1GB

Read a key: MemTable → L0 (all files) → L1 (1 file max) → L2 (1 file max) → ...
Write amplification: each byte is rewritten at each level crossing (~10x per level)
```

```bash
# Monitor LSM tree level stats in RocksDB (used by many HBase alternatives)
# HBase equivalent: check HFile count per region store
hbase shell
> hbase_htd_table = get_table 'mytable'
# List regions and their HFile counts
> list_regions 'mytable'
# High HFile count per region = high read amplification
# Rule of thumb: > 30 HFiles per region = trigger manual compaction
```

---

## Analogy 2: HBase Column Families — The Office Building Filing System

**The Story:**
An office building has multiple rooms: one for HR records, one for financial records,
one for legal documents, and one for email archives. Each room has its own filing cabinets,
climate control settings (some legal docs are kept at cool temperatures for preservation),
and access rules. When an employee needs their paycheck history, they only open the
financial room — they don't need to enter the HR or legal rooms. When IT needs to scan
email archives (huge files), it doesn't affect the climate-controlled legal room.

HBase column families are the rooms. All columns in a column family are physically stored
together in their own HFiles. When a query reads only one column family, it only touches
those files — other column family files are never opened.

**Connection to the database:**

```java
// WRONG: All 20 columns in one column family
// A query for just "email" must read a block containing all 20 columns
HColumnDescriptor singleCF = new HColumnDescriptor("data");
// Block contains: name, email, phone, address, dob, ...
// Block size default 64KB — query for 10-byte email reads 64KB

// RIGHT: Split by access pattern
HColumnDescriptor identityCF = new HColumnDescriptor("id");   // name, email, phone
HColumnDescriptor activityCF = new HColumnDescriptor("act");  // logins, purchases
HColumnDescriptor archiveCF = new HColumnDescriptor("arc");   // historical data

// Configuration per column family — different rooms have different settings
identityCF.setBlockCacheEnabled(true);        // Frequently accessed → cache
identityCF.setBlocksize(8192);                // Small blocks → faster point lookup
identityCF.setCompressionType(SNAPPY);        // Fast decompression

archiveCF.setBlockCacheEnabled(false);        // Rarely accessed → don't pollute cache
archiveCF.setBlocksize(524288);               // Large blocks → better sequential scan
archiveCF.setCompressionType(GZ);             // Better compression ratio → less storage
archiveCF.setTimeToLive(365 * 24 * 3600);    // Auto-delete after 1 year

// Query only identity column family — archive files never touched
Get get = new Get(rowKey);
get.addFamily(Bytes.toBytes("id"));  // Only opens "id" HFiles
Result result = table.get(get);
```

---

## Analogy 3: Row Key Hotspotting — The Supermarket Express Lane

**The Story:**
A supermarket opens 20 checkout lanes but designates lane 20 as the "express lane for
today's specials." Everyone buying the deal of the day queues in lane 20. Lanes 1-19
are empty. The cashier in lane 20 is overwhelmed; customers wait 20 minutes while
perfectly idle cashiers watch. The store has 20x the capacity it's using.

Sequential row key prefixes (like timestamps) send all new writes to the same HBase
region — the "express lane" at the end of the key space. The RegionServer hosting
the latest time range receives 100% of writes; others sit idle.

**Connection to the database:**

```
Sequential timestamp row keys (BAD):
"2024-01-15T12:00:00_event_001"   ──► Region-20 (latest time range)
"2024-01-15T12:00:01_event_002"   ──► Region-20
"2024-01-15T12:00:02_event_003"   ──► Region-20
                                       ← All 10K writes/second → Region-20
                                       ← Regions 1-19: zero traffic

SOLUTIONS: Distribute writes like customers across all 20 lanes

Option 1: Salt prefix (modulo distribution)
"0_2024-01-15T12:00:00_event_001"  ──► Region-1  (hash(event_001) % 20 = 0)
"7_2024-01-15T12:00:01_event_002"  ──► Region-7  (hash(event_002) % 20 = 7)
"15_2024-01-15T12:00:02_event_003" ──► Region-15 (hash(event_003) % 20 = 15)
COST: Range scans must query ALL 20 regions (scatter-gather)

Option 2: Field-based distribution (semantic salt)
"device-001_2024-01-15T12:00:00"   ──► Region based on device-001 hash
"device-002_2024-01-15T12:00:00"   ──► Region based on device-002 hash
BENEFIT: Device range scans go to one region; cross-device scans scatter-gather
COST: Some devices may be "hot" if they generate more events
```

```python
# Python: salt-based row key generator
import hashlib
import struct
from datetime import datetime

def generate_row_key(entity_id: str, timestamp: datetime,
                     num_regions: int = 20) -> bytes:
    """
    Salt = hash(entity_id) % num_regions → 1 byte
    Entity ID → 16 bytes (UUID or padded string)
    Inverted timestamp → 8 bytes (newest first)
    Unique suffix → 4 bytes (prevents collision at same millisecond)
    """
    salt = int(hashlib.md5(entity_id.encode()).hexdigest(), 16) % num_regions
    ts_millis = int(timestamp.timestamp() * 1000)
    inverted_ts = (2**63 - 1) - ts_millis  # Inverted for newest-first scan

    row_key = struct.pack('>B16sQ', salt,
                          entity_id.encode().ljust(16)[:16],
                          inverted_ts)
    return row_key
```

---

## Analogy 4: HBase Bloom Filters — The Library Card Catalog

**The Story:**
A library has 1,000 filing cabinets (HFiles) and millions of books. When you ask for
"The Great Gatsby," the librarian doesn't open all 1,000 cabinets. Instead, each cabinet
has a card on its drawer listing the INITIALS of books inside — a compact summary. If the
summary says "this cabinet definitely has no book starting with 'T-H-E'," the librarian
skips it. If the summary says "might have it," the librarian checks. The summary CAN
be wrong about "might have it" (false positive) but NEVER wrong about "definitely doesn't."

This is a Bloom filter. For each HFile, HBase maintains a Bloom filter in memory. Before
reading an HFile from disk, the Bloom filter is consulted. If it says "not present," the
HFile is skipped entirely — a disk read is saved.

**Connection to the database:**

```
READ PERFORMANCE WITH AND WITHOUT BLOOM FILTERS:

Without Bloom filter (10 HFiles per region):
GET row="user:1001"
├── Read HFile-1 → NOT FOUND (disk I/O wasted)
├── Read HFile-2 → NOT FOUND (disk I/O wasted)
├── Read HFile-3 → NOT FOUND (disk I/O wasted)
├── ... × 7 more files ...
└── Read HFile-10 → FOUND
Total: 10 disk reads for 1 row lookup

With Bloom filter (1% false positive rate):
GET row="user:1001"
├── Bloom(HFile-1) → "NOT PRESENT" (skip — 0 disk I/O)
├── Bloom(HFile-2) → "NOT PRESENT" (skip — 0 disk I/O)
├── Bloom(HFile-3) → "POSSIBLY PRESENT" → Read → NOT FOUND (false positive)
├── Bloom(HFile-4..9) → "NOT PRESENT" (skip)
└── Bloom(HFile-10) → "POSSIBLY PRESENT" → Read → FOUND
Total: 2 disk reads instead of 10 → 80% reduction in disk I/O
```

```java
// Configure Bloom filter per column family
HColumnDescriptor cf = new HColumnDescriptor("user_data");

// ROW bloom filter: useful for row-level point lookups
// "Does this HFile contain ANY data for row 'user:1001'?"
cf.setBloomFilterType(BloomType.ROW);

// ROWCOL bloom filter: more precise, useful when querying specific columns
// "Does this HFile contain column 'email' for row 'user:1001'?"
cf.setBloomFilterType(BloomType.ROWCOL);

// Check Bloom filter effectiveness in HBase shell
// High "blockCacheHitCaching" + low "fsReadTime" = Bloom filters working well
```

---

## Analogy 5: Compaction Strategies — The Library Book Reorganization

**The Story:**
A library receives new books continuously. They're placed on a "new arrivals" shelf (L0)
in no particular order. When the new arrivals shelf fills up, someone must sort them into
the main collection.

**Method 1 (Size-tiered — "group similar piles"):** Combine shelves of similar size. Four
small piles → one medium pile. Four medium piles → one large section. Fast to organize
(simple grouping), but a book might be in any of several sections of the same size. Finding
one book means checking multiple sections.

**Method 2 (Leveled — "one precise location"):** After sorting, each book has exactly ONE
correct location in the entire library. Finding it is fast (it's in the A-M section, fiction,
shelf 42). But maintaining this organization is expensive — every new book requires
reshuffling existing books to make room.

**Connection to the database:**

```
SIZE-TIERED COMPACTION (optimize for writes):
┌────────────────────────────────────────────────────────────────────────┐
│ Tier 1: [1MB A-Z][1MB A-Z][1MB A-Z][1MB A-Z] → merge → [4MB A-Z]     │
│ Tier 2: [4MB A-Z][4MB A-Z][4MB A-Z][4MB A-Z] → merge → [16MB A-Z]    │
│                                                                        │
│ ✓ Low write amplification (each byte rewritten few times)              │
│ ✗ Reads may need to check multiple files at same tier (all cover A-Z) │
│ Best for: append-only logs, time-series, bulk ingest                   │
└────────────────────────────────────────────────────────────────────────┘

LEVELED COMPACTION (optimize for reads):
┌────────────────────────────────────────────────────────────────────────┐
│ L1: [A-M, 10MB][N-Z, 10MB]        ← Each key in exactly ONE file      │
│ L2: [A-C][D-F][G-I]...[X-Z]       ← 10x more files than L1           │
│                                                                        │
│ ✓ Reads require O(num_levels) file checks maximum                     │
│ ✗ High write amplification (new key at A reshuffles A-file at L1, L2) │
│ Best for: random reads, frequently updated records, user profiles      │
└────────────────────────────────────────────────────────────────────────┘
```

The critical insight for time-series: neither pure leveled nor pure size-tiered is
optimal. Time-Window Compaction (Cassandra TWCS) partitions data by time window,
applies size-tiered within each window, and NEVER compacts across windows. Old data
(last week, last month) is never rewritten — write amplification for historical data
is effectively 1x.

---

## Analogy 6: HBase RegionServer and Regions — The Bank Branch Model

**The Story:**
A national bank has 20 branch offices (RegionServers). Each branch manages accounts
in a specific range — branch 1 handles accounts A-F, branch 2 handles G-L, etc.
A branch manager (HMaster) assigns account ranges to branches and handles splits
when a branch becomes too busy. Account operations go directly to the assigned branch
— the head office (HMaster) is not involved in day-to-day transactions.

If a branch burns down (RegionServer crash), the head office reassigns its account
ranges to other branches from the backup records (WAL files in HDFS). Those branches
replay the recent transaction logs (WAL) to recover the in-progress work.

**Connection to the database:**

```
HBase Architecture:
┌────────────────────────────────────────────────────────────────────────┐
│ HMaster (Head Office)                                                  │
│  ├── Assigns regions to RegionServers                                  │
│  ├── Handles RegionServer failure recovery                             │
│  ├── Coordinates splits and merges                                     │
│  └── NOT in read/write path (transactions bypass head office)          │
│                                                                        │
│ ZooKeeper (Corporate Phonebook)                                        │
│  ├── Tracks which RegionServer is "alive"                              │
│  ├── Stores location of hbase:meta table                               │
│  └── Enables HMaster failover (HMaster also registers in ZK)          │
│                                                                        │
│ RegionServer-1 (Branch Office 1)     RegionServer-2 (Branch Office 2) │
│  ├── Region: users_A-F               ├── Region: users_G-L            │
│  │   ├── MemStore (in-memory)         │   ├── MemStore                 │
│  │   ├── HFiles in HDFS               │   └── HFiles in HDFS           │
│  │   └── WAL (HDFS)                  └── Region: users_M-R             │
│  └── Region: transactions_2024                                         │
│                                                                        │
│ HDFS (Bank Vault)                                                      │
│  ├── All HFiles (SSTables) stored here with 3x replication            │
│  └── All WAL segments stored here                                      │
└────────────────────────────────────────────────────────────────────────┘

Region split trigger:
When a Region grows > 10GB (default), it splits into two child Regions.
The parent Region's key range is divided at the median key.
HMaster assigns the two new Regions (possibly to different RegionServers).
```

```python
# Monitor region distribution across RegionServers
import happybase  # Python HBase client

connection = happybase.Connection('hmaster-host', port=9090)
# Use HBase REST API to check region assignments
import requests

regions = requests.get(
    'http://hmaster-host:16010/api/v1/namespaces/default/tables/sensor_data/regions',
    headers={'Accept': 'application/json'}
).json()

server_to_regions = {}
for region in regions:
    server = region['regionServerName']
    server_to_regions.setdefault(server, []).append(region['name'])

# Print load distribution
for server, regions_list in server_to_regions.items():
    print(f"{server}: {len(regions_list)} regions")
    # Ideal: all servers have similar region counts
    # Imbalance: one server has 50 regions, others have 2
    # Fix: admin.balancer() to trigger HMaster rebalancing
```
