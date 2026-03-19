# Chapter 09: LLMOps & Production — Diagram Explanations

## Diagram 1: Complete LLMOps Stack

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     COMPLETE LLMOPS PRODUCTION STACK                   │
└─────────────────────────────────────────────────────────────────────────┘

                              USER REQUEST
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  LAYER 1: INPUT GUARDRAILS                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ PII Detection│  │Topic Filter  │  │Prompt Inject.│  │Length Check│  │
│  │ + Redaction  │  │(blocklist)   │  │Detection     │  │< 4096 tok  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                   │ PASS
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  LAYER 2: CACHING                                                       │
│  ┌────────────────────────┐    ┌──────────────────────────────────────┐  │
│  │  Exact Cache (Redis)   │    │  Semantic Cache (Vector Search)      │  │
│  │  Hash(query) → response│    │  embed(query) → similar cached resp  │  │
│  └────────────────────────┘    └──────────────────────────────────────┘  │
│                          HIT → return cached response                    │
│                          MISS → continue to model                       │
└──────────────────────────────────────────────────────────────────────────┘
                                   │ MISS
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  LAYER 3: MODEL ROUTING                                                  │
│                                                                          │
│  Query Classifier (fast, cheap)                                         │
│       │                                                                  │
│  ┌────┴──────────────────────────────────────────┐                      │
│  │  simple (80%)  │  moderate (15%)  │ complex (5%)│                    │
│  └────┬──────────────────────────────────┬────────┘                     │
│       ▼                                  ▼           ▼                  │
│  gpt-4o-mini               gpt-4o-mini          gpt-4o                 │
│  ($0.15/1M in)             ($0.15/1M in)        ($2.50/1M in)          │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  LAYER 4: RATE LIMIT MANAGEMENT                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │  Request Queue  →  Semaphore (RPM limit)  →  Token Bucket (TPM)   │  │
│  │  Retry Logic: exponential backoff (1s, 2s, 4s, 8s, 16s)          │  │
│  │  Fallback: alternate provider (Anthropic ↔ OpenAI ↔ Azure OAI)   │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                          [LLM API CALL]
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  LAYER 5: OUTPUT GUARDRAILS                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │ PII Scan     │  │ Schema Valid.│  │ Faithfulness │  │Toxicity    │  │
│  │ (output)     │  │ (if JSON)    │  │ Check        │  │Classifier  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  LAYER 6: OBSERVABILITY (every request)                                 │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  Langfuse/LangSmith trace: latency, tokens, cost, model used      │ │
│  │  Metrics: emit to Prometheus/Datadog                              │ │
│  │  Async: don't block response for observability                    │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
                              USER RESPONSE
```

---

## Diagram 2: LLM Evaluation Pipeline — Nightly Regression Testing

```
┌─────────────────────────────────────────────────────────────────────────┐
│               LLM EVALUATION & REGRESSION TEST PIPELINE                │
└─────────────────────────────────────────────────────────────────────────┘

COMPONENTS:
──────────────────────────────────────────────────────────────────────────

  ┌────────────────────────────────────┐
  │         GOLDEN DATASET             │
  │                                    │
  │  500 test cases, each with:        │
  │  - question (the query)            │
  │  - ground_truth (expected answer)  │
  │  - context (expected retrieval)    │
  │  - metadata (category, difficulty) │
  │                                    │
  │  Categories:                       │
  │  ├── factual_lookup (30%)          │
  │  ├── multi_hop (20%)               │
  │  ├── summarization (20%)           │
  │  └── edge_cases (30%)              │
  └────────────────────┬───────────────┘
                       │ nightly (2 AM)
                       ▼
  ┌────────────────────────────────────┐
  │      EVALUATION RUNNER             │
  │                                    │
  │  1. Run all 500 test cases         │
  │     against PRODUCTION endpoint    │
  │                                    │
  │  2. For each result compute:       │
  │     - faithfulness (RAGAS)         │
  │     - answer_relevancy (RAGAS)     │
  │     - context_precision (RAGAS)    │
  │     - LLM-as-judge score           │
  │     - latency_p95                  │
  │                                    │
  │  3. Compare to BASELINE scores     │
  │     (stored from last passing run) │
  └────────────────────┬───────────────┘
                       │
                       ▼
  ┌────────────────────────────────────┐
  │       REGRESSION DETECTOR         │
  │                                    │
  │  Pass criteria (ALL must pass):    │
  │  ✓ faithfulness ≥ 0.88            │
  │  ✓ answer_relevancy ≥ 0.85        │
  │  ✓ context_precision ≥ 0.80       │
  │  ✓ no single category drops >10%  │
  │  ✓ p95_latency < 3 seconds        │
  │                                    │
  └────────────────────┬───────────────┘
                       │
              ┌────────┴────────┐
              │                 │
            PASS              FAIL
              │                 │
              ▼                 ▼
  ┌───────────────────┐  ┌───────────────────────────────┐
  │  ✓ Update baseline│  │  ✗ Block deployment           │
  │  ✓ Generate report│  │  ✗ Alert on-call engineer     │
  │  ✓ Archive traces │  │  ✗ Create incident ticket     │
  └───────────────────┘  │  ✗ Attach failing test traces │
                         └───────────────────────────────┘

