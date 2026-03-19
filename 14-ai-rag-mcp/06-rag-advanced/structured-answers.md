# Chapter 06: Advanced RAG — Structured Answers

## Q1: Explain the complete hybrid search pipeline with RRF — how does it work end-to-end?

Hybrid search combines sparse (lexical) retrieval with dense (semantic) retrieval and fuses the ranked results. Here is a production-grade implementation:

```python
from rank_bm25 import BM25Okapi
from sentence_transformers import SentenceTransformer
import numpy as np
from typing import List, Tuple

class HybridRetriever:
    def __init__(self, documents: List[str], model_name: str = "BAAI/bge-large-en-v1.5"):
        self.documents = documents
        self.encoder = SentenceTransformer(model_name)

        # Sparse index: BM25 over tokenized documents
        tokenized = [doc.lower().split() for doc in documents]
        self.bm25 = BM25Okapi(tokenized)

        # Dense index: pre-compute embeddings (in prod: store in vector DB)
        self.doc_embeddings = self.encoder.encode(documents, normalize_embeddings=True)

    def retrieve_bm25(self, query: str, top_k: int = 50) -> List[Tuple[int, float]]:
        tokenized_query = query.lower().split()
        scores = self.bm25.get_scores(tokenized_query)
        top_indices = np.argsort(scores)[::-1][:top_k]
        return [(idx, scores[idx]) for idx in top_indices]

    def retrieve_dense(self, query: str, top_k: int = 50) -> List[Tuple[int, float]]:
        query_embedding = self.encoder.encode(query, normalize_embeddings=True)
        similarities = np.dot(self.doc_embeddings, query_embedding)
        top_indices = np.argsort(similarities)[::-1][:top_k]
        return [(idx, similarities[idx]) for idx in top_indices]

    def rrf_fusion(
        self,
        rankings: List[List[Tuple[int, float]]],
        k: int = 60
    ) -> List[Tuple[int, float]]:
        """Reciprocal Rank Fusion across multiple ranked lists."""
        rrf_scores = {}
        for ranking in rankings:
            for rank, (doc_id, _) in enumerate(ranking, start=1):
                rrf_scores[doc_id] = rrf_scores.get(doc_id, 0.0) + 1.0 / (k + rank)
        return sorted(rrf_scores.items(), key=lambda x: x[1], reverse=True)

    def retrieve(self, query: str, top_k: int = 10) -> List[Tuple[str, float]]:
        bm25_results = self.retrieve_bm25(query, top_k=50)
        dense_results = self.retrieve_dense(query, top_k=50)

        fused = self.rrf_fusion([bm25_results, dense_results], k=60)

        return [(self.documents[idx], score) for idx, score in fused[:top_k]]
```

**Key insight**: RRF with k=60 was empirically found to work across retrieval tasks. The k constant controls how much the rank differences matter at the top — lower k means the #1 document gets a much bigger boost over #2; higher k flattens the distribution. k=60 is a good compromise.

---

## Q2: Walk through a complete reranking pipeline using a cross-encoder

```python
from sentence_transformers import CrossEncoder
from typing import List, Tuple

class RerankedRetriever:
    def __init__(self, first_stage_retriever, reranker_model: str = "cross-encoder/ms-marco-MiniLM-L-6-v2"):
        self.retriever = first_stage_retriever
        self.reranker = CrossEncoder(reranker_model, max_length=512)

    def retrieve_and_rerank(
        self,
        query: str,
        first_stage_k: int = 50,
        final_k: int = 5
    ) -> List[Tuple[str, float]]:
        # Stage 1: fast approximate retrieval (ANN)
        candidates = self.retriever.retrieve(query, top_k=first_stage_k)

        # Stage 2: accurate cross-encoder reranking
        # Cross-encoder takes (query, document) pairs — no pre-computation possible
        pairs = [(query, doc) for doc, _ in candidates]
        rerank_scores = self.reranker.predict(pairs)

        # Sort by reranker score
        reranked = sorted(
            zip([doc for doc, _ in candidates], rerank_scores),
            key=lambda x: x[1],
            reverse=True
        )
        return reranked[:final_k]

# Cohere Rerank alternative (API-based, no GPU required)
import cohere

def cohere_rerank(query: str, documents: List[str], top_n: int = 5) -> List[dict]:
    co = cohere.Client("YOUR_API_KEY")
    response = co.rerank(
        model="rerank-english-v3.0",
        query=query,
        documents=documents,
        top_n=top_n,
        return_documents=True
    )
    return [
        {"document": result.document.text, "relevance_score": result.relevance_score}
        for result in response.results
    ]
```

