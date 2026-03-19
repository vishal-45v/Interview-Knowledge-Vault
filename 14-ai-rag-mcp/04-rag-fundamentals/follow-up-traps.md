# Chapter 04: RAG Fundamentals — Follow-Up Traps

Common follow-up traps in RAG system interviews. Each trap reveals a misconception that separates production engineers from tutorial-readers.

---

## Trap 1: "Larger chunks are better because they give the model more context"

**What most people say:** Bigger chunks contain more information, so the LLM has more to work with and gives better answers.

**Correct answer:** Larger chunks introduce the precision-recall tradeoff in retrieval. Larger chunks:
1. **Dilute relevance signals**: A 2000-token chunk on "database indexing" might contain one paragraph about B-trees, one about hash indexes, and one about query planning. A query about B-trees retrieves this chunk with lower similarity than a 200-token chunk focused solely on B-trees — the irrelevant content dilutes the embedding.
2. **Waste context window**: If only 10% of a large chunk is relevant to the query, you've used expensive context tokens for noise.
3. **Hurt the embedding**: Most embedding models have context windows of 512 tokens (Sentence-BERT) to 8192 tokens (BGE-M3). Beyond the model's context window, text is truncated — you're not actually encoding the full chunk.

The optimal chunk size depends on the embedding model's context window, the granularity of the query types, and the document structure. A practical heuristic: chunk size should match the typical "unit of retrieval" — for FAQs, that's one Q&A pair; for technical docs, that's one procedure or concept.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Common mistake: max chunk_size without considering embedding model limit
# sentence-transformers/all-MiniLM-L6-v2 has a 256-token effective limit
# Chunks > 256 tokens will be truncated during embedding → wasted tokens

# For all-MiniLM-L6-v2 (256 token limit):
splitter_small = RecursiveCharacterTextSplitter(
    chunk_size=256,        # ~200 words
    chunk_overlap=32,      # 12.5% overlap for continuity
    length_function=len,
)

# For BGE-large (512 token limit):
splitter_medium = RecursiveCharacterTextSplitter(
    chunk_size=512,
    chunk_overlap=64,
)

# For OpenAI text-embedding-3 (8192 token limit):
splitter_large = RecursiveCharacterTextSplitter(
    chunk_size=1500,  # Can go larger, but diminishing returns on retrieval precision
    chunk_overlap=150,
)
```

---

## Trap 2: "Cosine similarity is always the right metric for vector search"

**What most people say:** Cosine similarity measures the angle between vectors and is the standard for semantic similarity.

**Correct answer:** Cosine similarity and dot product give the same RANKING when embeddings are L2-normalized (which most modern embedding models do by default). The choice between cosine, dot product, and Euclidean distance matters in two specific cases:

1. **Non-normalized embeddings**: Some models (e.g., text-embedding-3-large with custom dimensions) do NOT normalize by default. Dot product with non-normalized vectors rewards both direction AND magnitude — larger magnitude (more "confident") vectors score higher even if direction is similar.

2. **HNSW index building**: Most vector databases allow you to choose the distance metric at index creation time. Choosing the wrong metric for your embedding model (cosine metric with an index built on Euclidean distance) produces subtly wrong results.

3. **Hybrid sparse-dense**: BM25 scores are not bounded — you cannot directly add a cosine similarity score ([-1, 1]) to a BM25 score (unbounded). Normalization is required before fusion.

```python
import numpy as np

# When cosine == dot product (L2-normalized vectors):
v1 = np.array([1.0, 2.0, 3.0])
v2 = np.array([4.0, 5.0, 6.0])

v1_norm = v1 / np.linalg.norm(v1)
v2_norm = v2 / np.linalg.norm(v2)

cosine_sim = np.dot(v1_norm, v2_norm)
dot_product = np.dot(v1_norm, v2_norm)  # Same! (already normalized)
print(f"Cosine: {cosine_sim:.4f}, Dot product: {dot_product:.4f}")  # Identical

