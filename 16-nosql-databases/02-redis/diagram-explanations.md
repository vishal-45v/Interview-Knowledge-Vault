# Chapter 02 — Redis: Diagram Explanations

ASCII diagrams of Redis architecture and data flow. Sketch these on whiteboards.

---

## Diagram 1: Redis Persistence Modes — Durability vs Performance

```
  DURABILITY (data safety on crash)
  HIGH │
       │  AOF always ●
       │              │
       │  AOF everysec ●
       │               │
       │  RDB + AOF hybrid ●
       │                   │
       │  RDB every 60s  ●
       │                 │
       │  RDB every 900s ●
  LOW  │                   No persistence ●
       └───────────────────────────────────────►
                                           PERFORMANCE (writes/sec)

  ┌──────────────────────────────────────────────────────────────┐
  │  RECOVERY TIME (10GB dataset):                              │
  │                                                              │
  │  RDB only:          10-30s  (binary format, fast parse)    │
  │  AOF only:          3-10min (replay all SET/ZADD/etc)      │
  │  RDB+AOF hybrid:    15-45s  (parse RDB + short AOF delta)  │
  │  No persistence:    <1s     (empty keyspace)               │
  └──────────────────────────────────────────────────────────────┘

  AOF FILE STRUCTURE (after BGREWRITEAOF with hybrid mode):
  ┌────────────────────────────────────────────────────────────┐
  │ REDIS0011 ← RDB header                                    │
  │ [binary RDB content — fast parse at startup]              │
  │ ...                                                        │
  │ *3\r\n$3\r\nSET\r\n... ← incremental AOF commands         │
  │ *5\r\n$4\r\nZADD\r\n... ← since last RDB snapshot         │
  └────────────────────────────────────────────────────────────┘
```

**Explanation:** The trade-off between durability and performance is controlled by when Redis
calls `fsync()`. AOF `always` calls `fsync` after every command (max durability, min throughput).
AOF `everysec` batches fsyncs (good balance). RDB relies on fork + background write (crash
window = time since last successful BGSAVE). The hybrid mode gives fast restart by combining
the compact RDB format with a short AOF tail.

---

## Diagram 2: Redis Sentinel Failover Flow

```
  NORMAL OPERATION:
  ┌──────────┐         REPLICATION         ┌──────────┐
  │  MASTER  │ ──────────────────────────► │ REPLICA1 │
  │  :6379   │ ──────────────────────────► │  :6380   │
  └──────────┘                             └──────────┘
       ▲                                   ┌──────────┐
       │ monitor                      ───► │ REPLICA2 │
  ┌────┴─────┐  ┌──────────┐  ┌──────────┐│  :6381   │
  │SENTINEL 1│  │SENTINEL 2│  │SENTINEL 3│└──────────┘
  │  :26379  │  │  :26379  │  │  :26379  │
  └──────────┘  └──────────┘  └──────────┘
  All 3 Sentinels PING master every down-after-milliseconds ms

  MASTER FAILURE DETECTION:
  ┌──────────────────────────────────────────────────────────────┐
  │  Step 1: S1 can't reach master → marks SDOWN (subjective)  │
  │  Step 2: S1 asks S2, S3: "Is master down for you too?"     │
  │  Step 3: S2 and S3 confirm → quorum=2 met → ODOWN          │
  │          (objective down)                                    │
  │  Step 4: S1 calls an election among Sentinels               │
  │  Step 5: Majority (2 of 3) vote S1 as failover leader       │
  │  Step 6: S1 sends SLAVEOF NO ONE to REPLICA1                │
  │  Step 7: S1 sends SLAVEOF new-master to REPLICA2            │
  │  Step 8: S1 publishes "+switch-master" on Pub/Sub channel  │
  │  Step 9: Clients subscribed to Sentinel discover new master │
  └──────────────────────────────────────────────────────────────┘

  POST-FAILOVER:
  ┌──────────┐         REPLICATION         ┌──────────┐
  │ REPLICA1 │ ─────────────────────────── │ REPLICA2 │
  │(NEW MSTR)│ ◄──── promoted ─────────── │  :6381   │
  │  :6380   │                             └──────────┘
  └──────────┘
  Old master (if it comes back) is auto-demoted to replica.
```

**Explanation:** Sentinel uses a two-level quorum: `quorum` Sentinels to *detect* failure,
and majority (`floor(N/2)+1`) of Sentinels to *authorise* the failover. With 3 Sentinels,
quorum=2, you need 2 to detect and 2 to authorise. One Sentinel can die and the system still
functions. This is why 3 is the minimum production Sentinel count.

---

## Diagram 3: Redis Cluster — Hash Slot Distribution and MOVED/ASK Protocol

