# Chapter 09: LLMOps & Production — Scenario Questions

1. Your RAG-based customer support application is serving 50,000 queries per day. The monthly LLM API bill is $28,000, and the CTO wants it under $8,000 without significant quality degradation. Walk through a systematic cost optimization plan: which techniques you would apply in what order, how you would measure quality impact at each stage, and your expected cost reduction from each technique.

2. Your team deployed a new version of the system prompt last Tuesday. On Wednesday, customer satisfaction scores dropped 8%. You need to determine if it was the prompt change that caused the drop. Describe your investigation process: what observability data you would pull, how you would compare pre/post metrics, and what a proper A/B test design for prompt changes looks like.

3. You're building a medical information chatbot where hallucinations about drug dosages could cause patient harm. Design a comprehensive safety stack: what guardrails you would implement at the input stage, the retrieval stage, the generation stage, and the output stage. How would you measure and monitor the safety of the system post-deployment?

4. Your production LLM application is hitting OpenAI rate limits during peak hours (8-10 AM). You're seeing errors that manifest as 5-second delays and 429 responses. Describe your complete rate limit management strategy: request queuing, exponential backoff implementation, provider load balancing, and any architectural changes to prevent this from recurring.

5. A new version of GPT-4o was released with significant capability improvements. Your team wants to migrate from GPT-4-turbo to GPT-4o for your production RAG system. Design a safe migration plan: how you would evaluate the new model against your golden dataset, what rollout strategy (canary, shadow, full migration) you would use, and what rollback criteria would trigger a reversion.

6. Your team wants to implement a prompt registry to manage 40 different prompts across 6 production services. Currently prompts are hardcoded in application code. Design the prompt registry system: data model, versioning scheme, deployment workflow (how prompts go from draft to production), A/B testing mechanism, and how it integrates with your observability stack.

7. You are building an LLM application that processes HR documents (performance reviews, compensation data) — PII and highly sensitive data. The legal team says this data cannot leave the corporate network, but your engineering team wants to use GPT-4o via the OpenAI API. Walk through the complete architecture options, the tradeoffs of each, and your recommendation.

8. Your company is fine-tuning Llama 3 (8B parameters) on 50,000 examples of domain-specific Q&A pairs for a legal document analysis task. The base model performs at 63% accuracy on your benchmark; you need 85%. Design the fine-tuning pipeline: data preparation steps, LoRA hyperparameter choices, training monitoring, evaluation methodology, and how you would compare the fine-tuned model against a RAG-based approach using GPT-4o.

9. You're tasked with building an LLM evaluation pipeline that runs nightly regression tests. The pipeline must: (a) run 500 test cases against your production endpoint, (b) compare results to a golden dataset, (c) detect regressions in faithfulness, relevancy, and tone, and (d) block deployment if any metric degrades by more than 5%. Design the complete pipeline architecture and the specific metrics/thresholds you would set.

10. A senior engineer proposes using semantic caching to reduce LLM API costs by 40% — similar queries get cached responses. You're concerned about staleness and incorrect cache hits. Design a semantic caching system that addresses these concerns: how you define "similar enough" (threshold, embedding model), TTL design, cache invalidation triggers, what query types you exclude from caching, and how you monitor cache quality in production.
