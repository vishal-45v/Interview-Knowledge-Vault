# Replication and High Availability — Structured Answers

Complete interview-ready answers with configuration, SQL, and architectural trade-off analysis.

---

## Q1: How does PostgreSQL streaming replication work end-to-end?

**Answer:**

Streaming replication streams WAL records from the primary to standbys in near real-time, allowing standbys to maintain a continuously updated copy of the primary's data.

**Primary configuration:**
```ini
# postgresql.conf on primary
wal_level = replica              # minimum level for streaming replication
max_wal_senders = 10             # max simultaneous WAL sender processes
wal_keep_size = 1024MB           # retain this much WAL if no replication slot
listen_addresses = '*'

# pg_hba.conf on primary
host  replication  replicator  10.0.0.0/8  scram-sha-256
```

**Create replication user:**
```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';
```

**Create standby from base backup:**
```bash
# On standby server:
pg_basebackup -h primary-host -U replicator -D /var/lib/postgresql/data \
  -Fp -Xs -P -R
# -Fp: plain format
# -Xs: stream WAL during backup (ensures no WAL gap)
# -P: show progress
# -R: create postgresql.auto.conf with standby settings
```

**This creates in postgresql.auto.conf on standby:**
```ini
primary_conninfo = 'host=primary-host port=5432 user=replicator ...'
```

**And creates standby.signal file (PostgreSQL 12+)** — presence of this file indicates standby mode.

**Monitoring replication on primary:**
```sql
-- View all connected standbys and their lag
SELECT
  application_name,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  write_lag,
  flush_lag,
  replay_lag,
  sync_state
FROM pg_stat_replication;
```

**Lag types:**
- `write_lag`: Time from primary flush to standby writing WAL to disk
- `flush_lag`: Time from primary flush to standby fsyncing WAL
- `replay_lag`: Time from primary flush to standby applying WAL to data files

**On the standby — verify replication is running:**
```sql
-- On standby
SELECT pg_is_in_recovery();  -- must return true
SELECT pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       pg_last_xact_replay_timestamp();
```

---

## Q2: Describe a complete Patroni HA cluster setup and failover behavior.

**Answer:**

Patroni is a template for PostgreSQL high availability that uses a distributed consensus store (etcd, Consul, or ZooKeeper) for leader election and configuration management.

**Architecture:**
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  node1      │     │  node2      │     │  node3      │
│  (primary)  │────►│  (standby)  │     │  (standby)  │
│  Patroni    │     │  Patroni    │     │  Patroni    │
└─────────────┘     └─────────────┘     └─────────────┘
       │                  │                   │
       └──────────────────┴───────────────────┘
                          │
                    ┌─────────────┐
                    │    etcd     │
                    │  (3 nodes)  │
                    └─────────────┘
```

**patroni.yml configuration:**
```yaml
scope: myapp-cluster
namespace: /service/
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.1:8008

etcd3:
  hosts:
    - etcd1:2379
    - etcd2:2379
    - etcd3:2379

bootstrap:
  dcs:
    ttl: 30                          # leader lease TTL (seconds)
    loop_wait: 10                    # how often Patroni checks health
    retry_timeout: 30
    maximum_lag_on_failover: 1048576 # 1MB max lag for failover candidate

  postgresql:
    use_pg_rewind: true              # use pg_rewind to resync old primary
    parameters:
      wal_level: replica
      hot_standby: on
      wal_log_hints: on              # required for pg_rewind
      max_wal_senders: 10
      max_replication_slots: 10
      synchronous_commit: remote_apply

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.1:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    replication:
      username: replicator
      password: secure_password
    superuser:
      username: postgres
      password: admin_password
```

**Failover sequence:**
1. Primary stops responding to etcd health checks
2. etcd lease expires (after ttl=30 seconds)
3. Patroni on standbys detects missing leader
4. Standby with least replication lag and within maximum_lag_on_failover wins election
5. Winner calls `pg_ctl promote` on its PostgreSQL instance
6. Other standbys connect to new primary via pg_rewind + replication
7. Load balancer detects new primary (via Patroni REST API /primary and /replica endpoints)

**HAProxy configuration for automatic routing:**
```
frontend postgres_write
  bind *:5000
  default_backend postgres_primary

