# Chapter 06: Advanced RAG — Diagram Explanations

## Diagram 1: Hybrid Search with RRF Pipeline

```
Query: "What is the optimal learning rate schedule for transformer training?"

┌─────────────────────────────────────────────────────────────────────────┐
│                         HYBRID RETRIEVAL PIPELINE                       │
└─────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────┐
                              │    Query    │
                              └──────┬──────┘
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
         ┌──────────────────┐               ┌──────────────────┐
         │   BM25 Retriever │               │  Dense Retriever │
         │  (sparse/lexical)│               │ (semantic/neural)│
         │                  │               │                  │
         │ Exact word match │               │ Meaning/concept  │
         │ TF-IDF weighted  │               │ ANN via HNSW     │
         └────────┬─────────┘               └────────┬─────────┘
                  │                                   │
         BM25 Ranked List                    Dense Ranked List
         1. doc_042 (score:8.2)              1. doc_019 (score:0.94)
         2. doc_107 (score:7.1)              2. doc_042 (score:0.91)
         3. doc_019 (score:6.8)              3. doc_203 (score:0.88)
         4. doc_388 (score:5.2)              4. doc_107 (score:0.85)
         5. doc_203 (score:4.9)              5. doc_512 (score:0.81)
                  │                                   │
                  └─────────────┬─────────────────────┘
                                ▼
                    ┌───────────────────────┐
                    │   RRF Fusion (k=60)   │
                    │                       │
                    │ doc_042: 1/(60+1) +   │
                    │         1/(60+2) =    │
                    │         0.0164+0.0161 │
                    │         = 0.0325 ← #1 │
                    │                       │
                    │ doc_019: 1/(60+3) +   │
                    │         1/(60+1) =    │
                    │         0.0159+0.0164 │
                    │         = 0.0322 ← #2 │
                    └───────────┬───────────┘
                                ▼
                    ┌───────────────────────┐
                    │    Reranker (opt.)    │
                    │  CrossEncoder / Cohere│
                    └───────────┬───────────┘
                                ▼
                    ┌───────────────────────┐
                    │  Top-5 Final Results  │
                    │  passed to LLM        │
                    └───────────────────────┘
```

**Key insight**: doc_042 appears as #1 in BM25 and #2 in dense — RRF promotes it to the top. doc_019 appears as #3 in BM25 and #1 in dense — RRF places it second. Documents appearing in both lists get a double boost regardless of their absolute scores.

---

## Diagram 2: Reranking Architecture — Two-Stage Retrieval

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TWO-STAGE RETRIEVAL + RERANKING                      │
└─────────────────────────────────────────────────────────────────────────┘

STAGE 1: Approximate Retrieval (Fast)
──────────────────────────────────────
Corpus: 10,000,000 documents

              Query Embedding
                    │
                    ▼
        ┌───────────────────────┐
        │  HNSW Index (ANN)     │  ← Pre-built at index time
        │  Time: ~10ms          │    Bi-encoder: doc vectors
        │  Recall: ~95%         │    stored as static floats
        └───────────┬───────────┘
                    │
                    ▼
            Top-50 Candidates
            (approximate, fast)


STAGE 2: Cross-Encoder Reranking (Accurate)
────────────────────────────────────────────
Input: [Query + Doc_1], [Query + Doc_2], ..., [Query + Doc_50]
                    │
                    ▼
        ┌───────────────────────┐
        │  Cross-Encoder Model  │  ← Full attention between
        │  (ms-marco-MiniLM)    │    query AND document tokens
        │  Time: ~150ms (GPU)   │    Cannot be pre-computed!
        │  Precision: very high │
        └───────────┬───────────┘
                    │
                    ▼
            Top-5 Reranked Results
            (accurate, passed to LLM)

LATENCY BREAKDOWN:
┌──────────────────────────────────────────────────────┐
│  Component          │  Latency  │  Scales with       │
├──────────────────────────────────────────────────────┤
│  Embedding query    │   20ms    │  query length      │
│  ANN search (HNSW)  │   10ms    │  O(log N)          │
│  Cross-encoder      │  150ms    │  num candidates    │
│  LLM generation     │  1500ms   │  output length     │
└──────────────────────────────────────────────────────┘
Total E2E: ~1.7 seconds (acceptable for most use cases)
```

---

## Diagram 3: RAPTOR Tree Structure

```
┌─────────────────────────────────────────────────────────────────────────┐
│               RAPTOR: RECURSIVE ABSTRACTIVE PROCESSING                  │
│                     FOR TREE-ORGANIZED RETRIEVAL                        │
└─────────────────────────────────────────────────────────────────────────┘