**Latency profile**: First-stage ANN retrieval ~5-20ms. Cross-encoder reranking over 50 candidates ~100-300ms on CPU, ~20-50ms on GPU. Cohere API reranking ~200-400ms including network. For latency-sensitive applications, use a smaller cross-encoder (MiniLM-L6) or Cohere's cloud API.

---

## Q3: Implement HyDE and explain when to use it vs. standard retrieval

```python
from openai import OpenAI
from sentence_transformers import SentenceTransformer
import numpy as np

client = OpenAI()
encoder = SentenceTransformer("BAAI/bge-large-en-v1.5")

def generate_hypothetical_document(query: str) -> str:
    """Generate a hypothetical document that would answer the query."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",  # cheap, fast — don't need full GPT-4 for this
        messages=[
            {
                "role": "system",
                "content": (
                    "Write a concise passage (2-3 paragraphs) that directly answers "
                    "the following question. Write as if it were an excerpt from a "
                    "technical document. Do not say 'I' or reference the question."
                )
            },
            {"role": "user", "content": query}
        ],
        temperature=0.7,
        max_tokens=300
    )
    return response.choices[0].message.content

def hyde_embed(query: str, n_hypothetical: int = 3) -> np.ndarray:
    """
    Generate multiple hypothetical docs and average their embeddings.
    Averaging reduces the impact of any single hallucinated document.
    """
    hypotheticals = [generate_hypothetical_document(query) for _ in range(n_hypothetical)]
    embeddings = encoder.encode(hypotheticals, normalize_embeddings=True)
    # Average and renormalize
    avg_embedding = np.mean(embeddings, axis=0)
    avg_embedding = avg_embedding / np.linalg.norm(avg_embedding)
    return avg_embedding

# When to use HyDE:
# GOOD:  "What are the best practices for managing technical debt in microservices?"
# GOOD:  "How does attention mechanism relate to human cognitive attention?"
# BAD:   "What is the boiling point of ethanol?"  (factual lookup, direct query works better)
# BAD:   Any query where hallucination risk is high (medical, legal with specific facts)
```

---

## Q4: Implement RAGAS evaluation for a complete RAG pipeline

```python
from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset
from typing import List

def evaluate_rag_pipeline(
    questions: List[str],
    answers: List[str],
    contexts: List[List[str]],  # list of retrieved chunks per question
    ground_truths: List[str]    # reference answers for recall computation
) -> dict:
    """
    faithfulness:    are answer claims supported by retrieved context? [0, 1]
    answer_relevancy: is the answer relevant to the question? [0, 1]
    context_precision: are retrieved chunks actually relevant? (precision) [0, 1]
    context_recall:   do retrieved chunks cover the ground truth? [0, 1]
    """
    data = {
        "question": questions,
        "answer": answers,
        "contexts": contexts,
        "ground_truth": ground_truths,
    }
    dataset = Dataset.from_dict(data)

    result = evaluate(
        dataset,
        metrics=[faithfulness, answer_relevancy, context_precision, context_recall],
    )

    return {
        "faithfulness": result["faithfulness"],
        "answer_relevancy": result["answer_relevancy"],
        "context_precision": result["context_precision"],
        "context_recall": result["context_recall"],
        "interpretation": interpret_scores(result),
    }

def interpret_scores(result: dict) -> str:
    issues = []
    if result["context_recall"] < 0.7:
        issues.append("RETRIEVER: not finding relevant documents (low recall)")
    if result["context_precision"] < 0.7:
        issues.append("RETRIEVER: returning too many irrelevant documents (low precision)")
    if result["faithfulness"] < 0.8:
        issues.append("GENERATOR: hallucinating beyond retrieved context (low faithfulness)")
    if result["answer_relevancy"] < 0.8:
        issues.append("GENERATOR: answers are off-topic or incomplete (low answer relevancy)")
    return "; ".join(issues) if issues else "All metrics acceptable"

# Diagnostic matrix:
# High faithfulness + Low context_recall  -> Retriever problem (not finding right docs)
# Low faithfulness + High context_recall  -> Generator problem (LLM ignoring context)
# Low context_precision                   -> Chunking/indexing problem (noise in results)
```

