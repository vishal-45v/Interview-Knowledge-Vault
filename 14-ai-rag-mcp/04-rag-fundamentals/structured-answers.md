# Chapter 04: RAG Fundamentals — Structured Answers

Complete answers for the most critical RAG engineering interview questions.

---

## Q1: RAG Architecture — Complete Pipeline Description

**Answer:**

RAG has two distinct pipelines with very different operational characteristics:

```
INDEXING PIPELINE (offline / batch):
─────────────────────────────────────

Raw Documents
    │
    ▼
Document Loaders
    │ (PDF parser, HTML stripper, DOCX extractor, etc.)
    │ Common failure: OCR quality in PDFs, table extraction
    ▼
Preprocessing & Cleaning
    │ (remove headers/footers, fix encoding, normalize whitespace)
    │ Common failure: boilerplate noise pollutes chunks
    ▼
Text Splitting / Chunking
    │ (recursive, semantic, markdown-aware, sentence)
    │ Parameters: chunk_size, chunk_overlap
    │ Common failure: splitting mid-sentence or mid-table
    ▼
Metadata Extraction
    │ (source, date, author, section, document_type)
    │ Critical for: filtered retrieval, version management
    ▼
Embedding
    │ (bi-encoder model: BGE, E5, OpenAI text-embedding-3)
    │ Common failure: embedding model truncates large chunks
    ▼
Vector Storage
    │ (Pinecone, Weaviate, Chroma, Qdrant, pgvector)
    │ (also store: original text, metadata, document_id)
    ▼
[Index ready for queries]


QUERY PIPELINE (online / real-time):
──────────────────────────────────────

User Query
    │
    ▼
Query Understanding
    │ (optional: query classification, entity extraction)
    │ (optional: query rewriting for better retrieval)
    ▼
Query Embedding
    │ (same embedding model as indexing — CRITICAL)
    │ Latency: 20-80ms for embedding inference
    ▼
Vector Search (+ optional metadata filter)
    │ (ANN search, top-k=5-20)
    │ Latency: 5-50ms for managed vector DB
    ▼
Optional: Re-ranking
    │ (cross-encoder: ColBERT, Cohere Rerank, BGE-Reranker)
    │ Latency: 100-500ms (expensive but quality improvement)
    ▼
Context Assembly
    │ (format chunks with source citations)
    │ (handle context window budget)
    ▼
LLM Generation
    │ (system prompt + retrieved context + user query)
    │ Latency: 500-2000ms (dominant)
    ▼
Post-processing
    │ (citation verification, output parsing, filtering)
    ▼
Response to User
```

```python
# Complete RAG implementation:
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.document_loaders import PyPDFLoader
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import Chroma

def build_rag_pipeline(pdf_path: str, persist_dir: str):
    # INDEXING PIPELINE
    loader = PyPDFLoader(pdf_path)
    documents = loader.load()

    splitter = RecursiveCharacterTextSplitter(
        chunk_size=512,
        chunk_overlap=64,
        separators=["\n\n", "\n", ". ", " ", ""],
    )
    chunks = splitter.split_documents(documents)

    # Add metadata for filtered retrieval
    for chunk in chunks:
        chunk.metadata["indexed_at"] = "2025-01-01"
        chunk.metadata["version"] = "2.0"

    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    vectorstore = Chroma.from_documents(
        chunks,
        embeddings,
        persist_directory=persist_dir
    )
    return vectorstore

def query_rag_pipeline(vectorstore, question: str, k: int = 5) -> str:
    # QUERY PIPELINE
    retriever = vectorstore.as_retriever(search_kwargs={"k": k})
    retrieved_docs = retriever.invoke(question)

    # Context assembly with citations
    context = "\n\n".join([
        f"[Source: {doc.metadata.get('source', 'unknown')}, "
        f"Page: {doc.metadata.get('page', 'N/A')}]\n{doc.page_content}"
        for doc in retrieved_docs
    ])

    prompt = f"""Answer the question using ONLY the provided context.
If the answer is not in the context, say "I cannot find this in the provided documents."
Always cite which source document your answer comes from.

Context:
{context}

Question: {question}

Answer:"""

    llm = ChatOpenAI(model="gpt-4o", temperature=0)
    return llm.invoke(prompt).content
```

---

## Q2: Chunking Strategies — When to Use Which

**Answer:**

