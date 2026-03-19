# Chapter 03 — Cassandra: Scenario Questions

Production-level Cassandra scenarios. Answer out loud before reviewing material.

---

1. **The tombstone tombstone**
   Your Cassandra cluster is experiencing increasingly slow reads on the `user_activity` table.
   The table stores user events with a TTL of 30 days, and you delete individual events when
   users request data erasure (GDPR). Monitoring shows read latency at p99 = 8 seconds,
   and `nodetool cfstats` reports tombstone_scanned_per_read averages over 5,000. Cassandra
   logs show `TombstoneOverwhelmingException`. Explain exactly what is happening, why deletes
   cause this problem, and provide a complete remediation plan including schema changes,
   compaction strategy changes, and operational steps.

2. **The hot partition meltdown**
   Your IoT telemetry platform stores sensor readings with the schema:
   `PRIMARY KEY (sensor_id, timestamp)`. One sensor (id: "gateway-001") is transmitting at
   10,000 writes/second, while the average sensor writes at 1/second. The node that owns the
   `gateway-001` partition is at 100% CPU and disk write throughput. All other nodes are idle.
   Identify the anti-pattern, redesign the partition key to distribute load, and explain what
   changes are needed in the application layer to support the new key design.

3. **The replication factor expansion**
   Your Cassandra cluster was deployed with RF=1 (no replication) for development and
   has been running in production for 6 months. Leadership discovers this and asks you to
   migrate to RF=3. There are 9 nodes in the cluster, 2TB of data, and the system handles
   100k writes/hour. Walk through the steps to safely expand the replication factor without
   downtime, including the exact CQL commands, the repair strategy, and the risks during
   the migration window.

4. **The QUORUM reads after a node failure**
   Your 6-node Cassandra cluster (RF=3) is configured with `consistency level QUORUM` for
   both reads and writes (W=2, R=2). Two nodes fail simultaneously due to a firmware bug.
   With 4 nodes remaining and RF=3, which partitions can still be read at QUORUM? Which
   cannot? After the nodes come back online, what is the state of their data, and what
   must you do before you can trust reads again?

5. **The compaction strategy migration**
   Your time-series metrics table has been running Size-Tiered Compaction Strategy (STCS)
   for 2 years. You now have 300 SSTables per partition, read latency is degrading as
   Cassandra must merge more SSTables per read, and disk space is 3x the actual data size.
   Explain why STCS is wrong for this workload, why TWCS (Time Window Compaction Strategy)
   is the right replacement, and the exact steps and risks of changing compaction strategy
   on a live table.

6. **The lightweight transaction bottleneck**
   Your team uses `IF NOT EXISTS` (a Cassandra LWT) to ensure unique usernames during
   registration. At peak load, you handle 500 registrations/second. After deploying a
   marketing campaign, traffic spikes to 5,000 registrations/second and latency explodes.
   LWT operations that normally take 5ms are now taking 3 seconds. Explain why LWT latency
   scales poorly under high concurrent load, what is happening at the Paxos protocol level,
   and how you would redesign the uniqueness check to avoid LWT.

7. **The multi-datacenter consistency bug**
   Your application writes with `LOCAL_QUORUM` to DC1 and reads with `LOCAL_QUORUM` from DC2.
   Users are reporting that they write data in one browser tab and immediately refresh to
   see nothing (stale read). The replication between DCs is asynchronous and DC2 is lagging
   by 200ms. Explain why this configuration creates this exact bug, what consistency level
   combination would fix it, what the performance cost of the fix is, and how you would
   implement a fallback for reads that allows eventual consistency for less critical data.

8. **The schema migration nightmare**
   Your team needs to add a new column `last_login_at` (timestamp) to the `users` table,
   which has 500 million rows across a 12-node cluster. In relational databases, an ALTER TABLE
   would lock the table and take hours. How does Cassandra handle schema changes? What is the
   actual operational risk of an ALTER TABLE in Cassandra, what happens to existing rows
   that don't have the new column, and how do you handle the dual-read period when both old
   and new application versions are running simultaneously?

9. **The ALLOW FILTERING production incident**
   A junior developer deployed a query `SELECT * FROM orders WHERE status = 'PENDING'` where
   `status` is a non-primary-key column with no secondary index. In development (1,000 rows),
   this ran fine. In production (500 million rows), it brought down the cluster. Explain exactly
   what `ALLOW FILTERING` does internally, why it causes full table scans across multiple nodes,
   and provide the correct schema design to support this query pattern efficiently.

10. **The Cassandra 5.0 upgrade planning**
    Your team is running Cassandra 3.11 and wants to upgrade to Cassandra 5.0. You have
    20 nodes, 5TB of data, RF=3, and a 24x7 SLA requiring < 1 minute of downtime per year.
    Walk through the rolling upgrade strategy, what compatibility concerns exist between
    3.11 and 5.0, what features in 5.0 justify the upgrade complexity, and how you validate
    each node after upgrade without impacting the remaining nodes.
