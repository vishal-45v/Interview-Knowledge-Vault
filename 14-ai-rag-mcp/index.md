# AI / RAG / MCP Interview Knowledge Vault

Senior-level interview preparation for AI engineering roles. Covers foundations through production system design. Each chapter contains theory questions, scenario questions, tricky follow-up traps, structured answers, analogies, and ASCII diagram walkthroughs.

---

## Chapter Index

### 01 — AI Fundamentals (`01-ai-fundamentals/`)
Core machine learning and deep learning concepts every senior AI engineer must own deeply. Covers supervised/unsupervised/RL paradigms, neural network internals (weights, biases, backpropagation, gradient descent), loss functions, regularization techniques, batch normalization, modern optimizers (Adam, AdamW), the attention mechanism from first principles, transformer architecture, embeddings (word2vec through contextual), transfer learning, and evaluation metrics (F1, BLEU, ROUGE, perplexity). This chapter tests whether you understand *why* things work, not just that they do.

### 02 — LLM Architecture (`02-llm-architecture/`)
Internals of large language models from tokenization to inference optimization. Covers autoregressive vs masked LMs, GPT/BERT/T5/BART architectural differences, tokenization algorithms (BPE, WordPiece, SentencePiece, tiktoken), KV cache mechanics, sampling strategies (temperature, top-p, top-k), inference optimization (quantization INT4/INT8, GGUF format, speculative decoding, flash attention, continuous batching), VRAM estimation, RLHF with reward models and PPO/DPO, and chat template formats (ChatML, Llama3). Critical for roles involving model serving or fine-tuning.

### 03 — Prompt Engineering (`03-prompt-engineering/`)
Systematic prompt design at the senior level — not tips and tricks, but engineering discipline. Covers zero-shot vs few-shot, chain-of-thought and self-consistency, ReAct, Tree of Thought, system prompt architecture, JSON/structured output modes, prompt injection and jailbreak vectors, prompt chaining, meta-prompting, constitutional AI, instruction tuning concepts, prompt compression (LLMLingua), and tool/function-calling prompts. Includes security and reliability angles that distinguish senior candidates.

### 04 — RAG Fundamentals (`04-rag-fundamentals/`)
Retrieval-Augmented Generation from architecture through quality measurement. Covers the full indexing and query pipeline, document loaders, chunking strategies (fixed, recursive, semantic, markdown-aware) and their tradeoffs, embedding model selection (OpenAI, BGE, E5, sentence-transformers), vector similarity metrics, top-k retrieval, naive vs advanced RAG patterns, the lost-in-the-middle problem, context stuffing limits, and retrieval quality metrics (Recall@k, MRR, NDCG). The chapter that separates engineers who have run RAG in production from those who have only read blog posts.

### 05 — Vector Databases (`05-vector-databases/`)
ANN algorithms and vector database internals for production systems. Covers HNSW, IVF, and PQ algorithms with their parameter tradeoffs, FLAT exact search when to use it, and deep dives into Pinecone (namespaces, sparse-dense hybrid), Weaviate (schema, BM25+vector hybrid), Chroma (local vs server), Qdrant (payload filtering, named vectors), FAISS index types, and pgvector (ivfflat vs hnsw index choices). Includes tenant isolation patterns, index rebuild strategies, and how to choose between managed and self-hosted options.

### 06 — Advanced RAG (`06-advanced-rag/`)
Production-grade RAG techniques beyond the naive retrieve-and-generate pattern. Covers query transformation (HyDE, multi-query, step-back prompting), re-ranking (cross-encoders, Cohere Rerank, LLM-as-judge), fusion retrieval and RRF (Reciprocal Rank Fusion), parent-child chunking, RAPTOR recursive summarization, self-RAG with retrieval reflection tokens, CRAG (corrective RAG), agentic RAG patterns, knowledge graph augmentation, and streaming RAG. For candidates who need to demonstrate they can debug and improve retrieval pipelines under real constraints.

### 07 — MCP Protocol (`07-mcp-protocol/`)
Anthropic's Model Context Protocol — the emerging standard for LLM-tool integration. Covers MCP architecture (hosts, clients, servers, transports), the full protocol lifecycle (initialize, tool-list, tool-call), tool schema design, resource and prompt primitives, stdio vs SSE transports, security model and sandboxing, MCP server implementation in Python and TypeScript, comparison with OpenAI function calling and LangChain tools, multi-server orchestration, and versioning strategy. Increasingly asked at companies building agent infrastructure.

### 08 — LLM Ops (`08-llm-ops/`)
Operationalizing LLMs and RAG systems in production. Covers model serving infrastructure (vLLM, TGI, Triton), load balancing and autoscaling for inference, latency vs throughput optimization, prompt caching strategies, cost management (token budgets, model routing), observability (LangSmith, Arize, Weights & Biases), A/B testing LLM responses, fine-tuning pipelines (LoRA/QLoRA, PEFT), dataset curation and deduplication, safety and content filtering, rate limiting, and multi-region deployment. The chapter for engineers running LLMs at scale.

### 09 — Evaluation and Testing (`09-evaluation-testing/`)
Systematic evaluation of LLMs and RAG systems — the discipline that separates production engineering from demos. Covers reference-based metrics (BLEU, ROUGE, BERTScore), reference-free evaluation (G-Eval, GPT-4 as judge), RAG-specific metrics (faithfulness, answer relevance, context precision/recall via RAGAS), red-teaming and adversarial testing, regression testing pipelines, human evaluation protocols, benchmark suites (MMLU, HellaSwag, HumanEval), hallucination detection, latency and cost profiling, and continuous evaluation in CI/CD.

### 10 — System Design (`10-system-design/`)
End-to-end system design for AI and RAG applications at senior/staff level. Covers designing enterprise RAG systems from scratch, multi-tenant knowledge bases, real-time vs batch indexing pipelines, hybrid search architectures, agentic systems with tool use and memory, latency budget allocation across pipeline stages, failure modes and circuit breakers, data freshness strategies, security (PII redaction, access control in retrieval), and cost-optimized architectures. Includes full design walkthroughs for common interview scenarios: "Design a company-internal Q&A bot" and "Design a code assistant."
