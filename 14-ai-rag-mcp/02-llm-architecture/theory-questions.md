# Chapter 02: LLM Architecture — Theory Questions

Senior-level questions on LLM internals, from tokenization through inference optimization. Expect follow-ups that probe whether you've actually worked with these systems.

---

1. Explain the fundamental difference between autoregressive (causal) language models and masked language models. Why is a GPT-style model preferred for generation tasks while BERT-style models were historically preferred for classification and embedding tasks?

2. Walk through the Byte-Pair Encoding (BPE) tokenization algorithm step by step. What are its failure modes for domain-specific text (e.g., code identifiers, chemical formulas), and how do modern tokenizers like tiktoken address them?

3. The KV cache is one of the most important optimizations in LLM inference. Explain exactly what is cached, at which point in the computation it is populated, and what the memory cost formula is as a function of sequence length, layer count, and model dimension.

4. Explain the difference between temperature, top-k, and top-p (nucleus) sampling. Mathematically, how does temperature transform the logit distribution before sampling? What happens as temperature approaches zero?

5. Speculative decoding uses a draft model to propose tokens that are then verified by the target model. Explain the acceptance/rejection mechanism. Under what conditions does speculative decoding provide no speedup?

6. INT8 quantization of a transformer model reduces memory by ~2x over float16. Walk through exactly what information is lost or approximated in the quantization process. What is "outlier-aware" quantization (as in LLM.int8()) and why is it necessary for LLMs specifically?

7. Explain flash attention. What problem does it solve that standard attention does not, and critically, does it change the mathematical output of attention? What specific hardware operation does it optimize?

8. RLHF (Reinforcement Learning from Human Feedback) involves three stages. Describe each stage, what is learned in each, and explain the PPO algorithm's role. Why is PPO chosen over simpler policy gradient methods?

9. DPO (Direct Preference Optimization) is described as an alternative to RLHF without a reward model. What is DPO actually optimizing, and what is the mathematical relationship it establishes between the policy and the optimal reward function?

10. Describe continuous batching in LLM inference. How does it differ from static batching, and what throughput improvement does it provide for a workload with highly variable output lengths?

11. What is the role of the system prompt in chat models, and how is it implemented at the model level? Can a sufficiently capable model truly distinguish between "system" and "user" instructions, or is this a convention enforced by training?

12. Explain the ChatML chat template format. Why do modern models need a structured chat template, and what failure mode occurs when a model fine-tuned with one template format is served with a different template format?

13. How does GPTQ differ from GGUF as quantization formats? What hardware targets does each optimize for, and what is the practical tradeoff in quality vs compression ratio between INT4 GPTQ and Q4_K_M GGUF?

14. The "lost in the middle" problem affects long-context LLMs. Describe the empirical finding and explain the architectural/attention reason why transformers exhibit this behavior. What mitigation techniques exist?

15. GPT-2 uses absolute positional embeddings. GPT-NeoX and LLaMA use RoPE (Rotary Position Embedding). Explain what RoPE is doing mathematically and why it enables better extrapolation to longer sequences than models saw during training.

16. Explain perplexity as it applies to LLMs. If a model has perplexity 15 on Wikipedia and perplexity 45 on arXiv papers, what does that tell you about the model's training data composition, and what would you expect to happen to both numbers if you continued training on arXiv papers?

17. What is the purpose of repetition penalty in language model sampling? Where exactly is it applied in the generation process (before or after softmax, on logits or probabilities)? What failure mode does an overly aggressive repetition penalty introduce?

18. Describe the training objective for a decoder-only language model like GPT. Why is this called "next token prediction" rather than "next word prediction"? How does this single objective give rise to capabilities like question answering, summarization, and translation without task-specific training?

19. What is the difference between model parallelism and tensor parallelism in the context of multi-GPU LLM inference? Which parallelism strategy is typically used for inference on a single multi-GPU node and why?

20. Explain the tokenization algorithm used by SentencePiece, and why it treats the tokenization problem as a language model optimization problem (unigram language model). How does this differ from BPE's bottom-up merge strategy?
