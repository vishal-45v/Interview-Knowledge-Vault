# Chapter 01: AI Fundamentals — Follow-Up Traps

Tricky follow-up questions interviewers use after initial answers. Each entry shows the trap (what naive candidates say) and the correct nuanced answer a senior engineer gives.

---

## Trap 1: "ReLU solves the vanishing gradient problem, right?"

**What most people say:** Yes, ReLU has a constant gradient of 1 for positive inputs, so gradients don't vanish.

**Correct answer:** ReLU solves vanishing gradients for active neurons but introduces the "dying ReLU" problem: if a neuron's pre-activation is negative for all training examples (e.g., due to a large negative bias or a bad weight update), its gradient is exactly zero and it never recovers. This is a form of permanent gradient death, not vanishing — it's worse because it's irreversible. Leaky ReLU, ELU, and GELU address this. Also, ReLU networks can still exhibit vanishing gradients in the backward pass through other layers if activations are sparse. The fix isn't "use ReLU" — it's residual connections (skip connections in ResNets), careful weight initialization (He initialization for ReLU), and gradient clipping.

```python
import torch
import torch.nn as nn

# Dying ReLU demonstration
layer = nn.Linear(10, 10)
# Force large negative bias
with torch.no_grad():
    layer.bias.fill_(-10.0)

x = torch.randn(100, 10)
out = torch.relu(layer(x))
print(f"Dead neurons: {(out.sum(0) == 0).sum().item()}/10")
# Output: Dead neurons: 10/10 — all neurons are dead
```

---

## Trap 2: "Adam is always better than SGD for training neural networks"

**What most people say:** Adam converges faster and handles different learning rates per parameter, so it's strictly better.

**Correct answer:** Adam converges faster in wall-clock time but often generalizes worse than SGD with momentum, especially on vision tasks. The reason is that Adam's adaptive learning rates can cause it to find sharp minima in the loss landscape — minima that are narrow and don't generalize well. SGD's noisier updates tend to escape sharp minima and find flatter minima which generalize better (the sharpness-generalization hypothesis). This is why ResNets and many vision models still use SGD + momentum with learning rate scheduling. AdamW partially addresses this by decoupling weight decay from the gradient-adaptive scaling, which prevents the adaptive scaling from undermining regularization.

```python
# Adam with L2 is NOT the same as AdamW
# In Adam + L2: the weight decay gets scaled by the adaptive lr estimates
# In AdamW: weight decay is applied directly, independent of gradient history

optimizer_adam_l2 = torch.optim.Adam(model.parameters(), lr=1e-3, weight_decay=1e-4)
# Effective decay = weight_decay / sqrt(v_t) — varies per parameter

optimizer_adamw = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-4)
# Effective decay = weight_decay * lr — constant, as intended
```

---

## Trap 3: "Dropout rate of 0.5 means keep 50% of neurons"

**What most people say:** Yes, and you should always use 0.5 for hidden layers.

**Correct answer:** Keep probability p=0.5 means 50% of neurons are dropped. But "always use 0.5" is wrong — dropout rate should be tuned per layer. Typically 0.1–0.3 for convolutional layers, 0.5 for large fully-connected layers, and often 0.1 for the input layer. More importantly: at inference, you must scale activations by p OR use inverted dropout (multiply by 1/p during training). PyTorch uses inverted dropout by default. Forgetting to set `model.eval()` during inference means dropout is still active, producing stochastic outputs every call — a real production bug.

```python
model.train()   # Dropout active: output is stochastic
out1 = model(x)
out2 = model(x)
assert not torch.allclose(out1, out2)  # Different each call

model.eval()    # Dropout disabled: output is deterministic
out3 = model(x)
out4 = model(x)
assert torch.allclose(out3, out4)  # Same each call

# THE BUG: calling model.train() before inference in a Flask endpoint
# Every request gets different output — intermittent failures, not reproducible
```

---

## Trap 4: "Higher BLEU score means better translation"

**What most people say:** BLEU is the standard metric for translation so higher is better.

**Correct answer:** BLEU correlates poorly with human judgment in several documented cases: (1) BLEU does not account for synonymy — "automobile" vs "car" gets zero credit even though both are correct. (2) BLEU penalizes short outputs via a brevity penalty but not long outputs that pad. (3) BLEU treats all n-gram matches equally regardless of meaning. (4) A model can get high BLEU by producing safe, generic translations that miss nuance. ROUGE (Recall-Oriented Understudy for Gisting Evaluation) is better for summarization because it's recall-oriented. For modern evaluation, BERTScore (semantic similarity via embeddings) or COMET (trained on human judgments) correlate better with human preferences. In production, track BLEU for regression, but evaluate quality on a held-out human-rated test set.

