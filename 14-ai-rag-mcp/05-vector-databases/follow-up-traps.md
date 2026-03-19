# Chapter 05: Vector Databases — Follow-Up Traps

Tricky follow-up questions that reveal whether candidates understand vector database internals vs just the API.

---

## Trap 1: "HNSW is always better than IVF for production"

**What most people say:** HNSW has better recall and query speed than IVF, so you should always use HNSW.

**Correct answer:** HNSW and IVF have different tradeoff profiles that make each better in specific scenarios:

**IVF wins when:**
- Dataset is very large (>100M vectors) and memory is constrained — IVF's memory footprint grows linearly, while HNSW's graph adds significant overhead (~M × 4 bytes per vector just for links)
- High-throughput batch queries matter more than low latency — IVF with GPU support (FAISS on GPU) has much higher throughput
- You need to add many vectors frequently — IVF allows easy insertion; HNSW insertions can degrade graph quality over time

**HNSW wins when:**
- Low single-query latency is critical (P99 < 10ms)
- Dataset is medium-scale (1M–50M vectors) and RAM is available
- Recall at small k is most important

The real answer: benchmark BOTH on your specific dataset, vector dimension, and query pattern before choosing.

```python
# Benchmarking both in FAISS:
import faiss
import numpy as np
import time

dim = 1536
n_train = 1_000_000
vectors = np.random.randn(n_train, dim).astype('float32')
query = np.random.randn(100, dim).astype('float32')

# HNSW
index_hnsw = faiss.IndexHNSWFlat(dim, 32)  # M=32
index_hnsw.hnsw.efConstruction = 200
t0 = time.time()
index_hnsw.add(vectors)
hnsw_build_time = time.time() - t0

t0 = time.time()
D, I = index_hnsw.search(query, k=10)
hnsw_query_time = (time.time() - t0) / 100 * 1000  # ms per query

# IVF
nlist = 1024
quantizer = faiss.IndexFlatL2(dim)
index_ivf = faiss.IndexIVFFlat(quantizer, dim, nlist)
t0 = time.time()
index_ivf.train(vectors)
index_ivf.add(vectors)
ivf_build_time = time.time() - t0

index_ivf.nprobe = 32
t0 = time.time()
D, I = index_ivf.search(query, k=10)
ivf_query_time = (time.time() - t0) / 100 * 1000

print(f"HNSW build: {hnsw_build_time:.1f}s, query: {hnsw_query_time:.2f}ms")
print(f"IVF  build: {ivf_build_time:.1f}s,  query: {ivf_query_time:.2f}ms")
# HNSW: typically 3-5x faster query but 2-3x more memory and slower build
```

---

## Trap 2: "FAISS stores vectors on disk like a real database"

**What most people say:** FAISS is a vector database, so it handles persistence, updates, and queries like any database.

**Correct answer:** FAISS is a **vector similarity search library**, not a database. Critical limitations:

1. **In-memory only**: FAISS loads all vectors into RAM for search. It does not support on-disk search natively (Disk-ANN does this, not FAISS).
2. **No metadata storage**: FAISS stores vectors and their integer IDs. All metadata (text, source, date) must be stored externally (in a database, files, etc.) and joined at query time.
3. **No native update/delete**: FAISS does not support updating or deleting individual vectors. To "delete", you must rebuild the index or use a "soft delete" with external ID mapping.
4. **No persistence API**: You serialize to disk with `faiss.write_index()` — a binary blob, not a queryable format.
5. **No query language**: No filtering, no SQL-like syntax. Just k-NN search.

FAISS is best used as the vector engine INSIDE a production system that adds metadata storage, persistence, and filtering on top:

```python
import faiss
import numpy as np
import sqlite3
import pickle

class FaissWithMetadata:
    """Production-grade wrapper: FAISS for search, SQLite for metadata."""

    def __init__(self, dim: int):
        self.dim = dim
        self.index = faiss.IndexHNSWFlat(dim, 32)
        self.index.hnsw.efConstruction = 200
        # SQLite for metadata (use PostgreSQL in real production)
        self.db = sqlite3.connect(":memory:")
        self.db.execute(
            "CREATE TABLE chunks (id INTEGER PRIMARY KEY, text TEXT, source TEXT, metadata TEXT)"
        )
        self.next_id = 0

    def add(self, text: str, vector: np.ndarray, source: str, metadata: dict):
        vector_id = self.next_id
        self.next_id += 1

        # Store in FAISS (vector only)
        self.index.add(vector.reshape(1, -1).astype('float32'))

        # Store in SQLite (metadata)
        self.db.execute(
            "INSERT INTO chunks VALUES (?, ?, ?, ?)",
            (vector_id, text, source, str(metadata))
        )
        self.db.commit()

    def search(self, query_vector: np.ndarray, k: int = 5, source_filter: str = None):
        distances, indices = self.index.search(
            query_vector.reshape(1, -1).astype('float32'), k
        )

        results = []
        for dist, idx in zip(distances[0], indices[0]):
            if idx == -1:  # FAISS returns -1 for unfilled slots
                continue
            row = self.db.execute(
                "SELECT text, source, metadata FROM chunks WHERE id = ?", (int(idx),)
            ).fetchone()
            if row and (source_filter is None or row[1] == source_filter):
                results.append({
                    "text": row[0],
                    "source": row[1],
                    "distance": float(dist)
                })
        return results

    def save(self, path: str):
        faiss.write_index(self.index, f"{path}.faiss")
        # Also save SQLite and other state...
```