```python
from langchain.text_splitter import (
    RecursiveCharacterTextSplitter,
    MarkdownTextSplitter,
    SentenceTransformersTokenTextSplitter,
)
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

# STRATEGY 1: Fixed-size chunking (baseline, fastest)
# Best for: homogeneous, well-structured prose documents
# Worst for: technical docs with code blocks, markdown headers
fixed_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,
)

# STRATEGY 2: Recursive character splitting (practical default)
# Best for: mixed format text, handles common structure separators
# The separators try: \n\n first, then \n, then " ", then char-by-char
recursive_splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,
    separators=["\n\n", "\n", ". ", " ", ""],
)

# STRATEGY 3: Markdown-aware splitting
# Best for: Confluence, Notion, README files, technical documentation
# Splits on headers, preserving document structure
md_splitter = MarkdownTextSplitter(chunk_size=512)

# STRATEGY 4: Token-based splitting (use when embedding has token limit)
# Best for: ensuring you don't exceed embedding model's token context
# sentence-transformers/all-MiniLM-L6-v2 has 256 token limit
token_splitter = SentenceTransformersTokenTextSplitter(
    model_name="sentence-transformers/all-MiniLM-L6-v2",
    chunk_overlap=32,
    tokens_per_chunk=256,
)

# STRATEGY 5: Semantic chunking (highest quality, slowest)
# Groups sentences that are semantically similar together
# Does NOT split mid-semantic-unit
# Cost: embeds every sentence during indexing
semantic_splitter = SemanticChunker(
    OpenAIEmbeddings(),
    breakpoint_threshold_type="percentile",  # or "standard_deviation"
    breakpoint_threshold_amount=95,  # Split when similarity drops below 95th percentile
)

# CHOOSING A STRATEGY:
# ┌─────────────────┬─────────────────────────────────────────────┐
# │ Document type   │ Recommended strategy                        │
# ├─────────────────┼─────────────────────────────────────────────┤
# │ FAQ pages       │ Split on Q: prefix (custom splitter)        │
# │ Markdown docs   │ MarkdownTextSplitter with header awareness  │
# │ Legal contracts │ Semantic + recursive (preserve paragraphs)  │
# │ Code + prose    │ Code-aware splitter (split on function def) │
# │ Short docs (<1K)│ Don't chunk — index whole document         │
# │ Financial tables│ Table-aware + summarize tables separately   │
# └─────────────────┴─────────────────────────────────────────────┘
```

---

## Q3: Retrieval Quality Metrics — Complete Evaluation Framework

**Answer:**

RAG evaluation has two separate concerns: retrieval quality and generation quality.

