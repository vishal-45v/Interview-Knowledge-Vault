# Chapter 09: LLMOps & Production — Structured Answers

## Q1: Build a complete LLM cost optimization stack

```python
import hashlib
import time
import redis
import numpy as np
from openai import AsyncOpenAI
from sentence_transformers import SentenceTransformer

client = AsyncOpenAI()
cache = redis.Redis(host="localhost", port=6379, decode_responses=True)
encoder = SentenceTransformer("all-MiniLM-L6-v2")  # fast, cheap embedding model

# ─────────────────────────────────────────────────────────
# TECHNIQUE 1: Exact cache (free for repeated queries)
# ─────────────────────────────────────────────────────────
def exact_cache_key(messages: list, model: str) -> str:
    content = str(messages) + model
    return f"exact:{hashlib.sha256(content.encode()).hexdigest()}"

# ─────────────────────────────────────────────────────────
# TECHNIQUE 2: Semantic cache (cheap for similar queries)
# ─────────────────────────────────────────────────────────
def semantic_cache_lookup(query: str, threshold: float = 0.97) -> str | None:
    query_embedding = encoder.encode(query).tolist()

    # In production: use Redis with vector search (RedisSearch + HNSW)
    cached_entries = cache.keys("sem_cache:*")
    for key in cached_entries:
        entry = cache.hgetall(key)
        cached_emb = np.frombuffer(bytes.fromhex(entry["embedding"]), dtype=np.float32)
        similarity = np.dot(query_embedding, cached_emb) / (
            np.linalg.norm(query_embedding) * np.linalg.norm(cached_emb)
        )
        if similarity > threshold:
            return entry["response"]
    return None

# ─────────────────────────────────────────────────────────
# TECHNIQUE 3: Model routing (route simple queries to cheap models)
# ─────────────────────────────────────────────────────────
async def classify_query_complexity(query: str) -> str:
    """Returns: 'simple' | 'moderate' | 'complex'"""
    # Use a local classifier or very cheap LLM call
    response = await client.chat.completions.create(
        model="gpt-4o-mini",  # cheapest model for classification
        messages=[
            {"role": "system", "content":
             "Classify this query complexity as exactly one of: simple|moderate|complex\n"
             "simple=factual lookup, moderate=some reasoning needed, complex=multi-step reasoning"},
            {"role": "user", "content": f"Query: {query}"}
        ],
        max_tokens=10,
        temperature=0,
    )
    return response.choices[0].message.content.strip().lower()

COMPLEXITY_TO_MODEL = {
    "simple": "gpt-4o-mini",      # $0.150/1M input, $0.600/1M output
    "moderate": "gpt-4o-mini",    # same — most queries are simple or moderate
    "complex": "gpt-4o",          # $2.50/1M input, $10.00/1M output
}

# ─────────────────────────────────────────────────────────
# TECHNIQUE 4: Batching (process multiple requests together)
# ─────────────────────────────────────────────────────────
async def batch_process(queries: list[str], model: str = "gpt-4o-mini") -> list[str]:
    """Process multiple queries in a single batch API call (50% cheaper, async)."""
    import json

    batch_requests = [
        {"custom_id": f"req-{i}", "method": "POST", "url": "/v1/chat/completions",
         "body": {"model": model, "messages": [{"role": "user", "content": q}]}}
        for i, q in enumerate(queries)
    ]

    # Write to JSONL file and submit batch
    batch_file_content = "\n".join(json.dumps(r) for r in batch_requests)
    # ... (submit via OpenAI batch API, poll for completion)
    # Results arrive within 24 hours at 50% reduced cost
    pass

# ─────────────────────────────────────────────────────────
# TECHNIQUE 5: Prompt compression (reduce input tokens)
# ─────────────────────────────────────────────────────────
def compress_context(context: str, query: str, target_tokens: int = 2000) -> str:
    """Extract only the most relevant sentences from context."""
    sentences = context.split(". ")
    if len(sentences) <= 10:
        return context

    # Score sentences by relevance to query
    query_emb = encoder.encode(query)
    sent_embs = encoder.encode(sentences)
    scores = np.dot(sent_embs, query_emb)

    # Keep top sentences up to target_tokens
    top_indices = np.argsort(scores)[::-1]
    compressed = []
    total_chars = 0
    for i in top_indices:
        if total_chars + len(sentences[i]) > target_tokens * 4:  # ~4 chars/token
            break
        compressed.append(sentences[i])
        total_chars += len(sentences[i])

    return ". ".join(sorted(compressed))  # restore original order

# COST IMPACT ESTIMATE (per 1M requests):
# Base cost (GPT-4o, no optimization):     $25,000
# + Exact caching (30% cache hit rate):    -$7,500
# + Semantic caching (additional 15%):     -$3,750
# + Model routing (60% → mini):            -$8,500
# + Prompt compression (30% token reduction): -$1,500
# Optimized cost:                           ~$3,750 (85% reduction)
```