SLICE-BASED REPORTING:
─────────────────────────────────────────────────────────────────────────
  Overall score:  0.89 ✓  |  Slice analysis reveals:
  ───────────────────────────────────────────────────────────
  factual_lookup: 0.95 ✓  |  Math questions:   0.71 ✗ REGRESSION
  multi_hop:      0.83 ✓  |  Legal questions:  0.87 ✓
  edge_cases:     0.79 ✗  |  ← needs investigation
  ───────────────────────────────────────────────────────────
  Overall passing masks a specific regression in edge cases + math!
```

---

## Diagram 3: Prompt Caching — Cache Hit vs. Miss

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PROMPT CACHING — TOKEN ECONOMICS                    │
└─────────────────────────────────────────────────────────────────────────┘

PROMPT STRUCTURE (well-structured for caching):
────────────────────────────────────────────────
  ┌──────────────────────────────────────────────────────┐
  │  [STABLE — cache this]                              │
  │  ┌────────────────────────────────────────────────┐ │
  │  │ System instructions: 2,000 tokens              │ │
  │  │ Retrieved documents: 8,000 tokens              │ │
  │  │ Few-shot examples:   1,500 tokens              │ │
  │  │                                                │ │
  │  │ Total cacheable prefix: 11,500 tokens          │ │
  │  └────────────────────────────────────────────────┘ │
  │                                                      │
  │  [VARIABLE — user's query, NOT cached]              │
  │  ┌────────────────────────────────────────────────┐ │
  │  │ User question: 15 tokens                       │ │
  │  └────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────┘

COST CALCULATION (Anthropic Claude pricing):
────────────────────────────────────────────

  FIRST REQUEST (cache miss):
  ┌────────────────────────────────────────────────────────┐
  │  11,500 cache write tokens × $3.75/1M = $0.0431       │
  │      15 input tokens       × $3.00/1M = $0.000045     │
  │  Total input cost: $0.043                              │
  └────────────────────────────────────────────────────────┘

  SUBSEQUENT REQUESTS (cache hit — within 5 min):
  ┌────────────────────────────────────────────────────────┐
  │  11,500 cache read tokens × $0.30/1M = $0.00345       │
  │      15 input tokens      × $3.00/1M = $0.000045      │
  │  Total input cost: $0.0035                             │
  │                                                        │
  │  Savings vs. no caching: 92% cheaper!                 │
  └────────────────────────────────────────────────────────┘

CACHE INVALIDATION TRIGGERS:
  ┌─────────────────────────────────────────────────────┐
  │  ✗ Any character change in cached prefix            │
  │  ✗ Cache TTL expires (5 minutes for Anthropic)      │
  │  ✗ New documents added to retrieval context         │
  │  ✓ Same system prompt + same documents + new query  │
  │    → CACHE HIT every time                          │
  └─────────────────────────────────────────────────────┘
```

---

