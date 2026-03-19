# Chapter 02: LLM Architecture — Structured Answers

Complete answers for the highest-priority LLM architecture interview questions.

---

## Q1: Explain GPT's decoder-only architecture and why it's used for generation

**Answer:**

GPT-style models use a decoder-only transformer stack with **causal (left-to-right) self-attention** — each token can only attend to itself and previous tokens, never future tokens. This is enforced via a causal attention mask that sets future positions to -infinity before softmax.

**Architecture components:**
```
Input tokens
    │
    ▼
Token Embedding + Positional Encoding
    │
    ▼
┌─────────────────────────────────────┐
│  Transformer Decoder Block × N      │
│                                     │
│  ┌──────────────────────────────┐   │
│  │  Causal Self-Attention       │   │
│  │  (can see: past and current) │   │
│  │  (cannot see: future)        │   │
│  └──────────────────────────────┘   │
│  LayerNorm + Residual                │
│  ┌──────────────────────────────┐   │
│  │  Feed-Forward Network        │   │
│  │  Linear(d) → GELU → Linear   │   │
│  └──────────────────────────────┘   │
│  LayerNorm + Residual                │
└─────────────────────────────────────┘
    │
    ▼
LM Head: Linear(d_model → vocab_size)
    │
    ▼
Logits → Softmax → Next token probabilities
```

**Training objective (next-token prediction / causal LM):**
```python
from transformers import GPT2LMHeadModel, GPT2Tokenizer
import torch

model = GPT2LMHeadModel.from_pretrained("gpt2")
tokenizer = GPT2Tokenizer.from_pretrained("gpt2")

# Input and labels are the SAME sequence, shifted by 1
text = "The transformer architecture is"
inputs = tokenizer(text, return_tensors="pt")
input_ids = inputs["input_ids"]

# During training: predict token[i+1] from tokens[0:i+1]
# Loss = cross-entropy averaged over all positions
outputs = model(input_ids=input_ids, labels=input_ids)
print(f"Loss: {outputs.loss.item():.4f}")  # NLL per token
print(f"Perplexity: {torch.exp(outputs.loss).item():.2f}")

# Generation: autoregressively sample one token at a time
generated = model.generate(
    input_ids,
    max_new_tokens=50,
    do_sample=True,
    temperature=0.8,
    top_p=0.95,
)
print(tokenizer.decode(generated[0]))
```

**Why decoder-only for generation:** The causal masking is exactly what generation requires — produce token t without knowing tokens t+1, t+2, ... Encoder models (BERT) cannot do this because they require the full input before producing representations. Encoder-decoder models (T5) can generate but require a separate encoder pass for input processing, adding latency and architectural complexity.

---

## Q2: Tokenization — BPE, WordPiece, and tiktoken

**Answer:**

**BPE (Byte-Pair Encoding):**

BPE starts with individual bytes/characters and iteratively merges the most frequent pair of adjacent tokens until vocabulary size is reached.

```python
def train_bpe(corpus, vocab_size):
    """Simplified BPE training."""
    # Start: split each word into characters
    vocab = {}
    for word, freq in corpus.items():
        chars = list(word) + ['</w>']
        vocab[tuple(chars)] = freq

    while len(get_all_tokens(vocab)) < vocab_size:
        # Count all adjacent pairs
        pairs = {}
        for word, freq in vocab.items():
            for i in range(len(word) - 1):
                pair = (word[i], word[i+1])
                pairs[pair] = pairs.get(pair, 0) + freq

        # Find most frequent pair
        best_pair = max(pairs, key=pairs.get)

        # Merge all occurrences of best_pair
        new_token = best_pair[0] + best_pair[1]
        vocab = merge_vocab(vocab, best_pair, new_token)

    return vocab

# GPT-2 uses BPE on bytes (not characters), making it byte-level BPE
# This means it can represent ANY UTF-8 text without unknown tokens
```

**BPE failure modes:**

