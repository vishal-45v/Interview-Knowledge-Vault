# Chapter 05 — Elasticsearch: Diagram Explanations

ASCII diagrams for Elasticsearch architecture, query processing, and data flow.

---

## Diagram 1: Elasticsearch Cluster Architecture with Node Roles

```
  ELASTICSEARCH CLUSTER (production recommendation: separate node roles)

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  MASTER NODES (3 dedicated — quorum = 2)                               │
  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
  │  │  master-1    │  │  master-2    │  │  master-3    │                 │
  │  │  (elected)   │  │  (standby)   │  │  (standby)   │                 │
  │  │              │  │              │  │              │                 │
  │  │ Manages:     │  │              │  │              │                 │
  │  │ - cluster    │  │              │  │              │                 │
  │  │   state      │  │              │  │              │                 │
  │  │ - shard      │  │              │  │              │                 │
  │  │   allocation │  │              │  │              │                 │
  │  │ - index      │  │              │  │              │                 │
  │  │   creation   │  │              │  │              │                 │
  │  └──────────────┘  └──────────────┘  └──────────────┘                 │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  COORDINATING NODES (2 — load balanced by client)                      │
  │  ┌──────────────────────────────────────────────────────────────────┐  │
  │  │  coord-1                          coord-2                        │  │
  │  │  - Receives client requests        - Fan-out queries to shards   │  │
  │  │  - Scatter-gather for Search       - Merge results from shards   │  │
  │  │  - Routes writes to primary shards - No data stored              │  │
  │  └──────────────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  DATA NODES (6 — hold primary and replica shards)                      │
  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
  │  │ data-1   │ │ data-2   │ │ data-3   │ │ data-4   │ │ data-5   │   │
  │  │ P0  R2   │ │ P1  R0   │ │ P2  R1   │ │ P3  R4   │ │ P4  R3   │   │
  │  │ (hot)    │ │ (hot)    │ │ (warm)   │ │ (warm)   │ │ (warm)   │   │
  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
  │  P0-P4 = primary shards, R0-R4 = replica shards                       │
  │  No primary and its replica reside on the same node (ES guarantee)    │
  └──────────────────────────────────────────────────────────────────────────┘

  CLUSTER STATUS:
  GREEN:  All primary AND replica shards assigned
  YELLOW: All primary shards assigned, some replicas unassigned
  RED:    Some primary shards unassigned (data UNAVAILABLE)
```

**Explanation:** Separating node roles prevents master nodes from being overwhelmed by data
operations (indexing, searching), which could cause master node instability and cluster
management failures. Coordinating nodes absorb the scatter-gather overhead of large queries,
protecting data nodes for actual data operations.

---

## Diagram 2: Document Indexing Lifecycle — From Write to Searchability

