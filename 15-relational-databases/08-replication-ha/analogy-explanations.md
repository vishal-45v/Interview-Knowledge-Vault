# Replication and High Availability — Analogy Explanations

Conceptual analogies for replication, failover, backup, and HA cluster management.

---

## Analogy 1: Streaming Replication — The Live News Transcript

**Story:**
A news anchor (the primary) is delivering a live broadcast. In a back room, a stenographer (the WAL sender) transcribes every word as it is spoken, then immediately faxes the transcript to a partner studio across town (the standby). The partner studio's operator (WAL receiver) receives the fax and reads it aloud to their local audience. The partner studio is always a few seconds behind — the lag depends on fax transmission time (network latency). If the main studio loses power (primary failure), the partner studio can immediately take over broadcasting from where they are — but the last few seconds of the original broadcast may be lost if the fax hadn't arrived yet.

**Database connection:**
```sql
-- Monitor the "fax lag" — how far behind is the standby?
SELECT
  application_name,
  replay_lag,                    -- time from primary write to standby apply
  pg_wal_lsn_diff(
    pg_current_wal_lsn(),
    replay_lsn
  ) AS lag_bytes
FROM pg_stat_replication;

-- The "stenographer" process on the primary:
-- pg_stat_activity shows a WAL sender process
SELECT pid, usename, application_name, state, query
FROM pg_stat_activity
WHERE backend_type = 'walsender';
```

**Why it matters:** Asynchronous replication (the default) means the fax hasn't been sent yet for the most recent seconds. If the primary crashes, those seconds are lost (the RPO). Synchronous replication holds the anchor's microphone until the partner studio confirms they received the fax — guaranteed zero loss but slower broadcast.

---

## Analogy 2: Synchronous vs Asynchronous Replication — The Surgeon's Dictation

**Story:**
A surgeon dictates operation notes for the medical record.

**Asynchronous (performance-first):** The surgeon dictates and the transcriptionist types as fast as possible but doesn't interrupt the surgeon. If the transcriptionist's computer crashes after the surgeon says "incision complete but before typing it", that note is lost. The surgeon never pauses — maximum throughput.

**Synchronous (safety-first):** After each sentence, the surgeon waits for the transcriptionist to say "recorded" before continuing. The operation notes are perfectly preserved even if the computer crashes between sentences. But the surgery takes longer because the surgeon is waiting.

**remote_apply (strictest):** The surgeon waits not just for the note to be written, but for the transcriptionist to have read it back and confirmed they understand it — double confirmation before proceeding.

**Database connection:**
```ini
# Asynchronous: maximum write throughput, small data loss window
synchronous_commit = off

# Local durability only, async to standby:
synchronous_commit = local

# Synchronous: standby must WRITE before ACK (small window for loss if standby crashes before fsync):
synchronous_commit = remote_write

# Full synchronous: standby must FLUSH to disk before ACK:
synchronous_commit = on  # or 'remote_apply' for applied confirmation

# Require at least 1 of 2 standbys to confirm:
synchronous_standby_names = 'ANY 1 (standby1, standby2)'
```

---

## Analogy 3: Replication Slots — The Reserved Seat Guarantee

**Story:**
A theater (the primary database) produces a newspaper each evening (WAL segments). Audience members (subscribers/standbys) collect the newspaper to stay informed. Normally, old newspapers are thrown out after 3 days. But if you have a reserved seat (replication slot), the theater holds YOUR newspapers indefinitely, even if you're away on vacation, even if the bin overflows. The theater won't throw out any newspaper that a reserved-seat holder hasn't collected yet — even if it means the storage room fills up completely.

**Database connection:**
```sql
-- Create a replication slot (reserve your seat):
SELECT pg_create_physical_replication_slot('my_replica_slot');

-- The slot holds WAL until the replica consumes it
-- Monitor how much WAL is being held:
SELECT
  slot_name,
  active,
  pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
  ) AS wal_retained,
  restart_lsn
FROM pg_replication_slots;

-- DANGER: If a consumer disappears, WAL accumulates forever
-- Set a safety limit (PostgreSQL 13+):
-- max_slot_wal_keep_size = 50GB  -- in postgresql.conf
-- Slot is invalidated (dropped) if retention exceeds this
```

**Why it matters:** The reserved-seat guarantee is both a feature (replica never misses WAL) and a risk (forgotten reservation fills the storage room). Always monitor slot lag and drop unused slots immediately.

---

## Analogy 4: Patroni and Leader Election — The Corporate Board Vote

**Story:**
A company (the cluster) needs exactly one CEO (the primary). The board of directors (etcd) keeps a record of who the current CEO is. The CEO must check in with the board every 30 seconds to "renew their lease" — proof that they are still active and in charge. If the CEO stops checking in (server crash, network failure), the board waits for the lease to expire, then calls an emergency election. Board members vote, and the candidate with the best track record (least replication lag) wins. The new CEO is announced to everyone, and the company continues operating. The old CEO, if they return from vacation, discovers they are no longer CEO and must become an employee (standby) under the new leadership.

