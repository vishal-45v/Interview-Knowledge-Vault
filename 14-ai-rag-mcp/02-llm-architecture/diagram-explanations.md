# Chapter 02: LLM Architecture — Diagram Explanations

ASCII diagrams for LLM internals: model families, inference pipeline, KV cache, and quantization.

---

## Diagram 1: LLM Family Comparison — Encoder-Only, Decoder-Only, Encoder-Decoder

```
ENCODER-ONLY (BERT, RoBERTa, DeBERTa)
─────────────────────────────────────

Input:  "The [MASK] sat on the mat"
           │
           ▼
    ┌─────────────────────────────────────────────────────────┐
    │         BIDIRECTIONAL SELF-ATTENTION                    │
    │  Every token can attend to EVERY other token            │
    │  (including future tokens)                              │
    │                                                         │
    │  "The" ◄──────────────────────────────► "mat"           │
    │         ◄───────────────────────────►                   │
    │                  ◄──────────────────►                   │
    │                           ◄──────────►                  │
    └─────────────────────────────────────────────────────────┘
           │
           ▼
    Output: contextual embedding for each token position
    Use for: classification, NER, QA, embeddings, NOT generation

DECODER-ONLY (GPT-2, GPT-4, LLaMA, Mistral, Phi)
─────────────────────────────────────────────────

Input:  "The cat"  →  generate: "sat"
           │
           ▼
    ┌─────────────────────────────────────────────────────────┐
    │         CAUSAL (LEFT-TO-RIGHT) SELF-ATTENTION           │
    │  Each token can ONLY attend to itself and past tokens   │
    │                                                         │
    │  "The" ─────────────────────────────────────────────►  │
    │  "cat" ◄──────── ──────────────────────────────────►   │
    │  "sat" ◄──────────────────── ────────────────────►     │
    │  Future: MASKED (set to -inf before softmax)           │
    └─────────────────────────────────────────────────────────┘
           │
           ▼
    Output: next token prediction at each position
    Use for: text generation, chat, code, instruction following

ENCODER-DECODER (T5, BART, mT5)
────────────────────────────────

Input: "Translate to French: The cat"
           │
           ▼
    ┌──────────────────┐
    │    ENCODER        │ (bidirectional attention over input)
    │  "Translate"     │
    │  "to"            │
    │  "French"        │
    │  ":"             │
    │  "The"           │
    │  "cat"           │
    └────────┬─────────┘
             │ encoder hidden states (used as K, V in decoder cross-attention)
             │
             ▼
    ┌──────────────────────────────────────┐
    │  DECODER                             │
    │  Causal self-attention (past outputs)│
    │  + Cross-attention (encoder states)  │
    │                                      │
    │  → "Le"  → "chat"  → </s>           │
    └──────────────────────────────────────┘
    Use for: translation, summarization, structured generation
```

---

## Diagram 2: KV Cache — Memory Growth During Generation

```
PROMPT: "Tell me about transformers"   (5 tokens)
         T1   T2   T3   T4   T5

STEP 0: Fill KV cache from prompt (one forward pass, all 5 tokens)
┌─────────────────────────────────────────────────────────────┐
│  Layer 1 KV Cache: K=[K1,K2,K3,K4,K5] V=[V1,V2,V3,V4,V5]  │
│  Layer 2 KV Cache: K=[K1,K2,K3,K4,K5] V=[V1,V2,V3,V4,V5]  │
│  ...32 layers...                                            │
└─────────────────────────────────────────────────────────────┘
Memory: 5 tokens × 512 KB/token (LLaMA-7B) = 2.5 MB

STEP 1: Generate token T6 (only compute Q,K,V for T6)
┌─────────────────────────────────────────────────────────────┐
│  Layer 1 KV Cache: K=[K1,K2,K3,K4,K5,K6] V=[...,V6]        │
│  Query from T6: Q6                                          │
│  Attention: softmax(Q6 × [K1...K6]^T / sqrt(d)) × [V1..V6] │
│             ←────────────── READ ENTIRE CACHE ──────────────│
└─────────────────────────────────────────────────────────────┘
Memory grows: now 6 tokens × 512 KB = 3 MB

STEP 100: Generate token T105
┌─────────────────────────────────────────────────────────────┐
│  Layer 1 KV Cache: K=[K1..K105] V=[V1..V105]                │
│  Memory: 105 × 512 KB = 52.5 MB for LAYER 1 ALONE           │
│  All 32 layers: 52.5 MB × 32 = 1.68 GB just for KV cache    │
└─────────────────────────────────────────────────────────────┘

MEMORY FORMULA:
KV_bytes = 2 × num_layers × num_kv_heads × head_dim × seq_len × dtype_bytes

LLaMA-2-7B  (MHA, 32 layers, 32 heads, dim=128, fp16):
  Per token = 2 × 32 × 32 × 128 × 2 = 524,288 bytes = 512 KB/token
  At 4K seq:  512KB × 4096 = 2.1 GB

LLaMA-2-70B (GQA, 80 layers,  8 heads, dim=128, fp16):
  Per token = 2 × 80 ×  8 × 128 × 2 = 327,680 bytes = 320 KB/token
  At 4K seq:  320KB × 4096 = 1.3 GB  ← LESS than 7B due to GQA!
```

