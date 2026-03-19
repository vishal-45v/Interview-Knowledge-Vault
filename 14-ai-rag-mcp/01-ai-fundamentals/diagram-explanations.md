# Chapter 01: AI Fundamentals — Diagram Explanations

ASCII architecture and flow diagrams for core AI/ML concepts. Use these to explain system internals clearly in whiteboard interviews.

---

## Diagram 1: The Transformer Architecture (Full)

```
INPUT SEQUENCE: ["The", "cat", "sat"]
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│                    INPUT EMBEDDING                        │
│   "The" → [0.2, -0.5, 0.8, ...]   (d_model = 512)        │
│   "cat" → [0.7,  0.1, -0.3, ...]                         │
│   "sat" → [-0.1, 0.9, 0.4, ...]                          │
└───────────────────┬───────────────────────────────────────┘
                    │
                    ▼
┌───────────────────────────────────────────────────────────┐
│              POSITIONAL ENCODING (added, not concat)      │
│   PE(pos=0) = [sin(0/1), cos(0/1), sin(0/100), ...]       │
│   PE(pos=1) = [sin(1/1), cos(1/1), sin(1/100), ...]       │
│   PE(pos=2) = [sin(2/1), cos(2/1), sin(2/100), ...]       │
└───────────────────┬───────────────────────────────────────┘
                    │
         ┌──────────┴──────────────────────┐
         │  ENCODER STACK (N=6 layers)     │
         │                                 │
         │  ┌───────────────────────────┐  │
         │  │  Multi-Head Self-Attention │  │
         │  │  (each token attends to   │  │
         │  │   all other tokens)       │  │
         │  └──────────┬────────────────┘  │
         │             │ residual + LayerNorm│
         │  ┌──────────▼────────────────┐  │
         │  │  Feed-Forward Network     │  │
         │  │  Linear → ReLU → Linear   │  │
         │  │  (d_model → 4*d_model → d_model)│
         │  └──────────┬────────────────┘  │
         │             │ residual + LayerNorm│
         │             │ (repeat N times)  │
         └─────────────┼───────────────────┘
                       │ encoder output (key/value for decoder)
         ┌─────────────▼───────────────────┐
         │  DECODER STACK (N=6 layers)     │
         │                                 │
         │  ┌───────────────────────────┐  │
         │  │  Masked Self-Attention    │  │
         │  │  (causal: can only see    │  │
         │  │   past tokens)            │  │
         │  └──────────┬────────────────┘  │
         │             │ residual + LayerNorm│
         │  ┌──────────▼────────────────┐  │
         │  │  Cross-Attention          │  │
         │  │  Q: from decoder          │  │
         │  │  K,V: from ENCODER output │  │  ◄── connects to encoder
         │  └──────────┬────────────────┘  │
         │             │ residual + LayerNorm│
         │  ┌──────────▼────────────────┐  │
         │  │  Feed-Forward Network     │  │
         │  └──────────┬────────────────┘  │
         │             │ residual + LayerNorm│
         └─────────────┼───────────────────┘
                       │
                       ▼
              ┌────────────────┐
              │  Linear Layer  │  (d_model → vocab_size)
              └───────┬────────┘
                      │
                      ▼
              ┌────────────────┐
              │    Softmax     │  → probability over vocabulary
              └────────────────┘
                      │
              OUTPUT: ["Le", "chat", "est", "assis"]
```

**Key observations:**
- Encoder processes ALL input tokens simultaneously (bidirectional attention)
- Decoder generates tokens one at a time, seeing only past outputs (causal masking)
- Cross-attention in decoder allows each output token to "query" the full encoder output
- Residual connections (skip connections) prevent vanishing gradients through N=6 layers
- LayerNorm after each sub-layer stabilizes activations

---

## Diagram 2: Scaled Dot-Product Attention Flow

