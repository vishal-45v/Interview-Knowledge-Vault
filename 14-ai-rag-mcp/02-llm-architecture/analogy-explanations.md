# Chapter 02: LLM Architecture — Analogy Explanations

Core LLM concepts explained through concrete everyday analogies.

---

## Analogy 1: Autoregressive Generation is a Typewriter with Photographic Memory

**The story:**

Imagine a typewriter where every letter you type is immediately photographed and stored. When you press the next key, the typewriter consults the photograph of everything typed so far — the entire document history — to decide which key to strike next. The typewriter always writes the next character, never the full word at once.

That's autoregressive generation. The model generates one token at a time, always conditioning on the full sequence of previously generated tokens. There's no "planning ahead" — just a very sophisticated pattern completion at each step.

**The implications:**

1. **Early mistakes compound**: If the model generates a wrong token early, every subsequent token conditions on that error. Like typing the wrong first letter and having no backspace.
2. **The model cannot revise**: Generation is strictly left-to-right. The model cannot "change its mind" about earlier tokens once generated.
3. **Beam search is a workaround**: Instead of one typewriter, run k typewriters in parallel, keep only the k best sequences at each step, and return the best final sequence.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

model = AutoModelForCausalLM.from_pretrained("gpt2")
tokenizer = AutoTokenizer.from_pretrained("gpt2")

# Each call adds exactly one token
input_ids = tokenizer.encode("The cat sat on", return_tensors="pt")

tokens_generated = []
with torch.no_grad():
    for step in range(10):
        outputs = model(input_ids)
        # outputs.logits shape: (batch=1, seq_len, vocab_size)
        next_token_logits = outputs.logits[:, -1, :]  # Only last position
        next_token = torch.argmax(next_token_logits, dim=-1, keepdim=True)
        tokens_generated.append(next_token.item())
        input_ids = torch.cat([input_ids, next_token], dim=-1)

print("Generated:", tokenizer.decode(tokens_generated))
```

---

## Analogy 2: KV Cache is Keeping Your Homework in a Folder

**The story:**

Imagine you are a student answering questions, and for each question you must first read a long textbook chapter. Without a filing system, you'd have to re-read the entire chapter from scratch for every new question — exhausting and slow.

With a filing system (KV cache), you read the chapter once, take notes (Key-Value pairs), and keep those notes in a folder. For each new question, you just consult your notes — you don't re-read the chapter.

The KV cache stores the "notes" (Key and Value matrices) computed for each token. When generating the next token, the model only computes fresh Q, K, V for the new token, then looks up the cached K, V for all previous tokens.

```python
# What gets cached: K and V matrices for each layer
# Shape: (batch_size, num_heads, seq_len, head_dim)

# Without cache (for illustration):
def slow_generate(model, input_ids, num_new_tokens):
    all_ids = input_ids
    for _ in range(num_new_tokens):
        # Recomputes ENTIRE sequence KV from scratch each step
        outputs = model(all_ids, use_cache=False)
        next_token = outputs.logits[:, -1:, :].argmax(-1)
        all_ids = torch.cat([all_ids, next_token], dim=1)
        # Cost: O(n) attention operations, grows with sequence length
    return all_ids

# With cache (what actually happens):
def fast_generate(model, input_ids, num_new_tokens):
    past_key_values = None
    all_ids = input_ids

    for step in range(num_new_tokens):
        if past_key_values is None:
            # First step: process full prompt, populate cache
            outputs = model(all_ids, use_cache=True)
        else:
            # Subsequent steps: only process the single new token
            # Pass ONLY the new token as input
            outputs = model(
                all_ids[:, -1:],           # Just the latest token
                past_key_values=past_key_values,  # Provide cached K, V
                use_cache=True
            )
        past_key_values = outputs.past_key_values  # Updated cache
        next_token = outputs.logits[:, -1:, :].argmax(-1)
        all_ids = torch.cat([all_ids, next_token], dim=1)
    return all_ids

# Memory cost: the folder gets bigger with each page you add
# 1000 tokens of KV cache for LLaMA-7B ≈ 512 MB — not trivial
```

---

## Analogy 3: Temperature is a Spice Level Dial for Text

**The story:**

You walk into a restaurant and order the house curry. The chef can make it mild, medium, or extra hot — it's the same recipe, just with different amounts of spice. Mild: safe, predictable, most people like it. Extra hot: exciting, surprising, occasionally inedible.

Temperature does the same thing to an LLM's output distribution. High temperature: adds "spice" — the model considers unlikely tokens more seriously, producing creative and surprising text. Low temperature: strips spice — the model sticks to the safest, most probable tokens, producing reliable but potentially boring output.

The critical insight: temperature doesn't teach the model new things. It adjusts how strongly the model commits to its learned probability estimates.

```python
import torch
import torch.nn.functional as F

