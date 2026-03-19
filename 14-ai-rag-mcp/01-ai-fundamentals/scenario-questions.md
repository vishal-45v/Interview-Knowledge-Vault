# Chapter 01: AI Fundamentals — Scenario Questions

Real-world engineering scenarios. Answer as a senior engineer making architectural and implementation decisions. Consider tradeoffs, failure modes, and production constraints — not just the happy path.

---

1. Your team is building a content moderation system for a social platform with 50M daily active users. You have 10,000 labeled examples of policy violations across 8 categories (hate speech, NSFW, spam, harassment, etc.) but the categories are severely imbalanced — spam is 60% of examples, some categories have fewer than 200 samples. You have budget for one GPU for training and must serve predictions under 50ms. How do you approach model selection, training strategy, loss function choice, and threshold calibration for each category?

2. You are the ML lead at a fintech company. You've trained a loan default prediction model that achieves 94% accuracy on your holdout set. Six months after deployment, the model's business performance (actual loan recovery rates) has degraded significantly despite the model's reported accuracy staying stable. What do you investigate first, and how do you instrument your system to catch this earlier in the future?

3. Your team is fine-tuning a pretrained transformer for a specialized legal document classification task. After three epochs, the model achieves excellent performance on your validation set. When you run it on actual court filings from a different jurisdiction than your training data, quality drops sharply. Walk through your debugging process and remediation strategy, considering both data and architectural interventions.

4. You are building a real-time recommendation system. The business wants to use a deep neural network to replace the existing matrix factorization approach. The DNN achieves better offline metrics (NDCG, MRR) in backtesting but you are worried about online performance. Design your A/B testing strategy, specify what metrics you will monitor, and describe the conditions under which you would roll back.

5. Your organization wants to train a proprietary embedding model for internal knowledge retrieval. You have 2TB of internal documents but no labeled query-document pairs. Describe your complete training strategy: what self-supervised objective(s) you would use, how you would construct training pairs, what evaluation framework you would use before moving to production, and how you would handle domain-specific vocabulary that is unlikely to be covered by existing pretrained models.

6. A model in production is exhibiting strange behavior: it performs well on most inputs but catastrophically fails on a specific pattern of inputs that your QA team discovered. You suspect a data issue in training. The model was trained on a dataset you inherited from a third-party vendor and you don't have direct visibility into all the training examples. How do you investigate the root cause, and what is your mitigation strategy if you cannot retrain from scratch?

7. Your team is evaluating whether to use a contrastive learning approach (like CLIP-style training) versus a supervised approach for a multi-modal document understanding system that pairs scanned pages with their text content. You have 500,000 document-text pairs that are weakly labeled (OCR output, not human verified). Make the case for and against each approach, and describe how you would run the evaluation to decide.

8. You are asked to reduce the inference latency of a transformer-based NLP model that is currently serving at 300ms P99. The business requires 100ms P99. You cannot change the model architecture. Describe in order of priority the techniques you would apply, what tradeoffs each introduces, and how you would measure whether the quality degradation from optimization is acceptable.

9. Your team is building an anomaly detection system for network security. You have abundant normal traffic logs but very few confirmed attack samples (class imbalance of roughly 10,000:1). A data scientist proposes training a supervised classifier. A second proposes using an autoencoder-based unsupervised approach. Evaluate both proposals rigorously, including failure modes specific to the security domain, and recommend a hybrid strategy.

10. You are inheriting a production ML system from a team that has left the company. The model is a gradient-boosted tree ensemble for predicting customer churn. There is minimal documentation. You notice the model was trained 18 months ago and has never been retrained. The feature engineering code references several database columns that no longer exist (they are being backfilled with zeros). Business stakeholders say the model "feels worse" but no formal metrics have degraded. Design your complete audit and remediation process, including how you decide whether to retrain or replace the model entirely.
