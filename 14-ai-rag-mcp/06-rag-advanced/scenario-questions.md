# Chapter 06: Advanced RAG — Scenario Questions

1. Your team is building a RAG system for a legal research platform with 10 million documents. Users frequently query using case names and citation numbers (e.g., "Roe v. Wade, 410 U.S. 113") but also ask abstract questions like "what precedents govern privacy rights in medical decisions?" How would you design the retrieval pipeline to handle both query types? Specify the exact components, their order, and the latency budget for each stage.

2. You're running a RAG system in production and RAGAS evaluation shows: faithfulness = 0.91, answer relevancy = 0.88, context precision = 0.62, context recall = 0.79. The product team says answers feel accurate but miss important nuances. Which metric is your smoking gun, what does it tell you about where the failure is, and what specific changes would you make to improve it?

3. Your company has a 500-page technical manual that's updated quarterly. Users ask highly specific questions like "what is the torque specification for the M8 bolt on the secondary heat exchanger?" Standard dense retrieval frequently returns the wrong section. Walk through how you would redesign the chunking, indexing, and retrieval strategy to achieve near-perfect precision on this type of question.

4. A financial services client needs a RAG system that must never hallucinate figures, and all answers must be traceable to specific document passages. The existing pipeline has faithfulness of 0.84. What architectural changes (retrieval, generation, post-processing) would you make to push faithfulness above 0.97? What are the latency tradeoffs?

5. Your team deployed a RAG system for a customer support bot. Six months later, the underlying product changed significantly. Without reindexing everything, how would you handle the stale knowledge problem? Describe a tiered indexing strategy, metadata versioning approach, and a fallback mechanism when retrieved context is outdated.

6. You need to build a system that answers multi-hop questions over a knowledge base of interconnected research papers. For example: "Which papers influenced the attention mechanism used in GPT-4?" Standard RAG fails on this. Walk through two different architectural approaches, their failure modes, and how you would evaluate which performs better.

7. Your RAG system works well for English but needs to support queries in 12 languages with documents in a mix of English and the user's language. Describe your multilingual embedding strategy, how you handle cross-lingual retrieval, and what evaluation challenges this introduces.

8. You're building an agentic RAG system for a data analyst who needs to query both structured (SQL databases) and unstructured (PDF reports) sources. The agent must decide which source to query and when to combine both. Design the tool set, the agent decision logic, and the failure recovery strategy.

9. A startup is deciding between fine-tuning a smaller model (7B parameters) on domain data vs. building a RAG system over a large foundation model (GPT-4o). The domain is cybersecurity threat intelligence with ~50,000 documents updated daily. Build the full technical and business case for which approach you'd recommend.

10. You are conducting a RAGAS evaluation and discover that context recall is 0.95 (excellent) but faithfulness is 0.71 (poor). This means the retriever is finding the right documents but the LLM is hallucinating anyway. What are the five most likely causes of this pattern, and for each, what is the specific fix?
