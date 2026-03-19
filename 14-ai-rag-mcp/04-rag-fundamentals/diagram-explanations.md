# Chapter 04: RAG Fundamentals — Diagram Explanations

ASCII diagrams for RAG pipeline architecture, chunking strategies, and evaluation frameworks.

---

## Diagram 1: Complete RAG Architecture — Indexing vs Query Pipeline

```
═══════════════════════════════════════════════════════════════════
                    INDEXING PIPELINE (OFFLINE)
═══════════════════════════════════════════════════════════════════

Raw Documents (PDFs, DOCX, HTML, Markdown, Confluence, Notion...)
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│  DOCUMENT LOADERS                                            │
│  PDF → PyPDF/PDFPlumber   HTML → BeautifulSoup              │
│  DOCX → python-docx       Notion → Notion API               │
│  Common bug: OCR quality, table extraction failures          │
└──────────────────────────────────┬───────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  PREPROCESSING                                               │
│  • Remove headers/footers/page numbers                       │
│  • Fix character encoding (UTF-8)                            │
│  • Normalize whitespace                                      │
│  • Extract metadata (title, author, date, section)           │
└──────────────────────────────────┬───────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  TEXT SPLITTING                                              │
│  Strategy depends on document type:                         │
│  • Recursive: ["\n\n", "\n", ". ", " ", ""]                 │
│  • Markdown-aware: split on ## headers                       │
│  • Semantic: group semantically coherent sentences           │
│  • Token-based: respect embedding model's token limit        │
│                                                              │
│  Parameters: chunk_size=512, overlap=64                     │
└──────────────────────────────────┬───────────────────────────┘
                                   │ chunks with metadata
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  EMBEDDING MODEL (bi-encoder)                                │
│  text → dense vector (e.g., 1536-dim for text-embedding-3)   │
│                                                              │
│  BGE, E5, OpenAI text-embedding-3, sentence-transformers     │
│  Batch processing: 100-1000 chunks per API call              │
└──────────────────────────────────┬───────────────────────────┘
                                   │ (vector, text, metadata)
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  VECTOR STORE                                                │
│  Pinecone / Weaviate / Qdrant / Chroma / pgvector / FAISS   │
│  Stores: vector + original text + metadata                   │
│  Index type: HNSW (default for most), IVFFlat for scale     │
└──────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════
                    QUERY PIPELINE (ONLINE / REAL-TIME)
═══════════════════════════════════════════════════════════════════

User Query: "What is the refund policy for enterprise customers?"
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│  QUERY PREPROCESSING (optional)                              │
│  • Query classification (simple fact vs complex synthesis)   │
│  • Query rewriting / expansion                               │
│  • Entity extraction (to augment metadata filters)           │
└──────────────────────────────────┬───────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  QUERY EMBEDDING                                             │
│  Same model as indexing! (critical for same vector space)    │
│  Latency: 20-80ms                                            │
└──────────────────────────────────┬───────────────────────────┘
                                   │ query vector
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  VECTOR SEARCH + METADATA FILTER                             │
│  ANN search: top-k=5 by cosine similarity                    │
│  Optional filter: {"doc_type": "policy", "version": "2024"} │
│  Latency: 5-50ms (managed DB), 1-5ms (local FAISS)          │
└──────────────────────────────────┬───────────────────────────┘
                                   │ top-k chunks
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  RE-RANKING (optional but recommended)                       │
│  Cross-encoder scores each (query, chunk) pair individually  │
│  Much more accurate than bi-encoder similarity               │
│  Latency: 100-500ms (expensive)                              │
└──────────────────────────────────┬───────────────────────────┘
                                   │ re-ranked, filtered chunks
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  CONTEXT ASSEMBLY                                            │
│  Format: [Source: doc_name, Page: N]\n{chunk_text}          │
│  Budget: fit within (context_window - system_prompt - query) │
└──────────────────────────────────┬───────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────┐
│  LLM GENERATION                                              │
│  system_prompt + context + user_query → answer               │
│  Latency: 500-2000ms (dominant cost in pipeline)             │
└──────────────────────────────────┬───────────────────────────┘
                                   │
                                   ▼
Answer: "Enterprise customers may request refunds within 60 days
        [Source: Enterprise_Policy_v3.pdf, Page 12]"
```

---

## Diagram 2: Chunking Strategy Comparison

