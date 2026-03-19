# MongoDB & Atlas — Interview Knowledge Vault

> MongoDB 7.x focus | Atlas Cloud Platform | 200+ interview questions across 10 chapters

---

## Section Overview

This section covers MongoDB from fundamentals through advanced Atlas cloud features. Questions range from junior (document model, CRUD) to staff-level (sharding strategy, Atlas Search tuning, CSFLE). Each chapter includes theory questions, scenario problems, follow-up traps interviewers use to catch memorizers, structured answers, analogies, and ASCII diagrams.

**MongoDB Version Coverage:** 7.x primary, with notes for 5.0/6.0 breaking changes (resharding, time-series, etc.)
**Atlas Coverage:** Atlas Free Tier through M10+ dedicated clusters, App Services, Atlas Search, Vector Search, Data Federation, Stream Processing

---

## Chapter Table

| # | Chapter | Topics | Questions | Difficulty |
|---|---------|---------|-----------|------------|
| 01 | [MongoDB Fundamentals](./01-mongodb-fundamentals/) | Document model, BSON, replica sets, sharding overview, CRUD basics, write concerns | 25 theory + 15 scenario + 12 traps | Junior–Mid |
| 02 | [Documents & Collections](./02-documents-collections/) | CRUD operators, update operators, bulk writes, validation, capped collections | 20 theory + 15 scenario + 12 traps | Junior–Mid |
| 03 | [Querying & Aggregation](./03-querying-aggregation/) | Query operators, aggregation pipeline, $lookup, $unwind, $facet, Atlas Search integration | 30 theory + 20 scenario + 12 traps | Mid–Senior |
| 04 | [Indexes & Performance](./04-indexes-performance/) | All index types, ESR rule, covered queries, explain(), Atlas Performance Advisor | 25 theory + 15 scenario + 12 traps | Mid–Senior |
| 05 | [Schema Design](./05-schema-design/) | Embedding vs referencing, 12 design patterns, anti-patterns, schema validation | 25 theory + 15 scenario + 12 traps | Senior |
| 06 | [Transactions & Consistency](./06-transactions-consistency/) | ACID transactions, write/read concerns, read preferences, retryable writes | 20 theory + 12 scenario + 10 traps | Senior |
| 07 | [Atlas Features](./07-atlas-features/) | Atlas tiers, App Services, Triggers, Functions, Data Federation, Vector Search, backups | 25 theory + 15 scenario + 12 traps | Mid–Senior |
| 08 | [Atlas Search](./08-atlas-search/) | Lucene-based search, $search operators, fuzzy, synonyms, facets, Vector Search | 25 theory + 15 scenario + 12 traps | Senior |
| 09 | [Replication & Sharding](./09-replication-sharding/) | Replica set internals, oplog, elections, shard key design, chunk migration, resharding | 25 theory + 15 scenario + 12 traps | Senior–Staff |
| 10 | [MongoDB Operations](./10-mongodb-operations/) | Tools (mongodump, mongostat), connection strings, pooling, profiler, CSFLE, Ops Manager | 20 theory + 12 scenario + 10 traps | Mid–Senior |

**Total: ~215 theory questions | ~149 scenarios | ~116 traps = 480+ interview touchpoints**

---

## Study Order

### Path 1 — Junior/Mid Backend Engineer (2–4 weeks)
```
01 → 02 → 03 → 04 → 06 (skip advanced) → 10 (tools only)
```

### Path 2 — Senior Backend Engineer (3–5 weeks)
```
01 → 02 → 03 → 04 → 05 → 06 → 09 → 07 (overview) → 10
```

### Path 3 — Staff/Principal / Solutions Architect (4–6 weeks)
```
All chapters in order. Deep-dive 05 (schema), 09 (sharding), 08 (Atlas Search), 07 (Atlas platform)
```

### Path 4 — Atlas / Cloud Platform Focus
```
01 (fast) → 03 (fast) → 07 → 08 → 09 (Global Clusters) → 06 (read preferences) → 10
```

---

## MongoDB Version Notes (7.x Focus)

| Version | Key Features Added |
|---------|--------------------|
| 4.0 | Multi-document ACID transactions (replica sets) |
| 4.2 | Distributed transactions (sharded clusters), wildcard indexes |
| 4.4 | Hedged reads, refineable shard keys |
| 5.0 | Live resharding, time-series collections, versioned API |
| 6.0 | Encrypted fields (QFBE), clustered collections, $densify/$fill aggregation |
| 7.0 | Compound wildcard indexes, Atlas Stream Processing GA, improved time-series performance |

---

## Key Atlas Services Covered

- **Atlas Clusters** — M0 Free → M10/M20/M30 Dedicated → Global Multi-Region
- **Atlas App Services** — Triggers, Functions, GraphQL, Data API, Device Sync
- **Atlas Search** — Full-text search (Lucene), autocomplete, facets, synonyms, highlighting
- **Atlas Vector Search** — Semantic/embedding-based kNN search ($vectorSearch)
- **Atlas Data Federation** — Query S3, Atlas clusters, HTTP endpoints via $sql/$mql
- **Atlas Stream Processing** — Real-time event processing on Kafka/Atlas streams
- **Atlas Charts** — Built-in visualization dashboards
- **Atlas Online Archive** — Tiered storage for cold data
- **Atlas Backup** — Continuous backup, PIT recovery, cross-region snapshots

---

## How to Use This Section

Each chapter folder contains 6 files:

| File | Purpose |
|------|---------|
| `theory-questions.md` | Core concepts — what interviewers ask to check knowledge depth |
| `scenario-questions.md` | Real-world design/debugging scenarios — used in system design rounds |
| `follow-up-traps.md` | Trick questions and common wrong answers interviewers use to filter candidates |
| `structured-answers.md` | Full answers with code examples — model responses you can adapt |
| `analogy-explanations.md` | Simple analogies for explaining concepts to non-engineers or in whiteboard sessions |
| `diagram-explanations.md` | ASCII architecture diagrams — use these in system design discussions |

---

## Quick Reference: Most-Asked Topics by Role

### Junior
- What is BSON? How is it different from JSON?
- What is an ObjectId? Why does MongoDB use it as the default `_id`?
- Explain the difference between `find()` and `findOne()`
- What is a replica set?
- What does `$set` do in an update?

### Mid-Level
- Explain the ESR rule for compound indexes
- What is a covered query?
- When would you embed vs reference documents?
- What is the aggregation pipeline? Walk through `$group` and `$lookup`
- What are write concerns and why do they matter?

### Senior
- Design a schema for an IoT sensor system ingesting 10k events/sec
- Explain shard key selection and the monotonically increasing ID problem
- How do multi-document transactions work and what are their performance costs?
- Describe the Atlas Search architecture and how it differs from a query index
- What is the Bucket Pattern and when would you use it?

### Staff / Architect
- How would you approach live resharding a 5TB sharded cluster with zero downtime?
- Explain causally consistent sessions and when they are necessary
- Design a multi-tenant SaaS schema strategy in MongoDB
- Compare Atlas Data Federation vs Atlas Online Archive for a data tiering use case
- How does Atlas Vector Search differ from Atlas Search? When would you use both together?
