# Chapter 05 — Schema Design: Scenario Questions

---

## Scenario 1: Blog Platform Schema

**Scenario**: You're designing the MongoDB schema for a blogging platform. Key requirements:
- Posts have a title, body (up to 50KB), tags (3–10 per post), and author info
- Posts have comments; top posts can have 10,000+ comments
- Comments can be nested (replies to comments), up to 3 levels deep
- The most common queries: latest posts, posts by tag, a post with its top 20 comments
- Authors rarely change their name or avatar

**Question**: How do you model posts, comments, and authors? What about the unbounded comment problem?

**Answer**:

```javascript
// Author collection (normalized — reference only)
{
  _id: ObjectId("..."),
  username: "alice_writes",
  displayName: "Alice Johnson",
  avatarUrl: "https://cdn.example.com/avatars/alice.jpg",
  bio: "Software engineer and writer.",
  followersCount: 4821
}

// Post collection — embed author SUBSET (Extended Reference Pattern)
// Embed top 20 comments, reference overflow
{
  _id: ObjectId("..."),
  slug: "mongodb-schema-design-2024",
  title: "MongoDB Schema Design Best Practices",
  body: "...(up to 50KB)...",
  tags: ["mongodb", "schema", "nosql"],
  author: {
    _id: ObjectId("..."),          // reference for joins if needed
    username: "alice_writes",      // denormalized subset
    displayName: "Alice Johnson",  // denormalized subset
    avatarUrl: "https://cdn.example.com/avatars/alice.jpg"
  },
  publishedAt: ISODate("2024-03-01T10:00:00Z"),
  updatedAt: ISODate("2024-03-05T14:30:00Z"),
  status: "published",
  stats: {
    views: 15420,
    likes: 892,
    commentsCount: 347           // pre-computed total
  },
  // Embed only TOP-LEVEL comments (first 20) — Outlier Pattern
  pinnedComments: [
    {
      _id: ObjectId("..."),
      author: { _id: ObjectId("..."), username: "bob", displayName: "Bob" },
      body: "Great article!",
      createdAt: ISODate("2024-03-01T11:00:00Z"),
      likes: 45,
      replyCount: 3
    }
    // max 20 entries — bounded!
  ],
  hasMoreComments: true          // flag: load more from comments collection
}

// Comments collection — for overflow + replies
{
  _id: ObjectId("..."),
  postId: ObjectId("..."),       // FK to post
  parentId: null,                // null = top-level; ObjectId = reply
  ancestorIds: [],               // Array of Ancestors Pattern for depth queries
  depth: 0,                      // 0=top, 1=reply, 2=reply-to-reply (max 3)
  author: { _id: ObjectId("..."), username: "carol", displayName: "Carol" },
  body: "This helped me a lot!",
  createdAt: ISODate("2024-03-02T09:00:00Z"),
  likes: 12,
  replyCount: 0,
  isDeleted: false
}

// Indexes
db.posts.createIndex({ tags: 1, publishedAt: -1 })      // posts by tag, recent first
db.posts.createIndex({ "author._id": 1, publishedAt: -1 })
db.posts.createIndex({ slug: 1 }, { unique: true })
db.posts.createIndex({ status: 1, publishedAt: -1 })

db.comments.createIndex({ postId: 1, parentId: 1, createdAt: -1 })  // thread view
db.comments.createIndex({ postId: 1, depth: 1, likes: -1 })          // top comments
db.comments.createIndex({ "author._id": 1, createdAt: -1 })
```

**Key decisions**:
- Author subset embedded (Extended Reference) — name + avatar don't change often
- Top 20 comments embedded for fast initial page load (Outlier Pattern handles viral posts)
- Comments collection for overflow — prevents document from growing unbounded
- `commentsCount` pre-computed (Computed Pattern) — avoids `$count` on every page load

---

## Scenario 2: E-Commerce Product Catalog

**Scenario**: You're building a product catalog for a marketplace. Products include:
- Electronics (CPU speed, RAM, storage, wattage)
- Clothing (size, color, material, gender)
- Books (ISBN, author, publisher, pages, genre)
- Furniture (dimensions, material, assembly required)

Each product type has ~15 unique attributes. You need to query: `{ "attributes.ram": "16GB" }`, faceted search by any attribute, and filter by price range. New product types are added monthly.

**Answer**:

```javascript
// Attribute Pattern — key-value pairs for polymorphic attributes
{
  _id: ObjectId("..."),
  sku: "LAPTOP-XPS-15-001",
  name: "Dell XPS 15 Laptop",
  category: "electronics",
  subCategory: "laptops",
  brand: "Dell",
  price: NumberDecimal("1299.99"),
  currency: "USD",
  inventory: {
    quantity: 45,
    reserved: 3,
    warehouse: "US-EAST-01"
  },
  images: [
    { url: "https://cdn.example.com/products/xps15-main.jpg", isPrimary: true },
    { url: "https://cdn.example.com/products/xps15-side.jpg", isPrimary: false }
  ],

  // Attribute Pattern — enables indexing any attribute
  attributes: [
    { k: "ram",        v: "16GB",        unit: null },
    { k: "storage",    v: "512GB",       unit: "SSD" },
    { k: "cpu",        v: "Intel i7",    unit: null },
    { k: "screen",     v: "15.6",        unit: "inches" },
    { k: "wattage",    v: 65,            unit: "W" },
    { k: "weight",     v: 1.86,          unit: "kg" }
  ],

  // Also keep top attributes as first-class fields for compound queries
  priceRange: "1000-1500",   // bucketed for facets

  createdAt: ISODate("2024-01-15T00:00:00Z"),
  isActive: true,
  schemaVersion: 2
}

// Indexes for Attribute Pattern
db.products.createIndex({ "attributes.k": 1, "attributes.v": 1 })
// Query: db.products.find({ attributes: { $elemMatch: { k: "ram", v: "16GB" } } })

db.products.createIndex({ category: 1, price: 1 })
db.products.createIndex({ brand: 1, category: 1, price: 1 })
db.products.createIndex({ isActive: 1, category: 1, "attributes.k": 1, "attributes.v": 1 })

// Faceted search with Atlas Search (or aggregation)
db.products.aggregate([
  { $match: { category: "electronics", isActive: true } },
  {
    $facet: {
      byBrand: [
        { $group: { _id: "$brand", count: { $sum: 1 } } },
        { $sort: { count: -1 } },
        { $limit: 10 }
      ],
      byPriceRange: [
        { $bucket: {
            groupBy: "$price",
            boundaries: [0, 100, 500, 1000, 2000, 5000],
            default: "5000+",
            output: { count: { $sum: 1 } }
        }}
      ],
      byAttribute: [
        { $unwind: "$attributes" },
        { $group: {
            _id: { key: "$attributes.k", value: "$attributes.v" },
            count: { $sum: 1 }
        }},
        { $sort: { count: -1 } },
        { $limit: 50 }
      ]
    }
  }
])
```

