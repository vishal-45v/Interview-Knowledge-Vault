# PostgreSQL Internals — Analogy Explanations

Conceptual analogies for complex internals topics. Each analogy includes a story, the database concept it maps to, and connection back to real SQL/config.

---

## Analogy 1: MVCC — The Library with Laminated Pages

**Story:**
Imagine a library where books can never be erased. When an editor wants to "update" a page, they don't erase the old text — they laminate the old page, write a date on the cover saying "superseded on Tuesday," and insert the new page next to it. Readers who walked in on Monday still read the old laminated page. Readers who walked in on Wednesday read the new page. The librarian (VACUUM) periodically collects all laminated pages that are old enough that nobody who is currently in the library could possibly be reading them, and shreds them.

**Database connection:**
This is exactly PostgreSQL MVCC. The "laminated pages" are dead tuple versions still in the heap. xmin is the "inserted on" date, xmax is the "superseded on" date. Your transaction snapshot determines which version you read. VACUUM is the librarian collecting old versions.

```sql
-- See both versions of a row during an UPDATE
BEGIN;
UPDATE products SET price = 99.99 WHERE id = 1;

-- In another session, the old version is still visible:
SELECT xmin, xmax, price FROM products WHERE id = 1;
-- Shows: xmin=100, xmax=150, price=49.99  (old, dead but not vacuumed)
--   AND: xmin=150, xmax=0,   price=99.99  (new, live)
COMMIT;
```

**Why it matters:** Unlike Oracle's undo segments, PostgreSQL's "laminated pages" live in the same table heap. This means table scans must skip over dead tuples, causing bloat and scan slowdowns — the cost of PostgreSQL's simpler crash recovery architecture.

---

## Analogy 2: WAL — The Accountant's Journal

**Story:**
A bank never modifies the main ledger directly. First, every transaction is written in a sequential journal: "At 2:34 PM, deducted $500 from account 1042, added $500 to account 2087." Only later is the main ledger updated. If the office catches fire mid-afternoon, the main ledger might be partially inconsistent — but the journal is intact, written sequentially on fireproof paper. Recovery means reading the journal from the last known good state and replaying each entry.

**Database connection:**
WAL is the journal. The heap files are the main ledger. fsync on the journal (WAL) is what makes PostgreSQL crash-safe — not fsync on the data files (which happens lazily during checkpoints). The checkpoint is the moment you say "the main ledger is fully caught up to journal entry #47,823."

```ini
# The "fireproof paper" guarantee:
fsync = on                      # WAL records flushed before commit ACK
synchronous_commit = on         # Extra guarantee: WAL written before client gets "OK"

# How often to "reconcile" journal to main ledger:
checkpoint_timeout = 10min      # At minimum every 10 minutes
max_wal_size = 8GB              # Or when journal reaches 8GB
```

**Why it matters:** Recovery time is bounded by the amount of WAL generated since the last checkpoint. Increasing max_wal_size reduces checkpoint I/O but increases recovery time — a real trade-off with measurable impact on RTO.

---

## Analogy 3: VACUUM — The Office Janitor Who Can't Move Furniture

**Story:**
Imagine an office floor where desks represent live data and the space between desks is dead — previously occupied but now cleared. The janitor (VACUUM) can sweep up the empty spaces and mark them as "available" on a floor map so new employees know where to sit. However, the janitor cannot actually shrink the floor or move the office walls in — the building lease remains the same size. To actually reduce the office footprint (return space to the landlord), you need a special "full renovation" (VACUUM FULL) that requires moving everyone out temporarily.

**Database connection:**
Regular VACUUM updates the FSM (the floor map of available space). It does NOT shrink the heap file on disk. VACUUM FULL rewrites the table into a new file and drops the old one — but requires ACCESS EXCLUSIVE lock (everyone moves out temporarily). pg_repack is like a renovation service that builds the new floor next to the old one without evicting anyone.

```sql
-- Regular VACUUM: sweep and mark free space (no table lock)
VACUUM orders;

-- VACUUM FULL: rewrite table, return disk space (full table lock)
VACUUM FULL orders;

-- Check if bloat is severe enough to justify VACUUM FULL
SELECT relname,
       pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
       pg_size_pretty(pg_relation_size(oid)) AS table_size
FROM pg_class
WHERE relkind = 'r'
ORDER BY pg_total_relation_size(oid) DESC LIMIT 10;
```

---

## Analogy 4: shared_buffers vs OS Page Cache — The Double Pantry

**Story:**
A restaurant has two storage areas for ingredients: a pantry inside the kitchen (shared_buffers) and a large walk-in refrigerator in the hallway (OS page cache). When a chef needs an ingredient, they first check the kitchen pantry. If it's not there, they go to the refrigerator. Frequently used ingredients end up in both places — the pantry has the prep-ready version, the refrigerator has the raw version. The restaurant manager (the planner) needs to know the combined capacity of both storage areas to decide whether to buy ingredients in bulk (sequential scan) or cherry-pick individual items (index scan).

**Database connection:**
shared_buffers is the kitchen pantry — PostgreSQL-managed, uses shared memory. The OS page cache is the walk-in fridge — kernel-managed. Data can live in both simultaneously. effective_cache_size is the manager's estimate of both combined — it does NOT allocate memory, it is a planning hint.

```ini
# Server with 128GB RAM:
shared_buffers = 32GB           # 25% of RAM, explicit PostgreSQL cache
effective_cache_size = 96GB     # 75% of RAM = shared_buffers + available OS cache
                                # Tells planner: "assume 96GB of data is cached"
                                # Planner uses this to decide index vs seqscan cost
work_mem = 256MB                # Per-operation sort/hash memory (kitchen counter space)
```

