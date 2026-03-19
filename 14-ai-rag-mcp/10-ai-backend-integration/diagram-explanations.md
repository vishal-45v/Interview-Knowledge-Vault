# Chapter 10: AI Backend Integration — Diagram Explanations

## Diagram 1: OpenAI vs. Anthropic API Message Format Comparison

```
┌─────────────────────────────────────────────────────────────────────────┐
│           OPENAI vs. ANTHROPIC API FORMAT COMPARISON                   │
└─────────────────────────────────────────────────────────────────────────┘

OPENAI CHAT COMPLETIONS (with tool call):
──────────────────────────────────────────

REQUEST:
{
  "model": "gpt-4o",
  "messages": [
    {"role": "system", "content": "You are a helpful assistant"},  ← system in messages[]
    {"role": "user",   "content": "What's the weather in Tokyo?"}
  ],
  "tools": [{
    "type": "function",
    "function": {
      "name": "get_weather",
      "description": "...",
      "parameters": { "type": "object", "properties": {...} }  ← "parameters"
    }
  }]
}

RESPONSE (tool call):
{
  "choices": [{
    "message": {
      "role": "assistant",
      "content": null,                    ← content is null when using tools
      "tool_calls": [{
        "id": "call_abc123",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\": \"Tokyo\"}"  ← JSON string, not object!
        }
      }]
    },
    "finish_reason": "tool_calls"
  }]
}

TOOL RESULT (must include in next request):
{"role": "tool", "tool_call_id": "call_abc123", "content": "22°C, sunny"}


ANTHROPIC MESSAGES API (with tool use):
─────────────────────────────────────────

REQUEST:
{
  "model": "claude-3-5-sonnet-20241022",
  "system": "You are a helpful assistant",  ← system is TOP-LEVEL, not in messages[]
  "messages": [
    {"role": "user", "content": "What's the weather in Tokyo?"}
  ],
  "tools": [{
    "name": "get_weather",
    "description": "...",
    "input_schema": {                       ← "input_schema" not "parameters"
      "type": "object",
      "properties": {...}
    }
  }]
}

RESPONSE (tool use):
{
  "role": "assistant",
  "content": [                              ← content is an ARRAY of blocks
    {"type": "text", "text": "Let me check..."},  ← optional text before tool
    {
      "type": "tool_use",                   ← "tool_use" not "tool_calls"
      "id": "toolu_01abc",
      "name": "get_weather",
      "input": {"city": "Tokyo"}            ← "input" is an OBJECT, not JSON string
    }
  ],
  "stop_reason": "tool_use"                ← "stop_reason" not "finish_reason"
}

TOOL RESULT (must include in next request):
{
  "role": "user",                           ← tool results go in USER role, not "tool"
  "content": [{
    "type": "tool_result",
    "tool_use_id": "toolu_01abc",
    "content": "22°C, sunny"
  }]
}
```

---

## Diagram 2: Complete RAG Streaming Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│            RAG STREAMING PIPELINE — END TO END                         │
└─────────────────────────────────────────────────────────────────────────┘

Browser                FastAPI Backend              OpenAI API
   │                        │                           │
   │─── GET /stream?q="..." ─►│                           │
   │                        │                           │
   │                        │──── embed(query) ─────────►│
   │                        │◄─── embedding ─────────────│
   │                        │                           │
   │                        │  [ANN search in VectorDB] │
   │                        │  top_k=5 chunks retrieved │
   │                        │                           │
   │                        │──── chat(prompt + chunks) ►│  stream=True
   │                        │                           │
   │◄── "data: {'text':'The'}" ───│◄─── chunk 1 ────────────│
   │◄── "data: {'text':' key'}" ──│◄─── chunk 2 ────────────│
   │◄── "data: {'text':' is'}" ───│◄─── chunk 3 ────────────│
   │◄── "data: {'text':' 42.'}" ──│◄─── chunk 4 ────────────│
   │◄── "data: {'type':'done'}" ──│◄─── [DONE] ─────────────│
   │                        │                           │

