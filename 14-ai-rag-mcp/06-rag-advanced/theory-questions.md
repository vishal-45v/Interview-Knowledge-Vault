# Chapter 06: Advanced RAG — Theory Questions

## Hybrid Search & Ranking

1. What is the fundamental difference between BM25 and dense vector retrieval? Under what conditions does BM25 outperform dense retrieval, and why?

2. Explain Reciprocal Rank Fusion (RRF). Walk through the formula `RRF(d) = Σ 1/(k + rank(d))` and explain why k=60 is a common default. What does changing k do to the fusion behavior?

3. You have a corpus where some documents contain very domain-specific acronyms (e.g., "CAGR", "EBITDA") that embeddings don't handle well. How would you design a hybrid retrieval pipeline to address this? What weight ratio would you apply between BM25 and dense scores?

4. What is a cross-encoder reranker and how does it differ from a bi-encoder? Why can't you use a cross-encoder for first-stage retrieval? What is the computational complexity tradeoff?

5. Cohere Rerank, BGE-Reranker-Large, and MonoT5 are all rerankers. What are the architectural differences? Under what latency/quality constraints would you choose one over another?

6. Describe the Hypothetical Document Embeddings (HyDE) technique. What is the underlying assumption about embedding space geometry that makes HyDE work? What are its failure modes?

7. What is query expansion in the context of RAG? How does it differ from multi-query retrieval? What are the risks of query expansion, and how do you mitigate context pollution?

8. Explain step-back prompting. How does it transform a specific query into an abstracted query, and why does this improve retrieval? Give an example with a factual question.

## Advanced Retrieval Architectures

9. Contrast parent-document retrieval with sentence-window retrieval. In what scenarios does each approach produce better results? What are the storage implications?

10. Explain RAPTOR (Recursive Abstractive Processing for Tree-Organized Retrieval). How does the tree structure enable multi-granularity retrieval? How does it handle the trade-off between specificity and breadth?

11. What is self-RAG? Describe the three types of special tokens it uses (IsREL, IsSUP, IsUSE). How does self-RAG decide whether to retrieve at all? How does it differ from standard RAG?

12. What is Corrective RAG (CRAG) and how does it use a retrieval evaluator? Describe the three paths CRAG can take (correct, ambiguous, incorrect). What happens when the evaluator deems retrieved docs irrelevant?

13. Describe agentic RAG. How does it differ from a standard RAG pipeline? What agent patterns (ReAct, Plan-and-Execute) are most commonly used for agentic RAG?

14. Explain Microsoft's GraphRAG. How does it extract entities and relationships? What is the community detection step and why is it important? Compare GraphRAG's strengths vs. standard vector RAG for different query types.

## Evaluation & Optimization

15. Explain the four core RAGAS metrics: faithfulness, answer relevancy, context precision, and context recall. For each, describe what it measures, what LLM calls it makes internally, and what score range is acceptable.

16. What is the difference between context precision and context recall in RAGAS? Which metric tells you about your retriever quality and which tells you about your generator quality?

17. Describe chunk deduplication strategies. What hashing or similarity approaches would you use to detect near-duplicate chunks? What are the risks of aggressive deduplication?

18. How do you design metadata filtering strategies for RAG at scale? Give examples of metadata schemas, pre-filtering vs. post-filtering, and the performance implications of each.

19. Multi-hop reasoning requires retrieving from multiple documents sequentially. Describe two architectures that support multi-hop reasoning and the failure modes specific to multi-hop versus single-hop retrieval.

20. When should you choose RAG over fine-tuning, and when should you choose fine-tuning over RAG? Frame this as a decision tree covering: knowledge currency, knowledge volume, behavioral change, latency, and cost.
