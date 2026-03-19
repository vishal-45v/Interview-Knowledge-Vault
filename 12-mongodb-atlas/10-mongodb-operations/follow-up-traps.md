# Chapter 10 — MongoDB Operations: Follow-Up Traps

---

## Trap 1: "updateMany Is Always Safe for Large Updates"

**Wrong answer:** "I'll just run `db.collection.updateMany({}, { $set: { region: 'US' } })` to update all 50M documents."

**Why it's wrong:**
- `updateMany` on 50M documents acquires an intent lock and holds it for the entire operation
- It generates 50M oplog entries all at once — this can overwhelm the oplog and cause replication lag on secondaries
- The operation cannot be interrupted cleanly — if it fails halfway, some documents are updated and some aren't (no rollback for updateMany unless in a transaction, and a 50M-document transaction would exceed the 16MB oplog limit)
- Long-running updateMany prevents other writes from proceeding efficiently

**Correct answer:** "For large bulk updates, use batched updates in a loop with throttling. Process documents in batches of 1,000-10,000, using `_id` as the cursor. Sleep briefly between batches. This limits I/O impact, allows other operations to proceed, and provides natural checkpointing (you can resume from the last processed `_id` after a failure)."

---

## Trap 2: "More Indexes Always Improve Performance"

**Wrong answer:** "I'll index every field that appears in a query filter to make everything fast."

**Why it's wrong:**
- Every index increases write overhead: each insert/update/delete must update all indexes on the collection
- Indexes consume RAM (WiredTiger cache) — too many indexes means working data gets evicted
- The query planner evaluates up to 64 candidate plans — too many indexes can slow plan selection
- Index storage costs money on Atlas (storage is metered)
- MongoDB uses only ONE index per query (with the exception of index intersection, which is rare)

**Correct answer:** "Indexes are a write tax. Each index on a collection makes every write slightly slower. The goal is the minimum number of indexes that satisfy your query patterns. Use compound indexes (which can satisfy multiple queries), partial indexes (smaller, for filtered queries), and regularly audit with `$indexStats` to remove unused indexes. A collection with 15 indexes but only 4 being used needs immediate cleanup."

---

## Trap 3: "The `_id` Field Is Always an ObjectId"

**Wrong answer:** "The `_id` field is a MongoDB-generated ObjectId — you can't change it."

**Why it's wrong:**
- `_id` can be ANY value: string, integer, ObjectId, UUID, custom document
- The only requirement: `_id` must be unique within the collection and cannot be null
- Custom `_id` values are often BETTER than ObjectIds for application use cases

**Correct answer:** "The `_id` field can be any immutable, unique value. In high-throughput systems, using a natural key (like `userId` for a users collection) as `_id` gives you free deduplication and avoids the need for a separate unique index on that field. For time-series data, custom ObjectId-like IDs with non-sequential components can prevent the write hotspot problem of sequential ObjectIds."

```javascript
// Examples of valid custom _id values:
db.users.insertOne({ _id: "user_abc123", email: "user@example.com" })
db.products.insertOne({ _id: "SKU-1234-RED-LG", name: "Red T-Shirt" })
db.sessions.insertOne({ _id: UUID(), createdAt: new Date() })
db.counters.insertOne({ _id: "orderCounter", seq: 0 })
```

---

## Trap 4: "Change Streams Guarantee Exactly-Once Delivery"

**Wrong answer:** "Change streams ensure each event is processed exactly once."

**Why it's wrong:**
- Change streams guarantee AT-LEAST-ONCE delivery, not exactly-once
- If your consumer crashes after processing but before saving the resume token, it will re-process the last event on restart
- Network failures or cursor timeouts can cause the same event to be re-delivered

**Correct answer:** "Change streams provide at-least-once delivery via resume tokens. To achieve exactly-once semantics, you must implement idempotent consumers: persist both the processing result AND the resume token atomically (in a MongoDB transaction). On restart, re-processing the same event with an idempotent consumer produces the same outcome — effectively achieving exactly-once semantics from the business logic perspective."