# These are real logit values from a language model for the next token
# after "The weather today is"
logits = {
    "sunny": 3.2,
    "nice": 2.8,
    "good": 2.5,
    "unpredictable": 1.1,
    "catastrophic": -0.5,
    "algorithmic": -2.1,
}

tokens = list(logits.keys())
raw = torch.tensor(list(logits.values()))

for temp in [0.1, 0.5, 1.0, 2.0]:
    probs = F.softmax(raw / temp, dim=-1)
    print(f"\nTemperature {temp}:")
    for tok, p in zip(tokens, probs):
        bar = "█" * int(p * 40)
        print(f"  {tok:15s}: {p:.3f} {bar}")

# Temperature 0.1: sunny=0.98, nice=0.02, rest≈0.00 (near-deterministic)
# Temperature 1.0: sunny=0.45, nice=0.34, good=0.17, unpred=0.04 (original)
# Temperature 2.0: sunny=0.24, nice=0.22, good=0.19, unpred=0.14 (exploratory)
```

**The "wrong spice" problem:** Code generation with temperature=1.5 produces syntactically invalid code. Creative writing with temperature=0.1 produces generic, clichéd sentences. Temperature is a tool, not a setting that's universally right or wrong.

---

## Analogy 4: BPE Tokenization is Like Abbreviation Evolution in Texting

**The story:**

Early text messaging charged per character. People started abbreviating: "be right back" → "brb", "laugh out loud" → "lol", "to be honest" → "tbh". Over time, the most frequent multi-character sequences got compressed into single symbols.

BPE does exactly this, but for text corpora. It starts with individual characters (or bytes). It then finds the most frequent pair of adjacent tokens and merges them into a single new token. Repeat until you hit the target vocabulary size.

```python
def bpe_encode_example():
    """
    Corpus: "low low low lower lower newest newest widest"
    Starting vocabulary: l, o, w, e, r, n, s, t, i, d, </w>
    """
    # Initial tokenization (words split into chars, </w> marks word end)
    words = {
        ("l", "o", "w", "</w>"): 3,    # "low" appears 3 times
        ("l", "o", "w", "e", "r", "</w>"): 2,  # "lower" appears 2 times
        ("n", "e", "w", "e", "s", "t", "</w>"): 2,  # "newest" appears 2 times
        ("w", "i", "d", "e", "s", "t", "</w>"): 1,  # "widest" appears 1 time
    }

    # BPE Iteration 1: count pairs
    # "l","o" appears: 3+2 = 5 times (most frequent!)
    # "o","w" appears: 3+2 = 5 times  (tied)
    # → merge (l, o) → lo
    words_iter1 = {
        ("lo", "w", "</w>"): 3,
        ("lo", "w", "e", "r", "</w>"): 2,
        ("n", "e", "w", "e", "s", "t", "</w>"): 2,
        ("w", "i", "d", "e", "s", "t", "</w>"): 1,
    }

    # BPE Iteration 2: now "lo","w" appears 3+2=5 times → merge to "low"
    words_iter2 = {
        ("low", "</w>"): 3,
        ("low", "e", "r", "</w>"): 2,
        ("n", "e", "w", "e", "s", "t", "</w>"): 2,
        ("w", "i", "d", "e", "s", "t", "</w>"): 1,
    }
    # Continue until target vocabulary size reached...

bpe_encode_example()

# Real usage with tiktoken:
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")

# Common English words become single tokens
print(enc.encode("the"))       # [1820] — one token
print(enc.encode("transformer"))  # [36521] — one token

# Rare/compound words split into subwords
print(enc.encode("CRISPR"))    # [62, 2200, 4805, 49, 8] — multiple tokens
# Each merge step in BPE training added the most frequent pair from pretraining corpus
# "CRISPR" is rare in web text, so it never got merged into a single token
```

---

## Analogy 5: Speculative Decoding is Like a Draft-Review Workflow

**The story:**

Imagine a junior writer (draft model) and a senior editor (target model). The senior editor is excellent but slow — reviewing one paragraph takes 30 minutes. The junior writer is fast but less accurate.

Old workflow: Senior editor writes and reviews every sentence (very slow, very accurate).

New workflow: Junior writer drafts 5 sentences quickly. Senior editor reviews all 5 in one sitting (same 30 minutes, since they read at the same speed). If the junior writer's draft is good, the senior editor accepts it and the output is 5x faster. If a sentence is wrong, the editor corrects it from that point.

Speculative decoding works identically. The draft model (junior writer) proposes k tokens. The target model (senior editor) verifies all k in a single forward pass — because verifying is cheaper than generating. Accepted tokens are kept; the first rejected token is corrected.

```python
# The math: why is verifying all k tokens ONE forward pass?
# - Standard generation: target model runs k separate forward passes
# - Speculative verification: target model runs 1 forward pass on input + k draft tokens
#   and reads off the predicted probabilities at each position simultaneously
#   Because transformers process ALL positions in parallel during the forward pass!

# This is the key insight: VERIFICATION is parallelizable, GENERATION is sequential

