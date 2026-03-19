# Chapter 01: AI Fundamentals — Structured Answers

Complete, interview-ready answers for the most important and frequently asked topics. Written at the level of a senior ML/AI engineer.

---

## Q1: Explain the transformer attention mechanism — from the formula to why it works

**Answer:**

The attention mechanism lets every position in a sequence directly attend to every other position, bypassing the sequential bottleneck of RNNs.

**The formula:**
```
Attention(Q, K, V) = softmax(QK^T / sqrt(d_k)) * V
```

Where:
- **Q** (queries): "What am I looking for?" — derived from the current position's representation
- **K** (keys): "What do I have to offer?" — derived from all positions
- **V** (values): "What do I actually give if attended to?" — derived from all positions

The dot product `QK^T` computes pairwise similarity scores between every query and every key. The `sqrt(d_k)` scaling prevents the softmax from entering a region with near-zero gradients when `d_k` is large (dot products grow with dimension due to random projection variance).

**Why sqrt(d_k) matters:**
```python
import torch
import torch.nn.functional as F

d_k = 512
q = torch.randn(1, 8, d_k)  # batch=1, seq=8, dim=512
k = torch.randn(1, 8, d_k)

scores_unscaled = torch.bmm(q, k.transpose(1, 2))
scores_scaled = scores_unscaled / (d_k ** 0.5)

# Without scaling: variance of scores grows linearly with d_k
# Softmax of large values -> one-hot distribution -> gradient vanishes
print(f"Unscaled softmax entropy: {F.softmax(scores_unscaled, -1).mean(-1).std():.4f}")
print(f"Scaled softmax entropy:   {F.softmax(scores_scaled, -1).mean(-1).std():.4f}")
```

**Multi-head attention** runs h=8 (or 16, 32) attention heads in parallel, each with its own W_Q, W_K, W_V projection matrices:
```python
class MultiHeadAttention(torch.nn.Module):
    def __init__(self, d_model, num_heads):
        super().__init__()
        assert d_model % num_heads == 0
        self.d_k = d_model // num_heads
        self.num_heads = num_heads
        self.W_q = torch.nn.Linear(d_model, d_model)
        self.W_k = torch.nn.Linear(d_model, d_model)
        self.W_v = torch.nn.Linear(d_model, d_model)
        self.W_o = torch.nn.Linear(d_model, d_model)

    def forward(self, q, k, v, mask=None):
        B, T, D = q.shape
        # Project and reshape to (B, num_heads, T, d_k)
        Q = self.W_q(q).view(B, T, self.num_heads, self.d_k).transpose(1, 2)
        K = self.W_k(k).view(B, T, self.num_heads, self.d_k).transpose(1, 2)
        V = self.W_v(v).view(B, T, self.num_heads, self.d_k).transpose(1, 2)

        scores = torch.matmul(Q, K.transpose(-2, -1)) / (self.d_k ** 0.5)
        if mask is not None:
            scores = scores.masked_fill(mask == 0, float('-inf'))
        attn = torch.softmax(scores, dim=-1)
        out = torch.matmul(attn, V)  # (B, num_heads, T, d_k)

        # Concatenate heads and project
        out = out.transpose(1, 2).reshape(B, T, D)
        return self.W_o(out)
```

Each head learns to attend to different aspects: some track syntactic dependencies (subject-verb agreement), others track semantic similarity, others positional proximity. The output projection W_o combines all heads' information.

---

## Q2: How does backpropagation work, and what are the practical failure modes?

**Answer:**

Backpropagation is the application of the chain rule to compute gradients of a scalar loss L with respect to all parameters in a computation graph.

**Forward pass:** Compute activations layer by layer, caching intermediate values.
**Backward pass:** Apply chain rule from output to input:
```
∂L/∂W_i = ∂L/∂a_i * ∂a_i/∂W_i
```

```python
# Simplified manual backprop for a 2-layer network
import numpy as np

def forward(X, W1, b1, W2, b2):
    z1 = X @ W1 + b1
    a1 = np.maximum(0, z1)   # ReLU
    z2 = a1 @ W2 + b2
    return z1, a1, z2

def backward(X, z1, a1, z2, y, W2):
    N = X.shape[0]
    # Softmax + cross-entropy gradient (combined for numerical stability)
    probs = np.exp(z2) / np.exp(z2).sum(axis=1, keepdims=True)
    dz2 = probs.copy()
    dz2[range(N), y] -= 1
    dz2 /= N

    dW2 = a1.T @ dz2
    db2 = dz2.sum(axis=0)
    da1 = dz2 @ W2.T
    # ReLU backward: gradient is zero where forward input was <= 0
    dz1 = da1 * (z1 > 0)
    dW1 = X.T @ dz1
    db1 = dz1.sum(axis=0)
    return dW1, db1, dW2, db2
```

