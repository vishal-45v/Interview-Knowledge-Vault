# 16 — NoSQL Databases: Interview Knowledge Vault

Senior-engineer and distributed-systems-architect level preparation covering the full
spectrum of NoSQL technology: theory, trade-offs, internals, and real-world design.

---

## Chapter Map

| # | Chapter | Core Focus |
|---|---------|------------|
| 01 | [NoSQL Fundamentals](01-nosql-fundamentals/) | CAP / PACELC / BASE vs ACID, consistency models, vector clocks, CRDTs, quorum math, Paxos/Raft |
| 02 | [Redis](02-redis/) | Data structures, persistence (RDB/AOF), Sentinel, Cluster, Pub/Sub, Streams, eviction, Lua |
| 03 | [Cassandra](03-cassandra/) | Ring topology, vnodes, tunable consistency, CQL data modeling, compaction, read/write paths |
| 04 | [DynamoDB](04-dynamodb/) | Partition/sort key design, GSI/LSI, capacity math, single-table design, DAX, Streams, transactions |
| 05 | [Elasticsearch](05-elasticsearch/) | Inverted index, Query DSL, aggregations, ILM, analyzers, relevance scoring, shard tuning |
| 06 | [Graph Databases](06-graph-databases/) | Property graphs, Neo4j Cypher, traversal algorithms, use cases vs relational JOINs |
| 07 | [Key-Value Stores](07-key-value-stores/) | Consistent hashing, DHT, etcd/Zookeeper, Riak, memcached vs Redis |
| 08 | [Wide-Column Stores](08-wide-column-stores/) | HBase, Bigtable lineage, row key design, column families, compaction, region splits |
| 09 | [Time-Series Databases](09-time-series-databases/) | InfluxDB, TimescaleDB, retention policies, downsampling, continuous queries, cardinality |
| 10 | [Database Selection](10-database-selection/) | Decision frameworks, polyglot persistence, migration strategies, cost modelling, real architectures |

---

## How to Use This Vault

1. **Theory Questions** — test recall of internals without hints; use as flashcard prompts.
2. **Scenario Questions** — simulate real on-site "design / debug" rounds; answer out loud before reading.
3. **Follow-up Traps** — the most valuable file; these are the exact questions that separate senior from staff.
4. **Structured Answers** — copy-pasteable, code-rich answers to anchor your explanations.
5. **Analogy Explanations** — use these to explain complex concepts to interviewers who appreciate clarity.
6. **Diagram Explanations** — sketch these on a whiteboard; they signal deep architectural fluency.

---

## Difficulty Progression

```
Chapter 01 ──► Foundations (mandatory for every other chapter)
Chapters 02-05 ──► Core technology deep-dives (high interview frequency)
Chapters 06-09 ──► Specialised stores (valuable differentiators)
Chapter 10 ──► Cross-cutting trade-offs (senior/staff level synthesis)
```

---

## Key Themes Across All Chapters

- **Consistency vs Availability** — every NoSQL choice is a point on this spectrum
- **Data modelling is query-first** — schema design begins with access patterns, not entities
- **Operational reality** — compaction, rebalancing, hotspots, and runaway tombstones matter as much as API
- **The "it depends" answer** — knowing *when* and *why* to reach for each tool is the real signal