Document: "2024 Annual Report — Acme Corp (200 pages)"

LEVEL 3 (Root — Global Summary)
─────────────────────────────────
        ┌───────────────────────────────────────────────────┐
        │  "Acme Corp achieved 24% revenue growth in 2024   │
        │   driven by AI product line expansion and strong  │
        │   APAC market penetration..."                     │
        └───────────────────────┬───────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼

LEVEL 2 (Section Summaries)
─────────────────────────────
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │ Financial    │    │ Product      │    │ Market       │
   │ Results      │    │ Strategy     │    │ Expansion    │
   │ Summary      │    │ Summary      │    │ Summary      │
   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘
          │                   │                   │
    ┌─────┴─────┐       ┌─────┴─────┐       ┌─────┴─────┐
    ▼     ▼     ▼       ▼     ▼     ▼       ▼     ▼     ▼

LEVEL 1 (Paragraph Summaries)
────────────────────────────────
 [Q1 rev] [Q2 rev] [Q3 rev]  [AI] [Cloud] [Mobile]  [US] [EU] [APAC]

LEVEL 0 (Raw Chunks — Leaf Nodes)
──────────────────────────────────
 chunk_001 chunk_002 chunk_003 ... chunk_847 chunk_848 chunk_849
 (actual paragraphs from the original document)


QUERY ROUTING:
──────────────
"What was Acme's total revenue in 2024?"
→ Level 2 (Financial Results Summary) answers this directly

"What does the APAC expansion section say about India specifically?"
→ Level 0 chunks (raw paragraphs about India) needed

"What is the overall narrative of Acme's 2024 performance?"
→ Level 3 (Root summary) gives the complete picture

All levels are indexed in the same vector store.
Query is run against all levels; best-scoring chunk wins.
```

---

## Diagram 4: Self-RAG Token Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SELF-RAG GENERATION FLOW                             │
└─────────────────────────────────────────────────────────────────────────┘

Query: "What were the key causes of the 2023 SVB bank collapse?"

Standard LLM generation:
─────────────────────────
  Query → LLM → "The Silicon Valley Bank collapsed due to..." (may hallucinate)

Self-RAG generation:
─────────────────────
  Query → LLM → generates [Retrieve] token? ← model decides!
                                │
                    YES: retrieve top passages
                                │
                                ▼
              ┌─────────────────────────────────┐
              │  Retrieved Passages:            │
              │  P1: "SVB held long-duration..."│
              │  P2: "The Federal Reserve rate..│
              │  P3: "Venture capital deposits..│
              └────────────┬────────────────────┘
                           │
                           ▼
  LLM generates [IsREL_P1: relevant] [IsREL_P2: relevant] [IsREL_P3: relevant]
                           │
                           ▼
  LLM generates answer token by token, then:
  [IsSUP: fully supported] ← all claims traceable to passages
                           │
                           ▼
  [IsUSE: 5] ← usefulness score on scale 1-5
                           │
                           ▼
  "SVB collapsed primarily due to [P1: duration mismatch in bond portfolio]
   combined with [P2: rising interest rates reducing bond values] and
   [P3: concentrated depositor base causing bank run when news broke]"

SPECIAL TOKENS:
┌────────────┬────────────────────────────────────────────────────────┐
│ [Retrieve] │ Model-generated: should I retrieve? (yes/no)          │
│ [IsREL]    │ Is retrieved passage relevant? (relevant/irrelevant)   │
│ [IsSUP]    │ Is answer supported by passage? (fully/partial/no)     │
│ [IsUSE]    │ Is overall response useful? (1-5 scale)               │
└────────────┴────────────────────────────────────────────────────────┘
These are GENERATED TOKENS, not API calls or classifiers.
The model must be fine-tuned to emit them correctly.
```

---

## Diagram 5: RAGAS Evaluation Framework

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     RAGAS FOUR-METRIC FRAMEWORK                         │
└─────────────────────────────────────────────────────────────────────────┘

Inputs required:
  Q  = Question asked by user
  A  = Answer generated by RAG system
  C  = Retrieved context chunks [C1, C2, C3, ...]
  GT = Ground truth answer (reference)

