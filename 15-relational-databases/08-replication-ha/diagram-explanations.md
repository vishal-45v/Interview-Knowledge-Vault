# Replication and High Availability — Diagram Explanations

ASCII diagrams illustrating replication architectures, failover sequences, and backup topologies.

---

## Diagram 1: PostgreSQL Streaming Replication Architecture

```
  PRIMARY (Read/Write)                     STANDBY (Read-only)
  ─────────────────────────               ──────────────────────────
  ┌─────────────────────┐                 ┌─────────────────────────┐
  │  Backend Processes  │                 │  Backend Processes       │
  │  (client queries)   │                 │  (read-only queries)     │
  └──────────┬──────────┘                 └──────────────────────────┘
             │ write                                  ▲
             ▼                                        │ replay
  ┌─────────────────────┐                 ┌──────────┴──────────────┐
  │  shared_buffers     │                 │  shared_buffers          │
  │  (dirty pages)      │                 │  (applying WAL changes)  │
  └──────────┬──────────┘                 └─────────────────────────┘
             │ WAL records                            ▲
             ▼                                        │
  ┌─────────────────────┐    stream WAL   ┌──────────┴──────────────┐
  │  WAL Buffers        │ ──────────────► │  WAL Receiver Process   │
  │                     │                 │  (walreceiver)           │
  └──────────┬──────────┘                 └──────────┬──────────────┘
             │ flush                                  │ write to disk
             ▼                                        ▼
  ┌─────────────────────┐                 ┌──────────────────────────┐
  │  WAL Sender Process │                 │  WAL Files               │
  │  (walsender)        │                 │  (on standby disk)       │
  └──────────┬──────────┘                 └──────────────────────────┘
             │ reads from
             ▼
  ┌─────────────────────┐
  │  WAL Segment Files  │───────────────► WAL Archive (S3)
  │  pg_wal/            │  archive_command
  └─────────────────────┘

  Synchronous commit flow:
  ┌──────────────────────────────────────────────────────────────────┐
  │  synchronous_commit = on:                                        │
  │  Client COMMIT → WAL flushed (primary) → ACK to client          │
  │                                                                  │
  │  synchronous_commit = remote_write:                              │
  │  Client COMMIT → WAL sent to standby (not fsynced) → ACK        │
  │                                                                  │
  │  synchronous_commit = remote_apply:                              │
  │  Client COMMIT → WAL applied on standby → ACK (maximum safety)  │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Diagram 2: Patroni Failover Sequence

```
  NORMAL OPERATION (t=0):
  ─────────────────────────────────────────────────────────────────
  node1 (PRIMARY) ──heartbeat every 10s──► etcd (leader=node1, ttl=30s)
  node2 (standby) ──streaming replication── node1
  node3 (standby) ──streaming replication── node1

  FAILURE DETECTED (t=10):
  ─────────────────────────────────────────────────────────────────
  node1 (PRIMARY) ── CRASH / NETWORK PARTITION ──X

  node2 and node3 detect no heartbeat from node1 in etcd
  They wait for TTL to expire...

  LEADER ELECTION (t=30):
  ─────────────────────────────────────────────────────────────────
  etcd TTL=30s expires → leader key deleted
  node2 (lag=0.2s)  ─── tries to claim leader key in etcd ───► WIN
  node3 (lag=3.1s)  ─── tries to claim leader key in etcd ───► LOSE
                        (node2 won first, node3 becomes standby)

  PROMOTION (t=31):
  ─────────────────────────────────────────────────────────────────
  node2: pg_ctl promote
  node2: becomes new PRIMARY, starts accepting writes
  node3: connects to node2 as new streaming replication source
  HAProxy: health check /primary now succeeds on node2 (port 8008)
  HAProxy: routes write traffic to node2

  RECOVERY OF OLD PRIMARY (t=+minutes):
  ─────────────────────────────────────────────────────────────────
  node1 comes back online
  Patroni detects it is no longer the etcd leader
  Patroni runs pg_rewind to resync node1 to node2's timeline
  node1 rejoins as standby of node2

  Timeline divergence handling:
  node1 timeline: ... TXN_A → TXN_B → TXN_C (wrote after partition)
  node2 timeline: ... TXN_A → TXN_B → TXN_D (new primary)
  pg_rewind finds divergence point, rewinds node1 to TXN_B, rebuilds from node2
  TXN_C on node1 is lost if it was never replicated to node2