**Key decisions**:
- Attribute Pattern accommodates unlimited new product types without schema migration
- Compound `$elemMatch` index on `attributes.k` + `attributes.v` enables efficient attribute queries
- Price kept as first-class field (not buried in attributes) for range index queries
- Atlas Search handles full-text search across all attributes efficiently

---

## Scenario 3: IoT Sensor Data

**Scenario**: 10,000 temperature sensors send readings every second. Each reading has: sensor ID, timestamp, temperature (°C), humidity (%), battery level (%). You need:
- Store 90 days of data (10,000 sensors × 86,400 seconds = 864 million readings/day)
- Query: average temperature per sensor per hour for the last 24 hours
- Alert: sensor readings outside threshold (>40°C or <0°C)
- Dashboard: last reading per sensor

**Question**: How do you store this efficiently? Compare three approaches.

**Answer**:

```javascript
// APPROACH 1: One document per reading (naive — BAD)
// 864M documents/day × 90 days = ~78 BILLION documents
// Index on sensorId + timestamp = huge index
{
  _id: ObjectId("..."),
  sensorId: "SENSOR-001",
  timestamp: ISODate("2024-03-01T10:00:00.000Z"),
  temp: 22.5,
  humidity: 65.2,
  battery: 98
}
// Problem: massive collection, poor write performance, huge indexes

// APPROACH 2: Bucket Pattern (RECOMMENDED for IoT)
// Group readings into hourly buckets — reduces doc count 3600x
{
  _id: ObjectId("..."),
  sensorId: "SENSOR-001",
  date: ISODate("2024-03-01T10:00:00.000Z"),  // hour truncated
  hour: 10,

  // Pre-computed stats (Computed Pattern combined)
  stats: {
    count:   3600,
    tempSum: 81090.5,
    tempMin: 21.8,
    tempMax: 23.4,
    tempAvg: 22.525,         // updated incrementally
    humidityAvg: 64.9
  },

  // Sparse readings array — only anomalies or sampled data
  anomalies: [
    { t: 1840, temp: 41.2 },   // t = seconds offset within hour
    { t: 2310, temp: -0.5 }
  ],

  // Raw data for recent hour (last hour kept detailed, older compressed)
  readings: [
    { t: 0,    temp: 22.5, hum: 65.2, bat: 98 },
    { t: 1,    temp: 22.6, hum: 65.1, bat: 98 },
    // ... 3600 readings (can exceed typical embedded limits — use carefully)
  ],

  nReadings: 3600
}
// 10,000 sensors × 24 hours = 240,000 documents/day (vs 864M)
// 90 days = 21.6 million documents total (vs 77.8 billion)

// APPROACH 3: MongoDB Time-Series Collections (MongoDB 5.0+) — BEST
db.createCollection("sensorReadings", {
  timeseries: {
    timeField:     "timestamp",    // REQUIRED: ISODate field
    metaField:     "sensorId",     // optional: groups/partitions data
    granularity:   "seconds"       // "seconds" | "minutes" | "hours"
  },
  expireAfterSeconds: 7776000      // 90 days TTL
})

// Insert individual readings — MongoDB handles bucketing internally
db.sensorReadings.insertMany([
  { sensorId: "SENSOR-001", timestamp: ISODate("2024-03-01T10:00:00Z"), temp: 22.5, humidity: 65.2, battery: 98 },
  { sensorId: "SENSOR-001", timestamp: ISODate("2024-03-01T10:00:01Z"), temp: 22.6, humidity: 65.1, battery: 98 }
])

// Query: hourly averages
db.sensorReadings.aggregate([
  { $match: {
      sensorId: "SENSOR-001",
      timestamp: { $gte: ISODate("2024-03-01T00:00:00Z"), $lt: ISODate("2024-03-02T00:00:00Z") }
  }},
  { $group: {
      _id: { hour: { $hour: "$timestamp" } },
      avgTemp: { $avg: "$temp" },
      maxTemp: { $max: "$temp" },
      minTemp: { $min: "$temp" },
      readings: { $sum: 1 }
  }},
  { $sort: { "_id.hour": 1 } }
])
```

**Comparison**:

| Approach | Documents/day | Storage | Query speed | Write speed |
|---|---|---|---|---|
| Per-reading | 864M | ~250GB/day | Slow | Slow |
| Bucket Pattern | 240K | ~5GB/day | Fast | Medium |
| Time-series | Auto-bucketed | ~3GB/day | Fastest | Fastest |

---

## Scenario 4: Social Network — User Relationships

**Scenario**: Building a social network. Users can follow other users (Twitter-style). Requirements:
- User A follows User B
- Query: "who does Alice follow?" (following list)
- Query: "who follows Alice?" (followers list)
- Query: "mutual friends between Alice and Bob"
- Some users (celebrities) have 50M+ followers

**Question**: Model the follow relationship. Handle the celebrity problem.

**Answer**:

```javascript
// APPROACH 1: Embedded arrays (wrong for large follow counts)
{
  _id: ObjectId("..."),
  username: "alice",
  following: [ ObjectId("bob"), ObjectId("carol") ],   // Alice follows these
  followers: [ ObjectId("dave"), ObjectId("eve") ]     // These follow Alice
}
// Problem: 50M followers = document > 16MB, unbounded array, write amplification

// APPROACH 2: Separate relationships collection (CORRECT)
// One document per directed edge
{
  _id: ObjectId("..."),
  followerId:  ObjectId("alice_id"),     // who is following
  followeeId:  ObjectId("bob_id"),       // who is being followed
  createdAt:   ISODate("2024-01-15T10:00:00Z")
}

// Indexes
db.follows.createIndex({ followerId: 1, followeeId: 1 }, { unique: true })  // deduplicate
db.follows.createIndex({ followeeId: 1, createdAt: -1 })                    // follower list
db.follows.createIndex({ followerId: 1, createdAt: -1 })                    // following list

// Follow operation (atomic)
db.follows.insertOne({ followerId: aliceId, followeeId: bobId, createdAt: new Date() })

// Unfollow operation
db.follows.deleteOne({ followerId: aliceId, followeeId: bobId })

// Keep denormalized counts on user document
db.users.updateOne({ _id: aliceId }, { $inc: { followingCount: 1 } })
db.users.updateOne({ _id: bobId },   { $inc: { followersCount: 1 } })

// Query: who does Alice follow?
db.follows.find({ followerId: aliceId }, { followeeId: 1 }).limit(50)

// Query: followers of Alice
db.follows.find({ followeeId: aliceId }, { followerId: 1 }).sort({ createdAt: -1 }).limit(50)

// Outlier Pattern for celebrities
{
  _id: ObjectId("..."),
  username: "elonmusk",
  followersCount: 52000000,   // pre-computed counter
  isCelebrity: true,           // flag to route queries differently
  // NO followers array — use relationships collection
}

// Mutual follows (intersection)
db.follows.aggregate([
  { $match: { followerId: aliceId } },            // who Alice follows
  { $lookup: {
      from: "follows",
      let: { followeeId: "$followeeId" },
      pipeline: [
        { $match: { $expr: {
            $and: [
              { $eq: ["$followerId", "$$followeeId"] },
              { $eq: ["$followeeId", aliceId] }     // and they follow Alice back
            ]
        }}}
      ],
      as: "mutualCheck"
  }},
  { $match: { mutualCheck: { $ne: [] } } },
  { $project: { followeeId: 1, _id: 0 } }
])
```

---

## Scenario 5: Inventory Management System

**Scenario**: Multi-warehouse inventory system. Products are stocked in multiple warehouses. Operations:
- Receive stock (increase quantity at warehouse)
- Pick order (decrease quantity, reserve stock)
- Transfer between warehouses
- Query: total quantity across all warehouses for a product
- Query: all products below reorder threshold at warehouse X

**Question**: Design the schema. How do you prevent overselling?

**Answer**:

```javascript
// Product document — Computed Pattern for total inventory
{
  _id: ObjectId("..."),
  sku: "WIDGET-A-001",
  name: "Blue Widget Type A",
  description: "...",
  category: "widgets",
  reorderThreshold: 100,
  reorderQuantity: 500,

  // Computed totals (updated on every stock movement)
  totalQuantity: 450,     // sum across all warehouses
  totalReserved: 30,      // sum of reserved quantities
  totalAvailable: 420,    // totalQuantity - totalReserved

  createdAt: ISODate("2024-01-01T00:00:00Z")
}

// Inventory document — one per product+warehouse combination
{
  _id: ObjectId("..."),
  productId: ObjectId("..."),
  warehouseId: "US-EAST-01",
  sku: "WIDGET-A-001",     // denormalized for query convenience

  quantity: 200,           // physical quantity on hand
  reserved: 15,            // quantity reserved for pending orders
  available: 185,          // quantity - reserved (computed, kept in sync)

  lastReceivedAt: ISODate("2024-03-01T08:00:00Z"),
  lastPickedAt: ISODate("2024-03-01T14:30:00Z"),
  location: "RACK-A3-SHELF2"   // physical location in warehouse
}

// Indexes
db.inventory.createIndex({ productId: 1, warehouseId: 1 }, { unique: true })
db.inventory.createIndex({ warehouseId: 1, available: 1 })   // reorder queries
db.inventory.createIndex({ sku: 1, warehouseId: 1 })

// Stock movement log (append-only audit trail)
{
  _id: ObjectId("..."),
  productId: ObjectId("..."),
  warehouseId: "US-EAST-01",
  sku: "WIDGET-A-001",
  type: "receive",          // receive | pick | reserve | transfer_in | transfer_out | adjust
  quantity: 100,            // positive = inbound, negative = outbound
  quantityBefore: 150,
  quantityAfter: 250,
  referenceType: "purchase_order",
  referenceId: ObjectId("..."),
  performedBy: ObjectId("..."),
  timestamp: ISODate("2024-03-01T08:00:00Z"),
  notes: "PO-2024-0301"
}

// Atomic reserve (prevent overselling)
async function reserveStock(productId, warehouseId, quantity, orderId) {
  const result = await db.inventory.findOneAndUpdate(
    {
      productId: ObjectId(productId),
      warehouseId: warehouseId,
      available: { $gte: quantity }    // ONLY update if enough available
    },
    {
      $inc: { reserved: quantity, available: -quantity }
    },
    { returnDocument: "after" }
  )

  if (!result) {
    throw new Error(`Insufficient stock: ${quantity} units not available`)
  }

  // Log the movement
  await db.stockMovements.insertOne({
    productId: ObjectId(productId),
    warehouseId: warehouseId,
    type: "reserve",
    quantity: quantity,
    referenceType: "order",
    referenceId: ObjectId(orderId),
    timestamp: new Date()
  })

  return result
}

// Query: products below reorder threshold at a warehouse
db.inventory.aggregate([
  { $match: { warehouseId: "US-EAST-01" } },
  { $lookup: {
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product"
  }},
  { $unwind: "$product" },
  { $match: { $expr: { $lt: ["$available", "$product.reorderThreshold"] } } },
  { $project: {
      sku: 1, warehouseId: 1, available: 1,
      reorderThreshold: "$product.reorderThreshold",
      reorderQuantity: "$product.reorderQuantity",
      shortfall: { $subtract: ["$product.reorderThreshold", "$available"] }
  }},
  { $sort: { shortfall: -1 } }
])
```

---

## Scenario 6: Multi-Tenant SaaS Application

**Scenario**: A project management SaaS (like Jira/Asana). Multiple organizations (tenants) use the platform. Each org has projects, tasks, users, and comments. Concerns:
- Complete data isolation between tenants
- Tenant A should never see Tenant B's data
- Some tenants have 10K users, some have 10
- Query performance must not degrade as tenant count grows

**Question**: Should you use one collection per tenant, or shared collections? How do you enforce isolation?

**Answer**:

```javascript
// APPROACH 1: Database per tenant (strong isolation, operational complexity)
// - db name: "tenant_acme", "tenant_globocorp"
// - Pros: complete isolation, easy backup per tenant, custom indexes
// - Cons: Atlas connection limit (500 dbs recommended max), operational overhead

// APPROACH 2: Shared collections with tenantId (recommended for most SaaS)
// All tenants in same collection, every document has tenantId

{
  _id: ObjectId("..."),
  tenantId: ObjectId("ACME_TENANT_ID"),   // REQUIRED on every document
  projectId: ObjectId("..."),
  title: "Implement OAuth2",
  description: "Add OAuth2 login flow",
  status: "in-progress",
  priority: "high",
  assigneeId: ObjectId("..."),
  labels: ["backend", "security"],
  createdAt: ISODate("2024-03-01T00:00:00Z"),
  dueDate: ISODate("2024-03-31T00:00:00Z")
}

// Every index MUST include tenantId as prefix
db.tasks.createIndex({ tenantId: 1, projectId: 1, status: 1 })
db.tasks.createIndex({ tenantId: 1, assigneeId: 1, dueDate: 1 })
db.tasks.createIndex({ tenantId: 1, status: 1, priority: 1, createdAt: -1 })

// Application-layer enforcement (middleware pattern)
class TaskRepository {
  constructor(tenantId) {
    this.tenantId = ObjectId(tenantId)
  }

  // All queries automatically scoped to tenant
  async findByProject(projectId) {
    return db.tasks.find({
      tenantId: this.tenantId,          // always prepended
      projectId: ObjectId(projectId)
    }).toArray()
  }

  async create(taskData) {
    return db.tasks.insertOne({
      ...taskData,
      tenantId: this.tenantId,          // always injected
      createdAt: new Date()
    })
  }
}

// Schema validation enforces tenantId presence
db.runCommand({
  collMod: "tasks",
  validator: {
    $jsonSchema: {
      required: ["tenantId", "projectId", "title", "status"],
      properties: {
        tenantId: { bsonType: "objectId" },
        title: { bsonType: "string", minLength: 1, maxLength: 500 }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})
```

---

## Scenario 7: Financial Ledger / Transaction History

**Scenario**: A digital wallet application. Users can send/receive money. Requirements:
- Every balance change must be recorded (immutable ledger)
- Current balance must be queryable instantly (no recalculation from history)
- Transactions between users must be atomic
- Regulatory requirement: 7 years of transaction history

**Answer**:

```javascript
// User wallet document
{
  _id: ObjectId("..."),
  userId: ObjectId("..."),
  currency: "USD",
  balance: NumberDecimal("1250.75"),    // current balance, always consistent
  version: 42,                           // optimistic locking version
  lastTransactionAt: ISODate("2024-03-01T15:30:00Z")
}

// Transaction document (immutable ledger entry)
{
  _id: ObjectId("..."),
  transactionId: "TXN-20240301-000001",  // human-readable reference
  type: "transfer",                       // deposit | withdrawal | transfer | fee | refund
  status: "completed",                    // pending | completed | failed | reversed

  // Double-entry bookkeeping: debit + credit
  entries: [
    {
      walletId: ObjectId("alice_wallet"),
      direction: "debit",
      amount: NumberDecimal("100.00"),
      balanceBefore: NumberDecimal("1350.75"),
      balanceAfter:  NumberDecimal("1250.75")
    },
    {
      walletId: ObjectId("bob_wallet"),
      direction: "credit",
      amount: NumberDecimal("100.00"),
      balanceBefore: NumberDecimal("850.00"),
      balanceAfter:  NumberDecimal("950.00")
    }
  ],

  fee: NumberDecimal("0.00"),
  description: "Payment for coffee",
  referenceId: "ORDER-2024-0301-1234",

  initiatedAt: ISODate("2024-03-01T15:30:00Z"),
  completedAt: ISODate("2024-03-01T15:30:01Z"),

  metadata: {
    ipAddress: "192.168.1.1",
    deviceId: "iphone-abc123",
    location: { type: "Point", coordinates: [-73.9857, 40.7484] }
  }
}

// Atomic transfer with multi-document transaction
async function transferFunds(fromWalletId, toWalletId, amount, description) {
  const session = client.startSession()

  try {
    await session.withTransaction(async () => {
      const amountDecimal = NumberDecimal(amount.toString())

      // Debit sender
      const sender = await db.wallets.findOneAndUpdate(
        {
          _id: ObjectId(fromWalletId),
          balance: { $gte: amountDecimal }  // sufficient funds check
        },
        { $inc: { balance: amountDecimal.negated(), version: 1 },
          $set: { lastTransactionAt: new Date() } },
        { session, returnDocument: "after" }
      )
      if (!sender) throw new Error("Insufficient funds or wallet not found")

      // Credit receiver
      const receiver = await db.wallets.findOneAndUpdate(
        { _id: ObjectId(toWalletId) },
        { $inc: { balance: amountDecimal, version: 1 },
          $set: { lastTransactionAt: new Date() } },
        { session, returnDocument: "after" }
      )
      if (!receiver) throw new Error("Recipient wallet not found")

      // Create immutable ledger entry
      await db.transactions.insertOne({
        transactionId: generateTransactionId(),
        type: "transfer",
        status: "completed",
        entries: [
          { walletId: ObjectId(fromWalletId), direction: "debit",  amount: amountDecimal,
            balanceBefore: sender.balance + amountDecimal, balanceAfter: sender.balance },
          { walletId: ObjectId(toWalletId),   direction: "credit", amount: amountDecimal,
            balanceBefore: receiver.balance - amountDecimal, balanceAfter: receiver.balance }
        ],
        description,
        initiatedAt: new Date(),
        completedAt: new Date()
      }, { session })
    })
  } finally {
    await session.endSession()
  }
}

// Archival for 7-year retention (Document Versioning approach)
// Use Atlas Data Archiving or move to cold storage after 1 year
db.transactions.createIndex(
  { completedAt: 1 },
  { expireAfterSeconds: 220752000 }   // 7 years = 7 × 365 × 24 × 3600
)
```

---

## Scenario 8: Content Management System (CMS) with Versioning

**Scenario**: A CMS where editors create/update articles. Requirements:
- Full edit history (revert to any version)
- Draft → Review → Published workflow
- Scheduled publishing (publish at future date)
- Multiple language versions per article

**Answer**:

```javascript
// Current (live) article document
{
  _id: ObjectId("..."),
  articleId: "article-uuid-123",       // stable business ID across versions
  slug: "getting-started-with-mongodb",

  currentVersion: 8,
  status: "published",                  // draft | in_review | scheduled | published | archived

  // Localized content using embedded map
  content: {
    "en": {
      title: "Getting Started with MongoDB",
      body: "...(full HTML/Markdown content)...",
      excerpt: "Learn MongoDB in 10 minutes.",
      metaTitle: "MongoDB Tutorial for Beginners",
      metaDescription: "Step-by-step guide to MongoDB."
    },
    "es": {
      title: "Primeros Pasos con MongoDB",
      body: "...",
      excerpt: "Aprende MongoDB en 10 minutos."
    }
  },

  defaultLocale: "en",
  availableLocales: ["en", "es"],

  author: { _id: ObjectId("..."), name: "Alice Johnson" },
  lastEditedBy: { _id: ObjectId("..."), name: "Bob Smith" },

  publishedAt: ISODate("2024-03-01T09:00:00Z"),
  scheduledAt: null,                    // null = not scheduled

  tags: ["mongodb", "tutorial", "nosql"],
  category: "tutorials",

  createdAt: ISODate("2024-02-20T10:00:00Z"),
  updatedAt: ISODate("2024-03-01T08:55:00Z")
}

// Version history document (Document Versioning Pattern)
{
  _id: ObjectId("..."),
  articleId: "article-uuid-123",
  version: 7,                           // version number

  // Snapshot of content at this version
  content: {
    "en": { title: "...", body: "..." }
  },

  changedBy: { _id: ObjectId("..."), name: "Alice Johnson" },
  changeType: "content_update",         // content_update | status_change | locale_added
  changeSummary: "Fixed typo in introduction",

  createdAt: ISODate("2024-02-28T16:00:00Z"),

  // Only keep diffs for large articles to save space (optional)
  // diff: { ... unified diff format ... }
}

// Indexes
db.articles.createIndex({ slug: 1 }, { unique: true })
db.articles.createIndex({ status: 1, scheduledAt: 1 })   // scheduler job
db.articles.createIndex({ tags: 1, status: 1, publishedAt: -1 })
db.articles.createIndex({ category: 1, status: 1, publishedAt: -1 })

db.articleVersions.createIndex({ articleId: 1, version: -1 })
db.articleVersions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 31536000 * 2 }  // keep 2 years of history
)

// Revert to previous version
async function revertToVersion(articleId, targetVersion, editorId) {
  const historicalVersion = await db.articleVersions.findOne({
    articleId, version: targetVersion
  })
  if (!historicalVersion) throw new Error(`Version ${targetVersion} not found`)

  const current = await db.articles.findOne({ articleId })

  // Save current state as a new version before reverting
  await db.articleVersions.insertOne({
    articleId,
    version: current.currentVersion,
    content: current.content,
    changedBy: { _id: ObjectId(editorId), name: "System" },
    changeType: "pre_revert_snapshot",
    changeSummary: `Auto-snapshot before revert to v${targetVersion}`,
    createdAt: new Date()
  })

  // Apply reverted content
  await db.articles.updateOne(
    { articleId },
    {
      $set: {
        content: historicalVersion.content,
        status: "draft",
        updatedAt: new Date(),
        lastEditedBy: { _id: ObjectId(editorId) }
      },
      $inc: { currentVersion: 1 }
    }
  )
}
```

---

## Scenario 9: Real-Time Gaming Leaderboard

**Scenario**: A mobile game with daily/weekly/all-time leaderboards. 5 million active players. Requirements:
- Update score atomically (score can only go up)
- Query top 100 players globally
- Query player's rank (position in leaderboard)
- Leaderboards reset daily and weekly

**Answer**:

```javascript
// Player score document — one per player per leaderboard period
{
  _id: ObjectId("..."),
  playerId: ObjectId("..."),
  username: "speedrunner99",
  avatarUrl: "https://cdn.example.com/avatars/speedrunner99.jpg",

  // Separate scores per time window
  scores: {
    allTime:  { score: 48250, updatedAt: ISODate("2024-03-01T14:00:00Z") },
    weekly:   { score: 1820,  updatedAt: ISODate("2024-03-01T14:00:00Z"), weekStart: ISODate("2024-02-26T00:00:00Z") },
    daily:    { score: 350,   updatedAt: ISODate("2024-03-01T14:00:00Z"), dayStart:  ISODate("2024-03-01T00:00:00Z") }
  },

  level: 47,
  country: "US"
}

// Indexes for leaderboard queries
db.leaderboard.createIndex({ "scores.allTime.score": -1 })
db.leaderboard.createIndex({ "scores.daily.score": -1, "scores.daily.dayStart": 1 })
db.leaderboard.createIndex({ "scores.weekly.score": -1, "scores.weekly.weekStart": 1 })
db.leaderboard.createIndex({ country: 1, "scores.allTime.score": -1 })   // country leaderboard

// Atomic score update (score can only increase)
db.leaderboard.updateOne(
  { playerId: ObjectId(playerId) },
  {
    $max: {
      "scores.allTime.score":  newScore,    // $max: only updates if newScore > current
      "scores.daily.score":    newDailyScore,
      "scores.weekly.score":   newWeeklyScore
    },
    $set: {
      "scores.allTime.updatedAt":  new Date(),
      "scores.daily.dayStart":     todayStart,
      "scores.weekly.weekStart":   weekStart
    }
  },
  { upsert: true }
)

// Top 100 query (fast — uses index)
db.leaderboard.find(
  { "scores.daily.dayStart": todayStart },
  { username: 1, avatarUrl: 1, "scores.daily.score": 1, country: 1 }
).sort({ "scores.daily.score": -1 }).limit(100)

// Player's rank (expensive — count docs with higher score)
// For exact rank: use Atlas Search or pre-compute rank in background job
const rank = await db.leaderboard.countDocuments({
  "scores.allTime.score": { $gt: playerScore }
}) + 1

// Daily reset: TTL index clears daily scores automatically
// OR: background job sets daily score to 0 at midnight
// BETTER: use a separate "dailyScores" collection with TTL
db.createCollection("dailyScores")
db.dailyScores.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 * 2 })  // 2-day TTL
db.dailyScores.createIndex({ score: -1, date: 1 })
```

---

## Scenario 10: Healthcare — Patient Records

