# Chapter 10: AI Backend Integration — Structured Answers

## Q1: Build a complete streaming LLM API endpoint with FastAPI + SSE

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
import asyncio
import json
from openai import AsyncOpenAI

app = FastAPI()
client = AsyncOpenAI()

async def stream_openai_response(query: str, context: str):
    """Generator that yields SSE-formatted chunks from OpenAI streaming."""
    try:
        response = await client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {"role": "system", "content": f"Answer based on this context:\n{context}"},
                {"role": "user", "content": query},
            ],
            stream=True,
            max_tokens=1024,
        )

        async for chunk in response:
            delta = chunk.choices[0].delta
            finish_reason = chunk.choices[0].finish_reason

            if delta.content:
                # SSE format: "data: {json}\n\n"
                payload = json.dumps({"type": "content", "text": delta.content})
                yield f"data: {payload}\n\n"

            if finish_reason == "stop":
                yield f"data: {json.dumps({'type': 'done'})}\n\n"
                return

            if finish_reason == "length":
                yield f"data: {json.dumps({'type': 'truncated', 'reason': 'max_tokens'})}\n\n"
                return

    except Exception as e:
        error_payload = json.dumps({"type": "error", "message": str(e)})
        yield f"data: {error_payload}\n\n"

@app.get("/stream")
async def stream_endpoint(query: str, request: Request):
    """SSE endpoint that streams LLM responses."""
    context = await retrieve_context(query)  # Your RAG retrieval here

    async def event_generator():
        async for chunk in stream_openai_response(query, context):
            # Check if client disconnected
            if await request.is_disconnected():
                break  # Stop generating when client cancels
            yield chunk

    return StreamingResponse(
        event_generator(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",          # Critical for Nginx to not buffer SSE
            "Connection": "keep-alive",
            "Transfer-Encoding": "chunked",
        },
    )

# Client-side consumption (Python):
import httpx

async def consume_stream(query: str):
    async with httpx.AsyncClient(timeout=60) as http_client:
        async with http_client.stream("GET", f"http://localhost:8000/stream?query={query}") as response:
            full_text = ""
            async for line in response.aiter_lines():
                if line.startswith("data: "):
                    data = json.loads(line[6:])
                    if data["type"] == "content":
                        print(data["text"], end="", flush=True)
                        full_text += data["text"]
                    elif data["type"] in ("done", "truncated"):
                        break
            return full_text
```

---

## Q2: Implement Anthropic API with tool use — complete conversation cycle

```python
import anthropic
import json

client = anthropic.Anthropic()

TOOLS = [
    {
        "name": "get_weather",
        "description": "Get current weather for a city. Returns temperature, conditions, and humidity.",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "City name, e.g. 'San Francisco, CA'"
                },
                "units": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "Temperature units"
                }
            },
            "required": ["city"]
        }
    }
]

def process_tool_call(tool_name: str, tool_input: dict) -> str:
    if tool_name == "get_weather":
        # Simulated weather API
        return json.dumps({
            "city": tool_input["city"],
            "temperature": 22,
            "units": tool_input.get("units", "celsius"),
            "conditions": "Partly cloudy",
            "humidity": 65
        })
    return "Tool not found"

def run_agent_with_tools(user_message: str) -> str:
    """Complete agent loop with Anthropic tool use."""
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-3-5-sonnet-20241022",
            max_tokens=1024,
            tools=TOOLS,
            messages=messages,
        )

        # Append assistant response to conversation
        messages.append({"role": "assistant", "content": response.content})

        # Check stop reason
        if response.stop_reason == "end_turn":
            # No tool calls — extract text response
            for block in response.content:
                if hasattr(block, "text"):
                    return block.text
            return ""

        if response.stop_reason == "tool_use":
            # Process tool calls and build tool_result blocks
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = process_tool_call(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result,
                        # "is_error": True,  # Set if tool call failed
                    })

            # Add tool results as user turn (required by Anthropic API)
            messages.append({"role": "user", "content": tool_results})
            # Loop continues → next response may have more tool calls or end_turn

result = run_agent_with_tools("What's the weather like in Tokyo right now?")
print(result)
```

---

## Q3: Implement structured output with Instructor and complex Pydantic models

```python
import instructor
from pydantic import BaseModel, Field, field_validator, model_validator
from typing import Optional, List
from openai import OpenAI
from enum import Enum

client = instructor.from_openai(OpenAI())

