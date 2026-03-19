# Chapter 04 — DynamoDB: Diagram Explanations

ASCII diagrams for DynamoDB architecture, single-table design, and access patterns.

---

## Diagram 1: DynamoDB Single-Table Design — E-Commerce Access Pattern Map

```
  TABLE: ECommerceTable
  ┌──────────────────┬──────────────────────────┬─────────────────────┐
  │ PK               │ SK                       │ Attributes          │
  ├──────────────────┼──────────────────────────┼─────────────────────┤
  │ CUST#alice       │ PROFILE                  │ name, email         │
  │ CUST#alice       │ ORDER#2024-01-15#001     │ status, total       │
  │ CUST#alice       │ ORDER#2024-01-14#002     │ status, total       │
  │ ORDER#001        │ METADATA                 │ customer_id, status │
  │ ORDER#001        │ ITEM#SKU-001             │ qty, unit_price     │
  │ ORDER#001        │ ITEM#SKU-002             │ qty, unit_price     │
  │ PRODUCT#SKU-001  │ METADATA                 │ name, price, stock  │
  │ PRODUCT#SKU-001  │ REVIEW#2024-01-10#user1  │ rating, text        │
  └──────────────────┴──────────────────────────┴─────────────────────┘

  ACCESS PATTERNS:
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Q1: Get customer           → GetItem(PK=CUST#alice, SK=PROFILE)   │
  │  Q2: Get customer orders    → Query(PK=CUST#alice,                  │
  │                               begins_with(SK, ORDER#), desc)        │
  │  Q3: Get order by ID        → GetItem(PK=ORDER#001, SK=METADATA)   │
  │  Q4: Get order items        → Query(PK=ORDER#001,                   │
  │                               begins_with(SK, ITEM#))               │
  │  Q5: Get product reviews    → Query(PK=PRODUCT#SKU-001,            │
  │                               begins_with(SK, REVIEW#))             │
  └──────────────────────────────────────────────────────────────────────┘

  GSI DESIGN (Global Secondary Indexes):
  ┌──────────────────┬─────────────────────────────────────────────────┐
  │ GSI              │ Purpose                                         │
  ├──────────────────┼─────────────────────────────────────────────────┤
  │ GSI1: GSI1PK,    │ Orders by status — admin dashboard              │
  │       GSI1SK     │ GSI1PK = STATUS#PENDING, GSI1SK = created_at   │
  ├──────────────────┼─────────────────────────────────────────────────┤
  │ GSI2: GSI2PK,    │ Products by category                            │
  │       GSI2SK     │ GSI2PK = CATEGORY#electronics, GSI2SK = SK     │
  └──────────────────┴─────────────────────────────────────────────────┘
```

**Explanation:** Single-table design uses hierarchical sort keys to create parent-child
relationships within a partition. Prefixed sort keys (`ORDER#`, `ITEM#`, `REVIEW#`) allow
efficient range queries that return only items of a specific type. Generic attribute names
(`GSI1PK`, `GSI1SK`) are reused across entity types to reduce the number of GSIs needed.

---

## Diagram 2: DynamoDB Partition Architecture and Throughput Limits