---

## Diagram 3: Sampling Strategies — From Greedy to Nucleus

```
MODEL OUTPUT LOGITS for next token after "The weather today is":
───────────────────────────────────────────────────────────────

Token          Raw Logit   Probability (temp=1.0)
────────────────────────────────────────────────
"sunny"           3.2         0.445  ████████████████████████
"nice"            2.8         0.298  ████████████████
"good"            2.5         0.220  ████████████
"cloudy"          1.1         0.062  ███
"unpredictable"   0.3         0.029  █
"terrible"       -0.8         0.009  ▌
"algorithmic"    -2.1         0.002  ▌
...many more...

GREEDY DECODING (temperature → 0):
  Always pick argmax → "sunny"
  Risk: repetition loops, mode collapse

TOP-K SAMPLING (k=3):
  ┌────────────────────────────┐
  │ Allowed: "sunny", "nice",  │  (top 3 by probability)
  │          "good"            │
  └────────────────────────────┘
  Renormalize → sample from these 3
  Risk: k=3 too restrictive on flat distributions

TOP-P / NUCLEUS SAMPLING (p=0.9):
  Cumulative probability scan:
  "sunny":  0.445  [total so far: 0.445]
  "nice":   0.298  [total so far: 0.743]
  "good":   0.220  [total so far: 0.963] ← crosses 0.9
  ┌──────────────────────────────────────┐
  │ Nucleus: {"sunny", "nice", "good"}   │ (captured 96.3% of mass)
  │ (dynamic size: 3 here, but adapts)   │
  └──────────────────────────────────────┘
  Advantage: adapts to distribution shape
  - On peaked distributions: small nucleus (safe)
  - On flat distributions: large nucleus (creative)

COMBINED (top_p=0.9, temperature=0.8):
  1. Scale logits: logits / 0.8 (sharpen distribution)
  2. Apply top-p filter (nucleus)
  3. Sample from remaining tokens
  This is the most common production setting.
```

---

## Diagram 4: Continuous Batching vs Static Batching

```
STATIC BATCHING (naive approach):
──────────────────────────────────
4 request slots. All must finish before next batch starts.

Time →  0   5   10  15  20  25  30  35  40  45  50
Slot 1: [Req A ─────────────────────────────────── done]
Slot 2: [Req B ─────────── done]                   [waiting...]
Slot 3: [Req C ─────────────────────── done]       [waiting...]
Slot 4: [Req D ───── done]                         [waiting...]
         ▲
         All 4 must complete before Req E,F,G,H start
         GPU idle for slots 2,3,4 while slot 1 finishes

GPU Utilization: ≈ 40% (lots of idle time waiting for stragglers)

CONTINUOUS BATCHING (vLLM, TGI approach):
──────────────────────────────────────────
Slots are refilled the moment a request completes.

Time →  0   5   10  15  20  25  30  35  40  45  50
Slot 1: [A ──────────────────────────────── done][E ─────────...]
Slot 2: [B ─────── done][E starts!]              ^refilled immediately
         ^             ^insert E here, don't wait for A
Slot 3: [C ─────────────────── done][F ───────...]
Slot 4: [D ─── done][G ──────────────── done][H ─...]
         ^    ^insert G here

GPU Utilization: ≈ 90% (nearly always has work to do)

IMPLEMENTATION CHALLENGE:
- Requests in the same batch have different sequence lengths at each step
- PagedAttention (vLLM) solves KV cache fragmentation:
  Instead of pre-allocating contiguous memory per request,
  allocates "pages" of KV cache like virtual memory in an OS
  → No wasted pre-allocated memory
  → Can serve 2-4x more concurrent requests
```

---

## Diagram 5: RLHF Pipeline — Three-Stage Training