**Scenario**: Electronic health records (EHR) system. Patients have:
- Demographics (name, DOB, contact info)
- Medical history (diagnoses, allergies, medications — historical, append-only)
- Visits (each visit has notes, prescriptions, lab results)
- Multiple doctors and care teams

Compliance: HIPAA, data must be auditable, no data deletion (soft delete only).

**Answer**:

```javascript
// Patient document — core demographics
{
  _id: ObjectId("..."),
  patientMRN: "MRN-2024-000001",       // Medical Record Number (unique, immutable)

  demographics: {
    firstName: "Alice",
    lastName: "Johnson",
    dateOfBirth: ISODate("1985-06-15T00:00:00Z"),
    gender: "female",
    ssn: "ENCRYPTED:AES256:BASE64STRING"   // encrypted at application layer
  },

  contact: {
    phone: "ENCRYPTED:...",
    email: "ENCRYPTED:...",
    address: {
      street: "ENCRYPTED:...",
      city: "New York",
      state: "NY",
      zipCode: "10001"
    }
  },

  // Active medical summary (Computed Pattern — updated on changes)
  activeSummary: {
    allergies: [
      { substance: "Penicillin", severity: "severe", reaction: "anaphylaxis" }
    ],
    currentMedications: [
      { name: "Metformin", dosage: "500mg", frequency: "twice daily", prescribedBy: ObjectId("...") }
    ],
    activeConditions: ["Type 2 Diabetes", "Hypertension"],
    bloodType: "O+",
    lastVisitDate: ISODate("2024-02-15T00:00:00Z")
  },

  careTeam: [
    { doctorId: ObjectId("..."), role: "primary_care_physician", since: ISODate("2020-01-01T00:00:00Z") }
  ],

  isActive: true,
  createdAt: ISODate("2020-01-15T10:00:00Z"),
  updatedAt: ISODate("2024-02-15T14:30:00Z"),
  schemaVersion: 3
}

// Visit document (separate collection — one per visit)
{
  _id: ObjectId("..."),
  visitId: "VIS-20240215-001234",
  patientId: ObjectId("..."),
  doctorId: ObjectId("..."),

  visitDate: ISODate("2024-02-15T10:00:00Z"),
  visitType: "follow_up",              // new_patient | follow_up | emergency | telehealth

  chiefComplaint: "Routine diabetes management follow-up",

  vitalSigns: {
    bloodPressure: { systolic: 128, diastolic: 82 },
    heartRate: 72,
    temperature: 98.6,
    weight: { value: 165, unit: "lbs" },
    height: { value: 65, unit: "inches" }
  },

  // SOAP notes
  assessment: "T2DM well-controlled. BP slightly elevated.",
  plan: "Continue Metformin. Recheck in 3 months. Dietary counseling referral.",

  diagnoses: [
    { code: "E11.9", system: "ICD-10", description: "Type 2 diabetes mellitus without complications" }
  ],

  prescriptions: [
    {
      medicationName: "Metformin",
      dosage: "500mg",
      frequency: "twice daily",
      quantity: 180,
      refills: 3,
      prescribedAt: ISODate("2024-02-15T11:00:00Z")
    }
  ],

  labOrders: [ ObjectId("...") ],      // references to lab results

  // Audit trail (append-only, never delete)
  auditLog: [
    { action: "created",     by: ObjectId("doctor_id"), at: ISODate("2024-02-15T10:00:00Z") },
    { action: "note_added",  by: ObjectId("doctor_id"), at: ISODate("2024-02-15T11:30:00Z") },
    { action: "signed",      by: ObjectId("doctor_id"), at: ISODate("2024-02-15T11:45:00Z") }
  ],

  isSigned: true,
  isDeleted: false                     // soft delete ONLY — HIPAA requirement
}

// Schema Versioning for EHR evolution
// When adding new required fields, use schemaVersion + lazy migration
db.patients.createIndex({ patientMRN: 1 }, { unique: true })
db.patients.createIndex({ "careTeam.doctorId": 1 })
db.patients.createIndex({ isActive: 1, "demographics.lastName": 1 })

db.visits.createIndex({ patientId: 1, visitDate: -1 })
db.visits.createIndex({ doctorId: 1, visitDate: -1 })
db.visits.createIndex({ isDeleted: 1, isSigned: 1, visitDate: -1 })
```

---

## Scenario 11: Ride-Sharing — Driver Location Tracking

**Scenario**: Ride-sharing app (Uber/Lyft model). Requirements:
- Drivers send GPS location every 5 seconds
- Find available drivers within 5km of a passenger
- Track a ride's path (sequence of GPS points)
- Query: how many drivers are in each city zone?

**Answer**:

```javascript
// Driver real-time location (hot data — updated every 5 seconds)
{
  _id: ObjectId("..."),
  driverId: ObjectId("..."),
  status: "available",                  // available | on_trip | offline

  currentLocation: {
    type: "Point",
    coordinates: [-73.9857, 40.7484]   // [longitude, latitude] — GeoJSON format
  },

  heading: 45,                          // degrees from north
  speed: 25,                            // mph

  vehicle: {
    type: "sedan",                      // sedan | suv | xl | luxury
    make: "Toyota",
    model: "Camry",
    licensePlate: "ABC-123",
    color: "silver"
  },

  rating: 4.87,
  totalRides: 2341,

  updatedAt: ISODate("2024-03-01T15:30:00.000Z")
}

// 2dsphere index for geospatial queries
db.driverLocations.createIndex({ currentLocation: "2dsphere" })
db.driverLocations.createIndex({ status: 1, currentLocation: "2dsphere" })

// Find available drivers within 5km
db.driverLocations.find({
  status: "available",
  currentLocation: {
    $nearSphere: {
      $geometry: {
        type: "Point",
        coordinates: [-73.9720, 40.7589]   // passenger location
      },
      $maxDistance: 5000                    // 5km in meters
    }
  }
}).limit(10)

// Ride route tracking (time-series collection for GPS path)
db.createCollection("rideRoutes", {
  timeseries: {
    timeField: "timestamp",
    metaField: "rideId",
    granularity: "seconds"
  },
  expireAfterSeconds: 2592000   // 30 days retention
})

db.rideRoutes.insertOne({
  rideId: ObjectId("..."),
  timestamp: ISODate("2024-03-01T15:30:00Z"),
  location: {
    type: "Point",
    coordinates: [-73.9857, 40.7484]
  },
  speed: 25,
  accuracy: 5                              // GPS accuracy in meters
})

// Reconstruct ride path
db.rideRoutes.find(
  { rideId: ObjectId(rideId) },
  { location: 1, timestamp: 1, speed: 1 }
).sort({ timestamp: 1 })

// Count drivers by city zone (aggregation)
db.driverLocations.aggregate([
  { $match: { status: "available" } },
  { $geoNear: {
      near: { type: "Point", coordinates: [-73.9857, 40.7484] },
      distanceField: "distanceMeters",
      maxDistance: 50000,          // 50km
      spherical: true
  }},
  { $bucket: {
      groupBy: "$distanceMeters",
      boundaries: [0, 1000, 5000, 10000, 20000, 50000],
      default: "far",
      output: {
        count: { $sum: 1 },
        avgRating: { $avg: "$rating" }
      }
  }}
])
```