```
  DynamoDB Table with 4 partitions (simplified)
  ┌──────────────────────────────────────────────────────────────────────┐
  │  KEY SPACE (hash of partition key, normalised to 0-1)               │
  │  ├───────────┼───────────┼───────────┼───────────┤                  │
  │  0          0.25        0.5         0.75         1.0                │
  │                                                                      │
  │  Partition 1  │  Partition 2  │  Partition 3  │  Partition 4       │
  │  [0 - 0.25)   │  [0.25 - 0.5) │  [0.5 - 0.75) │  [0.75 - 1.0)    │
  │               │               │               │                    │
  │  RCU limit:   │  RCU limit:   │  RCU limit:   │  RCU limit:       │
  │  3000/sec     │  3000/sec     │  3000/sec     │  3000/sec          │
  │               │               │               │                    │
  │  WCU limit:   │  WCU limit:   │  WCU limit:   │  WCU limit:       │
  │  1000/sec     │  1000/sec     │  1000/sec     │  1000/sec          │
  └──────────────────────────────────────────────────────────────────────┘

  Total table capacity: 12,000 RCU / 4,000 WCU — only if evenly distributed!

  HOT PARTITION SCENARIO:
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Key "popular_product" → hash → 0.42 → Partition 2                 │
  │                                                                      │
  │  Traffic: 8,000 writes/sec to popular_product                       │
  │                                                                      │
  │  Partition 2 limit:  1,000 WCU/sec                                  │
  │  Actual traffic:     8,000 WCU/sec                                  │
  │  Throttled:          7,000 WCU/sec  ← ProvisionedThroughputExceeded │
  │                                                                      │
  │  Adaptive capacity: DynamoDB can temporarily allow a partition to   │
  │  "borrow" unused capacity from other partitions. Helps for short   │
  │  bursts but NOT for sustained hot keys.                             │
  └──────────────────────────────────────────────────────────────────────┘

  WRITE SHARDING SOLUTION:
  popular_product#00 → Partition 1 (800 writes/sec)
  popular_product#01 → Partition 3 (800 writes/sec)
  popular_product#02 → Partition 2 (800 writes/sec)
  popular_product#03 → Partition 4 (800 writes/sec)
  ... × 10 shards → 8,000 writes/sec distributed across 10 partitions
```

**Explanation:** Each DynamoDB partition has hard per-partition throughput limits. The total
table capacity is the sum of all partition limits, but only if traffic is uniformly distributed.
Adaptive capacity provides temporary relief for burst traffic to hot partitions, but it cannot
overcome a sustained hot partition. Write sharding is the solution for consistently hot keys.

---

## Diagram 3: GSI vs LSI — Storage and Consistency Model

```
  BASE TABLE                        LSI (Local Secondary Index)
  PK: customer_id                   PK: customer_id (SAME)
  SK: created_at                    SK: total_amount (DIFFERENT)
  ┌─────────────────────────────┐   ┌─────────────────────────────────┐
  │ alice │ 2024-01-10 │ $99   │   │ alice │ $29.99  │ 2024-01-12 │  │
  │ alice │ 2024-01-12 │ $29   │   │ alice │ $99.00  │ 2024-01-10 │  │
  │ alice │ 2024-01-15 │ $200  │   │ alice │ $200.00 │ 2024-01-15 │  │
  │ bob   │ 2024-01-11 │ $150  │   │ bob   │ $150.00 │ 2024-01-11 │  │
  └─────────────────────────────┘   └─────────────────────────────────┘
  Consistent read: ✓ YES             Consistent read: ✓ YES
  Storage: base                      Storage: shared with base table
  Create: any time                   Create: at table creation ONLY
  Max 10GB per partition key         Max 10GB per partition key (SAME constraint)
  5 LSIs maximum per table

  GSI (Global Secondary Index)      COMPARISON SUMMARY
  PK: status                        ┌──────────────┬──────────┬──────────┐
  SK: created_at                    │              │  LSI     │  GSI     │
  ┌─────────────────────────────┐   ├──────────────┼──────────┼──────────┤
  │ PENDING  │ 2024-01-15 │ ... │   │ Partition    │ Same PK  │ Any PK   │
  │ SHIPPED  │ 2024-01-10 │ ... │   │ Consistency  │ Strong ✓ │ Eventual │
  │ SHIPPED  │ 2024-01-12 │ ... │   │ Create time  │ At init  │ Any time │
  │ PENDING  │ 2024-01-14 │ ... │   │ Storage cap  │ Shared   │ Separate │
  └─────────────────────────────┘   │ Throughput   │ Shared   │ Separate │
  Consistent read: ✗ NEVER          │ Max per table│ 5        │ 20       │
  Storage: SEPARATE table           └──────────────┴──────────┴──────────┘
  Create: any time
  20 GSIs maximum per table
```

**Explanation:** The critical difference: LSIs are stored *with* the base table partition,
sharing storage and the 10GB-per-partition limit. GSIs are separate distributed tables that
store a different projection of your data. GSIs have independent capacity, can be created or
deleted at any time, and support completely different access patterns. The trade-off is that
GSIs are always eventually consistent — you cannot get a strongly consistent read from a GSI.