---

## Trap 5: "mongodump Captures a Consistent Point-in-Time Snapshot"

**Wrong answer:** "I'll run mongodump to get a consistent backup of my entire database."

**Why it's wrong:**
- `mongodump` without `--oplog` is NOT point-in-time consistent across collections
- Collection A is dumped at T=0, Collection B is dumped at T=30s — they're inconsistent with each other
- Writes that occurred between dumping Collection A and Collection B are included in Collection B but not A
- This inconsistency matters for data that spans collections (e.g., orders + orderItems)

**Correct answer:** "For a consistent backup with mongodump, use `--oplog` flag. This captures the oplog delta during the dump and applies it at restore time, giving a consistent snapshot. Better yet: use Atlas Cloud Backup or filesystem snapshots (with `db.adminCommand({ fsync: 1, lock: true })`), which capture a consistent disk state atomically."

```bash
# Consistent dump with oplog capture:
mongodump --host primary:27017 --oplog --out /backup/
# Restores to a single consistent point in time
mongorestore --oplogReplay /backup/
```

---

## Trap 6: "WiredTiger Cache Should Be Set to Maximum Available RAM"

**Wrong answer:** "Set WiredTiger cache to all available RAM for maximum performance."

**Why it's wrong:**
- The OS and other processes need memory too
- If WiredTiger takes all RAM, the OS will swap, which is catastrophically slow
- MongoDB documentation recommends 50% of (RAM - 1GB), which is the default
- Atlas manages WiredTiger cache automatically — you shouldn't override it

**Correct answer:** "WiredTiger cache default is 50% of available RAM minus 1GB. For a dedicated 32GB database server, that's ~15.5GB. Leave the rest for the OS filesystem cache (which helps with disk I/O) and mongod process overhead. If you consistently see high cache pressure (cache hit ratio < 95%), the right fix is to upgrade RAM or reduce your working set size, not to simply max out the cache setting."

---

## Trap 7: "The `compact` Command Should Be Run Regularly"

**Wrong answer:** "I'll add a weekly cron job to run `compact` on all collections to keep them optimized."

**Why it's wrong:**
- On MongoDB 4.4+, WiredTiger performs online compaction automatically (incrementally, in the background)
- `compact` requires an exclusive lock on the collection — it BLOCKS all reads and writes during execution
- On Atlas, there's no way to run `compact` directly (Atlas manages storage)
- Running `compact` weekly on large collections would cause weekly outages

**Correct answer:** "WiredTiger handles compaction automatically and online. `compact` is rarely needed and should only be run manually when: (1) you've deleted a huge percentage of a collection (50%+) and storage hasn't been reclaimed after days, (2) you're on a self-managed instance during a scheduled maintenance window. On Atlas, storage efficiency is managed automatically."

---

## Trap 8: "All MongoDB Operations Are Atomic"

**Wrong answer:** "MongoDB is atomic, so I don't need transactions."

**Why it's nuanced:**
- Single-DOCUMENT operations are ALWAYS atomic (even complex updates with multiple operators)
- Multi-DOCUMENT operations (updateMany, insertMany) are NOT atomic as a whole
- Multi-collection operations require explicit transactions

**Correct answer:** "Single-document atomicity covers most use cases. A document that contains all related data (embedded model) can be updated atomically without transactions. But when you must atomically update multiple documents or multiple collections — like debiting one account and crediting another — you need explicit multi-document transactions. The schema design choice (embedded vs. referenced) directly determines whether you need transactions."

```javascript
// Single document: ALWAYS atomic
db.orders.updateOne(
  { _id: orderId },
  {
    $set: { status: "shipped" },
    $push: { statusHistory: { status: "shipped", ts: new Date() } },
    $inc: { updateCount: 1 }
  }
)
// All three operations succeed or fail together — no partial update possible

// Multi-document: NOT atomic without transaction
db.orders.updateOne({ _id: orderId }, { $set: { status: "shipped" } })
db.inventory.updateOne({ productId: pid }, { $inc: { shipped: 1 } })
// These two can partially succeed — need a transaction for atomicity
```

