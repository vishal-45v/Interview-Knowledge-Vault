# Chapter 05: Vector Databases — Diagram Explanations

ASCII diagrams for ANN algorithms and vector database architecture.

---

## Diagram 1: HNSW Graph Structure and Search

```
HNSW HIERARCHICAL GRAPH (3 layers, 8 vectors for illustration)

LAYER 2 (top, sparse):
  v1 ─────────────────────── v7
         \                  /
          v3            v6

LAYER 1 (middle):
  v1 ─── v2 ─── v3 ─── v5 ─── v7
    \         /     \       /
     v4─────         v6───

LAYER 0 (bottom, dense, ALL vectors):
  v1 ─ v2 ─ v3 ─ v4 ─ v5 ─ v6 ─ v7 ─ v8
       \   /     \ |  / \  /
        v4         v5    v7

QUERY: Find nearest to q★

STEP 1: Enter at Layer 2 entry point (v1)
        Greedy move: v1→v3→... stop when no neighbor is closer
        "Long-range navigation across the city"

STEP 2: Drop to Layer 1 at current position
        Greedy move: same process but with denser connections
        "Navigate within the district"

STEP 3: Drop to Layer 0
        Expand search with ef_search candidates
        "Search the immediate neighborhood thoroughly"

STEP 4: Return top-k from Layer 0 candidate set

VISUAL SEARCH PATH:
Layer 2: [v1] ──► [v3] ──► [v6]  (v6 is closest in layer 2)
Layer 1:           [v3] ──► [v5] ──► [v6]  (refine)
Layer 0: [v5] ──► [v6] ──► [v7] ──► [v8★] found!

PARAMETER EFFECTS:
M (connections): M=8  → sparser graph, less memory, lower recall
                 M=32 → denser graph, more memory, higher recall
                 8 bytes per connection × M connections × 2 directions

ef_construction: How many candidates to consider WHILE BUILDING
                 Higher → better graph quality → better recall
                 Cannot change after build

ef_search:       How many candidates to consider DURING QUERY
                 ★ Can change at query time without rebuilding ★
                 Higher → better recall, slower query
```

---

## Diagram 2: IVF — Inverted File Index Structure

```
IVF WITH nlist=4 CLUSTERS (simplified 2D example)

TRAINING PHASE: K-means to find cluster centroids
─────────────────────────────────────────────────

 ▲                  ● = vector
 │  ●●  [C1]  ●●   C = cluster centroid
 │  ●●●  ✦   ●
 │   ●  ● ●                    [C1] centroid coordinates
 │                              [C2] centroid coordinates
 │         [C2]    ●●●          [C3] centroid coordinates
 │          ✦   ●●●●            [C4] centroid coordinates
 │             ●●●
 │                     ●●  [C3]
 │         [C4]  ●●●●   ✦●●
 │          ✦  ●●  ●
 └────────────────────────────►

INDEXED DATA STRUCTURE (Inverted File):
─────────────────────────────────────────

Cluster C1: [v1, v2, v3, v4, v5, v8, v11, ...]   ← vectors assigned to C1
Cluster C2: [v6, v7, v9, v10, v13, v16, ...]      ← vectors assigned to C2
Cluster C3: [v12, v15, v17, v19, ...]              ← vectors assigned to C3
Cluster C4: [v14, v18, v20, v22, ...]              ← vectors assigned to C4

QUERY PHASE: q★ falls near C1 and C2
─────────────────────────────────────

nprobe=1: Only search C1 → fast but may miss nearest neighbor in C2!
nprobe=2: Search C1 AND C2 → slower but finds neighbors near the boundary

 ▲
 │  ●●  [C1]        ← nprobe=1: search here
 │  ●●●  ✦
 │   ●  ●●★q        ← query is near C1/C2 boundary
 │                  ← nprobe=2: also search C2
 │         [C2]  ●●●
 │          ✦  ●●●
 └───────────────────

RECALL vs SPEED TRADEOFF:
nprobe=1:    1/4 = 25% of vectors searched → fastest, lowest recall
nprobe=2:    2/4 = 50% of vectors searched
nprobe=4:    4/4 = 100% of vectors searched → exact search (slowest)

Rule: keep nprobe/nlist ratio constant when changing nlist!
```

---

## Diagram 3: Product Quantization — Memory Compression