---

## Scenario 12: E-Learning Platform — Course Progress

**Scenario**: Online learning platform. Students take courses with modules and lessons. Requirements:
- Track which lessons a student has completed
- Store quiz scores per attempt (multiple attempts allowed)
- Calculate course completion percentage
- Query: courses a student is currently taking
- Query: all students who haven't completed a course in 30+ days

**Answer**:

```javascript
// Course enrollment and progress — one document per student+course
{
  _id: ObjectId("..."),
  studentId: ObjectId("..."),
  courseId: ObjectId("..."),

  enrolledAt: ISODate("2024-01-15T00:00:00Z"),
  lastAccessedAt: ISODate("2024-02-28T14:00:00Z"),

  status: "in_progress",              // enrolled | in_progress | completed | dropped
  completionPercentage: 68,           // Computed Pattern — updated on each lesson complete
  completedAt: null,

  // Lesson progress (bounded — courses have max 200 lessons)
  lessonProgress: [
    {
      lessonId: ObjectId("..."),
      moduleId: ObjectId("..."),
      status: "completed",
      completedAt: ISODate("2024-02-01T10:00:00Z"),
      timeSpentSeconds: 1840,
      watchPercentage: 100
    },
    {
      lessonId: ObjectId("..."),
      moduleId: ObjectId("..."),
      status: "in_progress",
      lastWatchedAt: ISODate("2024-02-28T14:00:00Z"),
      watchPercentage: 45
    }
  ],

  // Quiz attempts (bounded — max 3 attempts per quiz × max 20 quizzes = 60 entries)
  quizAttempts: [
    {
      quizId: ObjectId("..."),
      lessonId: ObjectId("..."),
      attemptNumber: 1,
      score: 75,
      passed: false,
      answers: [ { questionId: ObjectId("..."), selected: 2, correct: false } ],
      submittedAt: ISODate("2024-02-05T11:00:00Z")
    },
    {
      quizId: ObjectId("..."),
      lessonId: ObjectId("..."),
      attemptNumber: 2,
      score: 90,
      passed: true,
      submittedAt: ISODate("2024-02-06T09:00:00Z")
    }
  ],

  certificate: null    // set when course completed
}

// Indexes
db.enrollments.createIndex({ studentId: 1, courseId: 1 }, { unique: true })
db.enrollments.createIndex({ studentId: 1, status: 1, lastAccessedAt: -1 })
db.enrollments.createIndex({ courseId: 1, status: 1, completionPercentage: -1 })
db.enrollments.createIndex(
  { status: 1, lastAccessedAt: 1 },
  { partialFilterExpression: { status: "in_progress" } }
)  // only index active enrollments

// Students inactive 30+ days
const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
db.enrollments.find({
  status: "in_progress",
  lastAccessedAt: { $lt: thirtyDaysAgo }
}, { studentId: 1, courseId: 1, lastAccessedAt: 1 })
```

---

## Scenario 13: Schema Migration in Production

**Scenario**: Your `users` collection has `fullName: "Alice Johnson"` (single field). The new app version needs `firstName: "Alice"` and `lastName: "Johnson"` separately. You have 50 million user documents. How do you migrate without downtime?

**Answer**:

```javascript
// Step 1: Schema Versioning — add schemaVersion field
// Don't touch existing data yet!
// Current docs: { _id, fullName: "Alice Johnson" }  ← schemaVersion missing (treated as v1)

// Step 2: Update application code to handle BOTH formats
function getUserName(user) {
  if (user.schemaVersion >= 2) {
    // New format
    return { firstName: user.firstName, lastName: user.lastName }
  } else {
    // Old format — parse on read
    const parts = (user.fullName || "").split(" ")
    return { firstName: parts[0] || "", lastName: parts.slice(1).join(" ") || "" }
  }
}

function saveUser(user, firstName, lastName) {
  return db.users.updateOne(
    { _id: user._id },
    {
      $set: {
        firstName,
        lastName,
        schemaVersion: 2          // upgrade on write
      },
      $unset: { fullName: "" }    // remove old field
    }
  )
}
// Deploy this code first — now app handles both v1 and v2 docs

// Step 3: Background migration script (runs during off-peak hours)
async function migrateUsers() {
  let migrated = 0
  const batchSize = 1000

  while (true) {
    // Find unmigrated docs (schemaVersion not set = v1)
    const batch = await db.users.find(
      { schemaVersion: { $exists: false } },
      { fullName: 1 }
    ).limit(batchSize).toArray()

    if (batch.length === 0) break   // done!

    const bulkOps = batch.map(user => {
      const parts = (user.fullName || "").split(" ")
      return {
        updateOne: {
          filter: { _id: user._id, schemaVersion: { $exists: false } },  // idempotent check
          update: {
            $set: { firstName: parts[0] || "", lastName: parts.slice(1).join(" ") || "", schemaVersion: 2 },
            $unset: { fullName: "" }
          }
        }
      }
    })

    await db.users.bulkWrite(bulkOps, { ordered: false })
    migrated += batch.length
    console.log(`Migrated: ${migrated} users`)

    // Small delay to reduce load
    await new Promise(r => setTimeout(r, 100))
  }

  console.log(`Migration complete: ${migrated} total`)
}

// Step 4: After all docs migrated, verify
const v1Count = await db.users.countDocuments({ schemaVersion: { $exists: false } })
console.log(`Remaining v1 docs: ${v1Count}`)  // should be 0

// Step 5: Create new index on firstName + lastName
db.users.createIndex({ lastName: 1, firstName: 1 })
// Step 6: Drop old fullName index if it existed
// Step 7: Optionally remove backward-compat code from app after full migration confirmed
```