```
DOCUMENT: "# Authentication\n\nUsers must provide credentials.\n
           The system uses OAuth 2.0.\n\n# Authorization\n\n
           Access control uses role-based policies..."

STRATEGY 1: Fixed-size (chunk_size=50 chars, overlap=0)
─────────────────────────────────────────────────────────
Chunk 1: "# Authentication\n\nUsers must provide"  ← cuts mid-sentence
Chunk 2: "credentials.\nThe system uses OAuth 2.0"
Chunk 3: ".\n\n# Authorization\n\nAccess control u"  ← crosses section boundary!
Chunk 4: "ses role-based policies..."

Problem: Chunk 3 mixes Authentication and Authorization concepts
         Semantic coherence destroyed

STRATEGY 2: Recursive character splitting (separators: [\n\n, \n, ". ", " "])
─────────────────────────────────────────────────────────────────────────────
  Attempt \n\n split first:
  Block 1: "# Authentication\n\nUsers must provide credentials.\n
            The system uses OAuth 2.0."
  Block 2: "# Authorization\n\nAccess control uses role-based policies..."

  Both blocks < chunk_size → done!
Chunk 1: "# Authentication\n\nUsers must provide credentials.\n
          The system uses OAuth 2.0."          ← coherent section

Chunk 2: "# Authorization\n\nAccess control uses role-based policies..."
          ← coherent section

Result: Sections stay together! Much better semantic coherence.

STRATEGY 3: Semantic chunking
──────────────────────────────
Sentence embeddings:
  S1: "Users must provide credentials."         → emb_1
  S2: "The system uses OAuth 2.0."              → emb_2
  S3: "Access control uses role-based policies" → emb_3

Similarity(emb_1, emb_2) = 0.82  ← HIGH: same topic (authentication)
Similarity(emb_2, emb_3) = 0.31  ← LOW: topic shift! → SPLIT HERE

Chunk 1: "Users must provide credentials. The system uses OAuth 2.0."
Chunk 2: "Access control uses role-based policies..."

Result: Splits exactly at semantic boundary (no header needed!)
Cost: Embeds EVERY sentence during indexing (expensive)
```

---

## Diagram 3: Retrieval Quality Metrics

```
RECALL@k: "Did we find what we needed?"
─────────────────────────────────────────
Given: 4 relevant documents [D1, D2, D3, D4]
Retrieved top-5: [D2, D7, D1, D9, D3]  (D7, D9 are irrelevant)

Recall@5 = |{D1,D2,D3} ∩ {D1,D2,D3,D4}| / |{D1,D2,D3,D4}|
         = 3 / 4 = 0.75

D4 was NOT retrieved → we might miss information from D4

MRR: "How quickly do we find the FIRST relevant document?"
──────────────────────────────────────────────────────────
Retrieved: [D7(irrelevant), D2(relevant), D1, D9, D3]
           rank:  1          rank: 2

MRR = 1/2 = 0.50  (first relevant hit was at rank 2)

If retrieved: [D2, D7, D1, D9, D3]:
MRR = 1/1 = 1.00  (first hit was relevant)

NDCG@k: "Are the MOST relevant docs ranked HIGHEST?"
────────────────────────────────────────────────────
Graded relevance: D1=3 (highly relevant), D2=2, D3=1, others=0

Retrieved: [D1, D2, D3, D7, D9]
DCG@3  = 2^3/log2(2) + 2^2/log2(3) + 2^1/log2(4)
       = 8/1 + 4/1.58 + 2/2
       = 8 + 2.52 + 1 = 11.52

Ideal: [D1, D2, D3, ...] → IDCG@3 = same = 11.52
NDCG@3 = 11.52 / 11.52 = 1.00

If retrieved in wrong order: [D3, D2, D1, D7, D9]
DCG@3  = 2^1/log2(2) + 2^2/log2(3) + 2^3/log2(4)
       = 1/1 + 4/1.58 + 8/2
       = 1 + 2.52 + 4 = 7.52
NDCG@3 = 7.52 / 11.52 = 0.65  ← penalized for wrong order

SUMMARY:
┌─────────┬──────────────────────────────┬────────────────────────┐
│ Metric  │ Best for                     │ Limitation             │
├─────────┼──────────────────────────────┼────────────────────────┤
│Recall@k │ Coverage evaluation          │ Ignores ranking order  │
│MRR      │ First-result quality         │ Only considers rank-1  │
│NDCG@k   │ Ranked list quality          │ Requires graded labels │
└─────────┴──────────────────────────────┴────────────────────────┘
```