backend postgres_primary
  option httpchk GET /primary
  server node1 10.0.1.1:5432 check port 8008
  server node2 10.0.1.2:5432 check port 8008
  server node3 10.0.1.3:5432 check port 8008

frontend postgres_read
  bind *:5001
  default_backend postgres_replicas

backend postgres_replicas
  balance roundrobin
  option httpchk GET /replica
  server node1 10.0.1.1:5432 check port 8008
  server node2 10.0.1.2:5432 check port 8008
  server node3 10.0.1.3:5432 check port 8008
```

---

## Q3: How do you implement and monitor a complete PITR backup strategy?

**Answer:**

Point-in-Time Recovery requires a base backup plus a complete unbroken WAL archive.

**Using pgBackRest (recommended tool):**
```ini
# /etc/pgbackrest/pgbackrest.conf

[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=4          # Keep 4 full backups
repo1-retention-diff=14         # Keep 14 differential backups
repo1-cipher-type=aes-256-cbc   # Encrypt backups
repo1-cipher-pass=secret_key

# S3 repository:
repo2-type=s3
repo2-path=/pgbackrest
repo2-s3-bucket=my-postgres-backups
repo2-s3-region=us-east-1
repo2-retention-full=30

log-level-console=info
log-level-file=detail

[myapp]
pg1-path=/var/lib/postgresql/data
pg1-port=5432
pg1-user=postgres
```

**postgresql.conf for archiving:**
```ini
archive_mode = on
archive_command = 'pgbackrest --stanza=myapp archive-push %p'
archive_timeout = 60            # Force WAL archive every 60 seconds even if not full
```

**Backup schedule (cron):**
```bash
# Full backup weekly (Sunday 1 AM)
0 1 * * 0 pgbackrest --stanza=myapp --type=full backup

# Differential backup daily (Mon-Sat 1 AM)
0 1 * * 1-6 pgbackrest --stanza=myapp --type=diff backup

# Continuous WAL archiving via archive_command (automatic)
```

**Performing a PITR restore:**
```bash
# Restore to specific point in time
pgbackrest --stanza=myapp --delta \
           --target-action=promote \
           --type=time \
           --target="2024-06-15 14:30:00 UTC" \
           restore

# Start PostgreSQL — it will replay WAL to the target time
pg_ctl start -D /var/lib/postgresql/data
```

**Monitoring archive completeness:**
```sql
-- On primary: check last successfully archived WAL
SELECT last_archived_wal, last_archived_time,
       last_failed_wal, last_failed_time,
       archived_count, failed_count
FROM pg_stat_archiver;

-- Alert condition: last_failed_time > last_archived_time
-- (meaning the last archive attempt failed)
```

**RPO/RTO calculation:**
- **RPO**: archive_timeout (max 60 seconds of data loss if a WAL segment is partially filled)
- **RTO**: restore time = base backup download time + WAL replay time
  - For a 500GB database on fast network: ~30 min download + ~5 min WAL replay
  - Improve RTO: keep a warm standby that already has recent base backup

---

## Q4: Explain table partitioning strategies with trade-off analysis.

**Answer:**

**RANGE partitioning (most common — time-series data):**
```sql
CREATE TABLE events (
  id BIGSERIAL,
  occurred_at TIMESTAMPTZ NOT NULL,
  event_type TEXT,
  user_id BIGINT
) PARTITION BY RANGE (occurred_at);

-- Create monthly partitions
CREATE TABLE events_2024_01 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
-- ... etc

-- Automatic partition creation with pg_partman extension
```

Trade-offs:
- Excellent partition pruning for date range queries
- Fast DROP TABLE for old partitions (data archival: O(1) vs DELETE: O(n))
- Poor for queries that span all dates (no pruning possible)
- Adding new partitions requires DDL (automate with pg_partman)

**LIST partitioning (categorical data):**
```sql
CREATE TABLE orders (
  id BIGSERIAL,
  region TEXT NOT NULL,
  customer_id BIGINT,
  total NUMERIC
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('US', 'CA', 'MX');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('DE', 'FR', 'GB', 'IT');
CREATE TABLE orders_apac PARTITION OF orders FOR VALUES IN ('JP', 'AU', 'SG');
```

Trade-offs:
- Good for routing data to specific tablespaces (EU data in EU region)
- Good for per-region compliance isolation
- Hard to rebalance if distribution changes
- Unknown values require a DEFAULT partition

**HASH partitioning (even distribution):**
```sql
CREATE TABLE user_sessions (
  id BIGSERIAL,
  user_id BIGINT NOT NULL,
  token TEXT,
  expires_at TIMESTAMPTZ
) PARTITION BY HASH (user_id);

CREATE TABLE user_sessions_0 PARTITION OF user_sessions
  FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE user_sessions_1 PARTITION OF user_sessions
  FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE user_sessions_2 PARTITION OF user_sessions
  FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE user_sessions_3 PARTITION OF user_sessions
  FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

Trade-offs:
- Even distribution prevents hot partitions
- NO partition pruning for range queries (hash is not ordered)
- Useful for spreading I/O across tablespaces/disks
- Hard to expand (changing modulus requires full rebuild)

**Indexes on partitioned tables:**
```sql
-- Indexes must be created on the parent table (PG11+) or each partition (PG10)
CREATE INDEX CONCURRENTLY idx_events_user_id ON events (user_id);
-- This automatically creates an index on each partition

-- Check all partition indexes:
SELECT indexrelid::regclass, indrelid::regclass
FROM pg_index
WHERE indrelid IN (
  SELECT inhrelid FROM pg_inherits WHERE inhparent = 'events'::regclass
);
```

---

## Q5: Design a backup strategy for a 5TB database with RPO < 5 minutes and RTO < 30 minutes.

**Answer:**

```
Architecture:
─────────────────────────────────────────────────────────
  Primary PostgreSQL (5TB)
    │
    ├── Streaming replication ──────► Warm Standby (1 minute lag max)
    │                                  (ready to promote in seconds)
    │
    └── WAL Archiving ───────────────► S3 (archive_timeout = 60s)
                                       (WAL segments every ~1 minute)

  Base Backup Schedule:
    Every 6 hours: pgBackRest incremental backup to S3
    Weekly: pgBackRest full backup to S3

  Restore Path for RTO < 30 minutes:
  ────────────────────────────────────
  Option A (preferred): Promote warm standby
    - Time: 30 seconds (promote standby, update DNS/VIP)
    - RPO: equals current replication lag (< 1 minute async)

  Option B (fallback): Restore from S3
    - Latest base backup (6 hours old) + WAL replay to now
    - Time estimate: 5TB @ 1GB/s = 85 minutes download → does NOT meet 30 min RTO
    - Improvement: Pre-restore incremental backup reduces download to ~100GB
    - Time with incremental: ~2 min download + 5 min replay ≈ 7 min total
```

```ini
# postgresql.conf critical settings
archive_mode = on
archive_command = 'pgbackrest --stanza=prod archive-push %p'
archive_timeout = 60          # Ensure WAL archived every 60 seconds

# Replication for warm standby
wal_level = replica
max_wal_senders = 5
```

```bash
# pgBackRest schedule:
# Full backup: weekly Sunday 1 AM
0 1 * * 0 pgbackrest --stanza=prod --type=full backup

# Incremental: every 6 hours
0 */6 * * * pgbackrest --stanza=prod --type=incr backup

# Verify backup integrity weekly:
0 3 * * 1 pgbackrest --stanza=prod verify
```

**Monitoring checklist:**
```sql
-- Archive health check (run every 5 minutes)
SELECT
  CASE
    WHEN last_failed_time > last_archived_time
      THEN 'CRITICAL: Archive failing'
    WHEN EXTRACT(EPOCH FROM (now() - last_archived_time)) > 120
      THEN 'WARNING: Archive stale > 2 min'
    ELSE 'OK'
  END AS archive_status,
  last_archived_wal,
  last_archived_time,
  last_failed_wal,
  last_failed_time
FROM pg_stat_archiver;
```

**Restore testing (monthly):**
```bash
# Spin up a test instance, restore to yesterday 3 PM, verify data:
pgbackrest --stanza=prod \
           --target="$(date -d yesterday '+%Y-%m-%d') 15:00:00" \
           --target-action=promote \
           --type=time \
           restore

pg_ctl start -D /tmp/restore-test
psql -c "SELECT COUNT(*) FROM orders" /tmp/restore-test  # verify record count
pg_ctl stop -D /tmp/restore-test
rm -rf /tmp/restore-test
```