```
Given: sequence of 4 tokens, each represented as 512-dim vector

STEP 1: Project to Q, K, V using learned weight matrices
─────────────────────────────────────────────────────────────
  Token embeddings                W_Q, W_K, W_V (learned)
  (4 × 512)               ──────────────────────────────────►
                                  Q (4 × 64)  [for head h]
                                  K (4 × 64)
                                  V (4 × 64)

STEP 2: Compute attention scores
─────────────────────────────────────────────────────────────
         Q (4×64)        K^T (64×4)
            │                │
            └───── × ────────┘
                    │
                    ▼
          Raw scores (4×4): each token's affinity for each other token
          ┌──────────────────────────────┐
          │     to: T1   T2   T3   T4    │
          │ from T1: 8.2  2.1  1.3  0.9  │
          │      T2: 1.4  7.8  3.2  2.1  │
          │      T3: 0.8  2.9  6.4  1.7  │
          │      T4: 1.2  1.8  2.1  9.1  │
          └──────────────────────────────┘
                    │
                    ▼
          Divide by sqrt(d_k=64) = 8.0
          ┌──────────────────────────────┐
          │     to: T1   T2   T3   T4    │
          │ from T1: 1.03 0.26 0.16 0.11 │
          │      T2: 0.18 0.98 0.40 0.26 │  ← more uniform after scaling
          │      T3: 0.10 0.36 0.80 0.21 │
          │      T4: 0.15 0.23 0.26 1.14 │
          └──────────────────────────────┘
                    │
                    ▼
          Softmax (row-wise) → attention weights (4×4), each row sums to 1
          ┌──────────────────────────────┐
          │     to: T1   T2   T3   T4    │
          │ from T1: 0.71 0.11 0.09 0.08 │ ← T1 mostly attends to itself
          │      T2: 0.09 0.67 0.16 0.10 │
          │      T3: 0.08 0.15 0.61 0.13 │
          │      T4: 0.07 0.09 0.12 0.72 │
          └──────────────────────────────┘
                    │
                    ▼
          Multiply by V (4×64) → output (4×64)
          output[i] = sum over j of (weight[i][j] * V[j])
          = weighted blend of all value vectors
```

---

## Diagram 3: Training Loop with Backpropagation

```
┌─────────────────────────────────────────────────────────────────┐
│                        TRAINING LOOP                            │
└─────────────────────────────────────────────────────────────────┘

   ┌──────────┐    ┌──────────────────────────────────────────┐
   │  Dataset │───►│  DataLoader (shuffle, batch, collate)    │
   └──────────┘    └──────────────────┬───────────────────────┘
                                      │ X_batch, y_batch
                                      ▼
                   ┌──────────────────────────────────────────┐
                   │           FORWARD PASS                   │
                   │                                          │
                   │  Layer 1: z₁ = W₁X + b₁                 │
                   │           a₁ = relu(z₁)                  │
                   │           ↓                              │
                   │  Layer 2: z₂ = W₂a₁ + b₂                │
                   │           a₂ = relu(z₂)                  │
                   │           ↓                              │
                   │  Output:  ŷ = softmax(W₃a₂ + b₃)         │
                   │           ↓                              │
                   │  Loss:    L = CrossEntropy(ŷ, y_batch)   │
                   └──────────────────┬───────────────────────┘
                                      │  L (scalar)
                                      ▼
                   ┌──────────────────────────────────────────┐
                   │           BACKWARD PASS                  │
                   │                                          │
                   │  ∂L/∂W₃ ◄── ∂L/∂ŷ ← dL/da₂ ← ...       │
                   │  ∂L/∂W₂ ◄── ∂L/∂a₂ × ∂a₂/∂W₂           │
                   │  ∂L/∂W₁ ◄── ∂L/∂a₁ × ∂a₁/∂W₁           │
                   │                                          │
                   │  (Chain rule propagates blame backward)  │
                   └──────────────────┬───────────────────────┘
                                      │  gradients for each W, b
                                      ▼
                   ┌──────────────────────────────────────────┐
                   │           OPTIMIZER STEP                 │
                   │                                          │
                   │  SGD:   W ← W - lr × ∂L/∂W              │
                   │  Adam:  W ← W - lr × m̂/(√v̂ + ε)         │
                   │  AdamW: W ← W(1-lr×λ) - lr × m̂/(√v̂ + ε) │
                   └──────────────────┬───────────────────────┘
                                      │
                                      │ (repeat for N epochs)
                                      ▼
                        ┌─────────────────────┐
                        │  Validation Check   │
                        │  val_loss < best?   │
                        │  → save checkpoint  │
                        └─────────────────────┘
```

---

## Diagram 4: Overfitting, Underfitting, and the Bias-Variance Tradeoff

```
LOSS vs MODEL COMPLEXITY
─────────────────────────────────────────────────────────────────

Loss
 │
 │  Training Loss ─ ─ ─ ─  (always decreases with complexity)
 │  Validation Loss ─────  (U-shaped)
 │
 │\                                              /
 │ \                                            /
 │  \          ┌──────────────────────┐        /
 │   \         │   SWEET SPOT         │       /
 │    \        │  Bias-Variance       │      /
 │     \_______│  Tradeoff Minimum    │_____/
 │             └──────────────────────┘
 │
 └────────────────────────────────────────────────── Model Complexity
  │UNDERFIT│         │GOOD FIT│            │OVERFIT│

UNDERFIT (high bias):              OVERFIT (high variance):
- Train loss high                  - Train loss low (near 0)
- Val loss high                    - Val loss high
- Model too simple                 - Model memorized training data
- "Can't learn the pattern"        - "Learned the noise"

FIX:                               FIX:
- More parameters                  - More data
- More training epochs             - Dropout, L1/L2 regularization
- Better features                  - Early stopping
- Reduce regularization            - Data augmentation
                                   - Reduce model size
```