```
STAGE 1: SUPERVISED FINE-TUNING (SFT)
──────────────────────────────────────
Base LLM ─── fine-tune on ──► SFT Model
             human-written
             demonstrations
             (high quality prompts + ideal responses)

STAGE 2: REWARD MODEL TRAINING
────────────────────────────────
┌────────────────────────────────────────────────────────────┐
│  Human Annotation                                          │
│  Prompt: "Explain quantum entanglement"                    │
│                                                            │
│  Response A: [SFT generated] "Quantum entanglement..."     │
│  Response B: [SFT generated] "Particles become linked..."  │
│                        ↓                                   │
│  Human: "A is better" ────────────────────────────────────►│
│                                                            │
│  (Repeat for thousands of prompt-response pairs)           │
└─────────────────┬──────────────────────────────────────────┘
                  │ preference data (prompt, chosen, rejected)
                  ▼
┌────────────────────────────────────────────────────────────┐
│  REWARD MODEL (RM)                                         │
│  Architecture: SFT model + scalar head                     │
│  Training: RM(chosen) > RM(rejected) via ranking loss      │
│  Output: scalar score ∈ ℝ for any (prompt, response) pair  │
└────────────────────────────────────────────────────────────┘

STAGE 3: PPO TRAINING
──────────────────────
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  Policy (π_θ)          Reference (π_ref, FROZEN)                 │
│  starts = SFT Model    starts = SFT Model                        │
│       │                       │                                  │
│       ▼                       ▼                                  │
│  Generate response       Compute log-probs of                    │
│  given prompt            policy's output under reference         │
│       │                       │                                  │
│       ▼                       │                                  │
│  RM scores response           │                                  │
│  r = RM(prompt, response)     │                                  │
│       │                       │                                  │
│       ▼                       ▼                                  │
│  PPO objective:                                                  │
│  maximize: r - β * KL(π_θ || π_ref)                             │
│            ↑               ↑                                     │
│         reward         KL penalty ("leash")                      │
│                        prevents reward hacking                   │
│                                                                  │
│  Update π_θ via PPO gradient step                                │
└──────────────────────────────────────────────────────────────────┘
       │
       ▼
  RLHF-trained model (ChatGPT, Claude, LLaMA-chat)
  - Follows instructions better
  - More helpful, honest, harmless
  - Aligns with human preferences (as captured by annotators)
```

---

## Diagram 6: Speculative Decoding — Draft-Verify Cycle

```
TARGET MODEL (70B, slow, high quality)
DRAFT MODEL  (7B, fast, decent quality)
k = 4 (draft 4 tokens before verifying)

STEP 1: DRAFT PHASE (4 sequential draft model calls, fast)
──────────────────────────────────────────────────────────
Prompt: "The capital of France is"

Draft call 1: draft_model → predicts "Paris"   [p_draft=0.95]
Draft call 2: draft_model → predicts "and"     [p_draft=0.72]
Draft call 3: draft_model → predicts "its"     [p_draft=0.61]
Draft call 4: draft_model → predicts "famous"  [p_draft=0.48]

Proposed sequence: ["Paris", "and", "its", "famous"]

STEP 2: VERIFY PHASE (1 target model call, parallel)
──────────────────────────────────────────────────────
Input to target: "The capital of France is" + ["Paris", "and", "its", "famous"]
ONE forward pass → target model predicts probability at each position simultaneously:

Position "Paris":   p_target("Paris")  = 0.97
                    accept prob = min(1, 0.97/0.95) = 1.0  → ACCEPT ✓

Position "and":     p_target(", ")     = 0.68  ← target prefers comma!
                    p_target("and")    = 0.18
                    accept prob = min(1, 0.18/0.72) = 0.25  → FLIP COIN
                    → say we REJECT (probability 75%)

Position "its":     Not reached (rejected at previous)

Correction: sample from adjusted distribution at "and" position:
            max(0, p_target - p_draft) / Z → samples ", "

RESULT: accepted ["Paris"] + corrected [", "] → 2 tokens this cycle

EXPECTED SPEEDUP:
If acceptance rate ≈ 0.8 per token:
E[tokens per target call] = 1 + 0.8 + 0.8^2 + 0.8^3 + 0.8^4 = 3.36
vs 1 token per target call without speculative decoding
Speedup: 3.36x (assuming draft calls are negligible cost)

WHEN SPECULATIVE DECODING FAILS:
- Draft model from different family (vocabulary mismatch)
- Highly creative/open-ended text (low acceptance rate)
- Very short outputs (overhead not worth it)
- Draft model too slow (no time advantage from parallelism)
```