**Practical failure modes:**

1. **Vanishing gradients:** Sigmoid/tanh saturate at extremes → gradient ≈ 0 → early layers don't update. Fix: ReLU, residual connections, careful init.

2. **Exploding gradients:** Gradients grow exponentially in deep/RNN networks. Fix: gradient clipping.
```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

3. **Dead ReLUs:** Neurons stuck at zero output permanently. Fix: Leaky ReLU, He initialization, lower learning rate.

4. **Numerical instability:** Computing `log(softmax(x))` directly overflows. Always use `F.log_softmax` + `F.nll_loss` or `F.cross_entropy`.

---

## Q3: Adam vs AdamW — what's actually different and when does it matter?

**Answer:**

Adam maintains per-parameter first and second moment estimates of gradients to adapt the learning rate:
```
m_t = β1 * m_{t-1} + (1-β1) * g_t          # 1st moment (momentum)
v_t = β2 * v_{t-1} + (1-β2) * g_t^2         # 2nd moment (RMS)
m̂_t = m_t / (1 - β1^t)                       # Bias correction
v̂_t = v_t / (1 - β2^t)                       # Bias correction
θ_t = θ_{t-1} - α * m̂_t / (sqrt(v̂_t) + ε)  # Update
```

**The L2 vs AdamW problem:**

When you add L2 regularization to Adam (`weight_decay` parameter in PyTorch's Adam), the weight decay gradient `λ*θ` is added to `g_t` BEFORE computing moments:
```
g_t_effective = g_t + λ * θ   # L2 on Adam
# This decay gets scaled by 1/sqrt(v̂_t)
# Parameters with small gradients get LESS regularization (wrong!)
```

AdamW applies weight decay AFTER the gradient update, decoupled from the adaptive scaling:
```
θ_t = θ_{t-1} - α * m̂_t / (sqrt(v̂_t) + ε) - α * λ * θ_{t-1}
# Weight decay is always α*λ, independent of gradient history
```

```python
# DON'T do this (Adam + L2 is not weight decay):
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3, weight_decay=1e-2)

# DO this (AdamW = proper weight decay):
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-2)

# For transformers, also use warmup + cosine decay:
from transformers import get_cosine_schedule_with_warmup
scheduler = get_cosine_schedule_with_warmup(
    optimizer,
    num_warmup_steps=500,
    num_training_steps=total_steps
)
```

**When it matters:** For large language models and transformers, AdamW consistently outperforms Adam+L2. The practical difference is measurable — on BERT fine-tuning tasks, AdamW typically gives 0.5–1.5% better downstream performance on GLUE benchmarks.

---

## Q4: Overfitting diagnosis and remediation at production scale

**Answer:**

Overfitting manifests when training loss diverges from validation loss, but diagnosing the root cause requires systematic investigation.

**Diagnosis framework:**
```python
import numpy as np
import matplotlib.pyplot as plt

def diagnose_model(train_losses, val_losses, train_accs, val_accs):
    """Diagnose training pathology from loss/accuracy curves."""
    final_train_acc = train_accs[-1]
    final_val_acc = val_accs[-1]
    generalization_gap = final_train_acc - final_val_acc

    # Epoch where val loss starts increasing
    val_losses_arr = np.array(val_losses)
    overfit_epoch = np.argmin(val_losses_arr)

    diagnostics = {
        "generalization_gap": generalization_gap,
        "best_val_epoch": overfit_epoch,
        "total_epochs": len(val_losses),
        "overfit_fraction": overfit_epoch / len(val_losses),
    }

    if generalization_gap > 0.15:
        if final_train_acc > 0.98:
            print("HIGH variance (classic overfitting): reduce model capacity, add dropout/L2, get more data")
        else:
            print("Check for label noise or data pipeline bugs")
    elif final_train_acc < 0.70 and final_val_acc < 0.70:
        print("HIGH bias (underfitting): increase model capacity, train longer, check learning rate")
    elif generalization_gap < 0.05:
        print("Well-fitted: monitor calibration and distribution shift")

    return diagnostics
```

**Remediation hierarchy (in order of trying):**

1. **Data augmentation** — cheapest fix for overfitting
2. **Early stopping** — checkpoint at best validation loss
3. **Dropout** — tune rate per layer
4. **L2 regularization / weight decay** — via AdamW weight_decay
5. **Reduce model capacity** — fewer layers, smaller hidden dim
6. **Get more data** — if budget allows
7. **Data cleaning** — label noise is frequently the root cause at scale

**What distinguishes senior engineers:** recognizing that a 38-point train/val gap might be caused by a data pipeline bug (e.g., augmentation applied only to training set leaking information, or test set contamination) rather than model pathology. Always check the data pipeline first.

---

## Q5: Evaluation metrics — knowing which metric to trust and when

**Answer:**

**Classification metrics:**

```python
from sklearn.metrics import (
    precision_recall_fscore_support,
    roc_auc_score,
    average_precision_score,
    confusion_matrix
)
import numpy as np

