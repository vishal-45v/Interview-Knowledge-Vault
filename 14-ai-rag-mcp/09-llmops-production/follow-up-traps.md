# Chapter 09: LLMOps & Production — Follow-Up Traps

## Trap 1: "LLM-as-judge is objective and unbiased"

**What most people say:** "Using GPT-4 as the evaluator is objective because it's an AI, not a human with biases."

**Correct answer:** LLM-as-judge has several well-documented biases: (1) **Self-preference bias** — GPT-4 rates GPT-4 outputs higher than equivalent outputs from Claude; (2) **Verbosity bias** — longer answers are rated higher even when a shorter answer is more accurate; (3) **Position bias** — in A/B comparisons, the first presented answer is rated higher; (4) **Sycophancy** — if you phrase the question "this answer is great, right?" the judge agrees; (5) **Threshold sensitivity** — the judge's calibration depends heavily on the prompt and temperature. Mitigations: use the judge model for relative comparisons (A vs. B) rather than absolute scores; calibrate against human labels; use multiple judges and take majority vote; run A/B in both orders and average. LLM-as-judge is still the best scalable option, but not because it's objective — because calibrated-LLM-as-judge correlates better with human judgment than BLEU/ROUGE.

---

## Trap 2: "Prompt caching works automatically and saves money"

**What most people say:** "I enabled prompt caching and my costs dropped 50% automatically."

**Correct answer:** Prompt caching requires deliberate prompt structure to maximize cache hits. Anthropic's caching (as of 2024) requires: (1) the cached prefix must be at least 1,024 tokens; (2) you must explicitly mark cache breakpoints with `cache_control: {"type": "ephemeral"}`; (3) the cached portion must be IDENTICAL across requests — even a single character difference busts the cache; (4) the cache TTL is 5 minutes for Claude (you must re-warm within 5 minutes). OpenAI's prompt caching (GPT-4o and later) is automatic but only applies to the system prompt and is context-sensitive. The practical implication: structure your prompts with the large, stable system context first (documents, instructions), and the variable user query at the end. If you put the document in the middle and the instructions vary, you get 0% cache hits.

```python
# WRONG structure for caching (instructions vary between calls):
prompt = f"Answer in {language}. Context: {large_document}. Question: {query}"
#            ^ varies       ^ stable (cacheable)  ^ varies
# Cache miss every time because the prefix changes

# CORRECT structure for caching:
messages = [
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": f"Context: {large_document}\n\nInstructions: Answer in English.",
                "cache_control": {"type": "ephemeral"}  # Mark stable prefix
            },
            {
                "type": "text",
                "text": f"Question: {query}"  # Variable suffix, not cached
            }
        ]
    }
]
```

---

## Trap 3: "LoRA adds parameters on top of the original model"

**What most people say:** "LoRA adds new layers to the model for fine-tuning, leaving original weights intact."

**Correct answer:** LoRA does NOT add new layers — it decomposes the weight UPDATE matrix into two low-rank matrices. For a weight matrix W of shape (d, d), instead of learning a full update ΔW (d×d parameters), LoRA learns two matrices A (d×r) and B (r×d) where r << d. The effective update is ΔW = BA. During inference, you merge: W_effective = W_original + BA — the merged model has the SAME size as the original. The memory savings come during TRAINING: instead of computing gradients for all d² parameters, you compute gradients only for r×d + d×r = 2rd parameters. For d=4096 and r=16: full fine-tuning needs 16M trainable parameters; LoRA needs 131K — a 122x reduction in trainable parameters. This enables fine-tuning on consumer GPUs.

```python
# LoRA math:
# d = 4096 (hidden dim), r = 16 (rank)
# Full fine-tuning: 4096 × 4096 = 16,777,216 parameters per weight matrix
# LoRA:             4096 × 16 + 16 × 4096 = 131,072 parameters
# Reduction: 128x fewer trainable parameters

from peft import LoraConfig, get_peft_model
config = LoraConfig(
    r=16,           # rank of the decomposition
    lora_alpha=32,  # scaling factor (alpha/r = effective learning rate scaling)
    target_modules=["q_proj", "v_proj"],  # which weight matrices to adapt
    lora_dropout=0.1,
    bias="none",
    task_type="CAUSAL_LM"
)
model = get_peft_model(base_model, config)
model.print_trainable_parameters()
# Output: trainable params: 2,621,440 || all params: 6,738,415,616 || trainable%: 0.04
```

---

## Trap 4: "Higher temperature = more creative, lower temperature = more deterministic"

**What most people say:** "For factual tasks, use temperature=0; for creative tasks, use temperature=1."

**Correct answer:** This is approximately correct but misses important nuances. Temperature 0 is NOT truly deterministic in most implementations — it selects the argmax token but floating-point non-determinism across GPU runs means you can get different results. Temperature controls the sharpness of the probability distribution over next tokens, NOT the factual accuracy of the output. You can get confidently wrong answers at temperature=0 (the model is certain but wrong). For factual RAG tasks, the right approach is not just low temperature — it's low temperature + a system prompt that instructs the model to say "I don't know" when uncertain + faithfulness checking. Also: for structured output (JSON extraction), temperature=0 is important for schema compliance; for creative tasks, temperature around 0.7-0.9 usually works better than 1.0 which can become incoherent.

---

## Trap 5: "Semantic caching is safe for all query types"

**What most people say:** "Semantic caching embeds the query, finds similar cached queries, and returns cached responses — it's always safe."