---

## Scenario 14: Product Reviews with Aggregated Ratings

**Scenario**: An e-commerce site where products have user reviews. Requirements:
- Products need average rating, rating distribution (1-5 stars count), and total review count
- These stats show on every product page (high-read traffic)
- New reviews are added frequently
- Reviews also need to be searchable by rating, date, verified purchase

**Answer**:

```javascript
// Product document — Computed Pattern for stats
{
  _id: ObjectId("..."),
  name: "Blue Widget",

  // Pre-computed review stats — updated on every new review
  reviewStats: {
    totalReviews: 1247,
    averageRating: NumberDecimal("4.32"),
    ratingDistribution: {
      "1": 45,
      "2": 63,
      "3": 189,
      "4": 398,
      "5": 552
    },
    verifiedPurchaseCount: 983,
    lastReviewAt: ISODate("2024-03-01T09:00:00Z")
  }
}

// Review document — separate collection
{
  _id: ObjectId("..."),
  productId: ObjectId("..."),
  authorId: ObjectId("..."),

  rating: 5,                  // 1-5 integer
  title: "Best widget I've ever bought!",
  body: "Excellent quality, fast shipping, exceeded expectations.",

  isVerifiedPurchase: true,
  orderId: ObjectId("..."),

  helpful: { votes: 45, total: 52 },   // helpful voting

  images: [
    "https://cdn.example.com/reviews/img-123.jpg"
  ],

  status: "approved",         // pending | approved | rejected | reported
  createdAt: ISODate("2024-03-01T09:00:00Z"),
  updatedAt: ISODate("2024-03-01T09:00:00Z")
}

// When a new review is added, update computed stats atomically
async function addReview(productId, rating, reviewData) {
  const session = client.startSession()

  await session.withTransaction(async () => {
    // Insert the review
    await db.reviews.insertOne({ productId: ObjectId(productId), rating, ...reviewData }, { session })

    // Update pre-computed stats atomically
    const ratingField = `reviewStats.ratingDistribution.${rating}`
    await db.products.updateOne(
      { _id: ObjectId(productId) },
      {
        $inc: {
          "reviewStats.totalReviews": 1,
          [ratingField]: 1
        },
        $set: { "reviewStats.lastReviewAt": new Date() }
      },
      { session }
    )

    // Recalculate average (can't do atomically with $inc — use background job or pipeline update)
    // Option: use pipeline update (MongoDB 4.2+) to recalculate averageRating in-place
  })

  await session.endSession()
}

db.reviews.createIndex({ productId: 1, status: 1, createdAt: -1 })
db.reviews.createIndex({ productId: 1, rating: 1, createdAt: -1 })
db.reviews.createIndex({ productId: 1, isVerifiedPurchase: 1, rating: -1 })
db.reviews.createIndex({ authorId: 1, createdAt: -1 })
// One review per customer per product
db.reviews.createIndex(
  { productId: 1, authorId: 1 },
  { unique: true, partialFilterExpression: { status: { $ne: "rejected" } } }
)
```

---

## Scenario 15: Real-Time Notifications System

**Scenario**: In-app notification system. Users receive notifications for: replies, likes, follows, system announcements. Requirements:
- Mark individual notifications as read
- Mark ALL notifications as read at once
- Show unread count in UI
- Notifications expire after 30 days
- Some notifications (system announcements) are broadcast to all users

**Answer**:

```javascript
// User notification inbox — one document per user+notification
{
  _id: ObjectId("..."),
  recipientId: ObjectId("..."),

  // Notification payload
  type: "reply",               // reply | like | follow | mention | system
  title: "Bob replied to your post",
  body: "This is a great tutorial! I learned a lot.",

  // Rich data for rendering
  actor: { _id: ObjectId("..."), username: "bob", avatarUrl: "..." },
  subject: {
    type: "post",
    id: ObjectId("..."),
    title: "Getting Started with MongoDB",
    url: "/posts/getting-started-with-mongodb"
  },

  isRead: false,
  readAt: null,

  priority: "normal",          // low | normal | high | urgent

  createdAt: ISODate("2024-03-01T10:00:00Z"),
  expiresAt: ISODate("2024-03-31T10:00:00Z")   // 30-day TTL
}

// Indexes
db.notifications.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })  // TTL
db.notifications.createIndex({ recipientId: 1, isRead: 1, createdAt: -1 }) // inbox query
db.notifications.createIndex(
  { recipientId: 1, createdAt: -1 },
  { partialFilterExpression: { isRead: false } }
)   // unread count (partial — only unread docs indexed here)

// Unread count (fast — uses partial index)
const unreadCount = await db.notifications.countDocuments({
  recipientId: ObjectId(userId),
  isRead: false
})

// Mark all read (updateMany with index)
await db.notifications.updateMany(
  { recipientId: ObjectId(userId), isRead: false },
  { $set: { isRead: true, readAt: new Date() } }
)

// System announcement to all users — DON'T create N documents
// Instead: use a global announcements collection + track dismissed per user
{
  // Global announcements collection
  _id: ObjectId("..."),
  type: "system_announcement",
  title: "Scheduled Maintenance Tonight",
  body: "The platform will be down from 2-4 AM UTC for maintenance.",
  priority: "high",
  targetAudience: "all",      // all | premium | free
  createdAt: ISODate("2024-03-01T10:00:00Z"),
  expiresAt: ISODate("2024-03-02T10:00:00Z")
}

// User dismissed announcements (small array — few global announcements at a time)
db.users.updateOne(
  { _id: ObjectId(userId) },
  { $addToSet: { dismissedAnnouncements: ObjectId(announcementId) } }
)

// Fetch user's notification feed (personal + global undismissed)
const [personal, global] = await Promise.all([
  db.notifications.find(
    { recipientId: ObjectId(userId) },
    { limit: 20, sort: { createdAt: -1 } }
  ).toArray(),
  db.announcements.find({
    _id: { $nin: user.dismissedAnnouncements || [] },
    expiresAt: { $gt: new Date() },
    $or: [{ targetAudience: "all" }, { targetAudience: user.plan }]
  }).toArray()
])
```
