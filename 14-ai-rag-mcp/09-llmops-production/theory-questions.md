# Chapter 09: LLMOps & Production — Theory Questions

## Evaluation Frameworks & Metrics

1. Compare RAGAS, TruLens, DeepEval, and Promptfoo as LLM evaluation frameworks. What are the primary use cases, deployment models (local vs. cloud), and metric coverage of each? When would you choose one over another?

2. Define hallucination rate as a metric. How is it operationally measured in a production RAG system? What is the difference between "closed-domain hallucination" (contradicting context) and "open-domain hallucination" (fabricating facts not in any context)?

3. What is the BLEU score and ROUGE score? Why are they insufficient for evaluating LLM outputs compared to LLM-as-judge approaches? What are the limitations of LLM-as-judge evaluation?

4. Describe how you would design a golden dataset for evaluating a RAG-based customer support bot. What are the dimensions of quality, how many examples do you need for statistical significance, and how do you handle the dataset's drift over time?

5. What is regression testing in the context of LLM systems? What specific challenges make LLM regression testing harder than traditional software regression testing? How do you define a regression threshold?

## Observability

6. What is a "trace" in LLM observability (LangSmith, Langfuse)? What is a "span"? How do you structure traces for a RAG pipeline vs. an agent pipeline? What metadata should you always capture?

7. Explain prompt caching in Anthropic's API and OpenAI's API. What are the conditions under which caching applies? What is the cost reduction, and what are the cache invalidation behaviors? When does caching hurt you?

8. What is token budgeting in production LLM systems? How do you calculate cost per request? Describe a model routing strategy where you route queries to smaller models when appropriate.

9. Describe the LangSmith tracing model: what information is captured at the run level, what at the span level? How do you use LangSmith feedback (thumbs up/down) to create training data or evaluation labels?

10. What is Helicone and how does it differ from LangSmith? What is its deployment model (proxy vs. SDK)? What specific observability gaps does Helicone address that LangSmith doesn't?

## Cost & Latency Optimization

11. Describe prompt caching in detail: which parts of the prompt qualify for caching, what is the minimum token threshold, what is the price reduction percentage, and how do you structure prompts to maximize cache hit rate?

12. What is response streaming and why is it important for user experience? How does SSE (Server-Sent Events) work for streaming LLM responses? What are the implementation challenges for streaming in a RAG pipeline (where you must retrieve BEFORE generating)?

13. Describe five concrete latency optimization techniques for production LLM applications. For each, quantify the expected improvement and the implementation complexity.

14. What is semantic caching for LLM responses? How does it work (embedding similarity threshold, TTL, cache key design)? What are its failure modes and when is it inappropriate to use?

## Guardrails & Safety

15. Compare Guardrails AI and NVIDIA NeMo Guardrails. What are the architectural differences? What types of validations can each perform (input validation, output validation, factual grounding)?

16. What is PII (Personally Identifiable Information) handling in LLM pipelines? Describe three implementation patterns: (a) PII detection and redaction before LLM call, (b) pseudonymization with re-identification after, (c) on-premise LLM deployment for PII data.

17. What is fine-tuning vs. LoRA vs. QLoRA? Explain the memory savings that LoRA provides and how it achieves them mathematically (low-rank decomposition). When would you use QLoRA over LoRA?

18. What is PEFT (Parameter-Efficient Fine-Tuning)? What methods does it include beyond LoRA (prefix tuning, prompt tuning, adapters)? In what scenarios does PEFT outperform full fine-tuning?

19. Describe LLM rate limit strategies: token bucket algorithms, retry with exponential backoff, request queuing, multiple API key rotation, and load balancing across model providers. What is the correct order to implement these?

20. What is a prompt registry/prompt management system? What problems does it solve that storing prompts in code doesn't? Describe the version control, A/B testing, and rollback features you would build.
