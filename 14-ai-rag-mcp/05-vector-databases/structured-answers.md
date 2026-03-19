# Chapter 05: Vector Databases — Structured Answers

Complete answers for the highest-priority vector database questions.

---

## Q1: HNSW Algorithm — From Data Structure to Query

**Answer:**

HNSW builds a layered graph structure where each layer is a "navigable small world" graph — a graph where any two nodes can be connected in a small number of hops (like 6 degrees of separation in social networks).

**The hierarchical structure:**
```
Layer 2 (sparse):    v1 ─────────────────── v42
                      \                    /
                       v17           v89
                        \           /
Layer 1 (medium):   v1 ─ v17 ─ v42 ─ v89 ─ v103
                        \   |  /   \   |
                         v23 v56  v71 v91
Layer 0 (dense):    [all vectors connected to M neighbors each]
```

- Higher layers: sparse long-range connections (coarse navigation)
- Layer 0: dense short-range connections (fine-grained search)

**Query algorithm:**
```
1. Start at the entry point in the highest layer
2. Greedy search: move to the neighbor closest to query vector
3. When no neighbor is closer than current node → drop to lower layer
4. Repeat until Layer 0 is reached
5. In Layer 0: expand search around current position using ef_search candidates
6. Return top-k results
```

```python
import faiss
import numpy as np

def build_hnsw_index(vectors: np.ndarray, M: int = 16, ef_construction: int = 200):
    """
    M: Number of bidirectional connections per node.
       Higher M = better recall, more memory, slower build
       Typical: M=16 (default), M=32 (higher quality)
       Memory per vector: M * 4 bytes * 2 (bidirectional) = 128 bytes at M=16

    ef_construction: Size of dynamic candidate list during build.
       Higher = better graph quality, much slower build time
       Typical: 100-200 for good quality, 64 for fast build
    """
    dim = vectors.shape[1]
    index = faiss.IndexHNSWFlat(dim, M)
    index.hnsw.efConstruction = ef_construction
    index.add(vectors)
    return index

def query_hnsw(index, query_vectors: np.ndarray, k: int = 10, ef_search: int = 50):
    """
    ef_search: Size of dynamic candidate list during search.
       Can be set AFTER build time (no rebuild needed)!
       Higher ef_search = better recall, slower query
       Must be >= k

    Rule of thumb: ef_search = 2-5 * k for ~99% recall
    """
    index.hnsw.efSearch = ef_search  # Can change at query time!
    distances, indices = index.search(query_vectors, k)
    return distances, indices

# PARAMETER INTERACTION TABLE:
# M     | ef_construction | ef_search | Recall@10 | Memory/vec | Build time
# 16    | 100             | 50        | 0.94      | 200 bytes  | 1x
# 32    | 200             | 100       | 0.98      | 400 bytes  | 4x
# 64    | 400             | 200       | 0.99+     | 800 bytes  | 16x
# Note: ef_search tuning at query time = free recall improvement
```

---

## Q2: Vector Database Selection Guide

**Answer:**

Choosing the right vector database depends on scale, operational model, existing infrastructure, and compliance requirements.