┌──────────────────┬────────────────────────────────────────────────────┐
│ Metric           │ What it measures                                   │
├──────────────────┼────────────────────────────────────────────────────┤
│ FAITHFULNESS     │ Are all claims in A supported by C?               │
│                  │ = |supported claims| / |total claims in A|        │
│                  │ Inputs: A + C                                      │
│                  │ High score = LLM not hallucinating beyond context  │
├──────────────────┼────────────────────────────────────────────────────┤
│ ANSWER RELEVANCY │ Is A actually relevant to Q?                      │
│                  │ = avg cosine_sim(embed(Q), embed(generated_Q_i))  │
│                  │ (generates N questions from A, checks similarity)  │
│                  │ High score = answer addresses the question asked   │
├──────────────────┼────────────────────────────────────────────────────┤
│ CONTEXT PRECISION│ Are the retrieved C chunks actually needed?       │
│                  │ = |relevant chunks| / |total retrieved chunks|    │
│                  │ Inputs: Q + C (checks each chunk's relevance)     │
│                  │ High score = retriever not returning noise         │
├──────────────────┼────────────────────────────────────────────────────┤
│ CONTEXT RECALL   │ Does C contain what GT needs?                     │
│                  │ = |GT sentences supported by C| / |GT sentences|  │
│                  │ Inputs: GT + C                                     │
│                  │ High score = retriever found the right documents   │
└──────────────────┴────────────────────────────────────────────────────┘

DIAGNOSTIC DECISION TREE:
──────────────────────────

         Context Recall < 0.7?
               │
        YES────┴────NO
        │                │
   Retriever fails    Context Precision < 0.7?
   (not finding              │
    right docs)       YES────┴────NO
                      │                │
              Too much noise      Faithfulness < 0.8?
              in retrieval                │
                               YES────┴────NO
                               │                │
                          LLM ignores      Answer Relevancy < 0.8?
                          context (hallucinates)   │
                                           YES────┴────NO
                                           │             │
                                     Prompt        All metrics
                                     engineering   acceptable ✓
                                     needed
```

---

## Diagram 6: GraphRAG vs. Standard RAG for Different Query Types

```
┌─────────────────────────────────────────────────────────────────────────┐
│              GRAPHRAG vs VECTOR RAG: QUERY TYPE ROUTING                 │
└─────────────────────────────────────────────────────────────────────────┘

GRAPHRAG INDEXING:
──────────────────
Documents → [LLM entity extraction] → Knowledge Graph
                                       ┌──────────────────────────────┐
                                       │  Entities:                   │
                                       │  (Elon Musk)──CEO──▶(Tesla)  │
                                       │      │                       │
                                       │   founded                    │
                                       │      │                       │
                                       │      ▼                       │
                                       │  (SpaceX)──contracts──▶(NASA)│
                                       └──────────────────────────────┘
                                       + Community Detection (Leiden algo)
                                       → Cluster: {Space Industry}
                                       → Cluster: {Electric Vehicles}

QUERY ROUTING:
──────────────
                    ┌─────────────────────────────────────────┐
                    │              User Query                 │
                    └───────────────────┬─────────────────────┘
                                        │
                    ┌───────────────────▼─────────────────────┐
                    │              Query Classifier            │
                    │  "local" = specific fact lookup         │
                    │  "global" = thematic / relational       │
                    └───────┬───────────────────────┬─────────┘
                            │                       │
                    local query               global query
                            │                       │
                            ▼                       ▼
               ┌────────────────────┐   ┌────────────────────────┐
               │   Vector RAG       │   │      GraphRAG          │
               │                    │   │                        │
               │ Fast ANN search    │   │ Graph traversal        │
               │ Chunk retrieval    │   │ Community summaries    │
               │ Direct answer      │   │ Multi-hop reasoning    │
               └────────────────────┘   └────────────────────────┘

Examples:
  "What was Tesla's revenue in Q3 2023?"       → Vector RAG (local)
  "What themes connect all of Elon Musk's ventures?" → GraphRAG (global)
  "How is NASA connected to Tesla?"            → GraphRAG (relational)
  "What is the boiling point of lithium?"      → Vector RAG (local/factual)

COST COMPARISON:
  Vector RAG indexing:  $0.0001/chunk (embed only)
  GraphRAG indexing:    $0.01-0.10/chunk (LLM calls for entity extraction)
  → GraphRAG is 100-1000x more expensive to index
  → Only justified for highly relational knowledge bases
```