# When they differ (non-normalized):
cosine_unnorm = np.dot(v1, v2) / (np.linalg.norm(v1) * np.linalg.norm(v2))
dot_unnorm = np.dot(v1, v2)
print(f"Unnorm Cosine: {cosine_unnorm:.4f}, Unnorm Dot: {dot_unnorm:.4f}")
# 0.9746 vs 32.0 — very different rankings for non-normalized vectors!
```

---

## Trap 3: "RAG eliminates hallucination"

**What most people say:** RAG grounds the model in retrieved facts, so it can't hallucinate.

**Correct answer:** RAG significantly reduces factual hallucination but introduces several NEW failure modes:

1. **Faithfulness failures**: Model has retrieved the correct information but does NOT stay grounded in it — it synthesizes across chunks and introduces unsupported claims.
2. **Attribution hallucination**: Model correctly states a fact but incorrectly attributes it to the wrong source document.
3. **Inter-chunk synthesis errors**: Model correctly understands individual chunks but makes errors when combining information across multiple chunks.
4. **Retrieval failure → hallucination**: If the relevant document isn't retrieved (recall failure), the model may hallucinate an answer rather than saying "I don't know."
5. **Overconfident refusal**: Model refuses to answer because retrieved context is incomplete, even when it could give a partial correct answer.

The correct claim is: "RAG reduces hallucination on facts present in the knowledge base, but introduces retrieval-dependent failure modes."

```python
# Detecting faithfulness failures with NLI:
from transformers import pipeline

nli_model = pipeline("text-classification", model="cross-encoder/nli-deberta-v3-base")

def check_faithfulness(retrieved_context: str, generated_answer: str) -> float:
    """
    Score whether generated_answer is entailed by retrieved_context.
    Returns: 1.0 = fully faithful, 0.0 = contradicts or unsupported
    """
    result = nli_model(f"{retrieved_context} </s> {generated_answer}")
    # Labels: ENTAILMENT, NEUTRAL, CONTRADICTION
    scores = {r['label']: r['score'] for r in result}

    # Weighted score: entailment - contradiction
    faithfulness = scores.get('ENTAILMENT', 0) - scores.get('CONTRADICTION', 0)
    return max(0.0, faithfulness)

# Prompt-level faithfulness enforcement:
FAITHFULNESS_PROMPT = """Answer the question using ONLY the information in the provided context.

Rules:
1. If the context contains the answer, state it directly with the source
2. If the context does NOT contain the answer, say exactly: "The provided documents do not contain information about [topic]"
3. Never use your general knowledge to supplement the context
4. Never say "according to the context" — instead cite the specific document

Context:
{context}

Question: {question}"""
```

---

## Trap 4: "Higher top-k always gives better answers"

**What most people say:** Retrieving more documents gives the model more information to work with.

**Correct answer:** Increasing k has diminishing returns and introduces new problems:

1. **Lost in the middle**: Liu et al. (2023) showed that LLMs perform significantly worse at retrieving information from the middle of a long context. Documents at positions 2-8 in a top-10 retrieval are the hardest for the model to use effectively.

2. **Noise introduction**: Beyond the ideal k, retrieved chunks become increasingly irrelevant. Adding noisy context can DECREASE answer quality compared to a smaller, higher-quality context.

3. **Context window exhaustion**: At k=20 with 500-token chunks, you're using 10,000 tokens just for context — leaving little room for reasoning.

4. **Precision vs recall tradeoff**: k=1 has high precision (the top result is almost always relevant) but low recall (may miss complementary information). k=20 has high recall but low precision.

The right approach: use adaptive k based on query type, measure precision@k for your specific workload, and apply re-ranking to maximize quality within a fixed context budget.

---

## Trap 5: "BM25 is outdated — dense retrieval is always better"

**What most people say:** Neural embeddings capture semantic meaning better than keyword matching, so BM25 is obsolete.

**Correct answer:** BM25 consistently outperforms dense retrieval on several query types that matter in practice:

1. **Exact entity lookup**: "Find the document mentioning 'RFC-4512-v3'" — BM25 will find this exactly; dense retrieval may rank semantically similar but unrelated documents higher.
2. **Rare terminology**: Technical jargon, product codes, chemical names, medical identifiers — terms too rare to have robust embedding representations.
3. **Negative queries**: "Documents NOT about Python" — dense retrieval doesn't handle negation; BM25 can be modified to penalize terms.
4. **Low data/cold start**: BM25 requires no training and works immediately; a dense model requires a trained embedding model.

The hybrid approach — combining BM25 scores with dense similarity scores (via Reciprocal Rank Fusion or learned weighting) — consistently outperforms either alone on BEIR benchmarks. Production RAG systems at Bing, Google, and Elasticsearch all use hybrid retrieval.

```python
from rank_bm25 import BM25Okapi
import numpy as np

