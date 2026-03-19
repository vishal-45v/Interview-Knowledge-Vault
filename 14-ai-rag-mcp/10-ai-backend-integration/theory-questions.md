# Chapter 10: AI Backend Integration — Theory Questions

## OpenAI API

1. What are the structural differences between the OpenAI Chat Completions API and the Assistants API? When would you use Assistants API over raw Chat Completions? What are the limitations and known issues of the Assistants API?

2. Explain OpenAI's streaming API. What is the `stream=True` parameter behavior? Walk through the structure of a streaming response: what does each `ChatCompletionChunk` contain? How do you reconstruct the full response from a stream? How do you handle tool calls in a streaming response?

3. Describe the OpenAI Batch API. What is the input format (JSONL)? What is the maximum batch size? What is the cost reduction compared to synchronous calls? What are the latency tradeoffs, and what use cases is it designed for?

4. Explain OpenAI function calling in the request payload: the `tools` array structure, `tool_choice` parameter options (`none`, `auto`, `required`, specific function), and parallel tool calls. What is the difference between `function` and `tool` in the API (historical context)?

5. How does the OpenAI embeddings API work? What are the model options (text-embedding-3-small, text-embedding-3-large, text-embedding-ada-002)? What are the dimension options for text-embedding-3 models and when would you use a reduced dimension?

## Anthropic API

6. Describe the Anthropic Messages API structure. What is the `system` parameter vs. the first `user` message? How does the `content` field differ from OpenAI's `content` (array of content blocks vs. string)?

7. Explain Anthropic's `tool_use` and `tool_result` content block format. Walk through a complete conversation where the model uses a tool: what does the request look like, what does the response look like, and what must the next request contain?

8. What is Anthropic's extended thinking (or "think" tokens / extended thinking mode in newer API versions)? What is the thinking budget parameter? How do thinking tokens affect billing?

9. Explain Anthropic's prompt caching in detail. What is the `cache_control` parameter? What is the minimum cache threshold (1024 tokens)? What is the pricing for cache writes vs. cache reads? How long does the cache persist?

## Frameworks

10. Explain LangChain Expression Language (LCEL). What are Runnables? What is the `|` pipe operator? How does `RunnableParallel` work? What is the difference between `.invoke()`, `.stream()`, and `.batch()`?

11. Compare LangChain's document loaders, text splitters, and retrievers. For each category, give three concrete examples. What is the difference between a retriever and a vector store in LangChain's abstraction?

12. Describe LlamaIndex's core abstractions: `VectorStoreIndex`, `QueryEngine`, `IngestionPipeline`, `NodeParser`, and response synthesizers. How does LlamaIndex's data model (Documents → Nodes) differ from LangChain's (Documents → Chunks)?

13. What is the LlamaIndex `IngestionPipeline`? What transformations can it include? How does it handle incremental ingestion (new documents added to an existing index)? What is the `docstore` used for in this context?

## Implementation Details

14. Walk through the implementation of SSE (Server-Sent Events) streaming for an LLM backend. What are the required HTTP headers? What is the event format? How does a client (browser or Python) consume the stream?

15. What is `tiktoken`? How do you accurately count tokens before making an OpenAI API call? What are the model-specific encoding differences between `cl100k_base` and `o200k_base`? Why does token counting matter for production systems?

16. Describe the Instructor library. How does it use Pydantic models to enforce structured output from LLMs? What is the `response_model` parameter? How does Instructor handle validation failures and retries?

17. How do you implement async LLM calls in Python? Compare `asyncio` with `aiohttp` vs. the `AsyncOpenAI` client. When does async matter for LLM applications, and when does it NOT provide meaningful benefit?

18. What is LiteLLM and how does it provide an OpenAI-compatible interface over 100+ models? What is the routing logic? How does it handle model-specific parameters that don't exist in the OpenAI spec?

19. Explain the OpenAI-compatible API that Ollama exposes. What endpoints does it implement? What are the limitations compared to the real OpenAI API? How do you use the same OpenAI client library to talk to Ollama?

20. Describe error handling best practices for LLM APIs. What are the specific error codes to handle (429, 500, 503)? What is content policy error handling? How do you handle context length exceeded errors (what is the recovery strategy)?