---

## Trap 9: "TTL Indexes Delete Documents Immediately"

**Wrong answer:** "Once I create a TTL index, expired documents are deleted immediately."

**Why it's wrong:**
- The TTL background thread runs every 60 seconds
- Documents expire 60 ± 60 seconds after their TTL time (could be up to 2 minutes late)
- Under heavy load, TTL deletions are delayed further (TTL thread is lower priority)
- During oplog replay (after failover or initial sync), TTL deletions are not replayed — they happen independently on each node

**Correct answer:** "TTL indexes delete expired documents on a best-effort basis, approximately every 60 seconds. For time-critical expiration (e.g., user sessions that must be invalid immediately after logout), don't rely on TTL alone — check the expiry timestamp in application code. TTL is great for garbage collection of truly expired data (logs, temp tokens) where a minute's delay is acceptable."

---

## Trap 10: "Aggregation Pipeline Stages Execute in Parallel"

**Wrong answer:** "MongoDB's aggregation pipeline parallelizes stages for better performance."

**Why it's wrong:**
- Pipeline stages execute SEQUENTIALLY — the output of stage N is the input to stage N+1
- Individual operations within a stage may use multiple threads internally (e.g., $group, $sort)
- The `$facet` stage is the ONLY stage that runs sub-pipelines concurrently (on the same input)
- `$lookup` is single-threaded per document (each document's lookup is sequential)

**Correct answer:** "Aggregation pipeline stages execute sequentially in order. This is why stage ordering matters — `$match` and `$project` early reduce the document count before expensive stages like `$lookup` and `$group`. The `$facet` stage is the exception: it fans out multiple sub-pipelines concurrently on the same input. For true parallel processing across the full pipeline, you'd need sharding (each shard processes in parallel and mongos merges results)."

---

## Trap 11: "You Can Always Use `allowDiskUse: true` to Fix Memory Issues"

**Wrong answer:** "If an aggregation fails with 'exceeded memory limit,' just add `allowDiskUse: true`."

**Why it's wrong:**
- `allowDiskUse: true` allows stages to spill to disk — but disk I/O is 100-1000x slower than memory
- A 10-second aggregation with allowDiskUse might take 10 minutes
- Atlas Free Tier (M0/M2/M5) does not support `allowDiskUse`
- It's a band-aid — the real fix is to reduce the document count earlier in the pipeline (`$match` earlier, `$project` to remove large fields before `$sort/$group`)

**Correct answer:** "`allowDiskUse: true` is a last resort for unavoidable large sorts. The right approach is: (1) `$match` as early as possible to reduce document count, (2) `$project` to remove large fields before memory-intensive stages, (3) ensure there's an index that makes `$sort` an IXSCAN rather than an in-memory sort, (4) for analytics, run against a secondary or use Atlas Data Federation which is optimized for large scans. `allowDiskUse` is acceptable for infrequent admin queries, not production hot paths."

---

## Trap 12: "Schema Validation Secures Against All Invalid Data"

**Wrong answer:** "I've added JSON Schema validation — my data is now guaranteed to be valid."

**Why it's wrong:**
- Schema validation only applies to WRITES (inserts and updates) — existing invalid data is not retroactively validated
- `validationLevel: "strict"` validates all writes, but `validationLevel: "moderate"` (the only safe choice when adding validation to existing collections with legacy data) only validates non-empty documents
- Validation happens AFTER the write (it rejects the write, but the document briefly exists in the operation stream before being rejected)
- Admin bypass: the `bypassDocumentValidation` option lets any write skip validation

**Correct answer:** "Schema validation is a defense in depth, not a complete data quality solution. Always validate at the application layer first (Mongoose/zod/Joi schema validation), then use MongoDB schema validation as a second guard. When adding validation to an existing collection, use `validationLevel: 'moderate'` and `validationAction: 'warn'` initially to identify existing violations without breaking writes. Migrate those violations, then switch to `validationAction: 'error'`."