```
  CLIENT writes document: PUT /products/_doc/1 {"name": "MacBook Pro"}

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  COORDINATING NODE                                                      │
  │  1. Receives request                                                    │
  │  2. Computes: shard = hash(document_id) % number_of_primary_shards     │
  │  3. Routes to primary shard owner (data-1 owns P0, doc goes to P0)     │
  └──────────────────────────────────────────────────────────────────────────┘
            │
            ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  PRIMARY SHARD (P0 on data-1)                                          │
  │                                                                         │
  │  ┌─────────────────────────────────────────────────────────────────┐   │
  │  │  In-memory indexing buffer (index.buffer_size, default 10%)    │   │
  │  │  [doc1 pending] [doc2 pending] [doc3 pending] ... ← new doc   │   │
  │  └─────────────────────────────────────────────────────────────────┘   │
  │  ┌─────────────────────────────────────────────────────────────────┐   │
  │  │  Translog (write-ahead log)                                     │   │
  │  │  [op1: index doc1] [op2: index doc2] [op3: index doc3] ← new  │   │
  │  │  fsync every 5s (default) — durability guarantee               │   │
  │  └─────────────────────────────────────────────────────────────────┘   │
  │                                                                         │
  │  t+0ms:  Document in buffer + translog. NOT searchable via Search.    │
  │          Searchable via GET /products/_doc/1 (reads from translog)    │
  │                                                                         │
  │  t+1s:   REFRESH — buffer written to new Lucene segment               │
  │          Document NOW SEARCHABLE via Search API                        │
  │  ┌─────────────────────────────────────────────────────────────────┐   │
  │  │  Lucene Segment (in-memory + mmap file)                        │   │
  │  │  [segment_1: doc1, doc2, doc3 — searchable]                    │   │
  │  └─────────────────────────────────────────────────────────────────┘   │
  │                                                                         │
  │  t+30min: FLUSH — translog fsynced to disk, cleared                   │
  │           Segment written to disk (durable)                            │
  │                                                                         │
  │  Background: SEGMENT MERGE — small segments consolidated              │
  │  [seg1][seg2][seg3][seg4] ──merge──► [seg_merged]                     │
  └──────────────────────────────────────────────────────────────────────────┘
            │
            │ Replication (async — after primary ACK)
            ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  REPLICA SHARD (R0 on data-2) — same process as primary               │
  └──────────────────────────────────────────────────────────────────────────┘
```

**Explanation:** The two-stage persistence model (buffer → segment on refresh, segment →
disk on flush) enables the 1-second near-real-time guarantee without fsyncing on every write.
The translog ensures documents are not lost between flushes. Understanding this model is
essential for debugging "document not found immediately after indexing" issues.

---

## Diagram 3: Query Execution — Scatter-Gather with Two-Phase Fetch

```
  QUERY: Search for "MacBook Pro", size=10, from=0

  Phase 1: QUERY PHASE (lightweight — collect matching doc IDs and scores)
  ┌──────────┐
  │Coordinator│ ──── Query("MacBook Pro") ─────────────────────────────────►
  │          │ ◄──── [doc1:score=0.9, doc7:score=0.7] ─── Shard P0 (data1) │
  │          │ ◄──── [doc3:score=0.85, doc9:score=0.6] ─── Shard P1 (data2) │
  │          │ ◄──── [doc2:score=0.8] ─────────────────── Shard P2 (data3) │
  │          │                                                               │
  │ Merges:  │ Global top 10 by score: doc1(0.9), doc3(0.85), doc2(0.8), ...│
  └──────────┘

  Phase 2: FETCH PHASE (heavyweight — retrieve full document _source)
  ┌──────────┐
  │Coordinator│ ──── Fetch([doc1,doc3,doc2,...]) ─── routes to shard owners ►
  │          │ ◄──── Full _source for doc1,doc2 ─── Shard P0 (data1)       │
  │          │ ◄──── Full _source for doc3       ─── Shard P1 (data2)       │
  │          │ ◄──── Full _source for doc2       ─── Shard P2 (data3)       │
  │          │                                                               │
  │ Returns: │ [{_score:0.9, _source:{...}}, {_score:0.85,...}, ...]       │
  └──────────┘ ──── Response ────────────────────────────────────────────► Client

  WHY TWO PHASES?
  ┌──────────────────────────────────────────────────────────────────────┐
  │  If we fetched full _source in phase 1:                             │
  │  Each shard would return 10 full documents (size=10 per shard)     │
  │  For 5 shards: 50 full documents transferred to coordinator        │
  │  Coordinator discards 40, returns 10                                │
  │                                                                      │
  │  With two phases:                                                   │
  │  Phase 1: each shard returns 10 × (doc_id, score) — tiny payload  │
  │  Phase 2: coordinator fetches only the 10 winners' full source     │
  │  Result: much less network transfer                                 │
  └──────────────────────────────────────────────────────────────────────┘
```

**Explanation:** The two-phase search (query + fetch) is Elasticsearch's mechanism for
efficiently returning the top N documents across distributed shards without transferring
all candidate documents' full data to the coordinator. This becomes especially important
for large `from + size` pagination — deep pagination forces shards to return `from + size`
candidates in phase 1, causing "deep pagination" performance degradation.

