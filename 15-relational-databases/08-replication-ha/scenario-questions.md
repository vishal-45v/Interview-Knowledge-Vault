# Replication and High Availability — Scenario Questions

Real-world production situations at the senior engineer / architect level.

---

1. Your primary PostgreSQL database (1TB, streaming replication to 2 standbys) undergoes an ungraceful failover at 2:47 AM. The DBA promotes standby-1, but your monitoring shows replication_lag was 38 seconds at time of failure. Application engineers ask: "How much data was lost?" Walk through exactly how you calculate the data loss, how you verify it, whether the lost transactions can be recovered from WAL archives, and what you tell the business stakeholders.

2. You have a 3-node Patroni cluster (1 primary, 2 standbys) with etcd for consensus. During a network partition, the primary loses connectivity to etcd but can still communicate with the application. Your monitoring shows the primary is still accepting writes. The standbys and etcd are on the same network segment and can communicate with each other. Explain exactly what Patroni should do in this scenario, what "fencing" means and why it matters, and what happens to writes that occurred on the old primary after the partition heals.

3. A product team wants to run heavy analytical queries (30–60 second runtime, full table scans of 500M-row tables) against the production database. They propose "just adding a read replica." Explain: what hot standby conflicts are, how these long analytical queries would affect the primary through the replica, what hot_standby_feedback does and its trade-offs, and the better architectural recommendation (dedicated analytics replica with different VACUUM settings, read replica in a different AZ, etc.).

4. Your PostgreSQL primary server has a replication slot named `logical_consumer_slot` that was created for a Debezium CDC (Change Data Capture) pipeline. The Debezium consumer was taken down 3 weeks ago for a "quick maintenance" and never restarted. You receive a PagerDuty alert: the primary disk is 95% full. Upon investigation, pg_replication_slots shows the slot has consumed 800GB of WAL. Explain exactly what happened, how to remediate immediately, and what alerting/monitoring you put in place to prevent recurrence.

5. Your company needs a PostgreSQL backup strategy meeting these SLAs: RPO < 5 minutes, RTO < 30 minutes, 30-day retention, and all backups must be stored off-site (S3). Design the complete backup architecture including: tool selection (pg_dump vs pg_basebackup vs WAL-E/pgBackRest), frequency of base backups, WAL archiving configuration, restore testing procedure, and how you verify backup integrity without a full restore test.

6. You are migrating a 2TB PostgreSQL 13 database to PostgreSQL 16 with zero downtime (the application cannot go down for more than 30 seconds). Describe the complete migration strategy using logical replication, including: publication/subscription setup, schema migration approach, handling of sequences, the cutover procedure, and how you verify data consistency after the cutover.

7. A multi-tenant SaaS application stores all tenants in a single PostgreSQL database. Tenant "Acme Corp" has 80% of the data (600M rows in their partitions). Query performance for Acme is degrading because their partition is being vacuumed every few minutes (hitting threshold), consuming significant I/O that affects all other tenants. Design a multi-tenant partitioning strategy that isolates Acme's data, allows per-tenant autovacuum tuning, and can support eventual migration of large tenants to dedicated databases.

8. Your application uses PostgreSQL with PgBouncer (transaction mode, pool_size=30). After deploying a new feature that uses LISTEN/NOTIFY for real-time updates, you discover the feature works locally but is completely broken in production — clients never receive notifications. Diagnose the root cause (LISTEN requires persistent session-level connections), explain why PgBouncer transaction mode breaks it, and propose two alternative architectures that would work.

9. A financial services company requires that no committed transaction ever be lost (RPO = 0). They currently use asynchronous streaming replication. Design the complete HA configuration that guarantees RPO = 0, including: synchronous_commit settings, synchronous_standby_names configuration with quorum, the impact on write latency and throughput, how the system behaves when the synchronous standby is down, and the monitoring required to detect when the guarantee is being temporarily violated.

10. You need to scale PostgreSQL reads for a global application with users in US, EU, and Asia-Pacific. The primary is in us-east-1. Design a multi-region read replica architecture including: cross-region WAL streaming configuration, application-level read routing (primary for writes, nearest replica for reads), consistency handling (what happens when a user writes and immediately reads from the replica), and monitoring for per-region replication lag.