PROBLEM: RAG requires retrieval BEFORE streaming starts.
─────────────────────────────────────────────────────────
User experience timeline:

  t=0ms   │ User sends query
  t=100ms │ Query embedded
  t=150ms │ ANN search returns 5 chunks           ← spinner here
  t=200ms │ First token from LLM arrives
  t=200ms │ "The"                                 ← streaming starts
  t=250ms │ " key"
  t=300ms │ " is"
  ...
  t=2000ms│ Complete response                     ← done

PERCEIVED LATENCY: Only 200ms before first token appears!
ACTUAL LATENCY:    200ms retrieval + 1800ms generation = 2000ms total
KEY INSIGHT: Streaming shifts perceived latency from 2000ms to 200ms
             by showing progress as it happens
```

---

## Diagram 3: LCEL Chain Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│              LCEL CHAIN TYPES AND EXECUTION MODELS                     │
└─────────────────────────────────────────────────────────────────────────┘

RUNNABLE SEQUENCE (sequential pipe):
─────────────────────────────────────
  chain = prompt | llm | output_parser

  Input: {"question": "What is RAG?"}
     │
     ▼ prompt.invoke()
  [ChatPromptValue with formatted messages]
     │
     ▼ llm.invoke()
  [AIMessage with generated text]
     │
     ▼ output_parser.invoke()
  "RAG is Retrieval Augmented Generation..."
     │
  Output: str


RUNNABLE PARALLEL (concurrent branches):
──────────────────────────────────────────
  from langchain_core.runnables import RunnableParallel

  chain = RunnableParallel({
      "context": retriever,     ← runs concurrently
      "question": RunnablePassthrough()  ← passes input through
  }) | prompt | llm | StrOutputParser()

  Input: "What is RAG?"
     │
     ├────────────────────────────────┐
     │ retriever.invoke()             │ RunnablePassthrough.invoke()
     │ (fetch relevant docs)          │ (return "What is RAG?" unchanged)
     │ ← ~100ms                      │ ← ~0ms
     └────────────────────────────────┘
     │ Both complete, merge to dict:
     │ {"context": "...docs...", "question": "What is RAG?"}
     ▼
  [Prompt formats merged dict]
     ▼
  [LLM generates answer]
     ▼
  Output: "RAG stands for..."


EXECUTION METHODS:
───────────────────
  ┌────────────────────────────────────────────────────────────────┐
  │ Method          │ Use case                  │ Returns          │
  ├────────────────────────────────────────────────────────────────┤
  │ .invoke(input)  │ Single sync call          │ Final output     │
  │ .ainvoke(input) │ Single async call         │ Final output     │
  │ .stream(input)  │ Stream tokens to user     │ Iterator[chunk]  │
  │ .astream(input) │ Async stream              │ AsyncIter[chunk] │
  │ .batch(inputs)  │ Process list concurrently │ List[output]     │
  │ .abatch(inputs) │ Async batch               │ List[output]     │
  └────────────────────────────────────────────────────────────────┘
  All methods available on ANY LCEL chain automatically!
```

---

## Diagram 4: Instructor Retry Loop for Structured Output