def hybrid_retrieval(query, documents, embedder, alpha=0.5, top_k=5):
    """
    Hybrid BM25 + dense retrieval with RRF fusion.
    alpha: weight on dense score (1-alpha on BM25)
    """
    # BM25 retrieval
    tokenized_docs = [doc.lower().split() for doc in documents]
    bm25 = BM25Okapi(tokenized_docs)
    bm25_scores = bm25.get_scores(query.lower().split())
    bm25_rankings = np.argsort(bm25_scores)[::-1]

    # Dense retrieval
    query_emb = embedder.encode([query])[0]
    doc_embs = embedder.encode(documents)
    dense_scores = np.dot(doc_embs, query_emb)
    dense_rankings = np.argsort(dense_scores)[::-1]

    # Reciprocal Rank Fusion (RRF)
    # Score = sum(1 / (k + rank)) for each method
    k = 60  # RRF constant (reduces impact of very high rankings)
    rrf_scores = np.zeros(len(documents))

    for rank, doc_idx in enumerate(bm25_rankings):
        rrf_scores[doc_idx] += 1 / (k + rank + 1)

    for rank, doc_idx in enumerate(dense_rankings):
        rrf_scores[doc_idx] += 1 / (k + rank + 1)

    final_rankings = np.argsort(rrf_scores)[::-1]
    return [(documents[i], rrf_scores[i]) for i in final_rankings[:top_k]]
```

---

## Trap 6: "Embedding all your documents once is sufficient"

**What most people say:** You embed your documents during indexing and the embeddings are stable — you don't need to re-embed.

**Correct answer:** Embeddings must be re-generated whenever:

1. **The embedding model changes**: Even a minor version update to your embedding model can produce embeddings in a different vector space. Queries embedded with the new model will not retrieve correctly against documents embedded with the old model.

2. **Chunking strategy changes**: If you change chunk size, overlap, or splitting method, all existing embeddings are for different text units and must be regenerated.

3. **Documents are updated**: Updated documents produce different embeddings. Stale embeddings of old versions will retrieve content that no longer exists.

4. **Domain fine-tuning**: If you fine-tune your embedding model on domain data, existing embeddings are from the un-fine-tuned model and must be regenerated.

The production implication: embedding model upgrades require full re-indexing — potentially hours of compute. Plan for this by versioning your indexes and supporting parallel queries during re-indexing (blue-green index deployment).

```python
class VectorIndexManager:
    def __init__(self, vector_db):
        self.db = vector_db

    def upgrade_embedding_model(self, new_embedder, old_namespace, new_namespace):
        """
        Blue-green index upgrade: re-embed all documents without downtime.
        """
        print(f"Starting re-indexing with new model...")

        # 1. Create new namespace/collection (don't touch old one)
        # 2. Re-embed all documents in batches
        docs = self.db.fetch_all_documents(namespace=old_namespace)

        batch_size = 100
        for i in range(0, len(docs), batch_size):
            batch = docs[i:i+batch_size]
            texts = [doc['text'] for doc in batch]
            new_embeddings = new_embedder.encode(texts)

            for doc, emb in zip(batch, new_embeddings):
                self.db.upsert(
                    namespace=new_namespace,
                    id=doc['id'],
                    vector=emb.tolist(),
                    metadata=doc['metadata']
                )

        # 3. Validate new index quality on test queries
        # 4. Switch query routing to new_namespace
        # 5. Delete old namespace after validation period
        print(f"Re-indexing complete. Validate before switching traffic.")
```

---

## Trap 7: "The generation model is a black box in RAG — just plug it in"

**What most people say:** RAG is model-agnostic — the retrieval is the important part.

**Correct answer:** The generation model's properties significantly affect RAG quality in ways that require specific prompt engineering per model:

1. **Context length**: Different models have different reliable context window sizes. GPT-4 reliably uses 16K tokens; some models degrade before their stated limit.

2. **Instruction following**: More instruction-tuned models are better at "answer ONLY from the context" instructions. Base models may synthesize from parametric knowledge instead.

3. **Citation behavior**: Models differ in their ability to faithfully cite sources and avoid fabricating citations. Claude models are notably more reliable at citation attribution than some alternatives.

4. **Hallucination under pressure**: When asked a question the retrieved context doesn't answer, some models say "I don't know" (correct) while others invent a plausible answer (incorrect). This is a model-specific property.

5. **Knowledge conflicts**: When retrieved context contradicts the model's parametric knowledge, models differ in whether they prioritize the context or their training.

The production decision: evaluate your generation model specifically for RAG scenarios — faithfulness, citation accuracy, knowledge conflict handling, and graceful refusal — not just on general benchmarks.
