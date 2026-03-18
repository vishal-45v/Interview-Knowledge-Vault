# Chapter 07 — Atlas Features: Diagram Explanations

---

## Diagram 1: Atlas Architecture Overview

```
ATLAS ARCHITECTURE: MANAGED LAYERS
══════════════════════════════════════════════════════════════════════

                        YOUR APPLICATION
                              │
                              │ MongoDB wire protocol (TLS)
                              │ connection string:
                              │ mongodb+srv://user:pass@cluster0.abc.mongodb.net/
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ATLAS CONTROL PLANE                          │
│  (Atlas UI / Atlas API / Atlas CLI)                             │
│                                                                  │
│  - Cluster provisioning and lifecycle management                 │
│  - Monitoring, alerting, Performance Advisor                    │
│  - Backup scheduling and restore operations                     │
│  - User management, IP allowlists                               │
│  - Auto-scaling triggers                                        │
└──────────────────────────────┬──────────────────────────────────┘
                               │
                               │ manages
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ATLAS DATA PLANE                             │
│              (runs on AWS / GCP / Azure)                        │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │  PRIMARY    │  │ SECONDARY 1 │  │ SECONDARY 2 │             │
│  │  (voting)   │  │  (voting)   │  │  (voting)   │             │
│  │ US-East-1a  │  │ US-East-1b  │  │ US-East-1c  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
│         │               │                │                       │
│         └───────────────┴────────────────┘                      │
│                    oplog replication                             │
│                                                                  │
│  Atlas-managed services (on the same infrastructure):           │
│  ┌──────────────────┐  ┌──────────────────┐                    │
│  │  Atlas Search    │  │  Atlas Backup    │                    │
│  │  (Lucene nodes) │  │  (S3 snapshots) │                    │
│  └──────────────────┘  └──────────────────┘                    │
└─────────────────────────────────────────────────────────────────┘

Atlas handles EVERYTHING below the "your application" line:
TLS certificates, patching, replica set management, backups,
storage scaling, monitoring, version upgrades
```

---

## Diagram 2: Atlas Backup — Snapshot + Oplog → Point-in-Time Restore

```
ATLAS CLOUD BACKUP: HOW PITR WORKS
══════════════════════════════════════════════════════════════════════

Timeline (Monday):

06:00 AM  ──── SNAPSHOT A ────── Full cluster snapshot taken (10 min to complete)
          │     (50GB snapshot stored in S3)
          │
          │     Continuous oplog shipping to S3:
          │     [oplog: 06:00→06:10] [oplog: 06:10→06:20] ... every ~10 minutes
          │
12:00 PM  ──── SNAPSHOT B ────── Second scheduled snapshot
          │     (incremental from Snapshot A)
          │
          │     [oplog: 12:00→12:10] [oplog: 12:10→12:20] ...
          │
02:47:30  ──── BUG DEPLOYED → DATA CORRUPTION
PM             (attacker/bug modifies 10,000 user records)
          │
02:47:45  └──── SOMEONE NOTICES THE BUG
PM

RESTORE TO 02:47:30 PM (30 seconds before corruption):
──────────────────────────────────────────────────────────────────────

Step 1: Atlas identifies nearest snapshot BEFORE target time
        → Snapshot B (12:00 PM) is nearest snapshot before 2:47:30 PM

Step 2: Restore cluster from Snapshot B
        (creates new cluster or overwrites existing)
        Time: 5-30 minutes depending on cluster size

Step 3: Replay oplog from 12:00 PM to 2:47:30 PM
        [12:00 oplog] → [12:10 oplog] → ... → [14:40 oplog] → [14:47:30 exact]
        Time: varies with oplog volume

Step 4: Cluster is in exact state it was in at 2:47:30 PM
        Bug never happened!
        Corruption rolled back!

RPO (Recovery Point Objective): ~1 minute (oplog shipping interval)
RTO (Recovery Time Objective):  30-120 minutes (snapshot restore + oplog replay)

BACKUP RETENTION POLICY (TYPICAL PRODUCTION):
──────────────────────────────────────────────────────────────────────
Hourly snapshots:  keep 7 days    → restore to any hour in the last week
Daily snapshots:   keep 30 days   → restore to any day in the last month
Weekly snapshots:  keep 3 months  → restore to any week in the last quarter
Monthly snapshots: keep 12 months → restore to any month in the last year
```

---

## Diagram 3: Atlas Online Archive — Hot/Cold Data Tiering