**Correct answer:** Semantic caching is UNSAFE for several query types: (1) **Time-sensitive queries** — "What is the current Bitcoin price?" should never be cached beyond 30 seconds; (2) **User-specific queries** — "What are my account details?" — a similar query from another user would return a different user's data; (3) **Transactional queries** — "Did my payment go through?" — the state changes between queries; (4) **Sensitive context queries** — "Based on the document I just uploaded, what..." — the document content changes the answer even if the question text is similar. The cache key must include: user_id (for personalized queries), context hash (if context was provided), and a timestamp bucket (for time-sensitive topics). The embedding similarity threshold must be very high (>0.98) to avoid false cache hits.

---

## Trap 6: "Guardrails AI validates outputs by running rules on the LLM's text"

**What most people say:** "Guardrails AI applies regex and keyword filters to LLM output to catch bad responses."

**Correct answer:** Guardrails AI does much more than regex: it supports a pipeline of validators that can include LLM-based checks, ML classifiers, regex, semantic similarity, and custom Python validators. More importantly, Guardrails AI supports *output correction* — if a validator fails, it can automatically re-prompt the LLM with instructions to fix the issue, rather than just blocking the output. The RAIL (Reliable AI Markup Language) spec defines the expected output structure, and Guardrails will attempt to parse, validate, and fix the output. The critical nuance: output validation via another LLM call doubles your latency and cost. In production, only run expensive validation on high-stakes outputs. Use cheap validators (regex, schema checking) first; LLM-based validators only when necessary.

```python
from guardrails import Guard
from guardrails.hub import ToxicLanguage, ValidJson

guard = Guard().use_many(
    ToxicLanguage(threshold=0.5, validation_method="sentence"),
    ValidJson(),
)

result = guard(
    openai.chat.completions.create,
    model="gpt-4o",
    messages=[{"role": "user", "content": "Generate a JSON user profile"}],
    max_retries=2,  # Guardrails will re-prompt up to 2 times on failure
)
```

---

## Trap 7: "You can always roll back a fine-tuned model"

**What most people say:** "If the fine-tuned model performs badly, just roll back to the base model."

**Correct answer:** Rolling back is straightforward technically, but the scenario where you CAN'T easily roll back is when you've also changed your data pipeline, prompt structure, or evaluation criteria to match the fine-tuned model's outputs. If your system has dependencies on the fine-tuned model's output format (e.g., it outputs structured JSON with specific keys that downstream services parse), rolling back to the base model breaks downstream systems. The correct deployment strategy: (1) Keep the base model serving traffic; (2) Shadow-deploy the fine-tuned model (receives same requests, responses NOT shown to users); (3) Compare shadow vs. base on automated metrics; (4) Gradually shift traffic (1% → 10% → 50% → 100%) with canary rollouts; (5) Never change the API contract without versioning. This way, rollback is truly reversible.

---

## Trap 8: "Nightly evaluation on 500 test cases is sufficient for regression testing"

**What most people say:** "Running 500 automated test cases nightly covers regression adequately."

**Correct answer:** 500 test cases can miss regressions for several reasons: (1) **Distribution shift** — if your golden dataset doesn't match current production query distribution, regressions in new query types go undetected; (2) **Edge case blindness** — regressions often hit tail cases that aren't in your golden set; (3) **Metric lag** — a 5% regression in faithfulness might not be statistically significant with 500 tests (you need ~600 tests to detect a 5% change at 95% confidence, but this depends on variance); (4) **Slow drift** — nightly evaluation misses intra-day regressions from model provider updates. Supplement nightly golden-set evaluation with: (a) 5% real-time LLM-as-judge on production traffic, (b) user feedback monitoring (thumbs down rate), (c) shadow evaluation on new model candidates, (d) slice-based evaluation (performance broken down by query category, user tier, etc.).

---

## Trap 9: "LangSmith is just logging for LangChain applications"

**What most people say:** "LangSmith is the logging tool you use when you're using LangChain."

**Correct answer:** LangSmith works with ANY LLM application — not just LangChain. It has native integrations and also works via the generic SDK. More importantly, LangSmith is not just logging — it is an LLM application development platform: (1) **Tracing**: hierarchical traces with spans, latency, token usage; (2) **Datasets**: create and manage evaluation datasets from production traces (one-click "add to dataset"); (3) **Evaluation**: run automated evaluations with custom evaluators on datasets; (4) **Annotation**: human annotation workflows for building training data and golden datasets; (5) **Experiments**: compare runs across different prompt versions or model configurations. The "logging" mental model undersells LangSmith — it's an end-to-end evaluation and dataset management platform. Langfuse is a strong open-source alternative with similar scope.

---

## Trap 10: "Model routing means always using the cheapest model first"

**What most people say:** "Route to GPT-4o-mini first; only escalate to GPT-4o if the cheap model fails."

**Correct answer:** Model routing is more nuanced than waterfall escalation. The optimal routing strategy depends on the query, not a fallback order. A well-designed router should: (1) **Classify query complexity** before routing — a simple factual question routes to a small model; a multi-step reasoning question routes directly to a large model; (2) **Route by task type** — code generation → code-specialized model; translation → translation-specialized model; (3) **Route by latency requirements** — real-time chat routes differently than batch processing; (4) **Consider context length** — if the query requires a 100K token context, route to a model that supports it regardless of cost. Waterfall escalation (small → large on failure) adds latency (you pay for two model calls on escalated requests) and the "failure" signal from the small model may be unreliable. A classifier that routes upfront based on query characteristics is often cheaper overall and lower latency than cascade escalation.
