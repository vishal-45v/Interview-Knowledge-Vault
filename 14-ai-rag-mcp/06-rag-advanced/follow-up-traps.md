# Chapter 06: Advanced RAG — Follow-Up Traps

## Trap 1: "RRF is better than weighted linear combination for hybrid search"

**What most people say:** "Yes, RRF is always better because it's parameter-free and doesn't require tuning the weights."

**Correct answer:** RRF is more robust *when you don't know the relative quality of your retrievers*, but it is NOT always better. RRF treats rank positions as the only signal — it discards the actual relevance scores entirely. If your dense retriever assigns very high scores to genuinely relevant documents and very low scores to irrelevant ones (high separation), weighted linear combination preserves that signal; RRF throws it away. The k=60 constant also means that a document ranked #1 gets score 1/61 ≈ 0.016, while a document ranked #60 gets 1/120 ≈ 0.008 — only a 2x difference for a 60-position gap. For production systems where you've measured your retrievers' quality, a tuned weighted combination often outperforms RRF. Use RRF as a strong baseline, but measure both.

```python
def rrf_score(rankings: list[list[str]], k: int = 60) -> dict[str, float]:
    scores = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank)
    return dict(sorted(scores.items(), key=lambda x: x[1], reverse=True))

def weighted_hybrid(dense_scores: dict, sparse_scores: dict, alpha: float = 0.7):
    # alpha controls dense weight; requires score normalization first
    all_ids = set(dense_scores) | set(sparse_scores)
    return {
        doc_id: alpha * dense_scores.get(doc_id, 0) + (1 - alpha) * sparse_scores.get(doc_id, 0)
        for doc_id in all_ids
    }
```

---

## Trap 2: "HyDE always improves retrieval performance"

**What most people say:** "HyDE generates a hypothetical answer and embeds it, which is always better because it's closer in embedding space to real documents."

**Correct answer:** HyDE can hurt performance in two important cases. First, if the LLM generates a confidently wrong hypothetical document, you embed hallucinated content and retrieve irrelevant documents — the error is amplified, not corrected. Second, for factual lookup queries ("What is the boiling point of water?"), HyDE is overkill and adds latency; the original query embeds fine. HyDE shines for abstract or vague queries where the embedding of the query itself is far from document embeddings. A better production approach is to run HyDE conditionally: classify the query, apply HyDE only for open-ended/conceptual queries, and use direct retrieval for specific factual queries.

```python
def conditional_hyde(query: str, llm, embedder) -> list[float]:
    query_type = classify_query(query)  # "factual" | "conceptual"
    if query_type == "conceptual":
        hypothetical = llm.generate(
            f"Write a paragraph that directly answers: {query}"
        )
        return embedder.embed(hypothetical)
    else:
        return embedder.embed(query)
```

---

## Trap 3: "More chunks retrieved = better recall"

**What most people say:** "Retrieve top-20 or top-50 to maximize the chance that the right document is in there."

**Correct answer:** Increasing k improves recall but degrades precision and, critically, LLM generation quality. When you flood the context with 50 chunks, the LLM suffers from "lost in the middle" — it pays less attention to relevant information buried in the middle of a long context. Studies show LLM accuracy on relevant facts drops significantly when that fact is in the middle vs. beginning/end of context. The right approach is: retrieve a larger set (top-20 to top-50), then rerank with a cross-encoder, and finally pass only the top 3-5 post-reranked chunks to the LLM. The reranking step is where you compress without losing relevant content.

---

## Trap 4: "RAGAS faithfulness measures whether the answer is correct"

**What most people say:** "High faithfulness means the answer is accurate and factually true."

**Correct answer:** Faithfulness measures whether the answer is *grounded in the retrieved context*, not whether it is *factually correct in the real world*. If your retriever returns an incorrect document and the LLM faithfully summarizes that incorrect document, faithfulness will be 1.0 but the answer is wrong. Faithfulness = 0 means the LLM is hallucinating *beyond* the retrieved context. You need BOTH high faithfulness (grounded generation) AND high context recall (correct documents retrieved) to guarantee a correct answer. This is why RAGAS requires all four metrics together.

```python
# Faithfulness checks: are all claims in the answer supported by context?
# It does NOT check if the context is correct.
faithfulness_prompt = """
Given context: {context}
Given answer: {answer}
List each factual claim in the answer. For each, is it supported by the context?
Return: number of supported claims / total claims
"""
```

---

## Trap 5: "Cross-encoders are just better bi-encoders"

**What most people say:** "Cross-encoders are always superior, you should use them for retrieval if you can afford the compute."