```python
import numpy as np
from typing import List, Tuple

# RETRIEVAL METRICS:

def recall_at_k(relevant_docs: List[str], retrieved_docs: List[str], k: int) -> float:
    """
    Recall@k: What fraction of relevant documents were retrieved in top-k?
    Interpretation: "Did we find what we needed?"
    Range: [0, 1]. Target: > 0.85 for production RAG.
    """
    retrieved_set = set(retrieved_docs[:k])
    relevant_set = set(relevant_docs)
    if not relevant_set:
        return 1.0  # No relevant docs → perfect recall by convention
    return len(retrieved_set & relevant_set) / len(relevant_set)

def mrr(relevant_docs: List[str], retrieved_docs: List[str]) -> float:
    """
    Mean Reciprocal Rank: 1/rank_of_first_relevant_document.
    Interpretation: "How quickly do we find the first relevant document?"
    Range: (0, 1]. MRR=1.0 means the first result is always relevant.
    Best for: single-answer queries where we need the top-1 to be correct.
    """
    relevant_set = set(relevant_docs)
    for rank, doc in enumerate(retrieved_docs, start=1):
        if doc in relevant_set:
            return 1.0 / rank
    return 0.0  # No relevant document found

def ndcg_at_k(relevant_grades: dict, retrieved_docs: List[str], k: int) -> float:
    """
    NDCG@k: Normalized Discounted Cumulative Gain.
    Interpretation: "How well ordered are the relevant documents?"
    Accounts for graded relevance (grade 3 = highly relevant, grade 1 = marginally).
    Better than MRR when relevance is not binary.
    Range: [0, 1]. Target: > 0.70 for production RAG.
    """
    def dcg(rankings):
        return sum(
            (2 ** relevant_grades.get(doc, 0) - 1) / np.log2(rank + 2)
            for rank, doc in enumerate(rankings[:k])
        )

    actual_dcg = dcg(retrieved_docs)
    ideal_ranking = sorted(relevant_grades.keys(),
                          key=lambda d: relevant_grades[d], reverse=True)
    ideal_dcg = dcg(ideal_ranking)

    return actual_dcg / ideal_dcg if ideal_dcg > 0 else 0.0


# GENERATION METRICS (using RAGAS framework):

def faithfulness_score(context: str, answer: str, llm_judge) -> float:
    """
    Faithfulness: What fraction of claims in the answer are supported by context?
    High faithfulness = model is grounded in retrieved content.
    Low faithfulness = model is hallucinating or using parametric knowledge.
    """
    prompt = f"""Given the context and answer below, extract ALL factual claims
    in the answer. For each claim, determine if it is SUPPORTED, UNSUPPORTED, or
    CONTRADICTED by the context. Return JSON:
    {{"total_claims": N, "supported": M, "faithfulness": M/N}}

    Context: {context}
    Answer: {answer}"""

    result = llm_judge(prompt)
    return result["faithfulness"]

def answer_relevance_score(question: str, answer: str, embedder) -> float:
    """
    Answer relevance: Does the answer actually address the question?
    High = answer is on-topic and complete.
    Low = answer is off-topic, evasive, or addresses wrong aspect.
    """
    # Generate N hypothetical questions that the given answer could answer
    # If these hypothetical questions are semantically similar to the original,
    # then the answer is relevant.
    hypothetical_questions = generate_hypothetical_questions(answer, n=5)
    original_emb = embedder.encode([question])[0]
    hyp_embs = embedder.encode(hypothetical_questions)

    similarities = [
        np.dot(original_emb, h_emb) /
        (np.linalg.norm(original_emb) * np.linalg.norm(h_emb))
        for h_emb in hyp_embs
    ]
    return np.mean(similarities)

# Complete evaluation dashboard:
def evaluate_rag_system(test_cases: List[dict], rag_system) -> dict:
    """
    test_cases: [{"query": str, "relevant_docs": List[str], "expected_answer": str}]
    """
    retrieval_recalls = []
    mrrs = []
    faithfulness_scores = []
    relevance_scores = []

    for case in test_cases:
        retrieved, answer = rag_system.query(case["query"])
        retrieved_ids = [doc.id for doc in retrieved]

        retrieval_recalls.append(recall_at_k(case["relevant_docs"], retrieved_ids, k=5))
        mrrs.append(mrr(case["relevant_docs"], retrieved_ids))
        # ... compute generation metrics

    return {
        "recall@5": np.mean(retrieval_recalls),
        "mrr": np.mean(mrrs),
        "faithfulness": np.mean(faithfulness_scores),
        "answer_relevance": np.mean(relevance_scores),
    }
```

---

## Q4: Embedding Model Selection — Decision Framework

**Answer:**