---

## Trap 5: "Batch normalization makes training faster — just add it everywhere"

**What most people say:** BN normalizes activations, stabilizes training, and acts as regularization, so add it after every layer.

**Correct answer:** BN has critical failure modes: (1) With batch size 1 or very small batches (common in NLP sequence models), the batch statistics are too noisy to be useful. (2) BN breaks the independence between samples in a batch — the prediction for sample A depends on samples B, C, D in the same batch. This is problematic for tasks requiring deterministic per-sample predictions. (3) In autoregressive generation, you cannot batch across sequence positions. (4) BN running statistics must be synced across GPUs in distributed training (SyncBN) — forgetting this causes subtle divergence. Layer Norm (normalizes across features for a single sample) avoids all these issues, which is why transformers use it exclusively.

```python
# BN: statistics computed ACROSS batch, PER feature channel
# Shape: (batch, features) -> normalize over batch dimension
bn = nn.BatchNorm1d(256)  # Running mean/var updated during training

# LayerNorm: statistics computed ACROSS features, PER sample
# Shape: (batch, seq_len, features) -> normalize over feature dimension
ln = nn.LayerNorm(256)  # No running statistics needed

# The RMS variant used in LLaMA/Mistral (RMSNorm):
class RMSNorm(nn.Module):
    def __init__(self, dim, eps=1e-6):
        super().__init__()
        self.weight = nn.Parameter(torch.ones(dim))
        self.eps = eps

    def forward(self, x):
        rms = x.pow(2).mean(-1, keepdim=True).add(self.eps).sqrt()
        return x / rms * self.weight
```

---

## Trap 6: "Attention is O(n^2) — that's the bottleneck"

**What most people say:** Yes, attention scales quadratically with sequence length, which is the fundamental limitation.

**Correct answer:** The O(n^2) is specifically in the attention matrix computation and memory, not necessarily the dominant bottleneck in practice. For sequences up to ~4K tokens, the FFN layers are often more expensive in FLOPs. The quadratic problem is primarily memory — storing the n×n attention matrix. Flash Attention addresses this not by reducing FLOPs but by using tiled computation to avoid materializing the full n×n matrix in HBM (GPU high-bandwidth memory), achieving the same result with O(n) memory. Sparse attention (Longformer, BigBird), linear attention approximations, and state space models (Mamba/S4) are architectural responses. But Flash Attention is purely an implementation optimization — it doesn't change the math, just the memory access pattern.

---

## Trap 7: "Perplexity of 20 means the model is confused between 20 tokens at each step"

**What most people say:** Correct — perplexity is the average branching factor.

**Correct answer:** That interpretation is technically correct but incomplete in a way that traps people. The key caveat: perplexity is computed on a specific evaluation corpus, and the vocabulary matters. A model with perplexity 20 on Wikipedia English is not comparable to a model with perplexity 20 on legal documents or Python code — the underlying token distributions are different. More critically: two models with the same perplexity on the same corpus can perform very differently on downstream tasks because perplexity measures average-case prediction, not worst-case or task-specific prediction. A model that is very good at predicting function words ("the", "a", "is") but bad at content words will have low perplexity but poor task performance.

```python
import torch
from torch.nn import functional as F

def perplexity(model, tokenized_text, device):
    """Compute perplexity: exp(-1/N * sum(log P(token_i | context)))"""
    input_ids = tokenized_text.to(device)
    with torch.no_grad():
        outputs = model(input_ids, labels=input_ids)
        # outputs.loss is the mean NLL across tokens
        nll = outputs.loss
    return torch.exp(nll).item()

# Perplexity = exp(cross_entropy_loss)
# CE loss = -1/N * sum(log p(x_i | x_<i))
# PPL = exp(-1/N * sum(log p(x_i | x_<i)))
# Lower PPL = better (model is less "surprised" by the text)
```

---

## Trap 8: "Transfer learning always works — pretrain on large corpus, fine-tune on small dataset"

**What most people say:** Yes, this is the standard modern approach.