```python
# DECISION FRAMEWORK:

def choose_vector_db(
    num_vectors: int,
    dimension: int,
    qps: int,
    latency_p99_ms: int,
    infrastructure: str,  # "cloud", "on-prem", "existing-postgres"
    team_expertise: str,  # "ml", "backend", "devops"
    compliance: list,     # ["gdpr", "hipaa", "sox"]
    budget: str,          # "startup", "scaleup", "enterprise"
) -> str:
    """Not actual code — a structured decision guide."""
    pass

# DECISION MATRIX:
decision_matrix = {
    "prototype_or_small": {
        "condition": "num_vectors < 500K, team=small, no managed infra",
        "recommendation": "Chroma (local persistent)",
        "rationale": "Zero config, embedded mode, great LangChain integration",
        "code": """
import chromadb
client = chromadb.PersistentClient(path="./chroma_db")
collection = client.get_or_create_collection("documents")
collection.add(embeddings=embs, documents=texts, ids=ids)
results = collection.query(query_embeddings=[q_emb], n_results=5)
        """
    },

    "existing_postgres": {
        "condition": "already on PostgreSQL, < 5M vectors, unified queries needed",
        "recommendation": "pgvector with HNSW index",
        "rationale": "No new infra, transactional consistency, JOINs with relational data",
        "code": """
CREATE EXTENSION vector;
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops) WITH (m=16, ef_construction=64);
SELECT id, text, 1-(embedding<=>$1) AS score FROM chunks ORDER BY embedding<=>$1 LIMIT 10;
        """
    },

    "managed_cloud_medium": {
        "condition": "< 50M vectors, cloud-native, team wants managed ops",
        "recommendation": "Pinecone (serverless or pod-based)",
        "rationale": "Fully managed, excellent developer experience, namespace isolation",
        "watch_out": "Vendor lock-in, data residency limitations, cost at scale"
    },

    "open_source_production": {
        "condition": "need self-hosted, advanced filtering, < 500M vectors",
        "recommendation": "Qdrant",
        "rationale": "Rust-based (fast), payload filtering pre-search, Rust/Python/gRPC",
        "code": """
from qdrant_client import QdrantClient
from qdrant_client.models import VectorParams, Distance, PointStruct, Filter, FieldCondition, MatchValue

client = QdrantClient("localhost", port=6333)
client.create_collection("docs", vectors_config=VectorParams(size=1536, distance=Distance.COSINE))
client.upsert("docs", points=[PointStruct(id=1, vector=emb, payload={"source": "policy.pdf"})])
results = client.search(
    "docs", query_vector=q_emb, limit=10,
    query_filter=Filter(must=[FieldCondition(key="source", match=MatchValue(value="policy.pdf"))])
)
        """
    },

    "hybrid_search_focused": {
        "condition": "need strong BM25+vector hybrid, schema-first design",
        "recommendation": "Weaviate",
        "rationale": "Native hybrid search, GraphQL API, module ecosystem",
        "code": """
import weaviate
client = weaviate.Client("http://localhost:8080")
result = (
    client.query.get("Document", ["text", "source"])
    .with_hybrid(query="refund policy", alpha=0.5)  # 50% vector, 50% BM25
    .with_limit(10)
    .do()
)
        """
    },

    "massive_scale_offline": {
        "condition": "> 100M vectors, batch queries, GPU available, custom pipeline",
        "recommendation": "FAISS with custom wrapper",
        "rationale": "Maximum performance, GPU support, full control",
        "watch_out": "Not a database: need to build metadata storage, persistence, updates"
    }
}
```

---

## Q3: HNSW Parameters — Tuning for Production

**Answer:**

```python
import faiss
import numpy as np
from dataclasses import dataclass

@dataclass
class HNSWConfig:
    M: int                   # Connections per node (8-64, default 16)
    ef_construction: int     # Build-time candidate list (100-500, default 200)
    ef_search: int           # Query-time candidate list (>= k, default 50)

def benchmark_hnsw_params(vectors, queries, ground_truth_k10, configs):
    """
    Find the optimal HNSW config for your dataset.
    ground_truth_k10: exact top-10 results for each query (from FLAT index)
    """
    flat_index = faiss.IndexFlatL2(vectors.shape[1])
    flat_index.add(vectors)
    _, gt = flat_index.search(queries, 10)

    results = []
    for config in configs:
        index = faiss.IndexHNSWFlat(vectors.shape[1], config.M)
        index.hnsw.efConstruction = config.ef_construction
        index.add(vectors)
        index.hnsw.efSearch = config.ef_search

        import time
        t0 = time.time()
        _, retrieved = index.search(queries, 10)
        query_time_ms = (time.time() - t0) / len(queries) * 1000

        # Recall@10: fraction of ground truth top-10 that we retrieved
        recall = sum(
            len(set(retrieved[i]) & set(gt[i])) / 10
            for i in range(len(queries))
        ) / len(queries)

        # Memory: approximate HNSW overhead
        memory_bytes = vectors.nbytes + config.M * 4 * 2 * len(vectors)

        results.append({
            "config": config,
            "recall@10": recall,
            "query_ms": query_time_ms,
            "memory_gb": memory_bytes / 1e9,
        })
        print(f"M={config.M:3d} ef_c={config.ef_construction:4d} ef_s={config.ef_search:4d} "
              f"→ recall={recall:.3f} latency={query_time_ms:.2f}ms")
    return results

# Representative configs to test:
configs = [
    HNSWConfig(M=8,  ef_construction=64,  ef_search=32),   # Speed-optimized
    HNSWConfig(M=16, ef_construction=128, ef_search=64),   # Balanced (typical default)
    HNSWConfig(M=32, ef_construction=200, ef_search=100),  # Quality-optimized
    HNSWConfig(M=64, ef_construction=400, ef_search=200),  # Max quality (expensive)
]

# Tuning strategy:
# 1. Set ef_construction HIGH during build (quality of graph structure)
#    You build once, but query millions of times — invest in build quality
# 2. Start with M=16, increase to M=32 if recall is below 95%
# 3. Tune ef_search at query time to hit your latency budget at target recall
#    ef_search is the ONLY parameter you can tune without rebuilding!
```