```python
# SELECTION CRITERIA (in priority order):

# 1. TASK TYPE (asymmetric vs symmetric search)
# Asymmetric: short query retrieves long document (most RAG)
#   → Use: BGE-large, E5-large, OpenAI text-embedding-3
#   → These are trained with query-document pairs (asymmetric objective)
# Symmetric: similar-length text pairs (dedup, clustering)
#   → Use: all-MiniLM-L6-v2, all-mpnet-base-v2
#   → These are trained with sentence pairs (symmetric objective)

# 2. DOMAIN ALIGNMENT
from sentence_transformers import SentenceTransformer

def evaluate_embedding_model(model_name: str, test_pairs: List[Tuple[str, str, int]]):
    """
    test_pairs: [(query, document, relevant_1_or_0), ...]
    Compute in-domain retrieval accuracy before committing to a model.
    """
    model = SentenceTransformer(model_name)

    queries = [p[0] for p in test_pairs]
    docs = [p[1] for p in test_pairs]
    labels = [p[2] for p in test_pairs]

    query_embs = model.encode(queries)
    doc_embs = model.encode(docs)

    similarities = []
    for qe, de in zip(query_embs, doc_embs):
        sim = np.dot(qe, de) / (np.linalg.norm(qe) * np.linalg.norm(de))
        similarities.append(sim)

    # AUC-ROC: can the model distinguish relevant from irrelevant?
    from sklearn.metrics import roc_auc_score
    auc = roc_auc_score(labels, similarities)
    print(f"{model_name}: AUC-ROC = {auc:.4f}")
    return auc

# 3. LATENCY AND COST
# Model               | Dims | Latency (batch=32) | Cost/1M tokens
# ────────────────────┼──────┼───────────────────┼───────────────
# all-MiniLM-L6-v2   |  384 | 10ms (local GPU)  | Free
# BGE-base-en-v1.5   |  768 | 25ms (local GPU)  | Free
# BGE-large-en-v1.5  | 1024 | 60ms (local GPU)  | Free
# E5-mistral-7b      | 4096 | 300ms (A100)       | Free (self-hosted)
# text-embedding-3-small | 1536 | 20ms (API)   | $0.02/1M tokens
# text-embedding-3-large | 3072 | 30ms (API)   | $0.13/1M tokens

# 4. MULTILINGUAL REQUIREMENTS
# BGE-M3: 100+ languages, supports dense + sparse + multi-vector
# multilingual-e5-large: strong cross-lingual retrieval
# paraphrase-multilingual-mpnet-base-v2: lightweight multilingual

# 5. CONTEXT WINDOW
# all-MiniLM-L6-v2: 256 tokens (hard limit)
# BGE-large-en-v1.5: 512 tokens
# OpenAI text-embedding-3: 8191 tokens
# E5-mistral-7b: 32768 tokens

# RECOMMENDATION BY USE CASE:
use_case_recommendations = {
    "general_rag_english": "text-embedding-3-small (cost-quality balance)",
    "high_quality_english": "text-embedding-3-large or BGE-large",
    "privacy_required": "BGE-large-en-v1.5 (self-hosted)",
    "multilingual": "BGE-M3 or multilingual-e5-large",
    "long_documents": "text-embedding-3-large (8K context) or E5-mistral-7b",
    "speed_critical": "all-MiniLM-L6-v2 (local, 384-dim, FAST)",
}
```

---

## Q5: The Lost-in-the-Middle Problem — Understanding and Mitigation

**Answer:**

Liu et al. (2023) demonstrated that when relevant information is placed in the middle of a long context (as opposed to the beginning or end), LLMs perform significantly worse at retrieving and using that information. Performance dropped from ~80% (information at start/end) to ~55% (information in the middle) on multi-document question answering tasks.

**Why it happens:** Transformers' attention patterns during fine-tuning on instruction-following tend to create strong attention to beginning tokens (system prompt) and recent tokens (end of context). Middle tokens receive relatively less attention weight.

```python
# MITIGATION STRATEGY 1: Reorder retrieved chunks (most relevant at boundaries)
def reorder_for_lost_in_middle(retrieved_chunks: list, query: str, embedder) -> list:
    """
    Place most relevant chunks at positions 1 and N (boundaries).
    Place less relevant chunks in the middle.
    """
    if len(retrieved_chunks) <= 2:
        return retrieved_chunks

    query_emb = embedder.encode([query])[0]
    similarities = []
    for chunk in retrieved_chunks:
        chunk_emb = embedder.encode([chunk.text])[0]
        sim = np.dot(query_emb, chunk_emb)
        similarities.append(sim)

    indexed = list(enumerate(similarities))
    sorted_by_relevance = sorted(indexed, key=lambda x: x[1], reverse=True)

    # Interleave: most relevant → positions 0 and -1
    # Second most relevant → positions 1 and -2
    reordered_indices = []
    left, right = [], []
    for i, (idx, _) in enumerate(sorted_by_relevance):
        if i % 2 == 0:
            left.append(idx)
        else:
            right.append(idx)

    reordered_indices = left + right[::-1]
    return [retrieved_chunks[i] for i in reordered_indices]


# MITIGATION STRATEGY 2: Reduce k — better to have 3 highly relevant chunks
# than 10 with the relevant one buried in the middle

# MITIGATION STRATEGY 3: Reranking — ensure top-k after reranking are
# actually the most relevant, then use a small k for the final context

# MITIGATION STRATEGY 4: Prompt instructions
ANTI_LOST_IN_MIDDLE_PROMPT = """You will be given multiple context documents.
IMPORTANT: Read ALL documents carefully, including those in the middle.
Do not rely only on the first or last document.
The relevant information may appear in any document."""
```