**Correct answer:** Cross-encoders cannot be used for first-stage retrieval because they require the query AND document as joint input — you cannot pre-compute document representations. This means you'd need to run inference for every (query, document) pair in your corpus at query time, which is O(N) inference calls where N is corpus size. For a 10M document corpus this is completely infeasible. Cross-encoders are only viable as rerankers over a small candidate set (typically top 50-200 from first-stage retrieval). The question "can I use a cross-encoder for retrieval" reveals a fundamental misunderstanding of the architecture. The correct mental model: bi-encoder = index-time document encoding + query-time query encoding + ANN search (fast); cross-encoder = inference over (query, document) pairs (slow, accurate).

---

## Trap 6: "GraphRAG is better than vector RAG"

**What most people say:** "GraphRAG extracts relationships so it's more powerful and should replace vector RAG."

**Correct answer:** GraphRAG is better for *global, relational, and thematic queries* (e.g., "What are the main themes across all these documents?" "How are entities A and B connected?"). Standard vector RAG is better for *specific, local queries* (e.g., "What does document X say about Y?"). GraphRAG has much higher indexing cost — it requires LLM calls to extract entities and relationships for every chunk, making it 10-100x more expensive to index. Microsoft's own benchmarks show GraphRAG dominates on "sensemaking" queries but is competitive with or inferior to vector RAG on specific retrieval queries. The right architecture in production is often a hybrid: vector RAG for specific queries + GraphRAG for global/analytical queries, with a query router deciding which path to use.

---

## Trap 7: "Self-RAG decides to retrieve or not based on query complexity"

**What most people say:** "Self-RAG uses some complexity classifier to decide whether to retrieve."

**Correct answer:** Self-RAG uses a special token called `[Retrieve]` that the model itself generates during inference — it's trained to emit this token when it determines retrieval would be helpful. After retrieval, it generates `[IsREL]` (is the retrieved passage relevant?), `[IsSUP]` (does the passage support the claim being generated?), and `[IsUSE]` (is the overall response useful?). The model is fine-tuned to generate these tokens, so the decision is embedded in the model weights, not a separate classifier. This is architecturally very different from having a routing layer. The implication is: self-RAG requires a specially fine-tuned model; you can't add this capability to an arbitrary LLM without training.

---

## Trap 8: "Parent-document retrieval solves the chunk size dilemma"

**What most people say:** "Parent-document retrieval is the best chunking strategy because you index small chunks but return large parent chunks, so you get the best of both worlds."

**Correct answer:** Parent-document retrieval solves the *retrieval precision* vs. *generation context* tradeoff, but introduces new problems. First, the parent chunk may still not have the right scope — a 2000-token parent might include the target sentence plus a lot of noise. Second, if multiple child chunks from different parents are retrieved, you may return multiple large parent chunks, quickly exhausting your context window. Third, parents must be stored in a docstore keyed by child chunk references, adding infrastructure complexity. Sentence-window retrieval (return k sentences around the matched sentence) is often more precise. The real decision is based on document structure: for well-structured technical docs (sections/subsections), parent-document is clean; for narrative text, sentence-window is usually better.

---

## Trap 9: "CRAG just re-queries the web when retrieval fails"

**What most people say:** "Corrective RAG falls back to web search when the local retriever doesn't find good results."

**Correct answer:** CRAG has three distinct paths based on the retrieval evaluator's confidence score: (1) **Correct** — confidence above upper threshold, use retrieved docs directly; (2) **Ambiguous** — confidence between thresholds, use retrieved docs but also perform web search and combine; (3) **Incorrect** — confidence below lower threshold, abandon retrieved docs entirely and use web search. The key architectural insight is the *retrieval evaluator* — a fine-tuned classifier (not another LLM call) that estimates document relevance. The web search in CRAG also goes through a knowledge refinement step to strip noise from search results before passing to the generator. It's not a simple fallback; it's a three-way conditional pipeline with quality control at each stage.

---

## Trap 10: "RAG vs fine-tuning is a clear binary choice"

**What most people say:** "Use RAG for dynamic knowledge, use fine-tuning for style/behavior changes."

**Correct answer:** The real decision is multidimensional, and the best systems often use both. Fine-tuning teaches the model *how to behave* (format, style, reasoning patterns, domain-specific terminology); RAG provides *what to know* (specific facts, current information). If you fine-tune without RAG, you need to retrain every time facts change. If you use RAG without fine-tuning, the model may not format outputs correctly, may not know domain vocabulary, and may not apply domain reasoning patterns. The nuanced decision tree: (1) If knowledge is updated daily → RAG required; (2) If you need to change output format/style significantly → fine-tune; (3) If domain has specialized vocabulary that base model misunderstands → fine-tune; (4) If knowledge volume exceeds context window → RAG required; (5) If latency is critical → fine-tune to reduce context length. Many production systems use RAG + LoRA fine-tuning together.