---

## Q4: Multi-Tenancy Patterns — Implementation and Tradeoffs

**Answer:**

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, Distance, PointStruct,
    Filter, FieldCondition, MatchValue,
    CreateCollection
)
import hashlib

# PATTERN 1: Separate collections per tenant
# Best for: large tenants (>1M vectors each), strict data isolation, GDPR
class TenantIsolatedDB:
    def __init__(self, client: QdrantClient, dim: int):
        self.client = client
        self.dim = dim

    def collection_name(self, tenant_id: str) -> str:
        return f"tenant_{hashlib.md5(tenant_id.encode()).hexdigest()[:8]}"

    def ensure_tenant_collection(self, tenant_id: str):
        name = self.collection_name(tenant_id)
        if not self._collection_exists(name):
            self.client.create_collection(
                name,
                vectors_config=VectorParams(size=self.dim, distance=Distance.COSINE)
            )
        return name

    def upsert(self, tenant_id: str, points: list):
        coll = self.ensure_tenant_collection(tenant_id)
        self.client.upsert(coll, points=points)

    def search(self, tenant_id: str, query_vector, top_k: int):
        coll = self.collection_name(tenant_id)
        return self.client.search(coll, query_vector=query_vector, limit=top_k)

# PROS: True data isolation, can have different configs per tenant,
#       easy per-tenant deletion, separate backup/restore
# CONS: Collection management overhead, cold start for new tenants,
#       doesn't scale to thousands of small tenants (collection overhead)


# PATTERN 2: Shared collection with payload filtering
# Best for: many small tenants (<100K vectors each), internal applications
class SharedCollectionDB:
    def __init__(self, client: QdrantClient, collection: str):
        self.client = client
        self.collection = collection

    def upsert(self, tenant_id: str, doc_id: str, vector, text: str):
        self.client.upsert(
            self.collection,
            points=[PointStruct(
                id=doc_id,
                vector=vector,
                payload={
                    "tenant_id": tenant_id,  # Mandatory field for isolation
                    "text": text,
                }
            )]
        )

    def search(self, tenant_id: str, query_vector, top_k: int):
        # Payload filter executes BEFORE scoring (not post-filter!)
        return self.client.search(
            self.collection,
            query_vector=query_vector,
            limit=top_k,
            query_filter=Filter(
                must=[FieldCondition(
                    key="tenant_id",
                    match=MatchValue(value=tenant_id)
                )]
            )
        )

    def delete_tenant(self, tenant_id: str):
        """Delete ALL vectors for a tenant (GDPR 'right to be forgotten')."""
        self.client.delete(
            self.collection,
            points_selector=Filter(
                must=[FieldCondition(key="tenant_id", match=MatchValue(value=tenant_id))]
            )
        )

# PROS: Simple management, efficient for small tenants
# CONS: No true isolation (a bug could expose cross-tenant data),
#       cannot have different vector configs per tenant,
#       deletion is slower (requires filter scan, not collection drop)


# PATTERN 3: Namespace-based (Pinecone-style)
# Best for: cloud-managed, medium tenants, simplicity
def pinecone_namespace_pattern(index, tenant_id: str, vectors, ids, texts):
    """Pinecone namespace: same index, isolated search space per namespace."""
    index.upsert(
        vectors=[
            {"id": f"{tenant_id}_{i}", "values": v, "metadata": {"text": t}}
            for i, (v, t) in enumerate(zip(vectors, texts))
        ],
        namespace=tenant_id  # All tenant data goes to dedicated namespace
    )

def pinecone_namespace_search(index, tenant_id: str, query_vector, top_k: int):
    return index.query(
        vector=query_vector,
        top_k=top_k,
        namespace=tenant_id,  # Only searches this tenant's vectors
        include_metadata=True
    )