1. **Rare domain terms split oddly**: "CRISPR" might tokenize as ["CR", "ISPR"] or ["C", "RISPR"] depending on training corpus
2. **Programming identifiers**: `snake_case_variable` might tokenize as ["snake", "_", "case", "_", "variable"] — 5 tokens instead of 1
3. **Numbers tokenize byte-by-byte**: "12345" → ["1", "2", "3", "4", "5"] in GPT-2, using 5 tokens for what should be a numeric concept

**tiktoken (OpenAI's modern BPE):**
```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4")

# Compare tokenization
text_normal = "The transformer architecture"
text_code = "snake_case_variable_name"
text_number = "12345678"

for text in [text_normal, text_code, text_number]:
    tokens = enc.encode(text)
    print(f"'{text}' → {len(tokens)} tokens: {[enc.decode([t]) for t in tokens]}")

# 'The transformer architecture' → 4 tokens
# 'snake_case_variable_name' → varies but typically 6-8 tokens
# '12345678' → 3-4 tokens (tiktoken handles numbers better than GPT-2)
```

---

## Q3: KV Cache — mechanics, memory cost, and GQA

**Answer:**

**What is cached and when:**

During autoregressive generation, attention requires Q, K, V matrices for ALL past tokens. Without caching, each new token requires recomputing K and V for the entire sequence. The KV cache stores K and V tensors for each layer so they only need to be computed once.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model = AutoModelForCausalLM.from_pretrained("gpt2")
tokenizer = AutoTokenizer.from_pretrained("gpt2")

prompt = "The quick brown fox"
inputs = tokenizer(prompt, return_tensors="pt")

# WITH KV cache (default behavior):
# First forward pass: compute K, V for all prompt tokens, cache them
# Subsequent passes: only compute Q, K, V for the new token; look up cached K, V
with torch.no_grad():
    outputs = model.generate(
        inputs["input_ids"],
        max_new_tokens=20,
        use_cache=True,  # DEFAULT - enables KV cache
        return_dict_in_generate=True,
    )
```

**Memory formula:**
```
KV_cache_bytes = 2 × num_layers × num_kv_heads × head_dim × seq_len × bytes_per_element

# For LLaMA-2-7B (float16):
# - 32 layers, 32 KV heads (MHA), head_dim=128, float16=2 bytes
KV_7B_per_token = 2 * 32 * 32 * 128 * 2 = 524,288 bytes = 512 KB/token

# For LLaMA-2-70B with GQA (float16):
# - 80 layers, 8 KV heads (GQA!), head_dim=128, float16=2 bytes
KV_70B_per_token = 2 * 80 * 8 * 128 * 2 = 327,680 bytes = 320 KB/token
# (actually cheaper per token than 7B despite larger model, because of GQA)
```

**GQA (Grouped Query Attention):**
```
MHA: Q heads = K heads = V heads = h    (most expensive KV cache)
GQA: Q heads = h, K heads = V heads = g where g < h (e.g., g=8, h=32)
MQA: Q heads = h, K heads = V heads = 1             (smallest KV cache)

# GQA example: 32 Q heads share 8 K/V heads (4 Q heads per K/V head)
# KV cache reduction: 32/8 = 4x smaller than MHA
# Quality: ~MHA quality at much lower memory cost (LLaMA-2-70B uses g=8)
```

---

## Q4: Sampling Strategies — Temperature, Top-k, Top-p

**Answer:**

**Temperature modifies the logit distribution before sampling:**
```python
import torch
import torch.nn.functional as F

logits = torch.tensor([2.0, 1.0, 0.5, 0.1, -0.5])  # Raw model output

def sample_with_temperature(logits, temperature=1.0):
    """Temperature scales logits BEFORE softmax."""
    # temperature < 1: sharpens the distribution (more confident, less creative)
    # temperature = 1: unchanged distribution
    # temperature > 1: flattens the distribution (more random/creative)
    # temperature → 0: approaches argmax (greedy/deterministic)
    scaled_logits = logits / temperature
    probs = F.softmax(scaled_logits, dim=-1)
    return torch.multinomial(probs, num_samples=1)

# Temperature effects:
for temp in [0.1, 0.5, 1.0, 1.5, 2.0]:
    probs = F.softmax(logits / temp, dim=-1)
    print(f"T={temp}: {probs.tolist()}")
# T=0.1: [0.99, 0.01, 0.00, 0.00, 0.00]  <- near-greedy
# T=1.0: [0.62, 0.23, 0.14, 0.09, 0.04]  <- original
# T=2.0: [0.38, 0.28, 0.21, 0.17, 0.12]  <- near-uniform

def nucleus_sampling(logits, top_p=0.9, temperature=1.0):
    """Top-p (nucleus) sampling: sample from smallest set covering p probability mass."""
    probs = F.softmax(logits / temperature, dim=-1)
    sorted_probs, sorted_indices = torch.sort(probs, descending=True)
    cumulative_probs = torch.cumsum(sorted_probs, dim=-1)

    # Remove tokens that would exceed the nucleus
    # Shift cumsum to include the token that crosses the threshold
    sorted_indices_to_remove = cumulative_probs - sorted_probs > top_p
    sorted_probs[sorted_indices_to_remove] = 0
    sorted_probs = sorted_probs / sorted_probs.sum()  # Renormalize

    sampled_index = torch.multinomial(sorted_probs, num_samples=1)
    return sorted_indices[sampled_index]

def top_k_sampling(logits, k=50, temperature=1.0):
    """Top-k: only sample from the k highest probability tokens."""
    probs = F.softmax(logits / temperature, dim=-1)
    top_k_probs, top_k_indices = torch.topk(probs, k=k)
    top_k_probs = top_k_probs / top_k_probs.sum()  # Renormalize
    sampled_index = torch.multinomial(top_k_probs, num_samples=1)
    return top_k_indices[sampled_index]
```

**When to use each strategy:**

| Strategy | Best for | Risk |
|----------|----------|------|
| Temperature low (0.2-0.5) | Factual Q&A, code generation | Repetitive, safe outputs |
| Temperature high (1.0-1.5) | Creative writing, brainstorming | Incoherence, hallucination |
| Top-p=0.95, temp=0.8 | General purpose (most common) | Balanced tradeoff |
| Top-k=50 | When distribution is peaky | Fixed k can be too many/few |
| Greedy (temp→0) | When determinism needed | Repetition loops common |

---

## Q5: RLHF and DPO — from preference data to aligned model

**Answer:**

**RLHF Pipeline:**

```
Stage 1: Supervised Fine-tuning (SFT)
─────────────────────────────────────
Base LLM → fine-tune on high-quality demonstrations → SFT model
(teaches the model what good outputs look like)

Stage 2: Reward Model Training
──────────────────────────────
Human annotators rank outputs: given prompt + [output_A, output_B], pick better
Train a reward model: RM(prompt, output) → scalar score
Architecture: same as SFT but final layer outputs scalar instead of logit distribution

Stage 3: PPO (Proximal Policy Optimization)
────────────────────────────────────────────
policy = SFT model (initialize from stage 1)
reference = frozen copy of SFT model (prevents policy from diverging too far)

PPO loss = E[RM(prompt, output)] - β * KL(policy || reference)
         ↑ maximize reward           ↑ don't stray too far from SFT model
```

```python
# DPO objective (Direct Preference Optimization):
# Given: (prompt x, chosen output y_w, rejected output y_l)
# Minimize:
# L_DPO = -E[ log σ( β * log[π_θ(y_w|x)/π_ref(y_w|x)]
#                  - β * log[π_θ(y_l|x)/π_ref(y_l|x)] ) ]

import torch
import torch.nn.functional as F

def dpo_loss(
    policy_chosen_logps: torch.Tensor,    # log probs of chosen output under policy
    policy_rejected_logps: torch.Tensor,  # log probs of rejected output under policy
    reference_chosen_logps: torch.Tensor, # log probs of chosen output under reference
    reference_rejected_logps: torch.Tensor,
    beta: float = 0.1,
) -> torch.Tensor:
    """
    DPO loss: No explicit reward model needed.
    The reward is implicitly defined by the ratio of policy to reference log probs.
    """
    # Implicit reward: β * log(π_θ/π_ref)
    chosen_reward = beta * (policy_chosen_logps - reference_chosen_logps)
    rejected_reward = beta * (policy_rejected_logps - reference_rejected_logps)

    # Loss: -log sigmoid(reward_chosen - reward_rejected)
    # Equivalent to: maximize P(chosen rated higher than rejected)
    loss = -F.logsigmoid(chosen_reward - rejected_reward).mean()
    return loss

# DPO advantages over RLHF:
# - No separate RM training
# - No RL loop (just supervised training on preference pairs)
# - More stable training
# DPO disadvantages:
# - Offline: can't improve by sampling from current policy
# - Sensitive to quality of preference data
```

---

## Q6: Inference Optimization — Continuous Batching and Speculative Decoding

**Answer:**

**Continuous Batching (used by vLLM, TGI):**

Static batching: process a batch of N requests together, but wait until ALL N finish before releasing GPU. If request 1 is 10 tokens and request 5 is 500 tokens, the GPU is idle processing request 5 while request 1's slot sits empty.

Continuous batching: as soon as one request in the batch finishes, immediately insert a new waiting request into that slot. The batch composition changes dynamically each iteration.

```
Static batching (4 slots):
Time 1: [Req1 ████ ██████████████████████████████]
Time 2: [Req2 ████ ████████████]
Time 3: [Req3 ████ ████████████████████████]
Time 4: [Req4 ████ █████]
→ Must wait for longest request before starting next batch
→ GPU utilization: ~40-60%

Continuous batching:
Slot 1: [Req1 ▶▶▶▶▶ done] [Req5 ▶▶▶▶▶▶▶▶▶▶▶▶▶▶▶]
Slot 2: [Req2 ▶▶▶▶▶▶▶▶ done] [Req6 ▶▶▶▶▶▶▶▶▶▶▶]
Slot 3: [Req3 ▶▶▶▶▶▶▶▶▶▶▶▶ done] [Req7 ▶▶▶▶▶]
Slot 4: [Req4 ▶▶▶ done] [Req8 ▶▶▶▶▶▶▶▶▶▶▶▶▶▶▶]
→ GPU always has work to do
→ GPU utilization: ~85-95%
```

**Speculative Decoding:**
```python
# Speculative decoding: use a small draft model to propose k tokens,
# then verify all k tokens in ONE forward pass of the large target model

def speculative_decode(target_model, draft_model, prompt, max_tokens, k=5):
    """
    k = speculation length (how many tokens to draft before verifying)
    Speedup = k * acceptance_rate (typically 2-4x for good draft models)
    """
    tokens = list(prompt)

    while len(tokens) < max_tokens:
        # Draft phase: generate k tokens with small model (fast)
        draft_tokens = []
        draft_probs = []
        for _ in range(k):
            draft_logits = draft_model(tokens + draft_tokens)
            draft_prob = softmax(draft_logits[-1])
            draft_token = sample(draft_prob)
            draft_tokens.append(draft_token)
            draft_probs.append(draft_prob[draft_token])

        # Verify phase: ONE forward pass of target model on all k draft tokens
        target_logits = target_model(tokens + draft_tokens)
        target_probs = [softmax(l) for l in target_logits[len(tokens):]]

        # Accept/reject each draft token sequentially
        accepted = 0
        for i in range(k):
            # Acceptance probability: min(1, target_prob / draft_prob)
            accept_prob = min(1.0, target_probs[i][draft_tokens[i]] / draft_probs[i])
            if random() < accept_prob:
                tokens.append(draft_tokens[i])
                accepted += 1
            else:
                # Reject: sample from adjusted distribution and stop
                adjusted = target_probs[i] - draft_probs[i]
                adjusted = {k: max(0, v) for k, v in adjusted.items()}
                tokens.append(sample(adjusted))
                break

        # If all accepted, target model also generates one bonus token
        if accepted == k:
            tokens.append(sample(target_probs[-1]))

    return tokens

# Speedup conditions:
# - Draft model must be 5-10x faster than target
# - Draft model must be from same family (matching vocabulary, similar style)
# - Works best for predictable text (code, structured output)
# - Provides NO speedup if draft tokens are all rejected (adversarial input)
```
