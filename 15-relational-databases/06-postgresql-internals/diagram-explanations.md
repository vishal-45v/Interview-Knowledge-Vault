# PostgreSQL Internals — Diagram Explanations

ASCII diagrams for visual learners covering the most important structural and process concepts.

---

## Diagram 1: PostgreSQL 8KB Heap Page Layout

```
┌─────────────────────────────────────────────────────────────────┐
│                     PAGE HEADER (24 bytes)                      │
│  pd_lsn (8)  pd_checksum (2)  pd_flags (2)                      │
│  pd_lower (2) pd_upper (2)  pd_special (2)  pd_pagesize_version │
├─────────────────────────────────────────────────────────────────┤
│            LINE POINTER ARRAY (grows downward ↓)                │
│  LP[1]: offset=8160, len=42, flags=NORMAL                        │
│  LP[2]: offset=8110, len=50, flags=NORMAL                        │
│  LP[3]: offset=0,    len=0,  flags=DEAD    ← recycled slot       │
│  LP[4]: offset=8060, len=48, flags=NORMAL                        │
│  ...                                                             │
│           pd_lower ──┐                                           │
├───────────────────────┼─────────────────────────────────────────┤
│     FREE SPACE        │  (pd_upper - pd_lower = available bytes) │
│           pd_upper ──┼─────────────────────────────────────────┤
│                       ↓                                          │
│            TUPLE DATA (grows upward ↑)                           │
│  Tuple 4: [xmin|xmax|cmin|cmax|null_bitmap|t_data]              │
│  Tuple 2: [xmin|xmax|cmin|cmax|null_bitmap|t_data]              │
│  Tuple 1: [xmin|xmax|cmin|cmax|null_bitmap|t_data]              │
├─────────────────────────────────────────────────────────────────┤
│                  SPECIAL AREA (index-specific)                   │
└─────────────────────────────────────────────────────────────────┘
  Total size: 8192 bytes (default, compiled at build time)
```

**Explanation:**
- pd_lower points to end of the line pointer array. pd_upper points to start of tuple data. Free space is the gap between them.
- Line pointers provide stable tuple addressing — when a tuple is moved (by HOT update), the LP entry is updated, not external indexes.
- System columns (xmin, xmax, cmin, cmax) are in every tuple header, enabling MVCC without separate undo storage.

---

## Diagram 2: MVCC Tuple Versions During UPDATE

```
  Transaction 100 INSERTs row:
  ┌──────────────────────────────────────┐
  │  xmin=100 │ xmax=0 │ name='Alice'   │  ← LIVE (visible to TXN >= 100)
  └──────────────────────────────────────┘

  Transaction 150 UPDATEs name to 'Alicia':
  ┌──────────────────────────────────────┐
  │  xmin=100 │ xmax=150 │ name='Alice' │  ← DEAD (xmax set, invisible after TXN 150)
  └──────────────────────────────────────┘
  ┌──────────────────────────────────────┐
  │  xmin=150 │ xmax=0   │ name='Alicia'│  ← LIVE (visible to TXN >= 150)
  └──────────────────────────────────────┘

  Snapshot at TXN 120 sees:  'Alice'   (xmin=100 < 120, xmax=150 > 120)
  Snapshot at TXN 160 sees:  'Alicia'  (xmin=150 < 160, xmax=0 = still live)
  Snapshot at TXN 80  sees:  nothing   (xmin=100 > 80, row didn't exist yet)

  After VACUUM runs (no TXN < 150 still open):
  ┌──────────────────────────────────────┐
  │  xmin=100 │ xmax=150 │ name='Alice' │  ← REMOVED, space reclaimed in FSM
  └──────────────────────────────────────┘
  ┌──────────────────────────────────────┐
  │  xmin=150 │ xmax=0   │ name='Alicia'│  ← Still LIVE
  └──────────────────────────────────────┘
```

**Explanation:**
Each UPDATE creates a new physical row version. Old versions accumulate until VACUUM determines they are invisible to all open transactions. This is why heavy UPDATE workloads bloat PostgreSQL tables without regular VACUUM.

---

## Diagram 3: WAL and Checkpoint Flow