**Correct answer:** Transfer learning can fail in well-documented ways: (1) Negative transfer — if the pretraining domain is too different (e.g., using ImageNet-pretrained features for medical histology), the pretrained features actively harm performance vs training from scratch. (2) Catastrophic forgetting — if you fine-tune all layers aggressively on a small dataset, the model forgets generalizable features and overfits. The fix is layer-wise learning rate decay (lower LR for early layers) or freezing early layers. (3) Label leakage through benchmark contamination — if your fine-tuning data overlaps with pretraining data, evaluation is invalid. (4) For very long context documents, pretrained models that saw only 512-token sequences during pretraining genuinely cannot transfer position-dependent features beyond that length.

```python
from transformers import AutoModel
import torch

model = AutoModel.from_pretrained("bert-base-uncased")

# Layer-wise learning rate decay for fine-tuning
# Prevents catastrophic forgetting of lower-level features
def get_layerwise_params(model, base_lr=2e-5, decay=0.9):
    param_groups = []
    layers = [model.embeddings] + list(model.encoder.layer)

    for i, layer in enumerate(layers):
        lr = base_lr * (decay ** (len(layers) - 1 - i))
        param_groups.append({"params": layer.parameters(), "lr": lr})

    return param_groups

# Layer 0 (embeddings): lr = 2e-5 * 0.9^11 ≈ 6.3e-6  (conservative)
# Layer 11 (top):       lr = 2e-5 * 0.9^0  = 2e-5    (aggressive)
```

---

## Trap 9: "L1 regularization always gives sparse weights"

**What most people say:** Yes, L1 drives weights to exactly zero, providing automatic feature selection.

**Correct answer:** L1 gives sparse solutions with gradient-based optimization only when the global optimum is truly at zero for some weights. In practice with mini-batch SGD, weights oscillate around zero rather than reaching it exactly. True sparsity with L1 is only guaranteed with coordinate descent (like in scikit-learn's Lasso). With neural networks using Adam, the adaptive learning rates can prevent some weights from reaching exactly zero. Furthermore, L1 on neural network weights is less principled than on linear model features because neural network weights don't have a direct feature-importance interpretation. For neural networks, structured pruning (zeroing entire neurons/attention heads based on magnitude or gradient) is more practical than L1-induced sparsity.

---

## Trap 10: "A model with high precision and high recall is always the best model"

**What most people say:** Yes, high precision and recall means the model is performing well on both sides.

**Correct answer:** This ignores calibration. A model can have high precision and recall on the test set but be poorly calibrated — meaning the predicted probabilities don't reflect actual likelihoods. For example, a model that outputs 0.9 probability for 60% of positives and 0.1 for 60% of negatives can have excellent precision/recall at a 0.5 threshold but be useless for decision-making that depends on probability magnitudes. Calibration is measured with ECE (Expected Calibration Error) or reliability diagrams. Critical in medical diagnosis, fraud detection, and any system where a downstream decision depends on the probability score, not just the binary classification. Post-hoc calibration with Platt scaling or isotonic regression is often applied after training.

```python
from sklearn.calibration import calibration_curve
import matplotlib.pyplot as plt

def plot_calibration(y_true, y_prob, n_bins=10):
    """Reliability diagram: perfectly calibrated = diagonal line."""
    fraction_of_positives, mean_predicted_value = calibration_curve(
        y_true, y_prob, n_bins=n_bins
    )
    # Perfect calibration: fraction_of_positives == mean_predicted_value
    # ECE = mean(|fraction_pos - mean_pred| * bin_weight)

    ece = sum(
        abs(fo - mp) * (len(y_true) // n_bins) / len(y_true)
        for fo, mp in zip(fraction_of_positives, mean_predicted_value)
    )
    return ece  # Lower is better; 0.0 = perfect calibration

# Platt scaling: fit a logistic regression on top of raw model scores
from sklearn.linear_model import LogisticRegression
calibrator = LogisticRegression()
calibrator.fit(raw_scores_val.reshape(-1, 1), y_val)
calibrated_probs = calibrator.predict_proba(raw_scores_test.reshape(-1, 1))[:, 1]
```

---

## Trap 11: "The attention mechanism lets the model look at all positions equally"

**What most people say:** Yes, that's the whole point — it can attend to any position.

**Correct answer:** The attention mechanism computes a weighted sum over all positions, but the weights are determined by the query-key similarity — they are not uniform. Critically, in a decoder with causal masking, future positions are masked to -infinity before softmax, ensuring autoregressive generation cannot "see the future." Furthermore, the choice of what to attend to is learned, not given — early in training, attention patterns are approximately uniform (the model hasn't learned to focus), which is why warmup scheduling is important. Multi-head attention learns different attention patterns per head: some heads learn syntactic dependencies, others semantic, others positional proximity. Visualizing attention weights with tools like BertViz reveals this specialization.