class RiskLevel(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class Product(BaseModel):
    name: str = Field(description="Product name as stated in the document")
    annual_revenue_usd: Optional[float] = Field(
        default=None,
        description="Annual revenue for this product line in USD. Null if not mentioned."
    )
    market_share_pct: Optional[float] = Field(
        default=None,
        ge=0, le=100,
        description="Market share percentage (0-100). Null if not mentioned."
    )
    growth_yoy_pct: Optional[float] = Field(
        default=None,
        description="Year-over-year growth percentage. Can be negative. Null if not mentioned."
    )

class CompanyAnalysis(BaseModel):
    company_name: str
    fiscal_year: int = Field(ge=2000, le=2030)
    total_revenue_usd: float = Field(gt=0)
    total_revenue_unit: str = Field(description="Unit: 'million' or 'billion'")
    products: List[Product] = Field(min_length=1)
    risk_level: RiskLevel
    key_risks: List[str] = Field(max_length=5, description="Top risk factors, max 5")
    analyst_notes: Optional[str] = None

    @field_validator("fiscal_year")
    @classmethod
    def validate_fiscal_year(cls, v: int) -> int:
        if v < 2015:
            raise ValueError("Only analyzing reports from 2015 onwards")
        return v

    @model_validator(mode="after")
    def validate_revenue_consistency(self) -> "CompanyAnalysis":
        """Check product revenues don't exceed total revenue."""
        product_revenues = [
            p.annual_revenue_usd for p in self.products
            if p.annual_revenue_usd is not None
        ]
        if product_revenues:
            total_product_revenue = sum(product_revenues)
            total = self.total_revenue_usd
            if self.total_revenue_unit == "billion":
                total *= 1_000_000_000
            elif self.total_revenue_unit == "million":
                total *= 1_000_000
            if total_product_revenue > total * 1.05:  # 5% tolerance
                raise ValueError(
                    f"Sum of product revenues ({total_product_revenue}) "
                    f"exceeds total revenue ({total})"
                )
        return self

def extract_company_analysis(document_text: str) -> CompanyAnalysis | None:
    try:
        result = client.chat.completions.create(
            model="gpt-4o",
            response_model=CompanyAnalysis,
            max_retries=2,
            messages=[
                {
                    "role": "system",
                    "content": "Extract financial data from the document. "
                               "Use null for information not present in the document. "
                               "Do not make up data."
                },
                {"role": "user", "content": f"Extract data from:\n\n{document_text}"}
            ]
        )
        return result
    except Exception as e:
        print(f"Extraction failed after retries: {e}")
        return None
```

---

## Q4: Build a complete async batch processing pipeline

```python
import asyncio
import time
from openai import AsyncOpenAI
from typing import NamedTuple

client = AsyncOpenAI()

class ProcessingResult(NamedTuple):
    doc_id: str
    embedding: list[float] | None
    summary: str | None
    category: str | None
    error: str | None

async def embed_document(text: str) -> list[float]:
    response = await client.embeddings.create(
        model="text-embedding-3-small",
        input=text[:8191],  # max input length
    )
    return response.data[0].embedding

async def summarize_document(text: str) -> str:
    response = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Summarize in 2 sentences."},
            {"role": "user", "content": text[:10000]},
        ],
        max_tokens=100,
    )
    return response.choices[0].message.content

async def classify_document(text: str) -> str:
    response = await client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "system", "content": "Classify as: technical|business|legal|other"},
            {"role": "user", "content": text[:2000]},
        ],
        max_tokens=10,
        temperature=0,
    )
    return response.choices[0].message.content.strip().lower()

async def process_single_document(
    doc_id: str,
    text: str,
    semaphore: asyncio.Semaphore
) -> ProcessingResult:
    """Process a single document with rate limiting via semaphore."""
    async with semaphore:  # Limit concurrent requests
        try:
            # Run all three calls concurrently for this document
            embedding, summary, category = await asyncio.gather(
                embed_document(text),
                summarize_document(text),
                classify_document(text),
            )
            return ProcessingResult(doc_id, embedding, summary, category, None)
        except Exception as e:
            return ProcessingResult(doc_id, None, None, None, str(e))

