# Chapter 08: Wide-Column Stores — Scenario Questions

1. Your team runs HBase for a financial audit log system storing 500 billion rows of
   transaction events (50TB total). You observe that all write traffic goes to a single
   RegionServer — the other 19 RegionServers are idle. A `hbase hbck` scan shows all
   regions for the audit log table are concentrated on one server. What is the root
   cause, how do you diagnose it definitively, and what is your immediate remediation
   and long-term row key redesign?

2. Your HBase cluster is experiencing 30-second read latency spikes every 4 hours.
   Investigation shows the spikes correlate with major compaction jobs starting on
   multiple RegionServers simultaneously. The cluster serves a real-time fraud detection
   system requiring p99 latency under 100ms. How do you redesign the compaction schedule,
   tune the compaction parameters, and implement read request handling during compaction?

3. You are designing an HBase schema for a smart city IoT platform that ingests
   10 million sensor readings per minute from 500,000 devices. Each reading includes
   device_id, timestamp, sensor_type, and value. You need to support: (1) fetch the
   latest 100 readings for a specific device, (2) fetch all readings for a device
   in a time range, (3) run hourly aggregations across all devices of a given sensor
   type. Design the row key, column family structure, and explain the access pattern
   trade-offs.

4. Your team is migrating a 5TB MySQL database to HBase to support horizontal write
   scaling. The MySQL schema has a 200-column events table that is written to at 100K
   rows/second and read sparsely (only specific columns per query). During the HBase
   migration design phase, your architect proposes putting all 200 columns in one column
   family. Walk through why this is wrong, the correct column family design, and the
   performance implications of your redesign.

5. Your HBase cluster experienced a ZooKeeper quorum loss (3 of 5 ZooKeeper nodes failed
   simultaneously). HBase is completely unavailable — no reads or writes. The WAL data
   on the remaining RegionServers is intact. Describe the recovery procedure step by step,
   including the risks of split-brain scenarios and how you verify data integrity after
   recovery.

6. You are implementing a multi-tenant HBase cluster where 20 teams share the same physical
   cluster. Team A runs nightly bulk loads (100GB/hour), Team B runs real-time queries
   (p99 < 50ms), and Team C runs analytics scans. Team A's bulk loads are saturating
   the HDFS write bandwidth and causing Team B's latency to spike to 5 seconds. How do
   you isolate workloads in HBase, and what cluster-level resource controls can you apply?

7. A Google Bigtable table in your production system is experiencing row key hotspotting
   despite using a salted row key scheme. Investigation reveals that your salt only has
   4 values (0-3) but Bigtable automatically split the table to 32 tablets. Half the
   tablets have no traffic while 4 tablets are saturated. How does this happen, and
   how do you redesign the salting to work correctly with Bigtable's dynamic tablet
   splitting?

8. Your time-series data in HBase shows severe read amplification — simple point lookups
   require reading 15-20 HFiles per RegionServer before finding the data. EXPLAIN outputs
   show many StoreFile reads per row lookup, and Bloom filter hit rates are under 30%.
   Diagnose the root cause, and explain what configuration changes and data model changes
   would bring Bloom filter effectiveness to above 90%.

9. You are choosing between HBase and Cassandra for a new user activity log system
   (10K writes/sec, 2K reads/sec, 1TB/year growth, 3-year retention with downsampling).
   The team has strong Java expertise but no existing Cassandra or HBase infrastructure.
   You're on AWS. Walk through your complete evaluation framework and provide a
   recommendation with quantitative justification.

10. Your HBase cluster has 50 RegionServers, 500 tables, and 50,000 regions. You notice
    that RegionServer GC pause times are climbing — p99 GC pause is now 8 seconds, causing
    WAL write stalls and client timeouts. Diagnose the possible causes (too many open
    HFile handles, LRU BlockCache off-heap issues, MemStore pressure) and describe
    the JVM tuning and HBase configuration changes to bring GC pauses below 1 second.