# Acceptance criterion for token i:
# if U ~ Uniform(0,1) < min(1, p_target(x_i) / p_draft(x_i)):
#     accept x_i
# else:
#     reject; sample from adjusted_distribution = max(0, p_target - p_draft) / Z

# Expected tokens per target model forward pass:
# E[accepted] = sum_{k} product_{i<k}(min(1, p_T(x_i)/p_D(x_i)))
# For a good draft model, this is 3-5 tokens per target model call
# vs 1 token per target model call without speculative decoding
```

---

## Analogy 6: RLHF is Training a Dog with a Human Judge

**The story:**

You want to train a dog to perform tricks that humans find delightful — but "delightful" is hard to encode as a simple reward function. You can't just reward "sits in 3 seconds" because that's too narrow.

Stage 1 (SFT): Watch professional dog trainers and copy their techniques (supervised fine-tuning on demonstrations).

Stage 2 (Reward Model): Ask judges to rank different trick performances: "Was performance A, B, or C better?" Learn a judge preference model from thousands of these rankings.

Stage 3 (RL): Train the dog using the learned judge as feedback. The dog tries a trick, the (now automated) judge scores it, and the dog adjusts its behavior to maximize the judge's score. A "leash" (KL penalty) prevents the dog from doing crazy tricks that fool the automated judge but would horrify a real human.

```python
# Conceptual RLHF training loop
def rlhf_training_step(policy_model, reference_model, reward_model, batch):
    """
    policy_model: model being trained (starts from SFT checkpoint)
    reference_model: frozen SFT model (the "leash")
    reward_model: trained to predict human preference scores
    """
    prompts = batch["prompts"]

    # Sample from policy
    generated_outputs = policy_model.generate(prompts, do_sample=True)

    # Get reward from reward model
    rewards = reward_model(prompts, generated_outputs)  # scalar per sample

    # Compute KL penalty: don't stray too far from SFT reference
    policy_logprobs = policy_model.log_probs(prompts, generated_outputs)
    reference_logprobs = reference_model.log_probs(prompts, generated_outputs)
    kl_penalty = (policy_logprobs - reference_logprobs).mean()

    # PPO objective: maximize reward, penalize KL divergence from reference
    beta = 0.1  # KL coefficient — controls how far policy can drift
    objective = rewards.mean() - beta * kl_penalty

    loss = -objective  # Minimize negative objective
    loss.backward()

    # The leash (KL penalty) prevents "reward hacking":
    # Without it, the model would find ways to get high reward model scores
    # (e.g., always outputting "I'm happy to help!")
    # that don't actually correspond to genuinely better outputs
```

---

## Analogy 7: Quantization is Compressing a Photograph

**The story:**

A RAW camera photograph has 16 bits per color channel per pixel — very precise, huge file size. A JPEG uses 8 bits per channel with lossy compression — smaller file, some visual quality lost. A heavily compressed JPEG uses even fewer bits — fast to send, but you might see artifacts.

Quantization does the same thing to model weights. FP32 (32-bit floating point) is the "RAW" format. FP16 is the "8-bit JPEG" — much smaller, essentially no quality loss for inference. INT8 is "heavy compression" — 4x smaller than FP32, small but measurable quality loss. INT4 is "max compression" — 8x smaller, potentially noticeable quality loss for sensitive tasks.

The challenge: unlike photographs where every pixel has similar importance, model weights are heterogeneous. Some weights have large values (outliers) that carry significant information — quantizing them too aggressively causes disproportionate quality loss.

```python
# LLM.int8(): outlier-aware quantization for LLMs
# Key finding (Dettmers et al., 2022): in LLMs, 0.1% of weight dimensions
# contain outlier values (>6x the standard deviation) that are CRITICAL
# for model quality. Quantizing these outliers to INT8 destroys performance.

# LLM.int8() solution: mixed-precision decomposition
# 1. Identify outlier columns in the weight matrix (threshold = 6.0)
# 2. Keep outlier columns in float16 (high precision)
# 3. Quantize non-outlier columns to int8 (low memory)
# 4. Compute: result = int8_matmul(non_outlier_W, non_outlier_X)
#                    + fp16_matmul(outlier_W, outlier_X)

from transformers import AutoModelForCausalLM
import torch

# Load model with automatic INT8 quantization
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    load_in_8bit=True,          # Uses LLM.int8()
    device_map="auto",
    torch_dtype=torch.float16,
)
# Result: 7B model uses ~7GB instead of 14GB
# Quality: typically <1% degradation on downstream benchmarks

# For even smaller: 4-bit via bitsandbytes NF4 (QLoRA format)
model_4bit = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-hf",
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",  # NormalFloat4: optimal for normally distributed weights
    bnb_4bit_compute_dtype=torch.float16,
    device_map="auto",
)
# Result: ~4GB for 7B model. QLoRA fine-tunes using this 4-bit base.
```
