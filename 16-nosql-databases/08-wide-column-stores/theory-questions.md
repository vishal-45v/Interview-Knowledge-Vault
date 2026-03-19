# Chapter 08: Wide-Column Stores — Theory Questions

1. Explain the wide-column store data model hierarchy: keyspace → table → row → column
   families → columns. How does a "column family" differ from a SQL table column, and
   why is the term "wide-column" potentially misleading?

2. Describe the HBase architecture: HMaster, RegionServer, ZooKeeper, and HDFS. What
   is the role of each component, and what happens during a RegionServer failure?

3. What is an HBase Region? How does a Region split work, and what is the "region split
   storm" problem that occurs after a major compaction or bulk load?

4. Explain the difference between minor compaction and major compaction in HBase. What
   does each do, what resources does each consume, and when would you trigger a major
   compaction manually?

5. What is a Bloom filter in HBase? How does it work probabilistically, and how does it
   reduce read amplification for row-key point lookups?

6. Explain the HBase Write Ahead Log (WAL/HLog). What data goes into the WAL, when is
   it written, and how is it used for crash recovery?

7. Describe the LSM-Tree (Log-Structured Merge Tree) data structure. Walk through the
   write path (memtable → L0 → L1 → Ln) and explain what happens during compaction at
   each level.

8. Define write amplification, read amplification, and space amplification in the context
   of LSM trees. How does the compaction strategy (leveled vs size-tiered) affect each
   of these three metrics?

9. What is leveled compaction vs size-tiered compaction in Cassandra/HBase? When does
   each perform better, and what workload characteristics should guide your choice?

10. Describe Google Bigtable's original architecture. What is the role of the Master,
    Tablet Servers, and the Chubby lock service? How do tablets map to SSTables on GFS?

11. What are HBase coprocessors? Describe the difference between Observer and Endpoint
    coprocessors. What risks do coprocessors introduce in a shared HBase cluster?

12. Explain the HBase block cache. What are the BlockCache implementations (LRUBlockCache
    vs BucketCache vs CombinedBlockCache), how does each work, and when would you use
    an off-heap BucketCache?

13. How does HBase row key design affect performance? Explain sequential key hotspotting
    and describe three techniques for distributing writes across RegionServers.

14. Compare HBase, Cassandra, and Google Bigtable/Spanner as wide-column stores. What
    are the key architectural differences in terms of consistency model, replication,
    and query capability?

15. What is the Bigtable paper's concept of "tablet server" failure recovery? How does
    the Master know a tablet server has failed, and how is recovery different from
    HBase's approach?

16. Explain HBase's support for ACID transactions. What are the transaction guarantees
    at the row level vs the multi-row level, and what library extends HBase to support
    multi-row ACID (Tephra/Omid)?

17. What is a "tombstone" in an LSM-tree-based database? How are tombstones propagated
    during compaction, and what is the "tombstone accumulation" problem that causes
    read latency degradation?

18. Describe the write path in HBase in detail: from a client Put() call through WAL,
    MemStore, HFile flush, and eventual compaction. At what points can data loss occur,
    and how does each safeguard prevent it?
