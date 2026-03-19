# Chapter 01: AI Fundamentals — Theory Questions

Senior-level questions covering ML paradigms, neural network internals, optimization, transformers, and evaluation. These questions test depth of understanding, not surface-level recall.

---

1. Explain the difference between supervised, unsupervised, and reinforcement learning. Give a concrete example of a production system where the wrong paradigm choice would cause a fundamental failure — not just a performance issue.

2. In backpropagation, why does the vanishing gradient problem specifically affect deep networks trained with sigmoid activations but not those using ReLU? What is the mathematical reason, and what failure mode does it produce during training?

3. Cross-entropy loss and MSE loss are both used in classification. Under what exact conditions is MSE a poor choice for a classification problem even if you treat the outputs as probabilities? Where does the gradient landscape break down?

4. You have a model with 99% training accuracy and 61% validation accuracy. Walk through your complete diagnostic process — what are the possible root causes and how do you distinguish between them programmatically?

5. Explain L1 vs L2 regularization. Beyond the "L1 induces sparsity" heuristic: what is the geometric reason L1 produces sparse solutions and L2 does not? How does this affect feature selection in practice?

6. Dropout is described as an ensemble method. Explain precisely what ensemble it is approximating, and why this approximation holds. Does dropout behave differently at inference time versus training time, and what bug does forgetting this cause?

7. Batch normalization operates differently during training and inference. Explain exactly what statistics it uses in each phase, where those statistics come from, and what happens to model predictions if you accidentally leave BN in training mode during inference on a production model serving small batch sizes.

8. Compare SGD with momentum, Adam, and AdamW. Why does Adam sometimes generalize worse than SGD despite converging faster? What is the theoretical explanation for AdamW's advantage over Adam + L2 regularization?

9. Explain the scaled dot-product attention formula: Attention(Q,K,V) = softmax(QK^T / sqrt(d_k)) * V. Why is the scaling by sqrt(d_k) necessary? What specific failure mode occurs at large d_k without it?

10. Multi-head attention runs h attention heads in parallel. Why is this not simply redundant computation? What does each head theoretically learn, and how do the heads' outputs get combined?

11. Positional encoding in the original transformer uses sinusoidal functions. Explain why this specific choice was made over learned positional embeddings. What property of sinusoidal encodings allows the model to generalize to sequence lengths not seen during training?

12. Word2Vec produces static embeddings. BERT produces contextual embeddings. Explain the architectural difference that produces contextual representations, and give an example where a static embedding of the word "bank" would cause a downstream system to fail in a way that a contextual embedding would not.

13. Explain contrastive loss (as used in SimCLR/CLIP). How does it differ structurally from cross-entropy, and why is it better suited for learning general-purpose representations without labeled data?

14. Transfer learning from a pretrained model assumes distribution overlap between pretraining and fine-tuning data. Describe three scenarios where this assumption breaks down badly, and what symptoms appear in each case.

15. Explain perplexity as an evaluation metric for language models. What is it measuring geometrically/informationally? Why can two models with similar perplexity scores produce dramatically different quality outputs on downstream tasks?

16. Precision and recall trade off against each other. In a medical diagnosis system, you deliberately tune your threshold to maximize recall at the cost of precision dropping to 40%. A stakeholder asks you to also report F1. Explain why F1 is potentially misleading here and what metric you should actually report in its place.

17. The BLEU score is commonly used for machine translation evaluation. Describe two fundamental weaknesses of BLEU that make it a poor proxy for human judgment, and explain what ROUGE measures differently. When would you use one over the other?

18. Explain the relationship between entropy, cross-entropy, and KL divergence. In the context of training a neural network, what exactly is the cross-entropy loss minimizing in information-theoretic terms?

19. Layer normalization is used in transformers instead of batch normalization. What is the specific reason for this choice? Under what training conditions does batch normalization fail in sequence models where layer normalization does not?

20. Describe the encoder-decoder transformer architecture. Why do decoder layers have two attention sub-layers (self-attention and cross-attention) while encoder layers only have one? What does the cross-attention mechanism do, and how does it connect encoder representations to decoder generation?