```
  Client writes                WAL Buffers             WAL Segment Files
  ─────────────               ──────────────          ─────────────────────
  INSERT/UPDATE/DELETE  ──►   [WAL record 1]  ──►    000000010000000000000001
  (modifies shared_buffers)   [WAL record 2]          000000010000000000000002
                              [WAL record 3]  ──►    000000010000000000000003
                                    │
                              on COMMIT + fsync()
                                    │
                                    ▼
                           WAL flushed to disk ──► client gets ACK

  Background Writer / Checkpoint
  ──────────────────────────────────────────────────────────────────
  shared_buffers (dirty pages):
  ┌────────┬────────┬────────┬────────┬────────┐
  │ Page A │ Page B │ Page C │ Page D │ Page E │  ← dirty pages
  └────────┴────────┴────────┴────────┴────────┘
       │         │        │
       ▼         ▼        ▼
  Heap files (on disk):   written lazily by bgwriter, or urgently by checkpoint
  ┌────────┬────────┬────────┐
  │orders  │orders  │orders  │
  │.0001   │.0002   │.0003   │
  └────────┴────────┴────────┘
                │
                ▼
       CHECKPOINT RECORD written to WAL
       pg_control updated: "safe to recover from LSN X"

  Crash Recovery:
  ──────────────
  1. Read pg_control → find last checkpoint LSN
  2. Replay all WAL records from checkpoint LSN to end of WAL
  3. Database is consistent → accept connections
```

**Explanation:**
WAL is written before heap files as the durability guarantee. Checkpoints bound recovery time. max_wal_size controls how much WAL can accumulate between checkpoints, which directly controls worst-case recovery time.

---

## Diagram 4: PostgreSQL Process Architecture

```
  ┌──────────────────────────────────────────────────────────────────┐
  │                        OS / Hardware                             │
  ├──────────────────────────────────────────────────────────────────┤
  │  ┌─────────────────────────────────────────────────────────┐     │
  │  │               SHARED MEMORY                             │     │
  │  │  ┌──────────────────┐  ┌──────────┐  ┌─────────────┐  │     │
  │  │  │  shared_buffers  │  │WAL bufs  │  │ Lock table  │  │     │
  │  │  │  (buffer pool)   │  │(wal_bufs)│  │(pg_locks)   │  │     │
  │  │  └──────────────────┘  └──────────┘  └─────────────┘  │     │
  │  └─────────────────────────────────────────────────────────┘     │
  │                                                                   │
  │  Postmaster (PID 1 of cluster)                                    │
  │  ├── Background Writer (bgwriter)  ← writes dirty pages lazily   │
  │  ├── Checkpointer                  ← runs checkpoint process      │
  │  ├── WAL Writer                    ← flushes WAL buffers          │
  │  ├── Autovacuum Launcher           ← spawns autovacuum workers    │
  │  │   ├── Autovacuum Worker 1                                      │
  │  │   ├── Autovacuum Worker 2                                      │
  │  │   └── Autovacuum Worker 3                                      │
  │  ├── Stats Collector               ← updates pg_stat_* views      │
  │  ├── WAL Sender (replication)      ← streams WAL to standby       │
  │  │                                                                 │
  │  ├── Backend PID 1234  ← one OS process per client connection     │
  │  ├── Backend PID 1235                                             │
  │  ├── Backend PID 1236                                             │
  │  └── ... (up to max_connections)                                  │
  └──────────────────────────────────────────────────────────────────┘

  Client connections:
  App1 ──────────────────────────────────────► Backend PID 1234
  App2 ──────────────────────────────────────► Backend PID 1235
  App3 ─────────► PgBouncer pool ─────────────► Backend PID 1234 (reused)
  App4 ─────────► PgBouncer pool ─────────────► Backend PID 1235 (reused)
```

**Explanation:**
Each client connection spawns a separate OS process (not a thread), sharing the shared memory segment. This architecture simplifies memory isolation but creates overhead per connection — motivating connection pooling. All background processes (bgwriter, checkpointer, autovacuum workers) also run as separate OS processes.

---

## Diagram 5: TOAST Storage Architecture