```
ATLAS ONLINE ARCHIVE: DATA TIERING ARCHITECTURE
══════════════════════════════════════════════════════════════════════

BEFORE ARCHIVING:
┌─────────────────────────────────────────────────────────────────┐
│ Atlas Cluster (M50, 2TB disk, $500/month)                       │
│                                                                  │
│ orders collection:                                               │
│ ├── orders from 2024 (last 90 days): 100GB   ← HOT, frequent   │
│ ├── orders from 2023: 400GB                  ← COLD, rare      │
│ ├── orders from 2022: 400GB                  ← COLD, very rare │
│ ├── orders from 2021: 400GB                  ← COLD, audit only│
│ └── orders from 2020-2017: 700GB             ← COMPLIANCE ONLY │
│                                                                  │
│ Total: 2TB, $500/month                                          │
└─────────────────────────────────────────────────────────────────┘

AFTER ARCHIVING (criteria: createdAt > 90 days):
┌──────────────────────────────┐    ┌─────────────────────────────┐
│ Atlas Cluster (M30, 200GB)   │    │ S3 Archive                  │
│ $200/month                   │    │ 1.8TB                       │
│                               │    │ $41/month                   │
│ orders from last 90 days      │    │                             │
│ (HOT data, indexed, fast)     │    │ All older orders            │
│                               │    │ (partitioned by date/user)  │
└──────────────┬────────────────┘    └──────────────┬──────────────┘
               │                                    │
               └──────────────┬─────────────────────┘
                              │ ATLAS DATA FEDERATION
                              │ Unified query endpoint:
                              │ data.mongodb.net
                              │
                              ▼
              Application queries both sources transparently!

db.orders.find({ createdAt: { $gte: 2017-01-01, $lt: 2024-01-01 } })
→ Data Federation routes:
  2024 data → Atlas Cluster (fast: 5ms)
  2017-2023 → S3 Archive (slow: 5-30 seconds)
→ Results merged and returned

COST SAVINGS:
Before: 2TB cluster × $0.25/GB = $500/month
After:  200GB cluster × $0.25 + 1.8TB archive × $0.023 = $50 + $41 = $91/month
SAVINGS: $409/month (82% reduction!) ← massive for long-lived data
```

---

## Diagram 4: Atlas Search vs Native $text — Feature Comparison

```
ATLAS SEARCH vs NATIVE $TEXT INDEX
══════════════════════════════════════════════════════════════════════

QUERY: User types "mongod performen" (two typos) looking for "MongoDB performance"

NATIVE $TEXT INDEX:
──────────────────────────────────────────────────────────────────────
db.articles.find({ $text: { $search: "mongod performen" } })

Processing:
"mongod" → exact match in index? NO (token doesn't exist) → no results
"performen" → exact match in index? NO (not in index) → no results
Result: []  ← 0 results! Useless for user with typo

ATLAS SEARCH (fuzzy + stemming):
──────────────────────────────────────────────────────────────────────
db.articles.aggregate([{
  $search: {
    text: {
      query: "mongod performen",
      path: ["title", "body"],
      fuzzy: { maxEdits: 1, prefixLength: 3 }
    }
  }
}])

Processing (Lucene BM25 algorithm):
"mongod" → 1 edit from "mongodb"? YES (d→db: 1 edit) → match!
"performen" → 1 edit from "performance"? "performan" → close, within fuzzy
             → stems to "perform" → matches "performance", "performer"
Result: [
  { title: "MongoDB Performance Tuning", score: 8.4 },
  { title: "MongoDB Index Performance",  score: 7.1 },
  { title: "MongoDB Query Optimization", score: 4.2 }
]

FEATURE MATRIX:
──────────────────────────────────────────────────────────────────────
Feature                   │ $text Index │ Atlas Search
──────────────────────────┼─────────────┼────────────────────────────
Fuzzy/typo tolerance      │     ✗       │ ✓ (maxEdits param)
Autocomplete              │     ✗       │ ✓ ($autocomplete)
Facet counts              │     ✗       │ ✓ ($searchMeta)
Relevance scoring         │  Basic(score)│ ✓ (BM25, boosts)
Custom analyzers          │     ✗       │ ✓ (custom tokenizers)
Geo + text combined       │     ✗       │ ✓
Highlighting              │     ✗       │ ✓ (show matched terms)
Phrase matching           │     ✓       │ ✓
Multiple collections      │     ✗       │ ✓
Synonym support           │     ✗       │ ✓
Sort by score             │     ✓       │ ✓
Available on M0 tier      │     ✓       │ ✗ (M10+ only)
Additional cost           │     ✗       │ ✓ (Lucene infrastructure)
Sync delay after write    │     sync    │ async (seconds)
```

---

## Diagram 5: Atlas Global Clusters — Zone-Based Sharding

