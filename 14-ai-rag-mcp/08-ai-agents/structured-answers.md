# Chapter 08: AI Agents — Structured Answers

## Q1: Implement a complete ReAct agent loop from scratch

```python
from openai import OpenAI
import json
from typing import Callable

client = OpenAI()

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "search_web",
            "description": "Search the web for current information. Returns top 3 relevant results.",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Search query"},
                },
                "required": ["query"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "calculate",
            "description": "Evaluate a mathematical expression. Input must be a valid Python expression.",
            "parameters": {
                "type": "object",
                "properties": {
                    "expression": {"type": "string", "description": "Python math expression e.g. '(42 * 1.08) + 15'"},
                },
                "required": ["expression"],
            },
        },
    },
]

def execute_tool(name: str, arguments: dict) -> str:
    """Dispatch tool calls to actual implementations."""
    if name == "search_web":
        # In production: call a real search API
        return f"Search results for '{arguments['query']}': [Result 1...] [Result 2...]"
    elif name == "calculate":
        try:
            # SAFETY: In production, use a sandboxed evaluator
            result = eval(arguments["expression"], {"__builtins__": {}})
            return str(result)
        except Exception as e:
            return f"Calculation error: {e}"
    return f"Unknown tool: {name}"

def react_agent(user_query: str, max_iterations: int = 10) -> str:
    """ReAct agent: Reasoning + Acting loop."""
    messages = [
        {
            "role": "system",
            "content": (
                "You are a helpful assistant that uses tools to answer questions. "
                "Think step-by-step. Use tools when needed. "
                "If you call a tool, wait for the result before reasoning further. "
                "When you have enough information, provide a final answer."
            ),
        },
        {"role": "user", "content": user_query},
    ]

    for iteration in range(max_iterations):
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=TOOLS,
            tool_choice="auto",
        )

        message = response.choices[0].message
        messages.append(message)  # Add assistant response to history

        # Terminal condition: no tool calls = agent has final answer
        if not message.tool_calls:
            return message.content

        # Process tool calls (may be parallel)
        tool_results = []
        for tool_call in message.tool_calls:
            arguments = json.loads(tool_call.function.arguments)
            result = execute_tool(tool_call.function.name, arguments)
            tool_results.append({
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": result,
            })

        messages.extend(tool_results)

    # Safety: max iterations reached
    return "I was unable to complete this task within the allowed number of steps."

# Usage
answer = react_agent("What is 15% of the current price of a Big Mac in the US?")
```

---

## Q2: Build a LangGraph agent with state management and conditional routing

```python
from langgraph.graph import StateGraph, END
from langgraph.prebuilt import ToolNode
from langchain_openai import ChatOpenAI
from langchain_core.messages import BaseMessage, HumanMessage
from typing import TypedDict, Annotated, Sequence
import operator

# State schema: typed, supports message accumulation
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], operator.add]
    iteration_count: int
    final_answer: str | None

# Tool definitions
from langchain_core.tools import tool

@tool
def search_customers(query: str) -> str:
    """Search the customer database by name or email. Returns matching customer records."""
    # Simulated DB lookup
    return f"Found customers matching '{query}': [{{'id': 'C001', 'name': 'Alice', 'email': 'alice@example.com'}}]"

@tool
def get_order_history(customer_id: str, limit: int = 5) -> str:
    """Retrieve recent orders for a customer by customer ID."""
    return f"Orders for {customer_id}: [{{order_id: 'O123', total: 45.00, status: 'delivered'}}]"

@tool
def process_refund(order_id: str, reason: str) -> str:
    """Process a refund for an order. Requires order_id and reason."""
    return f"Refund initiated for order {order_id}. Estimated 3-5 business days."

tools = [search_customers, get_order_history, process_refund]
tool_node = ToolNode(tools)

# LLM with tools bound
llm = ChatOpenAI(model="gpt-4o", temperature=0)
llm_with_tools = llm.bind_tools(tools)

# Agent node: calls LLM, gets response
def agent_node(state: AgentState) -> dict:
    messages = state["messages"]
    response = llm_with_tools.invoke(messages)
    return {
        "messages": [response],
        "iteration_count": state.get("iteration_count", 0) + 1,
    }

# Routing: should we call a tool or return the final answer?
def should_continue(state: AgentState) -> str:
    last_message = state["messages"][-1]

    # Safety: max iterations check
    if state.get("iteration_count", 0) >= 10:
        return "end"

    # If LLM made tool calls, route to tool executor
    if last_message.tool_calls:
        return "tools"

    # Otherwise, we have the final answer
    return "end"

# Build the graph
graph_builder = StateGraph(AgentState)
graph_builder.add_node("agent", agent_node)
graph_builder.add_node("tools", tool_node)
graph_builder.set_entry_point("agent")
graph_builder.add_conditional_edges(
    "agent",
    should_continue,
    {"tools": "tools", "end": END},
)
graph_builder.add_edge("tools", "agent")  # Loop back after tools

graph = graph_builder.compile()

# Run the agent
result = graph.invoke({
    "messages": [HumanMessage(content="Find customer alice@example.com and show her recent orders")],
    "iteration_count": 0,
    "final_answer": None,
})
print(result["messages"][-1].content)
```