---

## Diagram 4: Lost-in-the-Middle — Position Bias in Long Contexts

```
CONTEXT: 10 retrieved documents, ~500 tokens each (5000 total tokens)

LLM Attention Weight Pattern (observed empirically):
─────────────────────────────────────────────────────────────

Attention
Weight
 │
 │ ██  ← Strong: system prompt and first docs (primacy effect)
 │ ██
 │ █▌
 │ █
 │ ▌
 │ ▌    ← Weak: middle documents
 │ ▌
 │ █
 │ ██
 │ ███  ← Strong: last documents (recency effect)
 └──────────────────────────────────────────────────────────►
   Doc1 Doc2 Doc3 Doc4 Doc5 Doc6 Doc7 Doc8 Doc9 Doc10

Accuracy on answering questions from each position:
Position  1: 81%  ████████████████████████████████████████▌
Position  2: 74%  █████████████████████████████████████
Position  3: 69%  ██████████████████████████████████▌
Position  4: 62%  ███████████████████████████████
Position  5: 56%  ████████████████████████████
Position  6: 54%  ███████████████████████████
Position  7: 58%  █████████████████████████████
Position  8: 64%  ████████████████████████████████
Position  9: 71%  ███████████████████████████████████▌
Position 10: 78%  ███████████████████████████████████████

Source: "Lost in the Middle" (Liu et al., 2023)

MITIGATION: Reorder retrieved chunks
Before reordering: [R1, R2, R3, R4, R5] (by similarity score)
After reordering for boundary placement:
  [R1(most_rel), R3, R5, R4, R2(2nd_most_rel)]
   ^─ high attention                   ^─ high attention
        ^─ less attention for lower-relevance middle items
```

---

## Diagram 5: Hybrid Retrieval — BM25 + Dense with RRF Fusion

```
QUERY: "RFC-4512 LDAP authentication requirements"

SPARSE RETRIEVAL (BM25):                DENSE RETRIEVAL (embedding):
─────────────────────────               ──────────────────────────────
Finds exact keyword matches:            Finds semantic matches:

Rank 1: Doc_A (has "RFC-4512")         Rank 1: Doc_D ("LDAP spec overview")
Rank 2: Doc_C (has "authentication")   Rank 2: Doc_E ("directory auth guide")
Rank 3: Doc_B (has "LDAP")             Rank 3: Doc_A ("RFC-4512 authentication")
Rank 4: Doc_E (has "RFC")              Rank 4: Doc_B ("LDAP protocol")
Rank 5: Doc_F (marginal)               Rank 5: Doc_C ("OAuth vs LDAP")

RECIPROCAL RANK FUSION (RRF):
──────────────────────────────
k = 60 (RRF constant, reduces impact of extreme ranks)

Doc_A: BM25_rank=1 → 1/(60+1)=0.0164  |  dense_rank=3 → 1/(60+3)=0.0159
       RRF_score = 0.0164 + 0.0159 = 0.0323  ← appears in BOTH!

Doc_B: BM25_rank=3 → 1/(60+3)=0.0159  |  dense_rank=4 → 1/(60+4)=0.0156
       RRF_score = 0.0159 + 0.0156 = 0.0315

Doc_C: BM25_rank=2 → 1/(60+2)=0.0161  |  dense_rank=5 → 1/(60+5)=0.0154
       RRF_score = 0.0161 + 0.0154 = 0.0315

Doc_D: BM25_rank=N/A (not found) → 0   |  dense_rank=1 → 1/(60+1)=0.0164
       RRF_score = 0.0164

Doc_E: BM25_rank=4 → 1/(60+4)=0.0156  |  dense_rank=2 → 1/(60+2)=0.0161
       RRF_score = 0.0156 + 0.0161 = 0.0317

FINAL RANKING after RRF:
1. Doc_A (0.0323) ← boosted: relevant in BOTH retrieval systems
2. Doc_E (0.0317)
3. Doc_B (0.0315)
4. Doc_C (0.0315)
5. Doc_D (0.0164) ← semantic only match, lower score

Doc_A gets boosted to #1 because it was relevant to BOTH methods.
This is the key advantage of hybrid retrieval over either alone.
```
