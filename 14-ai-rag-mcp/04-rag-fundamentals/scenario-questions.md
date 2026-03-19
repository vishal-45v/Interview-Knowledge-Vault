# Chapter 04: RAG Fundamentals — Scenario Questions

Production RAG engineering decisions. These questions have no single correct answer — demonstrate reasoning, tradeoffs, and production awareness.

---

1. Your company is building a RAG system over 500,000 internal engineering documents (RFCs, design docs, postmortems, Confluence pages, GitHub issues). The documents were written over 8 years and span 15 different teams with inconsistent formatting. Users report that the system retrieves documents from the wrong team or returns outdated information superseded by newer documents. Describe your complete remediation strategy, including metadata strategy, chunking approach, retrieval design, and freshness handling.

2. You are building a RAG system for a pharmaceutical company. The knowledge base contains drug interaction tables, clinical trial summaries, and regulatory filings. Users are medical professionals asking precise questions like "What is the maximum daily dose of metformin for patients with eGFR below 45?" Current retrieval quality is poor — the system often retrieves the wrong drug or wrong parameter. Walk through your full redesign of the chunking and retrieval strategy for this highly structured, precise-retrieval use case.

3. Your RAG system works well in English but your company has just expanded to 8 new countries. Documents are in French, German, Japanese, Portuguese, Arabic, and Korean. Users query in their native language. Describe your strategy for multilingual RAG, including embedding model selection, cross-lingual retrieval, and how you would evaluate quality per language.

4. A user of your customer support RAG system asks a question that requires synthesizing information from 7 different documents (e.g., "How do I migrate from v1 to v3 of the API, skipping v2?"). Current top-k=5 retrieval misses critical steps. You cannot increase k beyond 8 due to context window constraints. Describe your strategy for handling multi-hop questions that require synthesizing many sources.

5. Your RAG pipeline is running correctly but users are getting wrong answers 23% of the time. You instrument the pipeline and find: retrieval recall@5=0.89 (good), but faithfulness=0.71 (model is not staying grounded in retrieved content) and answer relevance=0.64 (answers don't fully address the question). These metrics suggest the problem is in the generation stage, not retrieval. Design a focused intervention plan targeting faithfulness and answer relevance specifically.

6. You are building a legal document Q&A system. The knowledge base contains contracts (average 40 pages each), court filings, and legal memos. Context window is 8K tokens. A lawyer asks: "What are all the termination conditions in the master services agreement with Acme Corp?" — a question requiring full-document comprehension. Current chunk-based retrieval misses termination conditions scattered across different sections. How do you approach full-document questions in a RAG system constrained by context length?

7. Your company uses a RAG system for internal HR policy questions. An employee asks about their "parental leave policy." The system retrieves 5 chunks, 3 of which are from the old policy (2021, now superseded), 2 from the current policy (2024). The LLM synthesizes both and gives an incorrect blended answer. Design a solution that handles document version conflicts, including both the data layer and the prompt layer.

8. You are the first AI engineer at a startup. You need to build a RAG system in 2 weeks with minimal infrastructure. The corpus is 5,000 PDF product manuals. You have budget for API calls but not for managed vector databases. Walk through your technology stack choices, chunking strategy, and the minimum viable evaluation framework you would implement.

9. Your RAG system latency is: document loading=50ms, embedding query=80ms, vector search=120ms, LLM generation=800ms, total=1050ms. The product requirement is P99 under 500ms. You cannot change the LLM. Walk through your complete latency optimization strategy, what you expect from each optimization, and which optimizations involve quality tradeoffs.

10. A customer reports that your RAG system "made up" a citation — it stated that a fact came from "Document X, Page 3" but when they looked it up, the fact wasn't there. This is a faithfulness failure but with a specific "hallucinated citation" flavor. Design both a detection mechanism and a prevention mechanism for citation hallucination in RAG systems.
