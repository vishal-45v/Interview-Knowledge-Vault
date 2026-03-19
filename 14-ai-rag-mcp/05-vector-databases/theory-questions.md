# Chapter 05: Vector Databases — Theory Questions

Senior-level questions on ANN algorithms, vector database internals, and production system design. Tests whether you understand the data structures and tradeoffs, not just the API.

---

1. Explain the HNSW (Hierarchical Navigable Small World) algorithm from first principles. What is the "hierarchical" aspect and what is the "navigable small world" property? Why does HNSW achieve sub-linear search time?

2. HNSW has three key parameters: M, ef_construction, and ef_search. Explain what each controls, what happens when you set each too low vs too high, and how they interact with each other. Which parameter can be tuned at query time without rebuilding the index?

3. IVF (Inverted File Index) partitions the vector space into Voronoi cells. Explain how IVF search works, what the nlist and nprobe parameters control, and the fundamental tradeoff between search speed and recall that these parameters expose.

4. Product Quantization (PQ) is used to compress vectors for approximate search. Explain how PQ works — what a "codebook" is, how vectors are decomposed, and why this enables approximate distance computation without reconstructing full vectors. What quality is sacrificed and how is it measured?

5. Compare HNSW and IVFFlat for a vector database with 10 million 1536-dimensional vectors. Consider: (a) index build time, (b) memory usage, (c) query latency at 99% recall, and (d) the impact of adding new vectors without rebuilding. Which is better for each scenario?

6. Exact search (FLAT) is never used in production — or is it? Describe the conditions under which FLAT exact search is actually appropriate, and explain why it sometimes outperforms HNSW in practical benchmarks on small-to-medium datasets.

7. Pinecone uses namespaces for data isolation. Explain the difference between using namespaces for multi-tenancy versus using metadata filters. What are the performance and isolation implications of each approach?

8. Weaviate supports hybrid search combining BM25 and vector search. Explain how Weaviate's hybrid search is implemented, how the `alpha` parameter works, and when you would use pure BM25 (alpha=0), pure vector (alpha=1), or hybrid.

9. Qdrant's "payload filtering" happens during vector search rather than as a post-filter. Explain why this matters for performance, and describe the implementation approach that allows efficient filtering without scanning all vectors.

10. Chroma is often used for local development but also supports a server mode. Explain the architectural differences between Chroma's in-memory client, persistent client, and HTTP client modes. What are the production limitations of Chroma versus purpose-built cloud vector databases?

11. FAISS's IndexHNSWFlat, IndexIVFFlat, and IndexIVFPQ serve different use cases. Describe the memory footprint, search accuracy, and query latency tradeoffs between them for a 1M vector, 768-dim corpus.

12. pgvector brings vector search to PostgreSQL. Compare ivfflat and hnsw indexes in pgvector: build time, query speed, recall, and operational complexity. When would you choose pgvector over a dedicated vector database?

13. Describe the "index rebuild problem" in production vector databases. Why does HNSW degrade with many updates (inserts + deletes), and what strategies exist to maintain index quality over time without taking the database offline?

14. Multi-tenancy in vector databases can be implemented at three levels: separate indexes, namespaces/collections, and metadata-filtered queries on a shared index. Describe the data isolation, performance, and operational complexity tradeoffs of each approach.

15. Sparse vectors (from BM25 or SPLADE) can be stored and searched in some vector databases. Explain how sparse vector search differs from dense vector search algorithmically, and what the "sparse-dense hybrid" approach in Pinecone achieves that neither alone can.

16. Vector database replication and consistency: if you have a Weaviate cluster with 3 replicas and you upsert a vector, what consistency guarantees do you have? Describe the eventual consistency implications for a real-time RAG system.

17. Explain the "curse of dimensionality" as it applies to ANN algorithms. At what dimensionality does the curse most impact HNSW performance, and what is the practical implication for choosing embedding model dimension for production vector search?

18. Describe the key considerations when migrating a production vector database from one provider to another (e.g., from Pinecone to Qdrant). What must be handled carefully, and what is the primary operational risk?

19. HNSW graph construction is non-deterministic due to its probabilistic layer assignment. What does this mean for reproducibility in a RAG system, and does it affect retrieval quality in practice?

20. Describe the difference between vector database "collections" in Chroma, "indexes" in FAISS, "namespaces" in Pinecone, and "classes" in Weaviate. Despite different terminology, what is the conceptual unit each represents, and what isolation guarantees does each provide?
