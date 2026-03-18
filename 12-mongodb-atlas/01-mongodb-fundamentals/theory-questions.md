# Chapter 01 — MongoDB Fundamentals: Theory Questions

---

## MongoDB vs RDBMS

**Q1. What are the fundamental differences between MongoDB and a relational database like PostgreSQL?**

Key differences:
- MongoDB stores data as flexible JSON-like documents; RDBMS stores data in fixed-schema tables with rows/columns
- MongoDB has no enforced schema by default (schema-on-read); RDBMS enforces schema-on-write
- MongoDB scales horizontally via sharding; traditional RDBMS scales vertically (scale-up)
- MongoDB has no native JOINs (uses `$lookup` or embedding); RDBMS is built around normalized JOINs
- RDBMS ACID transactions span rows/tables natively; MongoDB added multi-document ACID in version 4.0
- MongoDB supports nested/embedded documents and arrays as first-class fields
- No foreign key constraints in MongoDB by default

---

**Q2. What is the document model in MongoDB?**

A document is a JSON-like record stored in BSON format. Documents:
- Are self-describing — each document can carry its own structure
- Support nested documents (subdocuments) and arrays
- Can have different fields from other documents in the same collection
- Have a maximum size of 16 MB
- Always have an `_id` field that serves as the primary key

```javascript
// Example document
{
  _id: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),
  name: "Alice Johnson",
  email: "alice@example.com",
  age: 32,
  address: {
    street: "123 Main St",
    city: "Austin",
    zip: "78701"
  },
  tags: ["nodejs", "mongodb", "typescript"],
  createdAt: ISODate("2024-01-15T10:30:00Z")
}
```

---

**Q3. How does a MongoDB collection differ from an RDBMS table?**

| Aspect | RDBMS Table | MongoDB Collection |
|--------|------------|-------------------|
| Schema | Fixed, enforced | Flexible, optional validation |
| Row/Document size | Fixed column widths | Variable, up to 16MB |
| Relationships | FK constraints | Embedded docs or `$lookup` |
| Indexing | On columns | On fields, including nested fields |
| Null handling | NULL for missing | Field simply absent |

Collections are schema-less by default but can have JSON Schema validation applied.

---

## BSON

**Q4. What is BSON and why does MongoDB use it instead of raw JSON?**

BSON (Binary JSON) is MongoDB's serialization format. Advantages over JSON:
- **Lightweight**: more compact binary encoding
- **Traversable**: binary structure allows fast field scanning without full parse
- **Rich types**: supports types JSON doesn't have — `Date`, `ObjectId`, `BinData`, `Decimal128`, `Int32`, `Int64`, `Timestamp`, `Regex`, `JavaScript code`
- **Ordered**: fields have deterministic order within documents

JSON limitations BSON solves:
- JSON has only one number type (IEEE 754 float); BSON has `int32`, `int64`, `double`, `Decimal128`
- JSON has no native `Date` type (stores as strings); BSON has a proper UTC datetime type
- JSON cannot represent binary data natively; BSON has `BinData`

```javascript
// BSON types in mongosh
db.demo.insertOne({
  intField: NumberInt(42),           // int32
  longField: NumberLong("9007199254740993"), // int64
  decimalField: NumberDecimal("19.99"),       // Decimal128 (financial math)
  dateField: new Date(),             // UTC datetime
  binaryField: BinData(0, "aGVsbG8="), // Binary data
  regexField: /^hello/i,             // Regular expression
  objectIdField: ObjectId()          // ObjectId
})
```

---

**Q5. What are the differences between JSON and BSON?**

| Aspect | JSON | BSON |
|--------|------|------|
| Format | Text (UTF-8) | Binary |
| Number types | Only double | int32, int64, double, Decimal128 |
| Date type | No (use string) | Yes (UTC milliseconds) |
| Binary data | Base64 string | Native BinData |
| ObjectId | No | Yes |
| Size | Larger (text overhead) | Smaller for numbers/dates |
| Parse speed | Slower (text parse) | Faster (binary traversal) |
| Human readable | Yes | No (requires tooling) |

---

## ObjectId and _id

**Q6. What is an ObjectId and what information does it encode?**

ObjectId is a 12-byte BSON type used as the default `_id` value:

```
|--- 4 bytes ---|--- 5 bytes ------|--- 3 bytes ---|
  Unix timestamp   Random value       Incrementing counter
  (seconds)        (per process)      (within same second)
```