```
LOSS vs TRAINING EPOCHS (time-axis view)
─────────────────────────────────────────

Loss │
     │\   Training Loss
     │ \  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─▼
     │  \
     │   \   Val Loss
     │    \───────────────────────────┐
     │     \                          │
     │      \________________________ │╲_______________
     │                                │                ╲ VAL LOSS
     │      ▲                         │                  INCREASING
     │      │                         │                  (overfitting begins)
     │   BEST CHECKPOINT              │
     │   (save here!)                 │
     │                         EARLY STOPPING
     │                         TRIGGER
     └──────────────────────────────────────────────── Epochs
```

---

## Diagram 5: Multi-Head Attention — Parallel Subspace Learning

```
INPUT (4 tokens × 512 dims)
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                   MULTI-HEAD ATTENTION                      │
│                                                             │
│  Input projected 8 times with different W_Q, W_K, W_V      │
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐     ┌─────────┐    │
│  │ HEAD 1  │  │ HEAD 2  │  │ HEAD 3  │ ... │ HEAD 8  │    │
│  │ 64 dims │  │ 64 dims │  │ 64 dims │     │ 64 dims │    │
│  │         │  │         │  │         │     │         │    │
│  │ Attends │  │ Attends │  │ Attends │     │ Attends │    │
│  │ to:     │  │ to:     │  │ to:     │     │ to:     │    │
│  │ syntac- │  │ semantic│  │ position│     │ corefer-│    │
│  │ tic     │  │ similar-│  │ -al     │     │ ence    │    │
│  │ depend- │  │ ity     │  │ proxim- │     │         │    │
│  │ encies  │  │         │  │ ity     │     │         │    │
│  └────┬────┘  └────┬────┘  └────┬────┘     └────┬────┘    │
│       │            │            │               │          │
│       └────────────┴────────────┴───────────────┘          │
│                          │                                  │
│               CONCATENATE (4 × 512 = 4 × 8×64)             │
│                          │                                  │
│                    W_O (512×512)                            │
│                          │                                  │
│               OUTPUT (4 × 512)                              │
└─────────────────────────────────────────────────────────────┘

Why multiple heads?
- Each head sees the FULL sequence but through a different 64-dim projection
- Different projections capture different relationship types
- Head 1 might focus: "what does this verb's subject look like?"
- Head 2 might focus: "what semantically similar words are nearby?"
- Head 5 might focus: "which pronoun does 'it' refer to?"
- W_O learns how to combine all these perspectives
```

---

## Diagram 6: Word Embedding Space — Static vs Contextual

```
STATIC EMBEDDINGS (Word2Vec / GloVe): one point per word type
──────────────────────────────────────────────────────────────

2D projection of 300-dim embedding space:

  ▲
  │    queen ●          ● princess
  │         king ●   ● prince
  │
  │                  ● woman    ● girl
  │          ● man        ● boy
  │
  │  ● bank_river ── ── ──►  "bank" has ONE point
  │  ● bank_finance         (average of all uses)
  │                         ← PROBLEM: ambiguous
  │
  │  ● run(verb) ── ── ──►  "run" has ONE point
  │  ● run(noun_in_baseball)  (but meanings differ)
  │
  └──────────────────────────────────────────────►

CONTEXTUAL EMBEDDINGS (BERT / GPT): different point per occurrence
──────────────────────────────────────────────────────────────

"I went to the BANK to deposit money"
         "bank" → [0.2, 0.8, -0.3, ...]  ←── near "finance", "deposit"
                        │
                        ▼
"She sat on the BANK of the river"
         "bank" → [0.7, -0.1, 0.9, ...]  ←── near "river", "shore", "sit"

The SAME word gets DIFFERENT vectors depending on context.
This is produced by bidirectional attention: each token's
final representation is influenced by ALL other tokens.

Layer evolution for "bank" in BERT:
  Layer 0 (embedding):  → starts the same regardless of context
  Layer 6 (middle):     → syntactic context begins differentiating
  Layer 12 (top):       → fully context-dependent, semantically rich
```