## Diagram 4: Model Routing Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   MODEL ROUTING DECISION TREE                          │
└─────────────────────────────────────────────────────────────────────────┘

                           Incoming Query
                                 │
                   ┌─────────────▼───────────────┐
                   │    Context length > 50K?     │
                   └─────────┬─────────┬──────────┘
                           YES         NO
                            │           │
                            ▼           │
                  ┌──────────────────┐  │
                  │ claude-3-5-sonnet│  │
                  │ (200K context)   │  │
                  │ or gpt-4-turbo   │  │
                  └──────────────────┘  │
                                        ▼
                           ┌─────────────────────────┐
                           │   Query is time-sensitive│
                           │   (stock price, news)?   │
                           └──────────┬──────────┬────┘
                                    YES          NO
                                     │            │
                                     ▼            │
                            ┌─────────────┐       │
                            │ Do NOT cache│       │
                            │ Use fastest │       │
                            │ model       │       │
                            └─────────────┘       │
                                                  ▼
                                    ┌──────────────────────┐
                                    │  Requires code gen   │
                                    │  or complex math?    │
                                    └───────┬──────┬───────┘
                                          YES      NO
                                           │        │
                                           ▼        ▼
                                   ┌──────────┐  ┌────────────────┐
                                   │ gpt-4o   │  │  Complex multi-│
                                   │ or       │  │  step reasoning│
                                   │ claude   │  │  needed?       │
                                   │ -3.5     │  └───┬──────┬─────┘
                                   └──────────┘    YES      NO
                                                    │        │
                                                    ▼        ▼
                                            ┌──────────┐ ┌──────────────┐
                                            │  gpt-4o  │ │ gpt-4o-mini  │
                                            │ (strong) │ │ (80% cheaper)│
                                            └──────────┘ └──────────────┘

EXPECTED TRAFFIC DISTRIBUTION:
┌────────────────────────────────────────────────────────────┐
│ gpt-4o-mini:    65% of queries  (cheapest, fastest)        │
│ gpt-4o:         25% of queries  (complex reasoning/code)   │
│ claude-3.5:      8% of queries  (long context)             │
│ cache hits:      2% of queries  (near-zero cost)           │
│                                                            │
│ Without routing:  avg cost = $2.50/1M tokens (all GPT-4o) │
│ With routing:     avg cost = $0.65/1M tokens (74% savings) │
└────────────────────────────────────────────────────────────┘
```

---

## Diagram 5: LoRA vs Full Fine-tuning Memory Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    LORA vs FULL FINE-TUNING COMPARISON                 │
└─────────────────────────────────────────────────────────────────────────┘

FULL FINE-TUNING (Llama 3 8B):
──────────────────────────────
  Model weights:      8B params × 2 bytes (bf16) = 16GB
  Gradients:          8B params × 2 bytes         = 16GB
  Optimizer states:   8B params × 8 bytes (Adam)  = 64GB
                                                ──────────
  Total GPU VRAM needed:                          96GB
  Hardware required:  4x A100 80GB GPUs ($16/hr)

LORA FINE-TUNING (same model, r=16):
─────────────────────────────────────
  Model weights:      16GB (same, but frozen — no gradients)
  LoRA matrices A,B:  ~8M params × 2 bytes         = 16MB
  Gradients (LoRA):   ~8M params × 2 bytes         = 16MB
  Optimizer (LoRA):   ~8M params × 8 bytes         = 64MB
                                                ──────────
  Total GPU VRAM needed:                          ~17GB
  Hardware required:  1x A100 40GB GPU ($4/hr) ← 4x cheaper!

QLORA FINE-TUNING (4-bit quantized base model):
────────────────────────────────────────────────
  Model weights:      8B params × 0.5 bytes (4-bit) = 4GB
  LoRA matrices:                                     = 16MB
  Total GPU VRAM:                                   ~5GB
  Hardware required:  1x RTX 4090 24GB ($0.40/hr) ← consumer GPU!

MATHEMATICAL DECOMPOSITION:
────────────────────────────

  Weight matrix W (d×d):  shape [4096, 4096]

  Full fine-tuning update:
  ΔW = full matrix         shape [4096, 4096] = 16.7M params

  LoRA update (rank r=16):
  ΔW = B × A
  B: shape [4096, 16]  = 65,536 params
  A: shape [16, 4096]  = 65,536 params
  Total: 131,072 params = 128x fewer parameters per weight matrix

  During inference:
  W_effective = W_base + B × A
              = (frozen 4096×4096) + (4096×16 × 16×4096)
              ← adds only 131K parameters' worth of "knowledge"
              ← model size remains 8B — no inference cost increase
              ← merge and unload → identical to original model size
```