---

## Q2: Implement production LLM observability with Langfuse

```python
from langfuse import Langfuse
from langfuse.decorators import observe, langfuse_context
import time

langfuse = Langfuse(
    public_key="pk-lf-...",
    secret_key="sk-lf-...",
    host="https://cloud.langfuse.com",
)

@observe(name="rag_pipeline")
async def rag_pipeline(query: str, user_id: str) -> str:
    """Main RAG pipeline — creates parent trace."""
    langfuse_context.update_current_trace(
        user_id=user_id,
        tags=["production", "rag"],
        metadata={"query_length": len(query)},
    )

    # Retrieval span
    with langfuse_context.observe_span(name="retrieval") as span:
        start = time.time()
        chunks = await retrieve_documents(query)
        latency = time.time() - start
        span.update(
            metadata={
                "num_chunks": len(chunks),
                "retrieval_latency_ms": latency * 1000,
            }
        )

    # Generation span
    with langfuse_context.observe_span(name="generation") as span:
        response = await generate_answer(query, chunks)
        span.update(
            metadata={
                "model": response.model,
                "input_tokens": response.usage.prompt_tokens,
                "output_tokens": response.usage.completion_tokens,
                "cost_usd": calculate_cost(response.usage),
            }
        )

    return response.choices[0].message.content

# Add human feedback (connected to the trace)
def record_user_feedback(trace_id: str, score: int, comment: str = ""):
    """Record thumbs up/down feedback linked to a specific trace."""
    langfuse.score(
        trace_id=trace_id,
        name="user_satisfaction",
        value=score,  # 1 = thumbs up, 0 = thumbs down
        comment=comment,
        data_type="BOOLEAN",
    )

# Production monitoring queries (in Langfuse dashboard):
# - Average cost per trace by user segment
# - P50/P95/P99 latency by pipeline stage
# - Failure rate by model and error type
# - Token usage trending (catch runaway agents)
# - User satisfaction score correlation with trace metadata
```

---

## Q3: Implement LoRA fine-tuning pipeline

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, TrainingArguments
from peft import LoraConfig, get_peft_model, TaskType
from trl import SFTTrainer
from datasets import Dataset
import torch

def prepare_dataset(examples: list[dict]) -> Dataset:
    """Format training examples as instruction-following pairs."""
    formatted = []
    for ex in examples:
        text = (
            f"<|system|>\n{ex['system']}\n"
            f"<|user|>\n{ex['question']}\n"
            f"<|assistant|>\n{ex['answer']}\n"
        )
        formatted.append({"text": text})
    return Dataset.from_list(formatted)