---

## Diagram 4: Inverted Index Structure and Term Query Execution

```
  DOCUMENTS:
  Doc1: {"name": "Wireless Bluetooth Headphones", "brand": "Sony"}
  Doc2: {"name": "Noise Cancelling Headphones",   "brand": "Bose"}
  Doc3: {"name": "Wireless Earbuds",              "brand": "Apple"}

  INVERTED INDEX for "name" field (after analysis):
  ┌──────────────────┬──────────────────────────────────────────────────┐
  │  Term            │  Posting List (doc_id: tf, positions)           │
  ├──────────────────┼──────────────────────────────────────────────────┤
  │  wireless        │  [Doc1: tf=1, pos=[0]], [Doc3: tf=1, pos=[0]]   │
  │  bluetooth       │  [Doc1: tf=1, pos=[1]]                          │
  │  headphone(s)    │  [Doc1: tf=1, pos=[2]], [Doc2: tf=1, pos=[2]]   │
  │  noise           │  [Doc2: tf=1, pos=[0]]                          │
  │  cancel(ling)    │  [Doc2: tf=1, pos=[1]]                          │
  │  earbud(s)       │  [Doc3: tf=1, pos=[1]]                          │
  └──────────────────┴──────────────────────────────────────────────────┘

  QUERY: match(name="wireless headphones")
  Step 1: Analyze query → ["wireless", "headphone"]
  Step 2: Look up "wireless" posting list → {Doc1, Doc3}
  Step 3: Look up "headphone" posting list → {Doc1, Doc2}
  Step 4: Intersect (operator:and) → {Doc1}
          OR union (operator:or) → {Doc1, Doc2, Doc3}

  QUERY: term(brand="Sony")  ← keyword field, exact match, no analysis
  Step 1: No analysis — look up "Sony" exactly in brand posting list
  Step 2: Returns Doc1 only
  → term("sony") returns nothing (case-sensitive, no normalisation on keyword fields)

  FILTER CACHE (for repeated filter clauses):
  ┌──────────────────────────────────────────────────────────────────┐
  │  filter: term(brand="Sony")                                     │
  │  Cache key: {term: {brand: "Sony"}}                             │
  │  Cached value: BitSet [1, 0, 0] (Doc1=1, Doc2=0, Doc3=0)      │
  │  Subsequent query with same filter: AND with query results      │
  │  using bit operations — extremely fast                          │
  └──────────────────────────────────────────────────────────────────┘
```

**Explanation:** The inverted index lookup is O(log N) to find a term in the term dictionary,
then O(P) to read the posting list where P is the number of documents containing the term.
The critical insight: `term` queries on `keyword` fields are exact and cheap (no analysis,
direct inverted index lookup, filter-cacheable). `match` queries on `text` fields are
analysed (potentially producing multiple terms, merged with OR/AND).

---

## Diagram 5: ILM Phase Transitions and Tier Architecture