---

## Trap 3: "More clusters (nlist) in IVF means better accuracy"

**What most people say:** More IVF clusters means finer-grained partitioning, which means more accurate search.

**Correct answer:** IVF accuracy depends on the ratio of `nprobe` to `nlist`, not `nlist` alone. With `nlist=1024` and `nprobe=32`, you're searching 32/1024 = 3.1% of all vectors. With `nlist=4096` and `nprobe=32`, you're searching only 0.8% — worse recall despite more clusters.

The rule of thumb: `nprobe/nlist` should stay roughly constant to maintain recall. Increasing `nlist` must be paired with increasing `nprobe` proportionally.

The FAISS recommendation: `nlist = 4 * sqrt(n)` for a dataset of n vectors. For 1M vectors: `nlist ≈ 4000`. For 10M: `nlist ≈ 12600`.

```python
# IVF recall vs nprobe tradeoff:
# With nlist=1024, n=1M vectors:
# nprobe=1:   recall@10 ≈ 0.40 (very fast, poor quality)
# nprobe=8:   recall@10 ≈ 0.75
# nprobe=32:  recall@10 ≈ 0.92
# nprobe=128: recall@10 ≈ 0.99 (slow, near-exact)
# nprobe=1024: recall@10 = 1.00 (exact, same as FLAT)

# Key insight: nprobe=nlist → EXACT SEARCH (every cluster examined)
# IVF with nprobe=nlist == IndexFlatL2 in terms of results,
# but with MORE overhead due to cluster structure!

# So: too many clusters with too few probes = worse than brute force
```

---

## Trap 4: "Pinecone namespaces are just folders — there's no performance difference"

**What most people say:** Namespaces are an organizational feature, like folders in a file system.

**Correct answer:** Pinecone namespaces have direct performance implications. Queries to a namespace ONLY search vectors in that namespace — not the entire index. This means:

1. **Smaller effective search space** = faster query latency
2. **Better isolation** = no cross-tenant leakage
3. **Independent deletion** = can wipe a namespace without affecting others

The alternative — using metadata filters on a shared namespace — searches the FULL vector space and then post-filters. This is O(n) in the full dataset size, not O(namespace_size). For a 100-tenant system where each tenant has 1% of the data, namespace-based isolation gives ~100x smaller search space than filtered search.

However: namespaces in Pinecone share the same underlying index. For true isolation (data sovereignty, separate encryption keys), you need separate indexes or separate Pinecone projects.

```python
import pinecone
from pinecone import Pinecone

pc = Pinecone(api_key="your-api-key")
index = pc.Index("production-rag")

# APPROACH 1: Namespaces (recommended for multi-tenancy)
# Query only searches tenant-123's data
results = index.query(
    vector=query_embedding,
    top_k=10,
    namespace="tenant-123",  # O(tenant_data_size) search
    include_metadata=True,
)

# APPROACH 2: Metadata filter (correct but slower for large datasets)
results = index.query(
    vector=query_embedding,
    top_k=10,
    filter={"tenant_id": "tenant-123"},  # O(full_index_size) search + filter
    include_metadata=True,
)

# PERFORMANCE COMPARISON for 100 tenants, 10M total vectors, 100K per tenant:
# Namespace query:  searches 100K vectors → ~5ms
# Metadata filter:  searches 10M vectors, returns 100K, filters to tenant → ~500ms

# BUT: if a single tenant has very dense data and you need cross-tenant
# metadata filtering, you need metadata filters within a namespace.
# The two approaches compose: namespace for tenant isolation, metadata for sub-filtering.
```

---

## Trap 5: "pgvector is just for small datasets — it can't handle production scale"

**What most people say:** pgvector is a hobby project; for production you need a dedicated vector database.

**Correct answer:** pgvector with HNSW indexes (added in v0.5.0, November 2023) is production-ready for many use cases. Supabase, Neon, and many enterprises run pgvector at scale. The key threshold where dedicated vector databases win:

- **Scale**: pgvector HNSW works well up to ~5-10M vectors at 768-1536 dims. Beyond this, dedicated vector databases offer better performance through sharding, GPU acceleration, and purpose-built storage engines.
- **Recall**: pgvector HNSW achieves 95%+ recall at good performance — comparable to dedicated solutions.
- **Operational simplicity**: If you're already on PostgreSQL, pgvector means zero additional infrastructure, unified transactions (vector search + relational JOIN in one query), and no separate backup/monitoring strategy.