def train_with_lora(
    base_model_name: str = "meta-llama/Llama-3.1-8B",
    dataset: Dataset = None,
    output_dir: str = "./lora-output"
):
    # Load base model in 4-bit for QLoRA (even less memory)
    from transformers import BitsAndBytesConfig

    bnb_config = BitsAndBytesConfig(
        load_in_4bit=True,
        bnb_4bit_quant_type="nf4",        # NormalFloat4 quantization
        bnb_4bit_compute_dtype=torch.bfloat16,
        bnb_4bit_use_double_quant=True,   # Double quantization for extra savings
    )

    model = AutoModelForCausalLM.from_pretrained(
        base_model_name,
        quantization_config=bnb_config,  # Remove for full LoRA (not QLoRA)
        device_map="auto",
    )
    tokenizer = AutoTokenizer.from_pretrained(base_model_name)

    # LoRA configuration
    lora_config = LoraConfig(
        r=16,                        # Rank: higher = more capacity but more memory
        lora_alpha=32,               # Scaling: typically 2x rank
        target_modules=[             # Which weight matrices to add LoRA to
            "q_proj", "v_proj",      # Attention query and value projections
            "k_proj", "o_proj",      # Optional: add key and output projections
        ],
        lora_dropout=0.05,
        bias="none",
        task_type=TaskType.CAUSAL_LM,
    )

    model = get_peft_model(model, lora_config)
    model.print_trainable_parameters()
    # Trainable params: ~8M / 8B total = 0.1%

    # Training configuration
    training_args = TrainingArguments(
        output_dir=output_dir,
        num_train_epochs=3,
        per_device_train_batch_size=4,
        gradient_accumulation_steps=4,  # Effective batch size = 16
        learning_rate=2e-4,
        warmup_ratio=0.05,
        lr_scheduler_type="cosine",
        fp16=True,
        logging_steps=50,
        evaluation_strategy="steps",
        eval_steps=200,
        save_strategy="steps",
        save_steps=200,
        load_best_model_at_end=True,
        report_to="wandb",            # Track in Weights & Biases
    )

    trainer = SFTTrainer(
        model=model,
        args=training_args,
        train_dataset=dataset,
        tokenizer=tokenizer,
        dataset_text_field="text",
        max_seq_length=2048,
    )

    trainer.train()
    model.save_pretrained(output_dir)

    # Merge LoRA weights back into base model for inference (optional)
    # merged_model = model.merge_and_unload()  # Creates full model with LoRA baked in
    return model
```

---

## Q4: Build a prompt registry with versioning and A/B testing

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum
import json, hashlib, sqlite3, random

class PromptStatus(Enum):
    DRAFT = "draft"
    STAGING = "staging"
    PRODUCTION = "production"
    ARCHIVED = "archived"

@dataclass
class PromptVersion:
    prompt_id: str
    version: int
    content: str
    status: PromptStatus
    created_at: datetime
    metadata: dict
    sha256: str = ""

    def __post_init__(self):
        self.sha256 = hashlib.sha256(self.content.encode()).hexdigest()[:8]

class PromptRegistry:
    def __init__(self, db_path: str = "prompts.db"):
        self.conn = sqlite3.connect(db_path)
        self._init_db()

    def _init_db(self):
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS prompts (
                prompt_id TEXT,
                version INTEGER,
                content TEXT,
                status TEXT,
                created_at TEXT,
                metadata TEXT,
                sha256 TEXT,
                PRIMARY KEY (prompt_id, version)
            )
        """)
        self.conn.execute("""
            CREATE TABLE IF NOT EXISTS ab_tests (
                test_id TEXT PRIMARY KEY,
                prompt_id TEXT,
                variant_a_version INTEGER,
                variant_b_version INTEGER,
                traffic_split REAL,  -- 0.0 to 1.0, fraction for variant B
                start_date TEXT,
                end_date TEXT,
                active INTEGER
            )
        """)
        self.conn.commit()

    def publish(self, prompt_id: str, content: str, metadata: dict = None) -> PromptVersion:
        """Create a new version of a prompt."""
        current_version = self._get_latest_version(prompt_id)
        new_version = current_version + 1

        pv = PromptVersion(
            prompt_id=prompt_id,
            version=new_version,
            content=content,
            status=PromptStatus.DRAFT,
            created_at=datetime.utcnow(),
            metadata=metadata or {},
        )
        self.conn.execute(
            "INSERT INTO prompts VALUES (?,?,?,?,?,?,?)",
            (pv.prompt_id, pv.version, pv.content, pv.status.value,
             pv.created_at.isoformat(), json.dumps(pv.metadata), pv.sha256)
        )
        self.conn.commit()
        return pv

    def promote(self, prompt_id: str, version: int, target: PromptStatus):
        """Promote a prompt version: draft → staging → production."""
        current_status = self._get_status(prompt_id, version)
        valid_transitions = {
            PromptStatus.DRAFT: PromptStatus.STAGING,
            PromptStatus.STAGING: PromptStatus.PRODUCTION,
            PromptStatus.PRODUCTION: PromptStatus.ARCHIVED,
        }
        if valid_transitions.get(current_status) != target:
            raise ValueError(f"Invalid transition: {current_status} → {target}")

        if target == PromptStatus.PRODUCTION:
            # Archive previous production version
            self.conn.execute(
                "UPDATE prompts SET status=? WHERE prompt_id=? AND status=?",
                (PromptStatus.ARCHIVED.value, prompt_id, PromptStatus.PRODUCTION.value)
            )

        self.conn.execute(
            "UPDATE prompts SET status=? WHERE prompt_id=? AND version=?",
            (target.value, prompt_id, version)
        )
        self.conn.commit()

    def get_for_request(self, prompt_id: str, request_id: str = None) -> str:
        """Get the right prompt version for a request, respecting A/B tests."""
        active_test = self._get_active_ab_test(prompt_id)

        if active_test and request_id:
            # Deterministic assignment: same request_id always gets same variant
            hash_val = int(hashlib.md5(request_id.encode()).hexdigest(), 16) / (2**128)
            variant = "b" if hash_val < active_test["traffic_split"] else "a"
            version = (active_test["variant_b_version"] if variant == "b"
                      else active_test["variant_a_version"])
        else:
            version = self._get_production_version(prompt_id)

        row = self.conn.execute(
            "SELECT content FROM prompts WHERE prompt_id=? AND version=?",
            (prompt_id, version)
        ).fetchone()
        return row[0] if row else None
```