---

## Diagram 4: DynamoDB Capacity Planning — RCU/WCU Math

```
  READ CAPACITY MATH:
  ┌────────────────────────────────────────────────────────────────────┐
  │  1 RCU = 1 strongly consistent read of 1 item up to 4KB/sec      │
  │  1 RCU = 2 eventually consistent reads of 1 item up to 4KB/sec   │
  │                                                                    │
  │  Item size: 10KB                                                   │
  │  Reads/sec: 1000                                                   │
  │  Consistency: Strong                                               │
  │                                                                    │
  │  RCU per item  = ceil(10KB / 4KB) = 3                            │
  │  Total RCU     = 3 × 1000 = 3,000 RCU/sec                       │
  │                                                                    │
  │  Consistency: Eventual (default)                                   │
  │  RCU per item  = ceil(10KB / 4KB) / 2 = 1.5                     │
  │  Total RCU     = 1.5 × 1000 = 1,500 RCU/sec                     │
  └────────────────────────────────────────────────────────────────────┘

  WRITE CAPACITY MATH:
  ┌────────────────────────────────────────────────────────────────────┐
  │  1 WCU = 1 write of 1 item up to 1KB/sec                         │
  │                                                                    │
  │  Item size: 2.5KB                                                  │
  │  Writes/sec: 500                                                   │
  │                                                                    │
  │  WCU per item  = ceil(2.5KB / 1KB) = 3                           │
  │  Total WCU     = 3 × 500 = 1,500 WCU/sec                        │
  │                                                                    │
  │  TRANSACTIONAL (2x cost):                                         │
  │  TransactWriteItems: 3 × 2 = 6 WCU per item                     │
  │  TransactGetItems:   RCU × 2 per item                            │
  └────────────────────────────────────────────────────────────────────┘

  ON-DEMAND vs PROVISIONED COST COMPARISON:
  ┌────────────────────────────────────────────────────────────────────┐
  │  100 reads/sec × 1 RCU × 3600 seconds × 24 hours × 30 days      │
  │  = 259,200,000 read request units/month                           │
  │                                                                    │
  │  On-demand:     $0.25/million reads  = $64.80/month              │
  │  Provisioned:   100 RCU × $0.00013/RCU-hour × 720 hours          │
  │               = $9.36/month  ← 7x cheaper for predictable load   │
  │                                                                    │
  │  Break-even: On-demand is cheaper if actual usage < ~15% of      │
  │  provisioned capacity (very spiky traffic)                        │
  └────────────────────────────────────────────────────────────────────┘
```

**Explanation:** RCU/WCU math is a common interview calculation. The key formulas are
`ceil(item_size / 4)` for reads and `ceil(item_size / 1)` for writes, with the eventual
consistency halving for reads. Transactions double the cost. The cost comparison between
on-demand and provisioned modes depends on utilisation — provisioned is always cheaper for
predictable, sustained traffic; on-demand is better for zero-to-100 spikes.

---

## Diagram 5: DynamoDB Streams Architecture and Lambda Integration

```
  DynamoDB Table (Orders)
  ┌─────────────────────────────────────────────────────────────────────┐
  │  Item Change (INSERT/MODIFY/REMOVE)                                │
  │       │                                                             │
  │       ▼                                                             │
  │  DynamoDB Stream (24-hour rolling window)                          │
  │  ┌────────────────────────────────────────────────────────────┐   │
  │  │  Shard 1 (partition key range 0-0.25)                     │   │
  │  │  [Record1][Record2][Record3]...                           │   │
  │  ├────────────────────────────────────────────────────────────┤   │
  │  │  Shard 2 (partition key range 0.25-0.5)                   │   │
  │  │  [Record4][Record5]...                                     │   │
  │  ├────────────────────────────────────────────────────────────┤   │
  │  │  Shard 3 (partition key range 0.5-0.75)                   │   │
  │  │  [Record6][Record7][Record8]...                           │   │
  │  ├────────────────────────────────────────────────────────────┤   │
  │  │  Shard 4 (partition key range 0.75-1.0)                   │   │
  │  │  [Record9]...                                              │   │
  │  └────────────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────────────┘

  CONSUMERS:
  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
  │  Lambda A    │    │  Lambda B    │    │  Kinesis     │
  │  (Sync to ES)│    │  (Audit Log) │    │  (Analytics) │
  └──────────────┘    └──────────────┘    └──────────────┘
        │                   │                   │
        └───────────────────┴───────────────────┘
                            │
                DynamoDB Streams API
                (each consumer has independent shard iterator)

  ORDERING GUARANTEE:
  ┌────────────────────────────────────────────────────────────────────┐
  │  Within a shard: records are ORDERED by sequence number           │
  │  (records for the same partition key go to the same shard)        │
  │                                                                    │
  │  Across shards: NO ordering guarantee                             │
  │                                                                    │
  │  Implication: for a given item (partition key), all changes       │
  │  are delivered in order. For different items, ordering is         │
  │  not guaranteed across shards.                                    │
  └────────────────────────────────────────────────────────────────────┘
```