---

## Q3: Implement multi-agent orchestrator-subagent pattern

```python
from openai import OpenAI
from typing import Any
import json

client = OpenAI()

# SUBAGENTS: specialized agents with specific tool sets
def research_agent(task: str) -> str:
    """Subagent specialized in web research."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",  # cheaper model for subagents
        messages=[
            {"role": "system", "content": "You are a research specialist. Find accurate information using web search."},
            {"role": "user", "content": task},
        ],
        tools=[{"type": "function", "function": {"name": "search_web", ...}}],
    )
    return response.choices[0].message.content

def analysis_agent(task: str, data: str) -> str:
    """Subagent specialized in data analysis."""
    response = client.chat.completions.create(
        model="gpt-4o",  # stronger model for analysis
        messages=[
            {"role": "system", "content": "You are a data analyst. Analyze data and provide insights."},
            {"role": "user", "content": f"Task: {task}\n\nData to analyze:\n{data}"},
        ],
    )
    return response.choices[0].message.content

# ORCHESTRATOR: decides which subagent to use
ORCHESTRATOR_TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "delegate_to_research",
            "description": "Delegate a research task to the research subagent. Use when you need to find information.",
            "parameters": {
                "type": "object",
                "properties": {"task": {"type": "string", "description": "The research task"}},
                "required": ["task"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "delegate_to_analysis",
            "description": "Delegate data analysis to the analysis subagent. Use when you have data and need insights.",
            "parameters": {
                "type": "object",
                "properties": {
                    "task": {"type": "string"},
                    "data": {"type": "string", "description": "The data to analyze"},
                },
                "required": ["task", "data"],
            },
        },
    },
]

def orchestrator(user_query: str) -> str:
    """Orchestrator that delegates to specialized subagents."""
    messages = [
        {
            "role": "system",
            "content": (
                "You are an orchestrator that coordinates specialized agents. "
                "Break complex tasks into subtasks and delegate to the appropriate specialist. "
                "Synthesize results from multiple agents into a final comprehensive answer."
            ),
        },
        {"role": "user", "content": user_query},
    ]

    collected_results = {}

    for _ in range(5):  # max orchestration steps
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=messages,
            tools=ORCHESTRATOR_TOOLS,
            tool_choice="auto",
        )
        msg = response.choices[0].message
        messages.append(msg)

        if not msg.tool_calls:
            return msg.content  # Final synthesis

        for tc in msg.tool_calls:
            args = json.loads(tc.function.arguments)
            if tc.function.name == "delegate_to_research":
                result = research_agent(args["task"])
            elif tc.function.name == "delegate_to_analysis":
                result = analysis_agent(args["task"], args["data"])
            else:
                result = "Unknown subagent"

            collected_results[tc.function.name] = result
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})

    return "Orchestration exceeded maximum steps"
```

---

## Q4: Implement agent observability with LangSmith tracing