---

## Q5: RAG vs. Fine-Tuning Decision Framework

```python
def rag_vs_finetune_decision(requirements: dict) -> str:
    """
    requirements keys:
    - knowledge_update_frequency: "daily" | "monthly" | "static"
    - knowledge_volume_docs: int
    - needs_behavior_change: bool  (format, style, reasoning patterns)
    - latency_budget_ms: int
    - explainability_required: bool
    - budget_usd_per_month: float
    """
    score_rag = 0
    score_ft = 0
    reasons = []

    freq = requirements.get("knowledge_update_frequency", "monthly")
    if freq == "daily":
        score_rag += 3
        reasons.append("Daily updates: RAG strongly preferred (no retraining cost)")
    elif freq == "static":
        score_ft += 1
        reasons.append("Static knowledge: fine-tuning viable")

    vol = requirements.get("knowledge_volume_docs", 1000)
    if vol > 10_000:
        score_rag += 2
        reasons.append(f"{vol} docs exceeds context window: RAG required")

    if requirements.get("needs_behavior_change", False):
        score_ft += 2
        reasons.append("Behavior/format change: fine-tuning required")

    if requirements.get("latency_budget_ms", 2000) < 500:
        score_ft += 1
        reasons.append("Low latency (<500ms): fine-tuning reduces retrieval overhead")

    if requirements.get("explainability_required", False):
        score_rag += 2
        reasons.append("Explainability: RAG provides source attribution")

    decision = "RAG" if score_rag > score_ft else "Fine-Tuning"
    if abs(score_rag - score_ft) <= 1:
        decision = "Both (RAG + LoRA Fine-Tuning)"

    return f"Recommendation: {decision}\nReasons: {'; '.join(reasons)}"

# Real-world rule of thumb:
# - Knowledge changes     -> RAG
# - Behavior changes      -> Fine-tune
# - Both change           -> RAG + PEFT (LoRA)
# - Only style/format     -> Few-shot prompting first, fine-tune if insufficient
```

---

## Q6: Implement Parent-Document Retrieval

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.storage import InMemoryStore
from langchain.retrievers import ParentDocumentRetriever
from langchain_community.vectorstores import Chroma
from langchain_openai import OpenAIEmbeddings
from langchain.schema import Document

def build_parent_document_retriever(documents: List[Document]):
    embeddings = OpenAIEmbeddings()
    vectorstore = Chroma(embedding_function=embeddings)
    docstore = InMemoryStore()  # In prod: use Redis or a persistent KV store

    # Child splitter: small chunks for precise retrieval
    child_splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=20)

    # Parent splitter: larger chunks for complete context
    parent_splitter = RecursiveCharacterTextSplitter(chunk_size=2000, chunk_overlap=200)

    retriever = ParentDocumentRetriever(
        vectorstore=vectorstore,
        docstore=docstore,
        child_splitter=child_splitter,
        parent_splitter=parent_splitter,
        search_kwargs={"k": 5},  # retrieve top-5 parents
    )

    retriever.add_documents(documents)
    return retriever

# The flow:
# Index time: split docs into parents (2000 tokens) AND children (200 tokens)
#             embed children, store parents in docstore keyed by parent_id
# Query time: embed query, ANN search over child embeddings,
#             look up parent_id for each matched child, return parent chunks
# Result:     precise retrieval (child matched) + full context (parent returned)
```
