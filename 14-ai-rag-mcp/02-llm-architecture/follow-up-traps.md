# Chapter 02: LLM Architecture — Follow-Up Traps

Common follow-up traps in LLM architecture interviews. The "correct answer" goes beyond the textbook explanation.

---

## Trap 1: "Temperature=0 gives deterministic outputs"

**What most people say:** Setting temperature to 0 makes the model always pick the highest probability token, producing deterministic outputs.

**Correct answer:** Temperature=0 means greedy decoding — always pick argmax over logits. This is deterministic in theory, but in practice several factors break determinism:

1. **Floating point non-determinism:** GPU operations like matrix multiplications with float16 are not associative across different thread orderings. Reordering operations between runs can produce different logit values at the last decimal place, which changes the argmax for tokens with very similar probabilities.

2. **Batch size effects:** If your request is batched with other requests, the padding and attention mask patterns affect activations through tensor operations, producing different results than a single-item batch.

3. **Sampling library bugs:** Some inference frameworks apply top-p or top-k filtering BEFORE the temperature scaling. If top-p=0.95 and temperature=0, the top-p filter runs first and can change which tokens are available to argmax.

4. **Multi-GPU tensor parallelism:** Reduction operations across GPUs have ordering non-determinism.

```python
# To get truly reproducible outputs:
import torch
torch.manual_seed(42)
torch.cuda.manual_seed_all(42)
# Use deterministic algorithms (slower)
torch.backends.cudnn.deterministic = True
torch.backends.cudnn.benchmark = False
# Set do_sample=False AND temperature=1 (or omit temperature)
outputs = model.generate(
    input_ids,
    do_sample=False,       # Greedy, not sampling
    temperature=1.0,       # Ignored when do_sample=False, but set explicitly
    seed=42                # Some frameworks support this
)
```

---

## Trap 2: "KV cache makes inference O(1) per token"

**What most people say:** The KV cache stores previously computed key/value pairs, so each new token generation is constant time.

**Correct answer:** KV cache makes the attention computation O(1) in FLOPs per new token (you only compute Q, K, V for the new token and then look up cached K, V for past tokens). But the attention operation itself is still O(n) in memory bandwidth per new token — you must read all n cached K, V vectors from GPU memory to compute attention weights. This means:

- **Time**: Grows linearly with sequence length even with KV cache (because of memory read bandwidth)
- **Memory**: KV cache grows linearly: `2 * num_layers * num_heads * head_dim * seq_len * bytes_per_param`

For LLaMA-2 70B (80 layers, 8 GQA groups, 128 head dim, float16):
```
KV cache per token = 2 * 80 * 8 * 128 * 2 bytes = 327,680 bytes ≈ 320 KB/token
At 4096 tokens: 320KB * 4096 = 1.3 GB just for KV cache
At 32K tokens:  320KB * 32768 = 10.5 GB
```

This is why long context inference is memory-bound and why GQA (Grouped Query Attention) and MQA (Multi-Query Attention) exist — they reduce KV cache size by sharing K,V heads across Q heads.

---

## Trap 3: "RLHF is better than DPO because it has an explicit reward model"

**What most people say:** RLHF trains a reward model that can be inspected and improved separately, giving more control than DPO's implicit optimization.

**Correct answer:** This tradeoff is more nuanced. RLHF's reward model is a failure point: it can be "reward hacked" — the policy learns to score high on the reward model without actually being better (the reward model is an imperfect proxy for human preference). PPO then amplifies this by optimizing the policy against this flawed proxy, potentially producing outputs that humans rate worse.

DPO doesn't have a separate reward model but its optimization is equivalent to implicit reward modeling. The mathematical equivalence shown in the DPO paper is:
```
r*(x, y) = β * log[π_θ(y|x) / π_ref(y|x)] + β * log Z(x)
```

The advantage of DPO: simpler training pipeline, no RL stability issues (PPO is notoriously hard to tune), less compute. The disadvantage: less flexible (offline) and harder to iteratively improve by collecting more human feedback on model outputs (online RLHF can do this by sampling from the current policy).

In 2024 practice: most teams use DPO or its variants (IPO, SLiC, KTO) because PPO's instability makes it harder to scale. But RLHF with PPO is still state-of-the-art for very large models where the reward model generalization matters more.

---

## Trap 4: "Flash Attention makes attention faster by reducing computation"

**What most people say:** Flash Attention is a more efficient attention algorithm that reduces the number of operations.

**Correct answer:** Flash Attention does NOT reduce the number of FLOPs — it performs the exact same mathematical computation as standard attention. What it optimizes is **memory access patterns**. Standard attention materializes the full n×n attention matrix in GPU HBM (high-bandwidth memory), which requires O(n²) reads/writes to slow memory. Flash Attention uses **tiling** to keep computations in the much faster SRAM (on-chip cache), fusing the softmax computation with the attention output into a single kernel pass.

