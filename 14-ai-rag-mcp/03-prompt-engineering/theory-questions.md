# Chapter 03: Prompt Engineering — Theory Questions

Senior-level questions on prompt design, reliability engineering, security, and production prompt systems. Goes beyond basic prompting tips.

---

1. Explain the mechanistic difference between zero-shot prompting and few-shot prompting from the perspective of in-context learning. What is the current theoretical understanding of WHY few-shot examples help — are the model's weights being updated?

2. Chain-of-thought (CoT) prompting consistently improves performance on multi-step reasoning tasks. Explain the hypothesized mechanism by which "let's think step by step" elicits better reasoning. Does CoT actually improve reasoning, or does it improve the model's ability to OUTPUT correct answers by providing a scaffold for computation?

3. Self-consistency is an extension of chain-of-thought. Describe the technique precisely, what it costs computationally, and under what conditions it provides the largest quality improvement over single-pass CoT.

4. ReAct (Reason + Act) combines reasoning traces with tool calls. Explain the specific interleaving pattern of thought, action, and observation in a ReAct trace. What failure mode does the "reasoning" step prevent that simple tool-calling-without-thought would encounter?

5. Prompt injection attacks differ from traditional SQL injection. Explain the mechanism of a direct prompt injection and an indirect prompt injection (via retrieved content), and why the latter is particularly dangerous in RAG systems.

6. Structured output via JSON mode and via constrained decoding (grammar-guided sampling) are two different technical approaches to getting models to output valid JSON. Explain the difference and why constrained decoding guarantees valid JSON while JSON mode does not.

7. What is the difference between instruction tuning and prompt engineering? Can a well-engineered prompt on a base model match the performance of instruction tuning on the same model? Under what conditions and for what tasks?

8. Explain "lost in the middle" as it applies to few-shot examples in a prompt. Given a long prompt with 20 few-shot examples, which examples have the most influence on the model's output and why?

9. Constitutional AI (CAI) is Anthropic's technique for aligning model behavior via principles. Explain the two-phase CAI process (critique and revision) and how it differs from RLHF in its data generation process.

10. What is meta-prompting, and how does it differ from standard prompt engineering? Describe a concrete use case where meta-prompting provides a significant advantage over manually crafting prompts.

11. LLMLingua is a prompt compression technique. Explain how it works, what information it preserves and what it discards, and at what token budget reduction ratios does quality typically degrade.

12. Function calling (tool use) in OpenAI's API and Anthropic's API requires the model to output a structured invocation. Explain how function schemas are represented in the prompt and how the model "knows" to produce a valid function call versus a text response.

13. System prompt design for production applications involves more than writing instructions. Describe five structural principles that distinguish a production-grade system prompt from a naive one, focusing on edge cases, ambiguity handling, and failure mode prevention.

14. Explain the "primacy and recency bias" in long prompts. How do these biases interact with few-shot examples, and what prompt construction strategies can you use to mitigate their effect?

15. Describe the prompt structure used by Tree of Thought (ToT). How does it differ from self-consistency, and why does ToT require multiple model calls that must be orchestrated externally rather than being a prompting technique the model executes internally?

16. What is role prompting, and what is the empirical evidence for its effectiveness? Is "You are an expert senior software engineer" actually a meaningful instruction that changes model behavior, or is it placebo?

17. Explain the difference between "output format constraints" enforced through prompting versus those enforced through API features (JSON mode, structured outputs). Where can each approach fail?

18. What is "prompt chaining" and how does it differ from single-turn prompting? Design a prompt chain for a 3-step task: extract entities, assess sentiment per entity, summarize findings in a specific format.

19. Temperature for determinism: if you need a production system to produce consistent formatting in its outputs (e.g., always outputting valid JSON), is temperature=0 sufficient? What else must be controlled?

20. Describe the "verbosity" failure mode in prompted LLMs. How does explicit length control in a prompt (e.g., "respond in 3 sentences") compare in effectiveness to post-processing the output to enforce length constraints?