- **Bytes 0–3**: Unix timestamp in seconds (creation time extractable)
- **Bytes 4–8**: Random value unique per machine/process (avoids collisions)
- **Bytes 9–11**: Incrementing counter (allows ordering within same second)

```javascript
// Extracting timestamp from ObjectId
const id = ObjectId("64f1a2b3c4d5e6f7a8b9c0d1")
id.getTimestamp()  // ISODate("2023-09-01T12:00:03Z")

// Comparing ObjectIds (newer ObjectIds are "greater")
ObjectId("64f1a2b3...") < ObjectId("64f1a2b4...")  // true (older < newer)
```

---

**Q7. Why is `_id` immutable in MongoDB?**

The `_id` field serves as the primary key and is used:
- As the shard key reference in sharded clusters (in some configurations)
- As the B-tree root in the `_id` index
- As the oplog's reference for replication ordering

If `_id` could be changed, replication would break (secondaries use oplog entries referencing `_id`), and index corruption could occur. MongoDB enforces immutability at the storage engine level — any attempt to `$set` the `_id` field throws `ModificationNotAllowed`.

```javascript
// This WILL fail
db.users.updateOne({ _id: ObjectId("...") }, { $set: { _id: "new_id" } })
// WriteError: Performing an update on the path '_id' would modify the immutable field '_id'
```

---

**Q8. Can you use a custom value for `_id`? When should you?**

Yes. Any unique, immutable value works as `_id`:

```javascript
// Custom string _id (useful when natural key exists)
db.countries.insertOne({ _id: "US", name: "United States", population: 331000000 })

// Custom composite key (using embedded doc)
db.inventory.insertOne({ _id: { warehouseId: "W001", sku: "PROD-123" }, qty: 500 })

// UUID as _id
db.sessions.insertOne({ _id: UUID(), userId: ObjectId("..."), expiresAt: new Date() })
```

Use custom `_id` when:
- You have a natural unique key (ISO country code, username, SKU)
- You want to avoid a separate unique index
- You need human-readable IDs for external APIs

Avoid when:
- ID must be auto-generated at scale across many writers (ObjectId is safer — no collisions)
- You cannot guarantee uniqueness upfront

---

## MongoDB Data Types

**Q9. List the important MongoDB/BSON data types and when to use each.**

```javascript
{
  // String
  name: "Alice",                          // UTF-8 string

  // Numbers
  age: NumberInt(32),                     // 32-bit integer (use for small counts)
  views: NumberLong("9007199254740994"),  // 64-bit integer (use for large counts)
  score: 98.6,                            // Double (64-bit float) — default for JS numbers
  price: NumberDecimal("19.99"),          // Decimal128 — use for financial values!

  // Boolean
  isActive: true,

  // Date
  createdAt: new Date(),                  // UTC datetime (stored as milliseconds)

  // ObjectId
  userId: ObjectId("64f1a2b3c4d5e6f7a8b9c0d1"),

  // Arrays
  tags: ["mongodb", "atlas"],

  // Embedded Document
  address: { city: "Austin", zip: "78701" },

  // Null
  middleName: null,

  // Binary Data
  profilePic: BinData(0, "base64encodeddata"),

  // Regular Expression
  pattern: /^hello/i,

  // JavaScript (rare, avoid in modern usage)
  validator: Code("function() { return this.age > 0; }"),

  // Timestamp (internal use — replication oplog)
  oplogTs: Timestamp(1, 1),

  // Min/Max Key (for sharding boundary markers)
  lowerBound: MinKey(),
  upperBound: MaxKey()
}
```

**Q10. Why should you use `NumberDecimal` for financial values instead of `double`?**

IEEE 754 double-precision floats cannot represent many decimal fractions exactly:

```javascript
// In JavaScript (and MongoDB doubles):
0.1 + 0.2  // 0.30000000000000004 — wrong!

// With Decimal128:
NumberDecimal("0.1") + NumberDecimal("0.2")  // NumberDecimal("0.3") — correct
```

Financial calculations with doubles accumulate rounding errors. `Decimal128` uses base-10 arithmetic with 34 significant digits — critical for prices, balances, and tax calculations.

---

## Collections and the Document Model

**Q11. What is a capped collection and when would you use one?**

A capped collection is a fixed-size circular buffer collection that:
- Has a maximum size in bytes and optionally maximum document count
- Automatically overwrites the oldest documents when full
- Preserves insertion order (no index needed for natural sort)
- Does NOT support deletion of individual documents
- Supports tailing cursors (like `tail -f` on a log file)