```
  CLUSTER TOPOLOGY (3 shards, RF=1 for simplicity):
  ┌─────────────────────────────────────────────────────┐
  │  Shard 1 (Node A :7000)  │  Slots 0 – 5460         │
  │  Shard 2 (Node B :7001)  │  Slots 5461 – 10922      │
  │  Shard 3 (Node C :7002)  │  Slots 10923 – 16383     │
  └─────────────────────────────────────────────────────┘

  CLIENT REQUEST: GET user:42
  Step 1: Client computes CRC16("user:42") % 16384 = slot 5649
  Step 2: Client cache says: slot 5649 → Node B :7001
  Step 3: Client sends GET to Node B

  MOVED REDIRECT (slot map is wrong):
  ┌──────┐  GET user:42    ┌────────┐
  │Client│ ──────────────► │ Node A │ ← client has stale map
  │      │ ◄────────────── │ :7000  │
  │      │  MOVED 5649      └────────┘  "slot 5649 is on Node B"
  │      │  127.0.0.1:7001
  │      │ ──────────────► ┌────────┐
  │      │                 │ Node B │ ← correct node
  │      │ ◄────────────── │ :7001  │
  │      │  "alice"         └────────┘
  │      │ (client updates its slot map: 5649 → Node B permanently)
  └──────┘

  ASK REDIRECT (slot is being migrated — do NOT update map):
  Slot 5649 is being migrated from Node B → Node C
  ┌──────┐  GET user:42    ┌────────┐
  │Client│ ──────────────► │ Node B │ ← slot is partially migrated
  │      │ ◄────────────── │ :7001  │
  │      │  ASK 5649        └────────┘  "try Node C but ask first"
  │      │  127.0.0.1:7002
  │      │ ──────────────► ┌────────┐
  │      │  ASKING          │ Node C │
  │      │  GET user:42     │ :7002  │
  │      │ ◄────────────── │        │
  │      │  "alice"         └────────┘
  └──────┘  (client does NOT update slot map — migration may not be complete)
```

**Explanation:** MOVED is a permanent redirect that updates the client's slot cache. ASK is
a temporary redirect during slot migration that instructs the client to send an ASKING command
first (to bypass normal slot ownership checks) and NOT update its map. Clients that handle
MOVED but not ASK will fail during resharding operations.

---

## Diagram 4: Redis Streams Consumer Group Architecture

```
  STREAM: "orders:events"
  ┌────────────────────────────────────────────────────────────────┐
  │  ID           │ Fields                                        │
  ├───────────────┼────────────────────────────────────────────────┤
  │ 1704067200001 │ order_id=101, event=OrderPlaced               │
  │ 1704067200002 │ order_id=102, event=OrderPlaced               │
  │ 1704067200003 │ order_id=101, event=PaymentProcessed          │
  │ 1704067200004 │ order_id=103, event=OrderPlaced               │
  │ 1704067200005 │ order_id=102, event=OrderShipped              │
  └────────────────────────────────────────────────────────────────┘

  CONSUMER GROUP: "order-processor"
  ┌──────────────────────────────────────────────────────────────────┐
  │  last-delivered-id: 1704067200003                               │
  │                                                                  │
  │  Consumers:                                                      │
  │  ┌────────────┬─────────────────────────────────────────────┐  │
  │  │ Consumer   │ PEL (Pending Entry List)                    │  │
  │  ├────────────┼─────────────────────────────────────────────┤  │
  │  │ worker-1   │ [1704067200001 ← delivered, not acked]      │  │
  │  │ worker-2   │ [1704067200002, 1704067200003 ← not acked]  │  │
  │  │ worker-3   │ [] ← empty (all acked or not yet delivered) │  │
  │  └────────────┴─────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────┘

  PROCESSING FLOW:
  worker-1 calls XREADGROUP → receives msg 1704067200001
       │
       ├─ Processing succeeds → XACK → removed from PEL
       └─ Processing fails → stays in PEL → XAUTOCLAIM recovers it

  CONSUMER CRASH RECOVERY:
  worker-2 crashes with 2 messages in PEL
       │
       └─ After PENDING_TIMEOUT ms, another consumer calls XAUTOCLAIM:
          XAUTOCLAIM orders:events order-processor worker-3 30000 0-0 COUNT 100
          ↓
          Messages transferred from worker-2's PEL → worker-3's PEL
          worker-3 reprocesses messages (must be idempotent!)
```

**Explanation:** Consumer groups track which messages have been delivered (the `last-delivered-id`)
and which have been acknowledged (the PEL). Messages in the PEL are "in flight" — delivered but
not yet confirmed processed. XAUTOCLAIM (Redis 7.0) or XCLAIM (older) transfers ownership of
stuck messages to healthy consumers after a timeout. Idempotent processing is essential because
a crashed consumer may have processed a message before crashing but before acking it.

---

## Diagram 5: Redis Sorted Set Internal Structure