```python
from langsmith import Client, traceable
from langsmith.run_helpers import get_current_run_tree
import os

os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "lsv2_..."
os.environ["LANGCHAIN_PROJECT"] = "production-agent"

ls_client = Client()

@traceable(name="agent_tool_call", run_type="tool")
def traced_tool_call(tool_name: str, arguments: dict) -> str:
    """Wrapper that automatically creates a LangSmith span for each tool call."""
    result = execute_tool(tool_name, arguments)

    # Add metadata to the current run
    run_tree = get_current_run_tree()
    if run_tree:
        run_tree.add_metadata({
            "tool_name": tool_name,
            "success": not result.startswith("Error"),
        })
    return result

@traceable(name="react_agent_run", run_type="chain")
def traced_react_agent(user_query: str) -> str:
    """Main agent run — creates a parent trace containing all child spans."""
    # All nested @traceable calls become child spans of this parent
    return react_agent(user_query)

# What you see in LangSmith UI:
# Parent trace: react_agent_run (total time, total tokens, final output)
#   ├── LLM call (gpt-4o, 850 tokens, 230ms)
#   ├── tool_call: search_web (45ms)
#   ├── LLM call (gpt-4o, 1200 tokens, 310ms)
#   ├── tool_call: calculate (2ms)
#   └── LLM call (gpt-4o, 650 tokens, 180ms) → final answer

# Production metrics to monitor:
AGENT_METRICS = {
    "avg_iterations_per_task": "should be < 7 for most tasks",
    "tool_error_rate": "track per-tool error rates separately",
    "context_overflow_rate": "% of runs hitting max tokens",
    "task_success_rate": "% of runs with non-error final answer",
    "p95_latency_seconds": "95th percentile total agent run time",
    "cost_per_run_usd": "sum of all LLM token costs per run",
}
```

---

## Q5: Design idempotent tools and handle failure modes

```python
import hashlib
import time
from functools import wraps

# IDEMPOTENCY: same inputs always produce same state (safe to retry)

# NON-IDEMPOTENT (dangerous for agents that retry):
def send_email_bad(to: str, subject: str, body: str) -> str:
    """PROBLEM: If the agent retries on timeout, duplicate emails are sent."""
    email_client.send(to=to, subject=subject, body=body)
    return "Email sent"

# IDEMPOTENT (safe to retry):
def send_email_good(to: str, subject: str, body: str, idempotency_key: str | None = None) -> str:
    """Safe to retry — same key = same email, never duplicated."""
    if idempotency_key is None:
        # Generate deterministic key from content
        content = f"{to}:{subject}:{body}"
        idempotency_key = hashlib.sha256(content.encode()).hexdigest()[:16]

    if redis_client.get(f"email_sent:{idempotency_key}"):
        return f"Email already sent (idempotency_key: {idempotency_key})"

    email_client.send(to=to, subject=subject, body=body)
    redis_client.setex(f"email_sent:{idempotency_key}", 86400, "1")  # 24h TTL
    return f"Email sent (idempotency_key: {idempotency_key})"


# FAILURE MODE HANDLING: Circuit breaker for tool retries
class AgentToolCircuitBreaker:
    def __init__(self, max_failures: int = 3, cooldown_seconds: int = 60):
        self.failures = {}
        self.max_failures = max_failures
        self.cooldown_seconds = cooldown_seconds

    def is_open(self, tool_name: str) -> bool:
        if tool_name not in self.failures:
            return False
        count, last_fail = self.failures[tool_name]
        if count >= self.max_failures:
            if time.time() - last_fail > self.cooldown_seconds:
                del self.failures[tool_name]  # Reset after cooldown
                return False
            return True  # Circuit is open — don't call this tool
        return False

    def record_failure(self, tool_name: str):
        count = self.failures.get(tool_name, (0, 0))[0]
        self.failures[tool_name] = (count + 1, time.time())

circuit_breaker = AgentToolCircuitBreaker()

def safe_tool_call(tool_name: str, arguments: dict) -> str:
    if circuit_breaker.is_open(tool_name):
        return f"Tool '{tool_name}' is temporarily unavailable. Try a different approach."
    try:
        result = execute_tool(tool_name, arguments)
        return result
    except Exception as e:
        circuit_breaker.record_failure(tool_name)
        return f"Tool error: {str(e)}. Consider an alternative approach."
```