**Explanation:** DynamoDB Streams use a shard architecture similar to Kinesis. Each shard
corresponds to a range of the DynamoDB partition key space. Records for the same partition
key always go to the same shard, preserving ordered delivery for a given item. Multiple
independent consumers can each maintain their own position (shard iterator) in the stream,
enabling fan-out patterns (same data changes triggering multiple downstream processes).

---

## Diagram 6: Sort Key Patterns for Query Flexibility

```
  SORT KEY PATTERN EXAMPLES:
  ┌──────────────────────────────────────────────────────────────────┐
  │ PK = CUST#alice                                                 │
  │                                                                  │
  │ SK options for different query patterns:                         │
  │                                                                  │
  │  Simple prefix separation:                                       │
  │  ORDER#2024-01-15T10:00:00Z    ← begins_with("ORDER#")         │
  │  PAYMENT#2024-01-15T10:05:00Z  ← begins_with("PAYMENT#")       │
  │  PROFILE                       ← exact match                    │
  │                                                                  │
  │  Hierarchical:                                                   │
  │  ORDER#2024#Q1#001             ← Query by year: begins_with     │
  │                                   ("ORDER#2024#")               │
  │  ORDER#2024#Q2#002             ← Query by quarter: begins_with  │
  │                                   ("ORDER#2024#Q2#")            │
  │                                                                  │
  │  Composite for range + equality:                                 │
  │  ORDER#PENDING#2024-01-15#001  ← Query pending orders by date:  │
  │  ORDER#SHIPPED#2024-01-10#002  │  begins_with("ORDER#PENDING#") │
  │                                   AND SK >= date_range_start    │
  └──────────────────────────────────────────────────────────────────┘

  SORT KEY RANGE OPERATIONS:
  ┌──────────────────────────────────────────────────────────────────┐
  │  begins_with(SK, "ORDER#")         ← All orders                 │
  │  begins_with(SK, "ORDER#2024")     ← Orders in 2024            │
  │  SK BETWEEN "ORDER#2024-01" AND    ← Orders in Jan 2024        │
  │            "ORDER#2024-01T99"                                   │
  │  SK > "ORDER#2024-01-15"           ← Orders after Jan 15       │
  │  SK = "PROFILE"                    ← Exact match               │
  └──────────────────────────────────────────────────────────────────┘

  IMPORTANT: Sort key is always lexicographic for strings!
  ┌────────────────────────────────────────────────────────────────────┐
  │  Correct:   "2024-01-09" < "2024-01-10"  (ISO date format)      │
  │  Incorrect: "9" > "10" (numeric string: '9' > '1' lexically)    │
  │                                                                   │
  │  For numeric sorting: pad with zeros                             │
  │  "00009" < "00010" ✓                                            │
  │                                                                   │
  │  Or use Number type for sort key (true numeric sort)            │
  └────────────────────────────────────────────────────────────────────┘
```

**Explanation:** Sort key patterns are the most powerful tool in DynamoDB schema design.
The `begins_with` and `between` operators on sort keys enable hierarchical data retrieval
without full-partition scans. ISO date format in sort keys enables time-range queries
naturally. Composite sort keys (prefixed with entity type + date + ID) allow retrieving
different entity types with different query operators using the same index.