```
┌─────────────────────────────────────────────────────────────────────────┐
│         INSTRUCTOR STRUCTURED OUTPUT — RETRY FLOW                     │
└─────────────────────────────────────────────────────────────────────────┘

  User Document Text
        │
        ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  ATTEMPT 1                                                      │
  │                                                                 │
  │  Prompt: "Extract data as JSON matching this schema..."         │
  │  + Pydantic model schema auto-injected by Instructor           │
  │                    │                                            │
  │                    ▼ LLM response                              │
  │  {                                                              │
  │    "company": "Acme Corp",                                     │
  │    "revenue": "four point two billion",  ← VALIDATION FAIL!    │
  │    "year": 2024                                                │
  │  }                                                              │
  │  Pydantic error: revenue must be float, got str                │
  └─────────────────────────────────────────────────────────────────┘
                                │ RETRY
  ┌─────────────────────────────▼───────────────────────────────────┐
  │  ATTEMPT 2                                                      │
  │                                                                 │
  │  Prompt: [ORIGINAL PROMPT]                                     │
  │  + "Previous attempt failed with error:                        │
  │     revenue must be float, got str                             │
  │     Please fix: revenue should be a number like 4200000000"   │
  │                    │                                            │
  │                    ▼ LLM response (corrected)                  │
  │  {                                                              │
  │    "company": "Acme Corp",                                     │
  │    "revenue": 4200000000.0,  ← VALID float!                    │
  │    "year": 2024                                                │
  │  }                                                              │
  │  Pydantic validation: PASS ✓                                   │
  └─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
                      CompanyAnalysis(
                        company="Acme Corp",
                        revenue=4200000000.0,
                        year=2024
                      )

COST ANALYSIS:
  Without retries (raw JSON mode):  12% parse failure rate
  With Instructor (max_retries=2):  ~0.5% failure rate
  Cost overhead:                    ~15% more tokens (retry prompts)
  Net: well worth the tradeoff for production use cases

WHEN TO STOP RETRYING:
  max_retries=2 is the recommended ceiling for most cases.
  Beyond that, the LLM usually doesn't have the information to satisfy
  the schema — it will hallucinate rather than admit missing data.
  Use Optional fields for uncertain information instead of retrying indefinitely.
```

---

## Diagram 5: Multi-Provider LLM Architecture with LiteLLM

```
┌─────────────────────────────────────────────────────────────────────────┐
│          MULTI-PROVIDER LLM ARCHITECTURE (LiteLLM + Routing)           │
└─────────────────────────────────────────────────────────────────────────┘

                    Your Application Code
                    (OpenAI format always)
                           │
                           ▼
         ┌─────────────────────────────────────────┐
         │              LiteLLM PROXY               │
         │                                         │
         │  ┌─────────────────────────────────┐    │
         │  │       ROUTING LAYER              │    │
         │  │                                 │    │
         │  │  simple/cheap → gpt-4o-mini     │    │
         │  │  complex      → gpt-4o           │    │
         │  │  long context → claude-3.5       │    │
         │  │  rate limited → fallback        │    │
         │  └─────────────────────────────────┘    │
         │                                         │
         │  ┌──────────────────────────────────┐   │
         │  │      TRANSLATION LAYER           │   │
         │  │                                  │   │
         │  │  OpenAI format → provider format │   │
         │  │  provider format → OpenAI format │   │
         │  └──────────────────────────────────┘   │
         └────────────────┬────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         ▼                ▼                ▼
  ┌────────────┐  ┌────────────┐  ┌────────────────────┐
  │  OpenAI    │  │  Anthropic │  │  Self-Hosted       │
  │  API       │  │  API       │  │  (vLLM / Ollama)   │
  │            │  │            │  │                    │
  │  gpt-4o    │  │  claude-   │  │  llama3.1:70b      │
  │  gpt-4o-   │  │  3-5-sonn. │  │  mistral-7b        │
  │  mini      │  │  claude-   │  │  codestral         │
  │  o1        │  │  3-haiku   │  │                    │
  └────────────┘  └────────────┘  └────────────────────┘

CONFIGURATION (config.yaml for LiteLLM proxy):
──────────────────────────────────────────────
  model_list:
    - model_name: fast          # your internal name
      litellm_params:
        model: gpt-4o-mini
        api_key: os.environ/OPENAI_KEY
        rpm: 500
    - model_name: powerful
      litellm_params:
        model: claude-3-5-sonnet-20241022
        api_key: os.environ/ANTHROPIC_KEY
        rpm: 100
    - model_name: powerful      # fallback for "powerful" if Claude is down
      litellm_params:
        model: gpt-4o
        api_key: os.environ/OPENAI_KEY

  router_settings:
    routing_strategy: least-busy
    fallbacks: [{"powerful": ["gpt-4o"]}]
    retry_policy: {max_retries: 2, retry_on: ["RateLimitError", "ServiceUnavailableError"]}
```
