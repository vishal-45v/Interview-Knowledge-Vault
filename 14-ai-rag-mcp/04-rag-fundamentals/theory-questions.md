# Chapter 04: RAG Fundamentals — Theory Questions

Senior-level questions on retrieval-augmented generation. Tests whether you understand the full stack — indexing, embedding, retrieval, and generation — not just the surface concept.

---

1. Describe the complete RAG architecture from document ingestion to answer generation. Distinguish between the indexing pipeline (offline) and the query pipeline (online), including every component in each and the failure modes specific to each stage.

2. Fixed-size chunking with overlap is the simplest chunking strategy. Explain why the chunk size and overlap are NOT arbitrary hyperparameters — how do they interact with the embedding model's context window, the LLM's context window, and the expected query types?

3. Recursive text splitting splits on progressively finer separators (\n\n, \n, " ", character). Explain what problem it solves that fixed-size chunking does not, and give a concrete example of a document where fixed-size chunking would produce dramatically worse retrieval results.

4. Semantic chunking groups sentences together based on embedding similarity rather than character count. Describe the algorithm, what computational cost it adds to the indexing pipeline, and what type of documents benefit most from it.

5. Explain the difference between cosine similarity, dot product similarity, and Euclidean distance as vector similarity metrics. Under what conditions are cosine and dot product equivalent? When would you prefer each in a production retrieval system?

6. The embedding model is one of the most critical choices in a RAG system. What specific factors determine whether an embedding model is appropriate for your retrieval task, beyond benchmark scores on MTEB?

7. Top-k retrieval retrieves the k most similar chunks. What is the impact of the choice of k on: (a) answer faithfulness, (b) answer relevance, (c) LLM context window usage, and (d) latency? What strategy do you use to choose k?

8. Explain "context stuffing" — what it is, why it causes problems, and what specific failure mode the "lost in the middle" paper (Liu et al., 2023) documented. What architectural patterns can mitigate this?

9. Describe the difference between sparse retrieval (BM25/TF-IDF) and dense retrieval (embedding-based). Give a concrete example of a query where sparse retrieval outperforms dense retrieval, and one where the reverse is true.

10. What is faithfulness vs answer relevance in RAG evaluation? They can diverge — a system can be high faithfulness but low relevance, or vice versa. Give a concrete example of each failure mode.

11. Explain Recall@k as a retrieval metric. Why is it a necessary but not sufficient metric for RAG quality, and what complementary metric do you need to evaluate re-ranking quality?

12. MRR (Mean Reciprocal Rank) and NDCG (Normalized Discounted Cumulative Gain) are both ranking metrics. What is the conceptual difference, and when would you choose NDCG over MRR for evaluating a RAG retrieval system?

13. What is the "lost in the middle" problem in RAG, and what does it reveal about how LLMs process long contexts? Give a specific prompt engineering or retrieval strategy to mitigate it.

14. Markdown-aware chunking treats headers, code blocks, and list items differently from body text. For a corpus of technical documentation, explain what problems naive character-based chunking creates and how markdown-aware chunking resolves them.

15. Describe the indexing pipeline for a multi-format document corpus that includes PDFs, DOCX files, HTML pages, and Slack messages. What specific challenges does each format introduce, and what preprocessing step is most frequently the root cause of poor retrieval quality?

16. Explain the difference between bi-encoder and cross-encoder retrieval models. Why are cross-encoders almost never used as the primary retrieval model in production, and what role do they serve instead?

17. What is "asymmetric semantic search" and how does it affect embedding model choice? Give an example of a RAG use case that is asymmetric and explain what embedding strategy you would use.

18. When should you use an open-source embedding model (like BGE or E5) versus a commercial embedding API (like OpenAI's text-embedding-3)? List at least four factors that affect this decision.

19. Explain context window management in a RAG system when the retrieved chunks + query + system prompt exceed the model's context limit. Describe at least three strategies to handle this gracefully.

20. What is the "knowledge conflict" problem in RAG? This occurs when retrieved content contradicts the LLM's parametric knowledge. How should the system be designed to handle this, and how do you evaluate how well it does?