The speedup comes from:
- **Reduced HBM I/O**: O(n) instead of O(n²) HBM reads/writes
- **Kernel fusion**: One GPU kernel instead of separate softmax + matmul kernels
- No change to the mathematical output (it's numerically identical)

Flash Attention 2 adds further optimizations: better parallelism across sequence length and reduced non-matrix-multiply FLOPs. Flash Attention 3 specifically targets Hopper architecture (H100) with asynchronous operations.

```python
# Standard attention: O(n^2) memory
# Materializes (seq_len x seq_len) attention matrix in HBM
attn_weights = torch.softmax(q @ k.T / scale, dim=-1)  # n^2 tensor
output = attn_weights @ v

# Flash Attention: Same output, O(n) memory
# PyTorch 2.0+ includes this as scaled_dot_product_attention:
from torch.nn.functional import scaled_dot_product_attention
output = scaled_dot_product_attention(q, k, v)  # Uses Flash Attention internally
```

---

## Trap 5: "Quantizing to INT4 gives a 4x speedup"

**What most people say:** INT4 uses 4 bits vs 16-bit float, so it's 4x smaller and faster.

**Correct answer:** INT4 quantization gives ~4x memory reduction, but the compute speedup depends on hardware support. INT4 matrix multiply is only natively fast on specific hardware (e.g., NVIDIA A100/H100 with INT4 tensor cores). On older hardware, INT4 must be dequantized to INT8 or float16 before computation, negating the compute speedup while keeping the memory savings.

More precisely:
- **Memory bandwidth benefit**: 4x reduction means weights load faster from HBM → real latency improvement (inference is often memory-bandwidth bound)
- **Compute benefit**: Architecture-dependent. May be 1x if no INT4 matmul support
- **Quality cost**: INT4 quantization typically causes 0.5-3% downstream quality degradation. LLMs are more sensitive to quantization in early layers and attention projection layers.

GGUF format (used by llama.cpp) uses "K-quants" like Q4_K_M which uses INT4 for most weights but INT6 for attention matrices, preserving quality at a small memory cost. This is a smarter allocation than uniform INT4.

---

## Trap 6: "The context window limit is just a hardware limitation"

**What most people say:** Context limits exist because models can't fit longer sequences in GPU memory.

**Correct answer:** Context limits have multiple causes, and "just add more GPU memory" doesn't solve all of them:

1. **Positional encoding range**: Models trained with positional encodings only up to position N genuinely perform poorly beyond N — the attention mechanism hasn't learned to interpret those positions. Memory is not the limiting factor here.

2. **Attention is O(n²) in memory**: Standard attention requires O(n²) memory, so 128K context requires 16x more memory than 32K. Flash Attention reduces this to O(n) but the KV cache is still O(n).

3. **Training data distribution**: Even if you extend the position encodings, if the model was only trained on documents with max length 2K, it has never learned to use information from position 50K. Fine-tuning is required.

4. **Long-range retrieval capability**: Even with architectural support, transformers empirically struggle to recall information from the very beginning of a long context ("lost in the middle" problem). This is a learning failure, not a hardware limitation.

---

## Trap 7: "BERT is better than GPT for embeddings because it's bidirectional"

**What most people say:** BERT sees the full context on both sides, so its embeddings encode more information than GPT's unidirectional embeddings.

**Correct answer:** This was true before 2022 but is no longer a reliable heuristic. Several factors complicate it:

1. Modern decoder-only models fine-tuned for embeddings (like E5-mistral-7b, GTE-Qwen2) outperform BERT-family models on MTEB benchmarks — despite being autoregressive, their scale compensates.

2. BERT's [CLS] token embedding is not inherently better for semantic search — it was fine-tuned specifically for sentence-level tasks in Sentence-BERT (SBERT) via contrastive learning. The raw BERT embedding without SBERT fine-tuning is often worse than averaged token embeddings.

3. For asymmetric tasks (short query vs long document), encoder models can actually underperform because they are trained with symmetric masking objectives.

4. The bidirectionality advantage is real for token-level tasks (NER, span extraction) where you need each token to see full context. For sentence-level embeddings, the scale of the model and the quality of contrastive fine-tuning dominate.

---

## Trap 8: "System prompts are secure — users can't override them"

**What most people say:** The system prompt defines the model's behavior and users cannot change it.

**Correct answer:** System prompts are enforced by training (RLHF/RLAIF) creating a tendency to follow them, but they are not cryptographically enforced or architecturally protected. They are text — the model processes the system prompt tokens the same way it processes user tokens. Several attack vectors exist:

1. **Prompt injection**: User input contains instructions that override or expand the system prompt ("Ignore all previous instructions...")
2. **Jailbreaking**: Adversarial prompts that bypass safety training (role-playing scenarios, hypothetical framing, base64 encoding of instructions)
3. **Indirect prompt injection**: Via retrieved documents in RAG — malicious documents tell the model to ignore the system prompt
4. **Context stuffing**: Filling the context with many examples of the undesired behavior before the instruction, using in-context learning to override fine-tuned behavior

Production defense requires: input sanitization, output scanning, canary tokens in system prompts to detect exfiltration, monitoring for policy violations, and not putting truly sensitive information (API keys, database schemas) in system prompts.

---

## Trap 9: "Bigger models always have lower perplexity"

**What most people say:** More parameters means better modeling of language, so bigger = lower perplexity.

**Correct answer:** Bigger models generally have lower perplexity on the same distribution, but several cases break this:

1. **Different training data**: A smaller model trained on carefully curated, domain-matched data can outperform a larger model on domain-specific perplexity.

2. **Different tokenizers**: Perplexity is measured per-token. A model with a larger vocabulary will naturally have lower token-level perplexity because tokens are longer/richer — this is not a fair comparison.

3. **Overfitting**: A model that has memorized training data has very low training perplexity but high test perplexity. This is measurable for very large models on very small fine-tuning sets.

4. **Quantization**: A quantized 70B model can have higher perplexity than a full-precision 7B model on out-of-distribution text.

The right statement: "For models with the same architecture, tokenizer, and training data, larger → lower perplexity. Across different settings, all bets are off."