```

---

## Diagram 3: PITR Restore Architecture

```
  BACKUP COMPONENTS:
  ──────────────────────────────────────────────────────────────────

  Time ─────────────────────────────────────────────────────────────►
       Sun         Mon         Tue         Wed         Thu
       1AM         1AM         1AM         1AM (crash!)

  Base Backups (weekly):
  ┌──────────┐                           ┌──────────┐
  │Full      │                           │Full      │
  │Backup    │                           │Backup    │
  │(500GB)   │                           │(510GB)   │
  └──────────┘                           └──────────┘

  WAL Archive (continuous, every ~1 minute):
  ─────────── WAL WAL WAL WAL WAL WAL WAL WAL WAL WAL WAL WAL ──────
               000001  000002  000003  ... 008472  008473  ←last before crash

  PITR to Wednesday 2:30 PM:
  ──────────────────────────
  1. Find latest base backup BEFORE Wed 2:30 PM → Monday 1AM full backup
  2. Restore base backup to new server
  3. Replay WAL segments from Monday 1AM → Wednesday 2:30 PM

  Recovery target options:
  ┌──────────────────────────────┬───────────────────────────────────┐
  │ recovery_target_time         │ Restore to exact timestamp        │
  │ recovery_target_lsn          │ Restore to exact WAL position     │
  │ recovery_target_xid          │ Restore to just after specific XID│
  │ recovery_target_name         │ Restore to named recovery point   │
  │ recovery_target = 'immediate'│ Restore to end of backup (fastest)│
  └──────────────────────────────┴───────────────────────────────────┘

  Recovery configuration (postgresql.conf):
  ┌───────────────────────────────────────────────────────────────────┐
  │ restore_command = 'aws s3 cp s3://backups/wal/%f %p'              │
  │ recovery_target_time = '2024-06-12 14:30:00 UTC'                  │
  │ recovery_target_action = 'promote'                                │
  │ recovery_target_inclusive = true                                  │
  └───────────────────────────────────────────────────────────────────┘
```

---

## Diagram 4: Table Partitioning — Query Plan with Pruning

```
  Partitioned table: events (4 quarterly partitions)
  ┌─────────────┬─────────────┬─────────────┬─────────────┐
  │ events_q1   │ events_q2   │ events_q3   │ events_q4   │
  │ Jan–Mar     │ Apr–Jun     │ Jul–Sep     │ Oct–Dec     │
  │ 50M rows    │ 45M rows    │ 48M rows    │ 52M rows    │
  │ 2.1GB       │ 1.9GB       │ 2.0GB       │ 2.2GB       │
  └─────────────┴─────────────┴─────────────┴─────────────┘

  Query: SELECT * FROM events WHERE occurred_at >= '2024-04-01'
                                AND occurred_at <  '2024-07-01'

  EXPLAIN output with partition pruning:
  ┌────────────────────────────────────────────────────────────────┐
  │  Append  (cost=... rows=45000000)                              │
  │  ├── Seq Scan on events_q1  ← PRUNED (excluded from plan)     │
  │  ├── Seq Scan on events_q2  ← SCANNED ✓                       │
  │  ├── Seq Scan on events_q3  ← PRUNED (excluded from plan)     │
  │  └── Seq Scan on events_q4  ← PRUNED (excluded from plan)     │
  │                                                                │
  │  Only 1.9GB scanned vs 8.2GB total → 4.3x improvement         │
  └────────────────────────────────────────────────────────────────┘

  WRONG: Query without partition key
  Query: SELECT * FROM events WHERE user_id = 42

  EXPLAIN output (NO pruning possible):
  ┌────────────────────────────────────────────────────────────────┐
  │  Append  (cost=... rows=195000000)                             │
  │  ├── Index Scan on events_q1 using idx_events_q1_user_id      │
  │  ├── Index Scan on events_q2 using idx_events_q2_user_id      │
  │  ├── Index Scan on events_q3 using idx_events_q3_user_id      │
  │  └── Index Scan on events_q4 using idx_events_q4_user_id      │
  │                                                                │
  │  All 4 partitions scanned — may be slower than single table   │
  │  due to planning overhead and 4 separate index lookups         │
  └────────────────────────────────────────────────────────────────┘