---

## Analogy 5: Checkpoint — The Evening Newspaper Press

**Story:**
A newspaper has reporters (backend processes) writing articles all day to scratch pads (WAL buffers). At the end of the day, the press operator (checkpoint process) prints everything from all the scratch pads into the actual newspaper (heap files). The press run happens continuously and is spread over the last 2 hours of the day (checkpoint_completion_target) to avoid all printers running simultaneously. If the building burns down mid-afternoon, the scratch pads are fireproof — the press operator can reconstruct everything from the last completed newspaper edition plus all the scratch pads since then.

**Database connection:**
Checkpoints write dirty shared_buffers to heap files. checkpoint_completion_target spreads this I/O to avoid spikes. The "last newspaper edition" is the previous checkpoint record in WAL. Recovery time = time to replay WAL from last checkpoint.

```ini
checkpoint_completion_target = 0.9    # Spread writes over 90% of checkpoint interval
checkpoint_timeout = 15min            # Maximum interval between checkpoints
max_wal_size = 16GB                   # Force checkpoint if WAL exceeds 16GB
log_checkpoints = on                  # Log to observe checkpoint frequency and duration
```

---

## Analogy 6: TOAST — The Airport Luggage System

**Story:**
An airline counter has a standard check-in queue for passengers with carry-on bags (small values). But passengers with oversized luggage (large values) are redirected to a special oversized baggage counter around the corner. The boarding pass (main tuple) has a claim stub pointing to where the big bag is stored. When the passenger lands, they can pick up their bag using the stub — the whole process is transparent from the passenger's perspective. But if thousands of passengers simultaneously go to retrieve oversized bags, the special counter becomes a bottleneck that the standard queue doesn't have.

**Database connection:**
TOAST is the oversized baggage system. Values over ~2KB are compressed and/or stored out-of-line in a separate pg_toast_<oid> table. The main tuple stores a pointer. Fetching TOASTed columns requires accessing the TOAST table — transparent but costly at scale.

```sql
-- See TOAST table for a relation
SELECT relname, reltoastrelid,
       (SELECT relname FROM pg_class WHERE oid = reltoastrelid) AS toast_table
FROM pg_class
WHERE relname = 'documents';

-- Check TOAST storage strategy per column
SELECT attname, atttypid::regtype, attstorage
FROM pg_attribute
WHERE attrelid = 'documents'::regclass
  AND attnum > 0;
-- attstorage: p=PLAIN, e=EXTERNAL, x=EXTENDED (compress+toast), m=MAIN
```

---

## Analogy 7: Connection Pooling with PgBouncer — The Receptionist Switchboard

**Story:**
A law firm has 500 clients who all want to speak with lawyers. The firm only has 20 lawyers. Without a receptionist, 500 clients would storm the building simultaneously, and most lawyers would spend all their time on hold rather than doing legal work. With a switchboard operator (PgBouncer), clients call the operator, who queues them and connects each call to a lawyer only when the lawyer is free. The client thinks they have a direct line to a lawyer at all times. The lawyers only handle one client at a time but are always productive. In session mode, one client monopolizes a lawyer for the entire appointment even during silence. In transaction mode, the lawyer can serve a different client between each question — far more efficient.

**Database connection:**
PgBouncer pools application connections to a smaller set of backend PostgreSQL processes. Transaction mode (recommended for most OLTP) returns the connection to the pool after each transaction, allowing much higher concurrency.

```ini
# pgbouncer.ini
[databases]
myapp = host=localhost port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction            # Return connection after each transaction
max_client_conn = 1000             # Allow 1000 application connections
default_pool_size = 25             # Only 25 actual PostgreSQL backends
min_pool_size = 5
reserve_pool_size = 5              # Emergency overflow connections
server_idle_timeout = 600          # Close idle backend connections after 10min
```

**Limitation of transaction mode:** Cannot use session-level features like `SET LOCAL`, advisory locks, or `LISTEN/NOTIFY` — the session state is not guaranteed to be the same connection on the next transaction.

---

## Analogy 8: autovacuum Cost-Based Delay — The Polite Street Sweeper

**Story:**
A city employs a street sweeper that runs continuously but is programmed to stop and rest whenever traffic gets heavy, to avoid blocking cars. The sweeper has a "work budget" per block (cost_limit). Sweeping a clean road costs 1 credit. Sweeping a dirty road costs 10. Washing down a particularly messy section costs 20. Once the budget is used up, the sweeper parks for 20 milliseconds (cost_delay) before resuming. In a quiet neighborhood, the sweeper runs almost continuously. On a busy highway, it pauses frequently — perhaps too frequently for the mess to be managed.

**Database connection:**
Autovacuum uses the same budget system. If autovacuum is too "polite" (high cost_delay, low cost_limit), it cannot keep up with dead tuple accumulation on busy tables. On fast NVMe storage, reducing cost_delay and increasing cost_limit allows autovacuum to be more aggressive without hurting OLTP performance.

```sql
-- Per-table autovacuum tuning for a high-write table
ALTER TABLE order_events SET (
  autovacuum_vacuum_cost_delay  = 2,     -- 2ms pause (default 20ms)
  autovacuum_vacuum_cost_limit  = 800,   -- larger budget (default 200)
  autovacuum_vacuum_scale_factor = 0.01  -- trigger at 1% dead tuples (default 20%)
);

-- Monitor autovacuum activity in real time
SELECT schemaname, relname, last_autovacuum, last_autoanalyze,
       n_dead_tup, autovacuum_count
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;
```
