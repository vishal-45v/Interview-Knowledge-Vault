# Chapter 05: Vector Databases — Scenario Questions

Production engineering decisions for vector database selection, configuration, and operation. Answer as a senior engineer who has operated these systems at scale.

---

1. You are designing the vector database layer for a multi-tenant SaaS application. Each customer has their own document collection. There are currently 500 customers with an average of 50,000 chunks each (25M total chunks). Some enterprise customers have 5M chunks; some SMB customers have 5,000. The access pattern is: each query is always constrained to a single customer's data. New customers are onboarded daily. Choose your vector database strategy and justify every design decision, including how tenant isolation is enforced and how you handle the customer size distribution.

2. Your RAG system has been running in production with Pinecone for 6 months. You are now asked to migrate to a self-hosted Qdrant cluster to reduce costs. The current Pinecone index has 8M vectors of 1536 dimensions with metadata attached. The migration must be completed with zero downtime. Design the complete migration strategy, including data export, re-indexing, traffic cutover, and rollback plan.

3. Your team is building a semantic search system for a legal research platform. The corpus is 500M legal documents (cases, statutes, regulations), each averaging 2000 tokens. The query pattern is: attorneys issue 10-50 queries per session, expect sub-second results, and require metadata filtering by jurisdiction, court level, and date range. Queries peak at 2000 QPS. Choose the vector database architecture, index type, and infrastructure to meet these requirements.

4. You have an existing pgvector setup on PostgreSQL for your RAG system (1M vectors, 768 dimensions). Query latency has increased from 50ms to 450ms over the past 3 months as the corpus grew. You need to bring latency back below 100ms P99 without migrating away from PostgreSQL. What are your options and how do you execute each?

5. You are building a recommendation system for a music streaming platform. The system needs to find "songs similar to this" from a catalog of 100 million songs, each represented as a 512-dimensional acoustic feature vector. Requirements: query latency under 50ms P99, 95%+ recall at top-50, ingest 100,000 new songs per day. FAISS is being evaluated as the vector engine. Design the FAISS index configuration and the surrounding system architecture.

6. A financial services client requires that their document vectors be stored in a vector database with the following constraints: (a) data must not leave their on-premise data center, (b) all vectors at rest must be encrypted, (c) queries must be audited with full query logging, (d) the system must tolerate the failure of any single node. Evaluate your vector database options for this compliance-constrained deployment.

7. Your RAG system uses Weaviate with a hybrid search setup. Users report that results for technical product queries (like "Part number XJ-4492 specifications") are poor while results for natural language queries work well. Walk through the diagnosis and resolution, including what you would change in the Weaviate hybrid configuration, the indexing pipeline, and the query-time behavior.

8. You need to add vector search capability to an existing monolithic application that already uses PostgreSQL extensively. The team has limited MLOps experience. The vector corpus is expected to grow from 500K to 5M vectors over 2 years. Should you use pgvector extension or a dedicated vector database? Make the case for both, quantify the tradeoffs, and give your recommendation with conditions.

9. Your Qdrant cluster is running at 85% memory utilization. You have 50M vectors of 1536 dimensions stored with float32 precision. You cannot add more hardware. Describe every option you have to reduce memory footprint, from lossless to lossy compression, including the quality impact of each and the implementation steps.

10. You are the first ML engineer at a startup. You need to build a RAG system prototype in one week. The corpus is 100K product descriptions. You have no cloud budget approved yet. Choose your vector database, justify why, and describe the exact code and configuration you'd use to get from raw text to searchable vectors in the minimum time.
