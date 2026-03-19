# Replication and High Availability — Follow-Up Traps

Tricky follow-up questions exposing common misconceptions about replication, HA, and backup strategies.

---

## Trap 1: "A synchronous standby guarantees zero data loss in all failure scenarios."

**What most people say:** With synchronous_commit = remote_apply, committed data is always on the standby.

**Correct answer:** Synchronous replication guarantees zero data loss only if the standby remains available. If the synchronous standby goes down (network partition, standby crash), PostgreSQL's default behavior is to hang indefinitely waiting for the synchronous standby to acknowledge writes — effectively making the primary read-only. You must configure this trade-off explicitly:

```sql
-- Check current synchronous standby configuration
SHOW synchronous_standby_names;

-- Configuration: require acknowledgment from at least 1 of 2 standbys
-- If BOTH standbys are down, writes will block!
synchronous_standby_names = 'ANY 1 (standby1, standby2)'

-- If the only synchronous standby is down, the primary becomes non-writable.
-- To allow writes to continue (accepting potential data loss), you must either:
-- 1. Demote the standby from synchronous status manually
-- 2. Set synchronous_commit = local for the session
-- 3. Use synchronous_standby_names = '' temporarily
```

**The nuance:** ANY 1 of N configuration means: as long as at least 1 standby is available, the primary can write. This is the recommended production configuration for true HA with synchronous replication.

---

## Trap 2: "Replication slots are always beneficial — they prevent data loss on standbys."

**What most people say:** Yes, use replication slots to ensure standbys never fall behind.

**Correct answer:** Replication slots are a double-edged sword. They guarantee that WAL files are never deleted until all slot consumers have consumed them — preventing the "replica fell too far behind and needed WAL that was already deleted" problem. But if a slot consumer (replica, logical consumer) disappears or falls far behind for any reason, the primary accumulates WAL indefinitely. A forgotten logical replication slot can fill the entire disk of the primary in days or weeks, taking down the entire production system:

```sql
-- Monitor replication slots for dangerous lag
SELECT
  slot_name,
  slot_type,
  active,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained,
  restart_lsn,
  confirmed_flush_lsn
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;

-- Alert if any slot retains more than 10GB of WAL
-- Drop inactive/orphaned slots immediately:
SELECT pg_drop_replication_slot('logical_consumer_slot');  -- if consumer is gone

-- Limit maximum WAL retention per slot (PostgreSQL 13+)
-- max_slot_wal_keep_size = 10GB  -- in postgresql.conf
-- Slot will be invalidated (dropped) if lag exceeds this limit
```

---

## Trap 3: "A daily pg_dump backup is sufficient for production databases."

**What most people say:** pg_dump creates a consistent backup that can restore the database.

**Correct answer:** pg_dump is not sufficient for most production use cases for several reasons:
1. **RTO**: Restoring a pg_dump of a 500GB database takes hours. During that time the system is down.
2. **RPO**: A daily backup means up to 24 hours of data loss on failure — unacceptable for most production systems.
3. **No PITR**: You cannot restore to "3:47 PM yesterday." You can only restore to the exact moment the dump was taken.
4. **Consistency during backup**: pg_dump uses a single transaction snapshot so the dump is consistent, but ongoing changes after the snapshot started are not captured.

The correct production backup strategy: **pg_basebackup** (full base backup every 1-7 days) + **continuous WAL archiving** (every WAL segment as it fills, achieving near-real-time archival). Together these provide PITR to any second within the retention window.

```bash
# Configure continuous WAL archiving
# postgresql.conf:
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
# or for S3:
archive_command = 'aws s3 cp %p s3://my-wal-archive/$(hostname)/%f'
```

---

## Trap 4: "Promoting a standby to primary is reversible — you can demote it back."

**What most people say:** You can promote, test, and then demote back if needed.

**Correct answer:** Promotion is a one-way operation in PostgreSQL. Once a standby is promoted, it begins accepting write transactions and generating its own WAL timeline. It cannot be "demoted" back to a standby of its former primary. To make it a standby again, you must either: (1) rebuild it from a fresh base backup of the new primary, or (2) if the old primary had fewer writes after the promotion point (unlikely in a real failover), use pg_rewind to resync it. pg_rewind can resync the old primary back to be a standby of the new primary by finding the divergence point and rewriting the diverged pages — but only if wal_log_hints=on or data checksums are enabled:

```bash
# After failover, resync the old primary to become a standby of the new primary:
pg_rewind --target-pgdata=/var/lib/postgresql/data \
          --source-server="host=new-primary port=5432 user=replicator"

# Requirements:
# - wal_log_hints = on on the old primary, OR data_checksums = on at initdb time
# - PostgreSQL version must be same on both
# - old primary must not be too far ahead (recent divergence)
```

---

## Trap 5: "pg_basebackup pauses the primary database while it copies files."

**What most people say:** Yes, pg_basebackup needs to pause writes to get a consistent snapshot.

**Correct answer:** pg_basebackup does NOT pause or lock the primary. It takes a consistent base backup while the primary continues serving writes. This is possible because PostgreSQL records the WAL position at backup start (with the `pg_start_backup()` function internally) and forces a checkpoint. Any page changes during the backup that create inconsistencies will be resolved during recovery by replaying WAL from the backup start position. The backup is consistent only when combined with the WAL generated during the backup — this is why pg_basebackup also archives the WAL generated during the backup. The primary experiences only the I/O load of reading all data files (significant for large databases) but no availability impact.

---

## Trap 6: "Hot standby conflicts only affect the standby — they don't impact the primary."

**What most people say:** The standby handles its own conflicts independently.

**Correct answer:** Hot standby conflicts DO affect the primary when hot_standby_feedback = on. Without hot_standby_feedback, the primary runs VACUUM and cleanup operations normally, potentially deleting rows that are still visible to long-running queries on the standby. The standby must cancel those queries (hot standby conflict). With hot_standby_feedback = on, the standby reports its oldest transaction XID to the primary, causing the primary's VACUUM to delay cleaning up rows that the standby might still need — effectively extending the primary's VACUUM horizon. This can cause table bloat on the PRIMARY because of long-running queries on the STANDBY. It is the same effect as having a long-running transaction on the primary itself.

```sql
-- Monitor hot standby conflicts on the standby
SELECT * FROM pg_stat_database_conflicts;
-- confl_tablespace, confl_lock, confl_snapshot, confl_bufferpin, confl_deadlock

-- Check if hot_standby_feedback is protecting the standby
SHOW hot_standby_feedback;

-- On the primary, check if standby's old_xmin is holding up VACUUM
SELECT
  application_name,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes
FROM pg_stat_replication;
```

---

## Trap 7: "PITR can restore a database to any point in time within the last 30 days."

**What most people say:** Yes, that's the point of continuous WAL archiving.

**Correct answer:** PITR can restore to any point in time for which you have: (1) a base backup taken BEFORE the target time, AND (2) all WAL files from the base backup time through the target time, WITHOUT gaps. A single missing WAL segment breaks the recovery chain. You can restore to the point just before the missing segment, but not to any later point. This is why WAL archival monitoring is critical — you must alert on any failed archive_command execution immediately. Also: PITR accuracy depends on the WAL archive being complete and accessible. If archiving to S3, any S3 outage or permission issue can silently break the archive chain, creating gaps you won't discover until disaster strikes.

```bash
# Verify WAL archive completeness (using pgBackRest):
pgbackrest --stanza=main info

# Test a PITR restore regularly — this is the only true validation:
# Restore to a test instance, verify data at the target timestamp,
# then discard the test instance. This should be automated monthly.
```

---

## Trap 8: "Logical replication replicates everything, including DDL changes."

**What most people say:** Logical replication replicates all changes from publication to subscription.

**Correct answer:** PostgreSQL's built-in logical replication does NOT replicate DDL (schema changes). It only replicates DML (INSERT, UPDATE, DELETE, TRUNCATE). This is one of the most important limitations to know. If you ALTER TABLE on the publisher to add a column, the subscriber does not automatically receive that DDL — you must manually apply the same ALTER TABLE on the subscriber before replication of that table's data resumes. Failure to do so will cause replication to fail with a schema mismatch error. Additionally, sequences are not replicated automatically — sequence values must be synchronized manually during cutover. Solutions for DDL replication: pglogical extension, Debezium CDC, AWS DMS, or application-level DDL deployment tools that apply to all nodes simultaneously.