```javascript
// Create a capped collection: 100MB, max 1,000,000 documents
db.createCollection("appLogs", {
  capped: true,
  size: 104857600,   // 100 MB in bytes
  max: 1000000
})

// Check if capped
db.appLogs.isCapped()  // true

// Tailable cursor (blocks and waits for new docs)
db.appLogs.find().tailable()
```

Use cases: application logs, event streams, audit trails, chat message history.

---

**Q12. What is the difference between embedded documents and references (DBRef) in MongoDB?**

**Embedded documents**: store related data inside the parent document

```javascript
// Embedded (denormalized)
{
  _id: ObjectId("..."),
  orderId: "ORD-001",
  customer: {
    name: "Alice",
    email: "alice@example.com",
    address: { city: "Austin", zip: "78701" }
  },
  items: [
    { sku: "WIDGET-A", qty: 2, price: NumberDecimal("9.99") }
  ]
}
```

**References**: store the `_id` of another document (manual reference)

```javascript
// Referenced (normalized)
{
  _id: ObjectId("..."),
  orderId: "ORD-001",
  customerId: ObjectId("customer_id_here"),  // reference
  items: [ObjectId("item1_id"), ObjectId("item2_id")]
}
// Requires separate query to fetch customer data
```

| Aspect | Embedded | Reference |
|--------|----------|-----------|
| Read performance | One query | Multiple queries or $lookup |
| Write performance | Single atomic update | Multiple updates (not atomic without transaction) |
| Data duplication | Yes | No |
| Good for | Data accessed together, 1:1, bounded 1:N | Large/unbounded related sets, many-to-many |
| Document size risk | Can hit 16MB limit | No |

---

## Replica Sets

**Q13. What is a MongoDB replica set?**

A replica set is a group of `mongod` instances that maintain the same data set:

- **Primary**: accepts all write operations, propagates changes to secondaries via the oplog
- **Secondary**: replicates from primary's oplog, can serve reads (with appropriate read preference)
- **Arbiter**: votes in elections but holds no data (used to achieve odd vote count without full data copy)

```javascript
// Check replica set status from mongosh
rs.status()

// Check which member is primary
rs.isMaster()  // or rs.hello() in newer versions

// Check replication lag
rs.printReplicationInfo()
rs.printSecondaryReplicationInfo()
```

Minimum viable replica set: 1 primary + 1 secondary + 1 arbiter (PSA) or 1 primary + 2 secondaries (PSS — preferred for production).

---

**Q14. What is the MongoDB oplog and why is it important?**

The oplog (operations log) is a special capped collection (`local.oplog.rs`) that records all data-modifying operations on the primary:

- Secondaries tail the oplog to replicate changes
- Each oplog entry is **idempotent** — can be safely re-applied multiple times
- The oplog has a configurable size; if a secondary falls too far behind (replication lag > oplog window), it enters `RECOVERING` state and requires a full resync
- Change Streams are built on top of the oplog

```javascript
// View oplog entries
use local
db.oplog.rs.find().sort({$natural: -1}).limit(5)

// Sample oplog entry
{
  ts: Timestamp(1693574403, 1),
  t: NumberLong(15),           // term number (election)
  h: NumberLong(0),
  v: 2,
  op: "u",                     // u=update, i=insert, d=delete, c=command
  ns: "mydb.users",
  ui: UUID("..."),
  o2: { _id: ObjectId("...") },  // query portion
  o: { $v: 2, diff: { ... } }    // modification
}
```

---

## Sharding Overview

**Q15. What is sharding in MongoDB and why is it needed?**

Sharding is MongoDB's horizontal scaling mechanism. A sharded cluster distributes data across multiple replica sets (shards):

- Enables datasets larger than a single server's capacity
- Distributes read/write throughput across multiple machines
- Each shard is itself a replica set for high availability

Key components:
- **Shards**: replica sets holding a subset of data
- **mongos**: query router (clients connect to mongos, not shards directly)
- **Config servers**: replica set storing cluster metadata, chunk assignments, and shard key ranges

```javascript
// Enable sharding on a database
sh.enableSharding("ecommerce")

// Shard a collection on the userId field (hashed for uniform distribution)
sh.shardCollection("ecommerce.orders", { userId: "hashed" })

// Check sharding status
sh.status()
```

---

## CAP Theorem

**Q16. Where does MongoDB sit in the CAP theorem?**

MongoDB is a **CP** system (Consistency + Partition Tolerance) with configurable tuning:

- **Default behavior**: With `w:majority` write concern and `r:majority` read concern, MongoDB guarantees strong consistency — reads always see the latest committed writes
- **Availability trade-off**: During network partitions, MongoDB may reject writes rather than risk split-brain scenarios (primary steps down without quorum)
- **Eventual consistency option**: With `w:1` write concern and secondary reads, you get higher availability with eventual consistency — reads from secondaries may lag behind primary

MongoDB is not "AP" by default — it prioritizes consistency when partitions occur. However, tuning write/read concerns lets you shift the trade-off point.

---

## CRUD Operations

**Q17. What is the difference between `find()` and `findOne()`?**

```javascript
// findOne() — returns a single document (or null)
const user = db.users.findOne({ email: "alice@example.com" })

// find() — returns a cursor (lazy iterator over results)
const cursor = db.users.find({ age: { $gte: 18 } })

// You must iterate the cursor:
cursor.forEach(doc => printjson(doc))
// OR
cursor.toArray()  // loads all into memory — be careful with large result sets
```

Key differences:
- `findOne()` adds an implicit `.limit(1)` internally
- `find()` returns a **Cursor** object — documents are fetched in batches (default batch size 101 for first batch, then 16MB chunks)
- `find()` supports chaining: `.sort()`, `.limit()`, `.skip()`, `.projection()`

---

**Q18. Explain the four core CRUD operations in MongoDB.**

```javascript
// CREATE
db.products.insertOne({ name: "Widget", price: NumberDecimal("9.99"), stock: 100 })
db.products.insertMany([
  { name: "Gadget", price: NumberDecimal("24.99") },
  { name: "Doohickey", price: NumberDecimal("4.99") }
])

// READ
db.products.find({ price: { $lt: NumberDecimal("10.00") } })
db.products.findOne({ name: "Widget" })

// UPDATE
db.products.updateOne(
  { name: "Widget" },
  { $set: { price: NumberDecimal("12.99") }, $inc: { stock: -1 } }
)
db.products.updateMany(
  { stock: { $lt: 10 } },
  { $set: { status: "low-stock" } }
)

// DELETE
db.products.deleteOne({ name: "Widget" })
db.products.deleteMany({ stock: 0 })
```

---

## Write Concerns

**Q19. What is a write concern in MongoDB?**

Write concern specifies the level of acknowledgment requested from MongoDB for write operations:

```javascript
// w:0 — Fire and forget (no acknowledgment)
db.logs.insertOne({ msg: "event" }, { writeConcern: { w: 0 } })

// w:1 — Primary acknowledged (default)
db.orders.insertOne({ total: 99.99 }, { writeConcern: { w: 1 } })

// w:majority — Majority of voting members acknowledged
db.payments.insertOne({ amount: 500 }, { writeConcern: { w: "majority", wtimeout: 5000 } })

// w:all — All replica set members acknowledged (avoid — one slow secondary blocks writes)
db.critical.insertOne({ data: "..." }, { writeConcern: { w: "all" } })

// j:true — Write must be written to journal on disk before acknowledgment
db.financial.insertOne({ txn: "..." }, { writeConcern: { w: "majority", j: true } })
```

| Write Concern | Performance | Durability | Use Case |
|--------------|-------------|------------|----------|
| w:0 | Fastest | None | Metrics, ephemeral logs |
| w:1 | Fast | Primary only | Most operations |
| w:majority | Slower | Survives primary failure | Financial, orders |
| j:true | Slowest | Survives crash | Maximum durability |

---

## Read Preferences

**Q20. What are MongoDB read preferences?**

Read preference controls which replica set members receive read operations:

```javascript
// primary (default) — always reads from primary
db.users.find().readPref("primary")

// primaryPreferred — primary if available, else secondary
db.users.find().readPref("primaryPreferred")

// secondary — always reads from a secondary
db.reports.find().readPref("secondary")  // may return stale data

// secondaryPreferred — secondary if available, else primary (good for analytics)
db.analytics.find().readPref("secondaryPreferred")

// nearest — lowest network latency member (regardless of primary/secondary)
db.catalog.find().readPref("nearest")  // good for geo-distributed deployments
```

Read preferences can be combined with **tag sets** to route reads to specific data center regions:

```javascript
db.users.find().readPref("secondary", [{ "datacenter": "us-east-1" }])
```

---

**Q21. What is the MongoDB Wire Protocol?**

The MongoDB Wire Protocol is the binary protocol used for communication between MongoDB clients (drivers) and the mongod/mongos server:

- Based on TCP/IP
- Uses **Opcodes** for different message types: `OP_MSG` (current), `OP_QUERY` (legacy), `OP_GET_MORE`, `OP_INSERT`, `OP_UPDATE`, `OP_DELETE`
- Modern drivers use `OP_MSG` (introduced in MongoDB 3.6) which is more flexible and supports sessions
- The protocol version is negotiated during the initial `hello`/`isMaster` handshake
- Connection compression (snappy, zlib, zstd) is negotiated in the handshake

Understanding the wire protocol matters for:
- Debugging network-level issues
- Understanding why driver version must match MongoDB server version
- MongoDB proxy/load balancer design

---

## mongosh Basics

**Q22. What are essential mongosh commands for day-to-day operations?**

```javascript
// Database operations
show dbs                         // list all databases
use myDatabase                   // switch to database (creates on first write)
db                               // show current database
db.dropDatabase()                // DANGER: drops current database

// Collection operations
show collections                 // list collections in current db
db.createCollection("users")     // explicit collection creation
db.users.drop()                  // drop a collection

// Document counts and stats
db.users.countDocuments()        // exact count
db.users.estimatedDocumentCount() // fast approximate count (uses metadata)
db.users.stats()                 // collection statistics
db.stats()                       // database statistics

// Index operations
db.users.getIndexes()            // list all indexes
db.users.createIndex({ email: 1 }, { unique: true })

// Administrative
db.serverStatus()                // server health and metrics
db.currentOp()                   // currently running operations
db.killOp(opid)                  // kill a specific operation
rs.status()                      // replica set status
sh.status()                      // sharding status
```

---

**Q23. How do you perform a basic query with filtering, projection, sort, and limit in mongosh?**

```javascript
// Full query with all modifiers
db.products.find(
  // Filter
  {
    category: "electronics",
    price: { $lte: NumberDecimal("100.00") },
    inStock: true
  },
  // Projection (1 = include, 0 = exclude)
  {
    name: 1,
    price: 1,
    brand: 1,
    _id: 0   // explicitly exclude _id
  }
)
.sort({ price: 1, name: 1 })   // sort by price ascending, then name ascending
.skip(20)                       // pagination: skip first 20
.limit(10)                      // return at most 10 documents

// Equivalent using aggregation (preferred for complex cases)
db.products.aggregate([
  { $match: { category: "electronics", price: { $lte: NumberDecimal("100.00") }, inStock: true } },
  { $project: { name: 1, price: 1, brand: 1, _id: 0 } },
  { $sort: { price: 1, name: 1 } },
  { $skip: 20 },
  { $limit: 10 }
])
```

---

**Q24. What is the difference between `db.collection.count()`, `countDocuments()`, and `estimatedDocumentCount()`?**

```javascript
// DEPRECATED: db.collection.count() — DO NOT USE
// Inaccurate after unclean shutdown with orphan documents in sharded clusters
db.users.count({ active: true })  // deprecated

// countDocuments() — accurate count using aggregation ($match + $group)
// Respects the filter, uses an index if available
db.users.countDocuments({ active: true })  // accurate, but slightly slower
db.users.countDocuments({})               // counts ALL documents accurately

// estimatedDocumentCount() — reads collection metadata (fast)
// Does NOT accept a filter, approximates based on metadata
db.users.estimatedDocumentCount()  // very fast, may be slightly off after crash
```

Use `countDocuments()` for accurate counts with a filter.
Use `estimatedDocumentCount()` when you need fast approximate total counts.

---

**Q25. What is Change Streams and how does it relate to MongoDB fundamentals?**

Change Streams allow applications to subscribe to real-time data changes in MongoDB collections, databases, or the entire deployment:

```javascript
// Watch for all changes on a collection
const changeStream = db.orders.watch()

// Watch with filter (only inserts and updates to high-value orders)
const changeStream = db.orders.watch([
  {
    $match: {
      operationType: { $in: ["insert", "update"] },
      "fullDocument.total": { $gte: 1000 }
    }
  }
])

// Process events
changeStream.forEach(change => {
  console.log("Change detected:", change.operationType)
  console.log("Document:", change.fullDocument)
  console.log("Resume token:", change._id)  // save for resuming after disconnect
})
```

Change Streams are built on the oplog and require:
- A replica set or sharded cluster (not available on standalone)
- MongoDB 3.6+ (collection-level), 4.0+ (database-level), 4.0+ (deployment-level)
- Read concern `majority` enabled (default)

Common uses: real-time notifications, cache invalidation, event-driven microservices, ETL pipelines.