**Database connection:**
```yaml
# Patroni's lease TTL — how long before leader considered dead:
ttl: 30              # 30 seconds without heartbeat = dead leader

# How often to check health and renew lease:
loop_wait: 10        # Check every 10 seconds

# Maximum replica lag allowed to be a failover candidate:
maximum_lag_on_failover: 1048576   # 1MB — too far behind = not eligible for promotion
```

```bash
# View cluster state:
patronictl -c /etc/patroni/patroni.yml list

# Manual switchover (planned maintenance):
patronictl -c /etc/patroni/patroni.yml switchover --master node1 --candidate node2

# Manual failover (force promote a specific node):
patronictl -c /etc/patroni/patroni.yml failover --master node1 --candidate node2 --force
```

---

## Analogy 5: PITR — The Movie Reel Archive

**Story:**
A film studio wants to be able to watch any scene from any movie at any point in time. They have two systems: (1) They create a complete master copy of each movie every week (base backup), and (2) they film a continuous "director's commentary" that records every edit decision made since the last master copy (WAL archive). To reconstruct the movie at any specific moment — say, Wednesday at 3:15 PM — they take the most recent master copy from Sunday, then replay the director's commentary from Sunday through Wednesday 3:15 PM. If even one reel of the director's commentary is missing, they can only reconstruct up to the missing reel, not beyond it.

**Database connection:**
```ini
# "Director's commentary" — continuous recording:
archive_mode = on
archive_command = 'aws s3 cp %p s3://backups/wal/%f'
archive_timeout = 60              # Record at least every 60 seconds

# Recovery — "replay the commentary to a specific time":
# postgresql.conf (during recovery)
restore_command = 'aws s3 cp s3://backups/wal/%f %p'
recovery_target_time = '2024-06-15 14:30:00 UTC'
recovery_target_action = 'promote'
```

```bash
# "Take a master copy" — weekly full base backup:
pgbackrest --stanza=prod --type=full backup

# "Reconstruct to Wednesday 3:15 PM":
pgbackrest --stanza=prod \
           --type=time \
           --target="2024-06-12 15:15:00 UTC" \
           restore
```

**Why it matters:** The unbroken chain of WAL archives is the film reels. One missing segment and you can only restore to just before the gap. This is why archive monitoring is critical — a failed archive_command silently breaks the recovery chain.

---

## Analogy 6: Table Partitioning — The Filing Cabinet System

**Story:**
A law firm has millions of case files. Without organization, finding or archiving any case requires searching through everything. Instead, they use a filing cabinet system: one drawer per year (RANGE partitioning by date), or one cabinet per client region (LIST partitioning by region). When a lawyer asks "show me all cases from 2022," the filing assistant goes directly to the 2022 drawer (partition pruning) without opening any other drawer. When the firm no longer needs 2019 cases, they roll the entire 2019 drawer into offsite storage instantly (DROP TABLE partition — O(1) versus deleting millions of rows one by one).

**Database connection:**
```sql
-- "Close the 2023 drawer" — instant data archival
-- vs DELETE FROM events WHERE year = 2023 (could take hours on billions of rows)
DROP TABLE events_2023;          -- instant, locks only the partition

-- Or detach for archival (keeps data accessible separately):
ALTER TABLE events DETACH PARTITION events_2023;

-- The benefit of sorted drawers (RANGE):
-- "Show me all cases from 2022-Q1 through 2022-Q2"
-- Only opens 2022-Q1 and Q2 drawers, ignores all others
SELECT * FROM events
WHERE occurred_at BETWEEN '2022-01-01' AND '2022-06-30';
-- EXPLAIN shows: only 2 of 20 partitions scanned
```

---

## Analogy 7: RPO vs RTO — The Hospital Emergency Plan

**Story:**
A hospital has two critical questions for disaster planning: "How old can our records be if we have to rebuild from a backup?" (RPO — Recovery Point Objective) and "How long can the hospital operate without its records before we restore them?" (RTO — Recovery Time Objective).

RPO = "We can tolerate losing the last 5 minutes of patient data" → means you need near-continuous backups (WAL archiving).

RTO = "We need patient records available within 30 minutes of a disaster" → means your restore procedure must complete in 30 minutes (warm standby ready to promote, or fast restore from nearby backup).

A nightly backup has RPO = 24 hours (up to a full day of data loss) and RTO = however long the restore takes (could be hours). This is fine for a test environment but catastrophic for a hospital or financial system.

**Database connection:**
```
Strategy           | RPO              | RTO
──────────────────────────────────────────────────────────
Nightly pg_dump    | up to 24 hours   | hours (full restore)
Daily base + WAL   | ~60 seconds      | hours (full restore)
  archiving        | (archive_timeout)|
Warm standby       | seconds to min   | <1 minute (promote)
  (async replication)|                |
Sync standby       | 0 (zero loss)    | <1 minute (promote)
  (sync replication)| (writes confirmed|
                   | on standby)      |
```

```sql
-- Measure your actual RPO (maximum WAL gap):
SELECT
  now() - last_archived_time AS current_archive_lag,
  last_archived_wal
FROM pg_stat_archiver;

-- Measure your actual RTO readiness (standby lag):
SELECT replay_lag, write_lag FROM pg_stat_replication;
```