# TRADEOFF SUMMARY:
# ┌──────────────┬────────────────┬───────────────┬─────────────────┐
# │ Pattern      │ Isolation      │ Scale limit   │ Best for        │
# ├──────────────┼────────────────┼───────────────┼─────────────────┤
# │ Separate coll│ Physical       │ ~1000 tenants │ Large tenants   │
# │ Shared+filter│ Logical only   │ Unlimited     │ Small tenants   │
# │ Namespace    │ Logical+perf   │ 1000s         │ Medium tenants  │
# └──────────────┴────────────────┴───────────────┴─────────────────┘
```

---

## Q5: Memory Optimization for Large Vector Indexes

**Answer:**

```python
import faiss
import numpy as np

# SCENARIO: 50M vectors × 1536 dimensions = 300GB in float32
# Strategy: Progressive compression, measure quality at each step

dim = 1536
n = 50_000_000

# OPTION 1: FP16 (half precision)
# Memory: 300GB → 150GB (2x reduction)
# Quality loss: essentially zero for retrieval tasks
# Implementation: store in float16, cast to float32 for distance computation
vectors_fp16 = vectors.astype(np.float16)
# Note: FAISS doesn't natively use fp16; need fp32 at compute time

# OPTION 2: Product Quantization (PQ)
# Memory: 300GB → ~25GB (12x reduction with M=8, nbits=8)
# Quality loss: ~5-10% recall drop at same nprobe
# M = number of sub-vectors (chunk dim by M): 1536/M = sub-vector dim
# nbits = bits per sub-vector centroid (8 bits = 256 centroids)
# Memory per vector: M * nbits/8 bytes = 8 * 1 = 8 bytes (vs 6144 bytes fp32!)

M_pq = 8          # 1536 / 8 = 192-dim sub-vectors
nbits = 8          # 256 centroids per sub-space
quantizer = faiss.IndexFlatL2(dim)
index_ivfpq = faiss.IndexIVFPQ(quantizer, dim, nlist=16384, M=M_pq, nbits=nbits)

# Must train on a representative sample before adding:
train_sample = vectors[:min(100_000, len(vectors))]
index_ivfpq.train(train_sample)
index_ivfpq.add(vectors)

print(f"IndexIVFPQ memory ≈ {(n * M_pq) / 1e9:.1f} GB")  # ~0.4 GB for 50M!

# OPTION 3: Scalar Quantization (SQ)
# Memory: 300GB → 75GB (4x reduction with 8-bit quantization of each float)
# Quality loss: minimal (<2% recall drop)
index_sq = faiss.IndexIVFScalarQuantizer(
    quantizer, dim, nlist=16384,
    qtype=faiss.ScalarQuantizer.QT_8bit  # 8-bit per dimension
)
index_sq.train(train_sample)
index_sq.add(vectors)

# OPTION 4: Qdrant with quantization
# Qdrant supports scalar and product quantization natively
from qdrant_client.models import (
    ScalarQuantizationConfig, ScalarType,
    ProductQuantizationConfig, CompressionRatio
)

# Scalar quantization: 4x memory reduction, < 1% recall loss
scalar_config = ScalarQuantizationConfig(
    type=ScalarType.INT8,
    quantile=0.99,  # Clamp outliers
    always_ram=True  # Keep quantized index in RAM even if original is on disk
)

# Product quantization: 32x memory reduction, ~5% recall loss
product_config = ProductQuantizationConfig(
    compression=CompressionRatio.X32,  # 32x compression
)

# COMPARISON TABLE:
# ┌─────────────────┬──────────────┬────────────────┬─────────────────┐
# │ Method          │ Memory (50M) │ Recall@10 loss │ Build overhead  │
# ├─────────────────┼──────────────┼────────────────┼─────────────────┤
# │ Full float32    │ 300 GB       │ 0%             │ None            │
# │ SQ8 (int8)      │ 75 GB        │ <1%            │ 5 min           │
# │ PQ (M=8, b=8)  │ 2.5 GB       │ 5-10%          │ 30 min          │
# │ PQ (M=16, b=8) │ 5 GB         │ 3-5%           │ 45 min          │
# └─────────────────┴──────────────┴────────────────┴─────────────────┘
```