```
  ZADD leaderboard 9500 "alice" 8200 "bob" 9900 "carol" 7800 "dave"

  DUAL STRUCTURE:
  ┌─────────────────────────────────────────────────────────────────┐
  │  HASH TABLE (for O(1) score lookup by member)                  │
  │  ┌──────────┬─────────┐                                        │
  │  │ "alice"  │  9500   │  ZSCORE leaderboard "alice" → 9500    │
  │  │ "bob"    │  8200   │  ZRANK leaderboard "bob"   → O(log N) │
  │  │ "carol"  │  9900   │                                        │
  │  │ "dave"   │  7800   │                                        │
  │  └──────────┴─────────┘                                        │
  └─────────────────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────────────────┐
  │  SKIP LIST (for O(log N) range queries)                        │
  │                                                                 │
  │  Level 3: dave(7800) ────────────────────────► carol(9900)    │
  │  Level 2: dave(7800) ─────────► alice(9500) ► carol(9900)    │
  │  Level 1: dave(7800) ► bob(8200) ► alice(9500) ► carol(9900) │
  │                                                                 │
  │  ZRANGE leaderboard 0 -1 (ascending):                         │
  │  Traverse Level 1 from left: dave, bob, alice, carol           │
  │                                                                 │
  │  ZRANGEBYSCORE leaderboard 9000 10000:                        │
  │  Skip to 9000+ using higher levels: alice(9500), carol(9900)  │
  └─────────────────────────────────────────────────────────────────┘

  ENCODING UPGRADE THRESHOLDS:
  ┌──────────────────────────────────────────────────────────────┐
  │  ≤ 128 members AND all members ≤ 64 bytes:                   │
  │  → listpack encoding (cache-friendly, compact)               │
  │                                                              │
  │  > 128 members OR any member > 64 bytes:                    │
  │  → skiplist + hashtable (full structure)                     │
  │                                                              │
  │  Config: zset-max-listpack-entries 128                      │
  │          zset-max-listpack-value   64                       │
  └──────────────────────────────────────────────────────────────┘
```

**Explanation:** Sorted Sets use two data structures simultaneously. The hash table enables
O(1) `ZSCORE` (score lookup by member name). The skip list enables O(log N) range operations
and rank queries. The skip list is kept sorted by score; within the same score, members are
sorted lexicographically. This dual-structure approach gives the best of both worlds at the
cost of storing each member in two places.

---

## Diagram 6: Redis Replication and Partial Resync

```
  MASTER (:6379)
  ┌─────────────────────────────────────────────────────────────┐
  │  replication backlog (circular buffer, default 1MB)        │
  │  ┌──────┬──────┬──────┬──────┬──────┬──────┬──────┐       │
  │  │cmd100│cmd101│cmd102│cmd103│cmd104│cmd105│cmd106│  ...   │
  │  └──────┴──────┴──────┴──────┴──────┴──────┴──────┘       │
  │  first_byte_offset: 100                                     │
  │  master_repl_offset: 106                                    │
  └──────────────────────────────────────────────────────────────┘
          │
          │ Async replication stream
          ▼
  REPLICA (:6380) — connected, at offset 106 ✓

  SCENARIO: REPLICA disconnects at offset 103

  [Replica offline — master continues to offset 250]

  Replica reconnects: sends PSYNC <replica_id> 103
  Master checks: is offset 103 >= first_byte_offset(100)? YES
  ──► PARTIAL SYNC: sends commands 103–250 only (delta)
  ──► Replica catches up quickly ✓

  SCENARIO: REPLICA disconnects at offset 5 (before backlog start)

  Replica reconnects: sends PSYNC <replica_id> 5
  Master checks: is offset 5 >= first_byte_offset(100)? NO
  ──► FULL RESYNC required:
      1. Master calls BGSAVE (forks, writes RDB)
      2. Master streams RDB file to replica (~minutes for large datasets)
      3. Master streams buffered commands since fork started
      4. Replica loads RDB, applies buffered commands
      5. Normal incremental replication resumes

  PREVENTING FULL RESYNC:
  ┌──────────────────────────────────────────────────────────────┐
  │  Set repl-backlog-size to cover max expected outage:        │
  │                                                             │
  │  Write rate (bytes/sec) × expected_outage_seconds           │
  │  = 10MB/s × 60s = 600MB backlog                            │
  │                                                             │
  │  CONFIG SET repl-backlog-size 629145600  # 600MB           │
  └──────────────────────────────────────────────────────────────┘
```

**Explanation:** The replication backlog is a circular buffer of recent write commands. PSYNC
enables partial resync when a replica reconnects within the backlog window. The backlog size
must be tuned to the expected maximum replica downtime multiplied by the write throughput.
A backlog that is too small causes frequent expensive full resyncs, which add significant
network and CPU load on the master (fork for RDB generation).