async def process_documents_batch(
    documents: dict[str, str],  # {doc_id: text}
    max_concurrent: int = 20    # Rate limit: max 20 docs in flight
) -> list[ProcessingResult]:
    semaphore = asyncio.Semaphore(max_concurrent)

    tasks = [
        process_single_document(doc_id, text, semaphore)
        for doc_id, text in documents.items()
    ]

    start = time.time()
    results = await asyncio.gather(*tasks, return_exceptions=False)
    elapsed = time.time() - start

    success = sum(1 for r in results if r.error is None)
    print(f"Processed {len(documents)} docs in {elapsed:.1f}s "
          f"({success}/{len(documents)} succeeded)")
    print(f"Sequential would take ~{len(documents) * 2:.0f}s; "
          f"speedup: {len(documents) * 2 / elapsed:.1f}x")

    return results

# Speedup estimate for 1000 documents:
# Sequential: 1000 docs × 2s avg = ~33 minutes
# Async (20 concurrent): ~1000 * 2 / 20 = ~100 seconds (20x speedup)
# Bottleneck: OpenAI rate limits, not code concurrency
```

---

## Q5: Implement complete error handling for LLM APIs

```python
import asyncio
import logging
from openai import AsyncOpenAI, APIStatusError, APIConnectionError, APITimeoutError
from openai import RateLimitError, BadRequestError, AuthenticationError

logger = logging.getLogger(__name__)
client = AsyncOpenAI()

class LLMError(Exception):
    """Base class for LLM-related errors."""
    def __init__(self, message: str, retryable: bool, error_type: str):
        super().__init__(message)
        self.retryable = retryable
        self.error_type = error_type

async def call_llm_with_full_error_handling(
    messages: list[dict],
    model: str = "gpt-4o",
    max_tokens: int = 1024,
    max_retries: int = 3,
) -> str:
    for attempt in range(max_retries):
        try:
            response = await client.chat.completions.create(
                model=model,
                messages=messages,
                max_tokens=max_tokens,
                timeout=30.0,
            )

            finish_reason = response.choices[0].finish_reason
            content = response.choices[0].message.content

            if finish_reason == "length":
                logger.warning(f"Response truncated at {max_tokens} tokens")
                # Strategy: return truncated response with a note, or retry with summary
                return content + "\n[Response truncated — context may be incomplete]"

            if finish_reason == "content_filter":
                raise LLMError(
                    "Response blocked by content policy",
                    retryable=False,
                    error_type="content_policy"
                )

            return content

        except RateLimitError as e:
            wait_time = 2 ** attempt + 1  # 2s, 3s, 5s
            logger.warning(f"Rate limit hit (attempt {attempt+1}). Waiting {wait_time}s")
            if attempt < max_retries - 1:
                await asyncio.sleep(wait_time)
            else:
                raise LLMError(str(e), retryable=False, error_type="rate_limit")

        except BadRequestError as e:
            error_code = e.code if hasattr(e, "code") else ""
            if "context_length_exceeded" in str(e) or error_code == "context_length_exceeded":
                # Context too long: truncate and retry
                logger.warning("Context length exceeded. Truncating.")
                messages = truncate_messages(messages, target_reduction=0.3)
                continue  # retry with truncated messages
            else:
                raise LLMError(str(e), retryable=False, error_type="bad_request")

        except APIConnectionError as e:
            logger.error(f"Connection error (attempt {attempt+1}): {e}")
            if attempt < max_retries - 1:
                await asyncio.sleep(2 ** attempt)
            else:
                raise LLMError(str(e), retryable=False, error_type="connection")

        except APITimeoutError:
            logger.warning(f"Request timed out (attempt {attempt+1})")
            if attempt < max_retries - 1:
                await asyncio.sleep(1)
            else:
                raise LLMError("Request timed out after all retries", retryable=False, error_type="timeout")

        except AuthenticationError:
            raise LLMError("Invalid API key", retryable=False, error_type="auth")

def truncate_messages(messages: list[dict], target_reduction: float = 0.3) -> list[dict]:
    """Reduce context by removing middle assistant/tool messages."""
    import tiktoken
    enc = tiktoken.encoding_for_model("gpt-4o")

    # Keep system prompt and last 4 messages; summarize or drop middle
    if len(messages) <= 4:
        # Can't truncate further — truncate the longest message
        longest_idx = max(range(len(messages)),
                         key=lambda i: len(enc.encode(str(messages[i].get("content", "")))))
        msg = messages[longest_idx]
        if isinstance(msg["content"], str):
            token_target = int(len(enc.encode(msg["content"])) * (1 - target_reduction))
            tokens = enc.encode(msg["content"])[:token_target]
            messages[longest_idx] = {**msg, "content": enc.decode(tokens)}
        return messages

    # Keep first (system) + last 3 messages; drop middle
    return [messages[0]] + messages[-3:]
```