```

---

## Diagram 5: Multi-Region Read Replica Architecture

```
                        ┌─────────────────────┐
                        │   us-east-1          │
  Write traffic ───────►│   PRIMARY            │
  (all regions          │   PostgreSQL         │
   route writes         │   + PgBouncer        │
   to primary)          └──────────┬──────────┘
                                   │
                   ┌───────────────┼───────────────┐
                   │ WAL streaming  │               │
                   ▼               ▼               ▼
         ┌────────────────┐ ┌─────────────┐ ┌──────────────┐
         │  eu-west-1     │ │ ap-east-1   │ │ us-west-2    │
         │  Read Replica  │ │ Read Replica│ │ Read Replica │
         │  (EU reads)    │ │ (APAC reads)│ │ (US-W reads) │
         │  lag: ~50ms    │ │ lag: ~180ms │ │ lag: ~30ms   │
         └────────────────┘ └─────────────┘ └──────────────┘

  Application routing logic:
  ┌──────────────────────────────────────────────────────────────────┐
  │  func getDbConnection(op: "read" | "write") {                    │
  │    if (op === "write") return PRIMARY;                           │
  │    if (op === "read") {                                          │
  │      // Route to nearest replica by latency                     │
  │      return nearestReplica(userRegion);                          │
  │    }                                                             │
  │  }                                                               │
  │                                                                  │
  │  // Handle read-after-write consistency:                         │
  │  // After a write, force the next read to primary for N seconds  │
  │  if (recentlyWrote(userId, withinSeconds: 5)) {                  │
  │    return PRIMARY;  // avoid reading stale data from replica     │
  │  }                                                               │
  └──────────────────────────────────────────────────────────────────┘

  Replication lag monitoring:
  ┌──────────────────────────────────────────────────────────────────┐
  │  -- On primary: check per-region replica lag                     │
  │  SELECT                                                          │
  │    application_name AS replica,                                  │
  │    client_addr AS replica_ip,                                    │
  │    replay_lag,                                                   │
  │    pg_wal_lsn_diff(                                              │
  │      pg_current_wal_lsn(), replay_lsn                           │
  │    ) AS lag_bytes                                                │
  │  FROM pg_stat_replication                                        │
  │  ORDER BY replay_lag DESC;                                       │
  └──────────────────────────────────────────────────────────────────┘
```

---

## Diagram 6: Replication Slot Danger — WAL Accumulation

```
  NORMAL OPERATION — slot consumer active:
  ──────────────────────────────────────────
  Time ──────────────────────────────────────────────────────────────►
       9 AM         10 AM        11 AM        12 PM

  WAL generated: ████████████████████████████████████████████████████
  WAL consumed:  ████████████████████████████████████████████████████
  WAL retained:  ░░░░░░░░░░░░░░░░░░░░░░ (minimal — consumer is live)
  Disk usage:    ▓▓▓ (normal — ~500MB retained)

  CONSUMER GOES DOWN — slot becomes stale:
  ────────────────────────────────────────
       Day 1       Day 2        Day 3        Day 7

  WAL generated: ████████████████████████████████████████████████████
  WAL consumed:  ████ (stopped here — consumer offline)
  WAL RETAINED:  ████████████████████████████████████████████████████
                                              (ALL WAL since Day 1!)
  Disk usage:    ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ DISK FULL!

  ┌──────────────────────────────────────────────────────────────────┐
  │  Alert query (run every 5 minutes):                              │
  │                                                                  │
  │  SELECT slot_name, active,                                       │
  │    pg_size_pretty(                                               │
  │      pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)         │
  │    ) AS wal_retained                                             │
  │  FROM pg_replication_slots                                       │
  │  WHERE pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)       │
  │        > 10 * 1024 * 1024 * 1024;  -- alert at 10GB             │
  │                                                                  │
  │  Immediate remediation:                                          │
  │  SELECT pg_drop_replication_slot('stale_consumer_slot');         │
  │                                                                  │
  │  Prevention (postgresql.conf):                                   │
  │  max_slot_wal_keep_size = 10GB  -- auto-invalidate at 10GB      │
  └──────────────────────────────────────────────────────────────────┘
```