---

## Q5: Implement production guardrails with input/output validation

```python
import re
from typing import NamedTuple
from enum import Enum

class GuardrailResult(NamedTuple):
    passed: bool
    action: str        # "allow" | "block" | "modify"
    reason: str
    modified_content: str | None = None

class ProductionGuardrails:
    # PII patterns
    PII_PATTERNS = {
        "ssn": re.compile(r"\b\d{3}-\d{2}-\d{4}\b"),
        "credit_card": re.compile(r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b"),
        "email": re.compile(r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b"),
        "phone": re.compile(r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b"),
    }

    # Topic blocklist
    BLOCKED_TOPICS = [
        "how to hack", "bypass security", "generate malware",
        "make explosive", "synthesize drug",
    ]

    def validate_input(self, user_message: str) -> GuardrailResult:
        """Validate user input before sending to LLM."""

        # Check 1: Blocked topics
        lower_msg = user_message.lower()
        for topic in self.BLOCKED_TOPICS:
            if topic in lower_msg:
                return GuardrailResult(
                    passed=False, action="block",
                    reason=f"Blocked topic detected: '{topic}'"
                )

        # Check 2: PII detection and redaction
        modified = user_message
        pii_found = []
        for pii_type, pattern in self.PII_PATTERNS.items():
            matches = pattern.findall(user_message)
            if matches:
                pii_found.append(pii_type)
                modified = pattern.sub(f"[REDACTED_{pii_type.upper()}]", modified)

        if pii_found:
            return GuardrailResult(
                passed=True, action="modify",
                reason=f"PII redacted: {pii_found}",
                modified_content=modified,
            )

        # Check 3: Prompt injection patterns
        injection_signals = [
            "ignore previous instructions",
            "disregard your system prompt",
            "you are now",
            "act as if you are",
            "new instructions:",
        ]
        for signal in injection_signals:
            if signal in lower_msg:
                return GuardrailResult(
                    passed=False, action="block",
                    reason="Potential prompt injection detected"
                )

        return GuardrailResult(passed=True, action="allow", reason="Input passed all checks")

    def validate_output(self, llm_response: str, context: str = "") -> GuardrailResult:
        """Validate LLM output before returning to user."""

        # Check: No PII in output (model shouldn't echo back PII even if not redacted)
        for pii_type, pattern in self.PII_PATTERNS.items():
            if pattern.search(llm_response):
                modified = pattern.sub(f"[REDACTED]", llm_response)
                return GuardrailResult(
                    passed=True, action="modify",
                    reason=f"PII found in output: {pii_type}",
                    modified_content=modified,
                )

        # Check: Response length sanity (detect runaway generation)
        if len(llm_response) > 10_000:
            return GuardrailResult(
                passed=False, action="block",
                reason=f"Response too long ({len(llm_response)} chars). Possible generation error."
            )

        return GuardrailResult(passed=True, action="allow", reason="Output passed all checks")
```