def comprehensive_eval(y_true, y_pred_proba, threshold=0.5):
    y_pred = (y_pred_proba >= threshold).astype(int)
    p, r, f1, _ = precision_recall_fscore_support(y_true, y_pred, average='binary')

    metrics = {
        "precision": p,
        "recall": r,
        "f1": f1,
        # AUC-ROC: threshold-independent, but misleading with class imbalance
        "auc_roc": roc_auc_score(y_true, y_pred_proba),
        # AUC-PR: better for imbalanced classes
        "auc_pr": average_precision_score(y_true, y_pred_proba),
    }
    return metrics

# For imbalanced classes (fraud, medical):
# - AUC-PR is more informative than AUC-ROC
# - ROC looks good even if model ignores minority class
```

**NLP generation metrics:**

| Metric | What it measures | Failure mode |
|--------|-----------------|--------------|
| BLEU | n-gram precision vs reference | Penalizes valid synonyms, favors brevity |
| ROUGE-L | Longest common subsequence recall | Misses paraphrases |
| BERTScore | Semantic similarity via embeddings | Biased toward BERT vocabulary |
| Perplexity | Average prediction confidence | Not correlated with downstream task quality |
| RAGAS | RAG-specific: faithfulness, relevance | Requires LLM as judge (meta-evaluation) |

**Key principle for interviews:** Always state the metric's *failure mode* alongside its definition. "We use F1 because it balances precision and recall, but note that for our fraud detection use case with 0.1% positive rate, we actually monitor AUC-PR as the primary metric and F1 at our operating threshold as secondary."

---

## Q6: Transfer learning and fine-tuning strategy at production scale

**Answer:**

Transfer learning is not one technique — it's a spectrum of strategies with different compute/data/performance tradeoffs:

```
Frozen backbone + new head     (fast, cheap, lower ceiling)
         |
Layer-wise unfreezing          (progressive fine-tuning)
         |
Full fine-tuning               (expensive, high ceiling, needs large dataset)
         |
LoRA / QLoRA                   (parameter-efficient, modern standard)
```

**Implementation with layer-wise learning rates (prevents catastrophic forgetting):**

```python
from transformers import AutoModel
import torch

def fine_tune_with_layerwise_lr(model_name, num_labels, train_loader, val_loader):
    model = AutoModel.from_pretrained(model_name)
    head = torch.nn.Linear(model.config.hidden_size, num_labels)

    # Group parameters by layer depth
    optimizer_params = []
    base_lr = 2e-5
    decay_factor = 0.9

    # Embeddings: lowest LR (most general features, preserve them)
    optimizer_params.append({
        "params": model.embeddings.parameters(),
        "lr": base_lr * (decay_factor ** 12)
    })

    # Transformer layers: gradually increasing LR
    for i, layer in enumerate(model.encoder.layer):
        optimizer_params.append({
            "params": layer.parameters(),
            "lr": base_lr * (decay_factor ** (12 - i))
        })

    # Classification head: highest LR (randomly initialized)
    optimizer_params.append({
        "params": head.parameters(),
        "lr": base_lr * 10  # 10x higher for new head
    })

    optimizer = torch.optim.AdamW(optimizer_params, weight_decay=0.01)
    return optimizer
```

**When to use LoRA instead of full fine-tuning:**
- Dataset < 50K examples: LoRA wins (less overfitting risk)
- Memory constrained: LoRA uses 10–100x less memory
- Multiple domain adaptations needed: swap LoRA adapters, keep base frozen

```python
from peft import LoraConfig, get_peft_model, TaskType

config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,               # Rank: tradeoff between params and expressiveness
    lora_alpha=32,      # Scale: lora_alpha/r = effective learning rate scaling
    lora_dropout=0.1,
    target_modules=["q_proj", "v_proj"],  # Only attention projection matrices
    bias="none"
)
peft_model = get_peft_model(base_model, config)
peft_model.print_trainable_parameters()
# trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.062
```

**Interview insight:** A senior engineer knows that LoRA works because the weight update matrix ΔW can be decomposed as BA where B is (d×r) and A is (r×d), with r << d. The hypothesis is that the task-specific adaptation lives in a low-rank subspace of the full weight space — which empirically holds for most fine-tuning scenarios.