```sql
-- Logical replication setup
-- On publisher:
CREATE PUBLICATION mypub FOR TABLE orders, customers;

-- On subscriber:
CREATE SUBSCRIPTION mysub
  CONNECTION 'host=primary port=5432 dbname=mydb user=replicator'
  PUBLICATION mypub;

-- DDL on publisher (NOT automatically replicated):
ALTER TABLE orders ADD COLUMN notes TEXT;  -- Only on publisher!

-- Must manually run on subscriber too:
-- ALTER TABLE orders ADD COLUMN notes TEXT;  -- On subscriber
-- THEN replication of new rows with notes column will work
```

---

## Trap 9: "More read replicas always improve read throughput proportionally."

**What most people say:** 3 replicas = 3x read capacity.

**Correct answer:** Read replicas improve throughput only for stateless read queries that tolerate eventual consistency. Several factors limit scaling: (1) All replicas apply WAL from the primary — high write volume on the primary limits how many replicas can keep up (WAL sender process per replica adds CPU/network load). (2) Long-running queries on replicas cause hot standby conflicts (if hot_standby_feedback=off) or hold up primary VACUUM (if on). (3) Application routing complexity increases: if reads that depend on a recent write go to a lagging replica, they see stale data. (4) Connection pooling must be replica-aware. The real scaling limit is write throughput — no amount of read replicas helps if the primary is the bottleneck.

---

## Trap 10: "A pg_dump taken from a running database is always consistent."

**What most people say:** pg_dump acquires a transaction snapshot so it's always consistent.

**Correct answer:** A single pg_dump IS internally consistent — it uses a single transaction snapshot. However, if you dump multiple tables with separate pg_dump calls (or use pg_dumpall which dumps each database sequentially), each dump gets a different snapshot. For cross-table referential integrity, you need all tables at the same snapshot moment. The solution: use pg_dump with a single run (which wraps everything in one snapshot) or pg_dumpall for the full cluster. For live databases with active transactions, a consistent multi-table backup requires either a single pg_dump run or a file-system snapshot at the OS level. Also: restoring a pg_dump over a live database (to pg_restore --clean --if-exists) does NOT guarantee referential integrity during the restore — objects must be restored in dependency order.

---

## Trap 11: "Failover is automatic with any streaming replication setup."

**What most people say:** PostgreSQL streaming replication handles automatic failover.

**Correct answer:** PostgreSQL streaming replication is NOT self-managing. Raw streaming replication provides: WAL streaming, standby serving reads, and the ability to be manually promoted. It does NOT provide: automatic failure detection, automatic promotion, DNS/VIP failover, fencing of the old primary, or coordination between multiple standbys. Automatic failover requires an external HA manager: Patroni (most common), repmgr, or cloud-managed services (RDS Multi-AZ). Without these, a primary failure requires: human detection, manual `pg_ctl promote` on the chosen standby, manual application reconfiguration, and manual cleanup of the old primary. This process typically takes 5-30 minutes with a skilled DBA — unacceptable for most production SLAs.

---

## Trap 12: "Table partitioning always improves query performance."

**What most people say:** Partitioning lets queries skip most of the data (partition pruning) so performance improves.

**Correct answer:** Partitioning only improves performance when queries include the partition key in their WHERE clause (enabling partition pruning). Queries that do NOT filter on the partition key must scan ALL partitions — which can be SLOWER than a single non-partitioned table because: (1) planning time increases O(number of partitions) as the planner must consider each partition, (2) execution overhead of the Append node, (3) index statistics are per-partition, causing suboptimal cross-partition join planning. For a table with 500 monthly partitions, a query without the date filter scans all 500 partitions with 500 separate index lookups — potentially 10x slower than a single B-tree scan of the unpartitioned table. Partitioning is a maintenance tool as much as a performance tool — it enables fast partition-drop for data lifecycle management and targeted VACUUM.
