# Chapter 02: LLM Architecture — Scenario Questions

Engineering decisions at production scale. Answer as a senior engineer who has operated LLMs in real systems.

---

1. Your company is deploying a 70B parameter LLM for internal use. You have a budget of 4 x A100 80GB GPUs. The model in float16 would require approximately 140GB of VRAM — more than you have. Walk through your complete strategy for fitting this model into your hardware budget while minimizing quality loss. What quantization approach would you use, what inference framework, and what monitoring would you put in place to detect quality regression?

2. You are building a real-time customer support chatbot. The LLM endpoint currently has P50 latency of 800ms and P99 of 4.2 seconds. Product requirements are P50 under 300ms and P99 under 1.5 seconds. You cannot change the model (it is contractually fixed). Describe every optimization you would explore, the likely latency improvement from each, and the priority order you would tackle them.

3. You need to serve 5 different specialized variants of a base LLM to different user segments (medical, legal, financial, general, internal). Full fine-tuned models for each would cost too much to serve simultaneously. Design the serving architecture. Consider LoRA adapters, routing logic, and how you handle requests that don't clearly fit one segment.

4. Your team is evaluating whether to fine-tune a 7B LLM on 50,000 domain-specific Q&A pairs or use RAG with the base model. The domain is pharmaceutical regulatory documentation. Make the argument for both approaches, and describe what evaluation framework you would use to decide. What production characteristics would tip the balance toward fine-tuning vs RAG?

5. A user reports that your LLM-powered coding assistant is producing different answers to the same question on different calls. Investigation reveals the temperature is set to 0 but the outputs still vary. What are all the possible causes of non-deterministic output even with temperature=0, and how do you diagnose each?

6. You are building an LLM application that processes confidential legal documents. During internal testing you discover that the model occasionally reproduces verbatim text from its training data in its outputs (memorization/regurgitation). What are the privacy and legal implications? What technical measures can you take at the model level and the application level to mitigate this?

7. Your team is deciding between using a 70B model with high accuracy or a 7B model with lower accuracy for a production use case. The 7B model is 10x cheaper to operate. Design an experiment to measure the actual quality gap on your specific task and determine whether the quality difference justifies the cost. What is your decision framework?

8. You are asked to extend a model's context window from 4K to 32K tokens without retraining from scratch. Describe the techniques available (position interpolation, YaRN, rope scaling), their tradeoffs, and the fine-tuning approach you would use to stabilize the model at the new context length.

9. Your LLM inference service is seeing GPU utilization of 30% during peak hours despite the system being at capacity (requests are queuing). Explain the apparent contradiction and what architectural changes would improve throughput without adding hardware.

10. A product manager wants to add a "creativity slider" to your LLM application that users can control. Describe the technical implementation of this feature, what sampling parameters it would modulate, the exact range you would allow, and what guardrails you would implement to prevent low-quality outputs at the high creativity end.