```
ORIGINAL VECTOR (1536 dimensions, float32):
─────────────────────────────────────────────────────────────────────
[0.12, -0.45, 0.87, 0.23, -0.11, 0.67, ..., 0.33, -0.78]
 └──────────────────── 1536 floats ────────────────────┘
                     = 6,144 bytes (6 KB) per vector

STEP 1: Split into M=8 sub-vectors of 1536/8=192 dimensions each
─────────────────────────────────────────────────────────────────────

Sub-vec 1       Sub-vec 2       ...  Sub-vec 8
[0.12,-0.45,    [-0.11,0.67,    ...  [0.33,-0.78,
  0.87,0.23,...]   0.22,-0.44,...]     -0.12,0.55,...]
│192 dims│       │192 dims│            │192 dims│

STEP 2: For each sub-vector, find nearest centroid in its codebook
─────────────────────────────────────────────────────────────────────

Codebook 1 (256 centroids, each 192-dim):   Sub-vec 1 → Centroid #47
Codebook 2 (256 centroids, each 192-dim):   Sub-vec 2 → Centroid #183
...
Codebook 8 (256 centroids, each 192-dim):   Sub-vec 8 → Centroid #211

STEP 3: Store only centroid INDICES (1 byte each for 256 centroids)
─────────────────────────────────────────────────────────────────────

Compressed: [47, 183, 29, 74, 108, 55, 166, 211]
            └─────────── 8 bytes ───────────────┘

COMPRESSION RATIO:
Original:    6,144 bytes
Compressed:      8 bytes  → 768x compression!

DISTANCE COMPUTATION WITHOUT DECOMPRESSION:
─────────────────────────────────────────────────────────────────────

For query q and compressed vector v:
dist(q, v) ≈ dist(q₁, centroid₁[47])
           + dist(q₂, centroid₂[183])
           + dist(q₃, centroid₃[29])
           + ... (8 lookups total)

PRECOMPUTATION TABLE (computed once per query):
           Centroid 0  Centroid 1  ... Centroid 255
Sub-vec 1:   2.31        1.87    ...    0.44
Sub-vec 2:   0.98        3.12    ...    1.23
...

Each database vector: 8 table lookups → ~192x faster than exact computation
Quality: ~5-10% recall loss vs exact (acceptable for most RAG use cases)
```

---

## Diagram 4: Vector Database Comparison

```
VECTOR DATABASE SELECTION MATRIX

                    Chroma      pgvector    Pinecone    Qdrant      Weaviate    FAISS
                    ──────────────────────────────────────────────────────────────────
Managed service     Local/HTTP  Via RDS     Yes ✓       Self/Cloud  Self/Cloud  No
Max scale           ~10M        ~10M        ~1B         ~500M       ~500M       ~1B+
Hybrid search       No          No          Sparse+vec  No*         BM25+vec ✓  No
Pre-filter          No          Via SQL     Via metadata Via payload Via WHERE   No
Build time          Fast        Med         Managed     Med         Med         Slow
Delete support      Yes         Yes         Yes         Yes         Yes         No
Replication         Server mode Via PG      Managed     Built-in    Built-in    No
Best for            Dev/Proto   Existing PG Managed prod Self-hosted Hybrid srch Custom

* Qdrant added sparse vectors in v1.7

LATENCY AT DIFFERENT SCALES (approximate, varies by hardware and config):
─────────────────────────────────────────────────────────────────────────

Scale     │  FLAT   │  HNSW   │  IVFFlat │  IVFPQ
──────────┼─────────┼─────────┼──────────┼────────
  10K     │  0.5ms  │  0.8ms  │  0.3ms   │  0.2ms
 100K     │  5ms    │  1ms    │  1ms     │  0.5ms
   1M     │ 50ms    │  2ms    │  3ms     │  1ms
  10M     │ 500ms   │  5ms    │  8ms     │  3ms
 100M     │ 5000ms  │ 15ms    │ 20ms     │  8ms
   1B     │ N/A     │ 50ms    │ 60ms     │ 20ms

FLAT: exact search, no index overhead
HNSW: graph-based ANN, constant memory overhead per vector
IVFFlat: cluster-based ANN, low build time
IVFPQ: cluster-based + quantized, lowest memory
```

---

## Diagram 5: Qdrant Payload Filtering — Pre-filter Architecture

```
QUERY: "Find documents about authentication"
FILTER: { "tenant_id": "acme", "created_after": "2024-01-01" }

APPROACH A: POST-FILTER (naive, slow for selective filters)
─────────────────────────────────────────────────────────────

ANN Search (top_k=100)     Filter         Return top_k=10
────────────────────────  ──────────     ────────────────
Scan all vectors          tenant=acme?   [v12, v34, ...]
Find top-100 by similarity created>2024? (10 results after filter)
Return 100 results         Maybe returns
                           only 3 matching → need k>>10 to get 10!
                           ← PROBLEM: must oversample!

APPROACH B: PRE-FILTER (Qdrant, efficient for selective filters)
─────────────────────────────────────────────────────────────────

Payload Index Lookup      ANN Search         Return top_k=10
──────────────────────   ───────────────    ────────────────
B-tree on tenant_id  →   Only search        [v12, v34, ...]
Range index on date  →   vectors that       (10 results, all match)
                         match filter
                         (e.g., 50K of 1M)
                         ← ANN on 50K vecs, not 1M!

RESULT:
Pre-filter:  Search 50K vectors (5% of total) → fast, accurate
Post-filter: Search 1M vectors, discard 95% → slow, wasteful

QDRANT IMPLEMENTATION:
─────────────────────
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

results = client.search(
    collection_name="documents",
    query_vector=query_embedding,
    limit=10,
    query_filter=Filter(       # Qdrant applies this BEFORE vector search
        must=[
            FieldCondition(
                key="tenant_id",
                match=MatchValue(value="acme")
            ),
            FieldCondition(
                key="created_at",
                range=Range(gte="2024-01-01T00:00:00Z")
            )
        ]
    )
)
# Qdrant uses payload indexes (B-tree, hash, range) to efficiently
# determine which vectors to include in the ANN search candidates
# This is what makes filtered search O(filtered_set) not O(total)
```