```
ATLAS GLOBAL CLUSTERS: DATA RESIDENCY + LOW LATENCY
══════════════════════════════════════════════════════════════════════

                     Global Application Cluster
                     Shard key: { region: 1, _id: 1 }

US Zone                EU Zone                AP Zone
──────────             ──────────             ──────────
┌──────────┐           ┌──────────┐           ┌──────────┐
│ US-East  │           │ EU-West  │           │AP-SE     │
│ Primary  │           │ Primary  │           │ Primary  │
│  Shard A │           │  Shard B │           │  Shard C │
│          │           │          │           │          │
│ region:  │           │ region:  │           │ region:  │
│  "US"    │           │  "EU"    │           │  "AP"    │
└──────────┘           └──────────┘           └──────────┘
     │                      │                      │
     │◄─ US users ─────────►│◄─ EU users ─────────►│◄─ AP users ─►
     │   write/read locally  │   write/read locally  │  write/read
     │   < 5ms latency       │   < 5ms latency       │  < 5ms latency

WRITE ROUTING:
User in New York writes profile:
  { region: "US", _id: ObjectId(), name: "Alice" }
  → mongos reads region: "US" → routes to Shard A (US-East)
  → Alice's primary shard write: 5ms RTT ✓

User in Berlin writes profile:
  { region: "EU", _id: ObjectId(), name: "Klaus" }
  → mongos reads region: "EU" → routes to Shard B (EU-West)
  → Klaus's primary shard write: 5ms RTT ✓

GDPR COMPLIANCE:
  EU users' documents physically reside in EU-West data center
  No EU personal data crosses the Atlantic
  Meets GDPR Article 44 (transfer restrictions) ✓

CROSS-ZONE READ:
Alice (US zone) views Klaus's (EU zone) public profile:
  → mongos routes to Shard B (EU-West)
  → 120ms round-trip (cross-Atlantic)
  → Acceptable for rare cross-region reads (user profile views)

NON-ZONE-AWARE QUERY (global aggregation):
  db.users.aggregate([{ $group: { _id: "$country", count: { $sum: 1 } } }])
  → mongos queries ALL shards (US + EU + AP)
  → Merges results at mongos
  → Slow: involves all regions, all network hops
  → Use sparingly for global analytics
```

---

## Diagram 6: Atlas Vector Search — RAG Architecture

```
RAG (RETRIEVAL AUGMENTED GENERATION) ARCHITECTURE
══════════════════════════════════════════════════════════════════════

INGESTION PIPELINE (one-time setup):
──────────────────────────────────────────────────────────────────────

                    Your Documentation / Knowledge Base
                              │
                              ▼
                    Text Chunking (500-word chunks)
                              │
                              ▼
               AI Embedding Model (e.g., OpenAI Ada-002)
                              │
                              ▼
              1536-dimensional vector per chunk: [0.001, -0.023, ...]
                              │
                              ▼
                    MongoDB Collection:
         { title: "...", body: "...", embedding: [0.001, -0.023, ...] }
                              │
                              ▼
                    Atlas Vector Search Index
                    (numDimensions: 1536, similarity: "cosine")

QUERY PIPELINE (every user question):
──────────────────────────────────────────────────────────────────────

User: "How do I use MongoDB Atlas Search?"
      │
      ▼
AI Embedding Model → query vector: [0.024, -0.008, ...]
      │
      ▼
$vectorSearch:
  find 5 most similar chunks in the index
      │
      ▼
Retrieved chunks:
  [1] "Atlas Search integrates with Atlas using $search stage..." (score: 0.94)
  [2] "Creating a search index in Atlas UI..." (score: 0.91)
  [3] "Atlas Search supports fuzzy matching and autocomplete..." (score: 0.88)
  [4] "Difference between Atlas Search and native text index..." (score: 0.85)
  [5] "Atlas Search requires M10+ cluster tier..." (score: 0.82)
      │
      ▼
Prompt to LLM:
  System: "Answer based only on the provided context."
  Context: [chunk 1] + [chunk 2] + [chunk 3] + [chunk 4] + [chunk 5]
  User: "How do I use MongoDB Atlas Search?"
      │
      ▼
LLM response: "To use Atlas Search:
1. Create a search index in Atlas UI (requires M10+)
2. Use the $search aggregation stage...
[grounded in your actual documentation, not hallucinated]"

WHY VECTOR SEARCH > KEYWORD SEARCH FOR RAG:
──────────────────────────────────────────────────────────────────────
Query: "atlas full-text capabilities"
Keyword match: finds docs with exact words "atlas", "full-text", "capabilities"
Vector match:  finds docs about: Atlas Search, Lucene integration, $search stage
               even if those exact words don't appear together
→ Vector search understands MEANING, not just word matching
```