```sql
-- pgvector: production-grade setup for moderate scale

-- Enable extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create table with vector column
CREATE TABLE document_chunks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id UUID REFERENCES documents(id),
    content TEXT NOT NULL,
    embedding vector(1536),
    chunk_index INTEGER,
    metadata JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- HNSW index: fast queries, high recall
-- Best for: < 5M vectors, query latency < 50ms
CREATE INDEX chunks_embedding_hnsw ON document_chunks
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- IVFFlat index: lower memory, slower build
-- Best for: > 5M vectors, or when index build time is critical
CREATE INDEX chunks_embedding_ivfflat ON document_chunks
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);  -- Tune: ~sqrt(num_rows)

-- Query: vector search + metadata filter IN ONE SQL QUERY
-- This is the killer feature vs dedicated vector DBs
SELECT
    dc.id,
    dc.content,
    dc.metadata,
    1 - (embedding <=> $1) AS similarity
FROM document_chunks dc
JOIN documents d ON dc.document_id = d.id
WHERE
    d.tenant_id = $2          -- Metadata filter (exact match, uses B-tree)
    AND d.created_at > NOW() - INTERVAL '90 days'  -- Date filter
ORDER BY embedding <=> $1     -- Cosine distance ordering
LIMIT 10;

-- Performance note: set ef_search at query time for speed/recall tradeoff
SET hnsw.ef_search = 100;  -- Higher = more accurate, slower
```

---

## Trap 6: "Deleting vectors from HNSW is the same as from other indexes"

**What most people say:** You can delete vectors and the index updates correctly.

**Correct answer:** HNSW does not support true deletion efficiently. When you "delete" a vector from an HNSW index:

1. **FAISS HNSW**: Does not support deletion at all. You can mark IDs as deleted externally and filter post-search, but the deleted vector still participates in routing. Index quality degrades over time as deleted vectors create "dead ends" in the graph.

2. **Qdrant**: Uses "soft deletion" — marks vectors as deleted in a bitset but they remain in the graph. A background vacuum process eventually rebuilds affected graph segments.

3. **Weaviate**: Similar soft deletion with background optimization.

4. **The quality degradation problem**: In HNSW, each vector is a node in a graph with navigation edges. Deleting a node removes it from the result set but leaves its neighbors disconnected. If the deleted node was on many navigation paths, search quality for nearby vectors degrades because the routing graph has gaps.

The production strategy: use periodic full index rebuilds for datasets with high churn rates (>10% monthly deletion rate). For low-churn datasets, soft deletion with vacuum is acceptable.

```python
# Qdrant delete + check degradation:
from qdrant_client import QdrantClient
from qdrant_client.models import PointIdsList

client = QdrantClient(host="localhost", port=6333)

# "Delete" specific vectors (soft delete in Qdrant)
client.delete(
    collection_name="documents",
    points_selector=PointIdsList(points=[42, 43, 44]),  # IDs to delete
)

# Trigger optimization to rebuild affected graph segments
client.update_collection(
    collection_name="documents",
    optimizer_config={"indexing_threshold": 0},  # Force immediate re-indexing
)

# For high-churn datasets: scheduled full re-index
# 1. Create new collection: documents_v2
# 2. Re-insert all non-deleted vectors
# 3. Blue-green cutover
# 4. Delete documents (old collection)
```

---

## Trap 7: "The curse of dimensionality means higher dimensions = worse ANN performance"

**What most people say:** Higher-dimensional vectors are harder to search because of the curse of dimensionality — all points become equidistant.

**Correct answer:** This is partially true but misses important nuance. The "curse of dimensionality" does mean that in purely random high-dimensional data, the ratio of max to min distances between points converges to 1 as dimension increases — making nearest-neighbor search meaningless in theory. However:

1. **Embedding dimensions are NOT uniformly random**: Real embedding spaces have low intrinsic dimensionality — semantic information is concentrated in a lower-dimensional subspace despite the high nominal dimension. 1536-dim embeddings don't have 1536 "independent" dimensions of variation.

2. **ANN algorithms work well on real embeddings**: HNSW achieves excellent recall on 1536-dim OpenAI embeddings in practice. The theoretical curse doesn't bite hard on real distribution data.

3. **Where dimension DOES hurt**: Index build time (O(d) per comparison), memory usage (O(d × n)), and query latency all scale linearly with d. Going from 768 to 3072 dimensions doubles both memory and compute.

4. **The real tradeoff**: Larger embedding dimensions capture more semantic nuance (better embedding quality) but increase infrastructure costs. OpenAI's text-embedding-3 supports "dimensions" reduction — you can use 512 dims instead of 3072 and get 90%+ of the quality at 1/6 the storage and compute.

```python
import openai

# Matryoshka embeddings: embed at reduced dimension
response = openai.embeddings.create(
    model="text-embedding-3-large",  # Native 3072 dims
    input="The transformer architecture",
    dimensions=512,  # Truncate to 512 dims (Matryoshka property)
)
# 512-dim embedding, ~85-90% of full-dim quality
# 6x memory reduction, 6x faster distance computation
embedding = response.data[0].embedding  # length 512
```