```
  Main table: documents (oid = 16400)
  ┌────┬──────────────────────────────────────────────────────────┐
  │ id │  title (TEXT, small)  │  body (TEXT, large → TOAST)       │
  ├────┼───────────────────────┼──────────────────────────────────┤
  │  1 │  "Q3 Report"          │  [TOAST pointer: chunk_id=4200]   │
  │  2 │  "Budget 2024"        │  [TOAST pointer: chunk_id=4201]   │
  │  3 │  "Short note"         │  "Just a brief note"  ← fits!     │
  └────┴───────────────────────┴──────────────────────────────────┘
        ↑ heap page (8KB)                ↑ inline value           ↑ pointer

  TOAST table: pg_toast_16400
  ┌──────────────┬───────────────┬────────────────────────────────┐
  │ chunk_id     │ chunk_seq     │ chunk_data                      │
  ├──────────────┼───────────────┼────────────────────────────────┤
  │ 4200         │ 0             │ [first 2000 bytes of body doc1] │
  │ 4200         │ 1             │ [next  2000 bytes of body doc1] │
  │ 4200         │ 2             │ [final 1500 bytes of body doc1] │
  │ 4201         │ 0             │ [first 2000 bytes of body doc2] │
  └──────────────┴───────────────┴────────────────────────────────┘

  Storage strategies (per column, set via ALTER TABLE):
  ┌──────────┬───────────────┬──────────────────────────────────────┐
  │ Strategy │ Compress?     │ Out-of-line storage?                  │
  ├──────────┼───────────────┼──────────────────────────────────────┤
  │ PLAIN    │ Never         │ Never (error if value > page)         │
  │ EXTENDED │ Try first     │ If still > threshold after compress   │
  │ EXTERNAL │ Never         │ Yes (raw bytes out-of-line)           │
  │ MAIN     │ Try first     │ Only as last resort                   │
  └──────────┴───────────────┴──────────────────────────────────────┘
```

**Explanation:**
TOAST chunks are stored 2KB each in a separate table. Fetching a TOASTed value requires accessing the TOAST table with multiple heap reads — one per chunk. For a 10KB value split into 5 chunks, a single row fetch requires 5 extra heap page reads. On a query returning 10,000 rows with large text columns, this is 50,000 extra heap reads.

---

## Diagram 6: Free Space Map (FSM) and Visibility Map (VM)

```
  Heap file: orders (3 pages shown)
  ┌──────────────────────────┐
  │  Page 0  (8KB)           │
  │  [T1][T2][dead][T4][   ] │  ← some dead tuples, some free space
  │  Free: 1800 bytes        │
  └──────────────────────────┘
  ┌──────────────────────────┐
  │  Page 1  (8KB)           │
  │  [T5][T6][T7][T8][T9]   │  ← fully packed, all live tuples
  │  Free: 200 bytes         │
  └──────────────────────────┘
  ┌──────────────────────────┐
  │  Page 2  (8KB)           │
  │  [T10][T11][T12]         │  ← all live, all visible to everyone
  │  Free: 3600 bytes        │
  └──────────────────────────┘

  FSM (orders_fsm): tracks free space per page
  ┌────────┬─────────────────────────────────────────┐
  │ Page 0 │ ████████████░░░░░░░░░  (1800 bytes free)│
  │ Page 1 │ ███████████████████░  ( 200 bytes free) │
  │ Page 2 │ ████████░░░░░░░░░░░░  (3600 bytes free) │
  └────────┴─────────────────────────────────────────┘
  Purpose: INSERT scans FSM to find pages with enough space (avoids full table scan)

  VM (orders_vm): 2 bits per page
  ┌────────┬──────────────┬──────────────────────────────────────────┐
  │ Page 0 │ all_visible=0│ Has dead tuples or recently modified rows │
  │ Page 1 │ all_visible=0│ May have MVCC-invisible tuples            │
  │ Page 2 │ all_visible=1│ All tuples visible to ALL transactions    │
  └────────┴──────────────┴──────────────────────────────────────────┘
  Purpose:
  - VACUUM uses VM to skip pages where all_visible=1 (no dead tuples)
  - Index-only scans use VM: if all_visible=1, heap access skipped entirely
  - VACUUM sets all_visible bits; any update/delete clears the bit

  Index-only scan on Page 2:
  ┌────────────────────┐              ┌──────────────────┐
  │  B-tree index      │ ── lookup ──►│ VM bit = 1?      │
  │  key=50 → Page 2   │              │ YES → return from│
  │                    │              │ index entry only  │
  └────────────────────┘              └──────────────────┘
                                      (heap page NOT accessed)
```

**Explanation:**
The FSM and VM are companion files stored alongside the heap (orders, orders_fsm, orders_vm on disk). VACUUM maintains both. The VM's all_visible bits directly enable the index-only scan optimization — one of the highest-impact performance features for read-heavy workloads. A table that never gets VACUUMed will have no all_visible bits set, forcing every index-only scan to access the heap.