```
  TIME ──────────────────────────────────────────────────────────────────►

  Day 0:    Index created (logs-000001, hot node, 5 shards, RF=1)
  Day 1:    Rollover → logs-000002 (max_age=1d reached)
            logs-000001 still on hot node
  Day 2:    ILM transitions logs-000001 to WARM phase:
            - Migrate to warm nodes
            - Force merge to 1 segment per shard
            - Shrink to 1 primary shard

  ┌──────────────────────────────────────────────────────────────────────┐
  │  HOT TIER (SSDs, high CPU)           logs-000003 (today)           │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ writes: ████████████████████░░░░░  reads: █████████████░░░░  │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  ├──────────────────────────────────────────────────────────────────────┤
  │  WARM TIER (SSDs, medium CPU)        logs-000001, logs-000002     │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ writes: none  reads: ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  ├──────────────────────────────────────────────────────────────────────┤
  │  COLD TIER (HDDs / searchable snapshots, low CPU)  logs ≥30d old  │
  │  ┌───────────────────────────────────────────────────────────────┐  │
  │  │ writes: none  reads: ██░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  │  │
  │  └───────────────────────────────────────────────────────────────┘  │
  ├──────────────────────────────────────────────────────────────────────┤
  │  DELETED ≥90d:    logs purged automatically by ILM                 │
  └──────────────────────────────────────────────────────────────────────┘

  ROLLOVER ALIAS PATTERN:
  ┌──────────────────────────────────────────────────────────────────────┐
  │  "logs-active" alias always points to the CURRENT WRITE index:     │
  │                                                                      │
  │  Day 0: logs-active → logs-000001 (is_write_index: true)          │
  │  Day 1: logs-active → logs-000002 (is_write_index: true)          │
  │         logs-000001: read-only, transitioning to warm             │
  │  Day 2: logs-active → logs-000003 (is_write_index: true)          │
  │                                                                      │
  │  Application ALWAYS writes to "logs-active" alias — no code change │
  └──────────────────────────────────────────────────────────────────────┘
```

**Explanation:** ILM automates the entire lifecycle of time-series indices. The rollover
mechanism creates new indices automatically based on size, age, or document count thresholds.
The tier architecture places hot data on expensive fast hardware and cold data on cheap slow
hardware, reducing infrastructure cost by 5-10x for large-scale logging. The alias pattern
is the key to making this transparent to applications.

---

## Diagram 6: Elasticsearch bool Query Structure

```
  SEARCH: "running shoes" in "footwear" category, price <= $100, NOT "sold_out"

  {
    "query": {
      "bool": {
  ┌─── "must": [   ← Documents MUST match, CONTRIBUTES to score
  │     {
  │       "multi_match": {
  │         "query": "running shoes",
  │         "fields": ["name^3", "description"]
  │       }
  │     }
  │   ],
  │
  ├─── "filter": [ ← Documents MUST match, does NOT contribute to score
  │     {           ← CACHED in filter cache for repeated use
  │       "term": {"category": "footwear"}
  │     },
  │     {
  │       "range": {"price": {"lte": 100}}
  │     }
  │   ],
  │
  ├─── "should": [ ← If present, BOOSTS score but does not require match
  │     {           ← "minimum_should_match" controls how many must match
  │       "term": {"is_featured": true}  // featured products ranked higher
  │     },
  │     {
  │       "term": {"is_new": true}       // new products ranked higher
  │     }
  │   ],
  │   "minimum_should_match": 0,  // 0 = should clauses are optional boosts
  │
  └─── "must_not": [  ← Documents MUST NOT match, does NOT contribute to score
        {
          "term": {"status": "sold_out"}
        },
        {
          "term": {"is_discontinued": true}
        }
      ]
    }
  }

  SCORE CONTRIBUTION SUMMARY:
  ┌──────────────┬────────────────────────┬──────────────────────────────────┐
  │  Clause      │  Affects Score?        │  Purpose                        │
  ├──────────────┼────────────────────────┼──────────────────────────────────┤
  │  must        │  YES — adds to score   │  Required relevance criteria    │
  │  filter      │  NO  — binary pass/fail│  Exact criteria, cached         │
  │  should      │  YES — adds to score   │  Optional relevance boost       │
  │  must_not    │  NO  — binary exclude  │  Excludes matching docs         │
  └──────────────┴────────────────────────┴──────────────────────────────────┘
```

**Explanation:** The `bool` query is the fundamental building block of complex Elasticsearch
queries. The critical operational insight is the filter/cache distinction: `filter` clauses
are cached as bit sets per shard, making repeated exact-match conditions essentially free after
the first execution. Pure `filter` queries (no `must` or `should`) have constant relevance
scores of 1.0 for all matching documents. Mix `must` for scoring + `filter` for constraints.
