# Chapter 08: AI Agents — Follow-Up Traps

## Trap 1: "ReAct is the best agent architecture for complex tasks"

**What most people say:** "ReAct is the standard agent loop — Thought, Action, Observation — and it handles complex multi-step tasks well."

**Correct answer:** ReAct is excellent for short-horizon tasks (3-5 tool calls) but degrades significantly for long-horizon tasks. The fundamental limitation: ReAct re-evaluates the entire situation at every step using only the current context window. For tasks requiring 20+ steps, the early observations become distant and get less attention (lost in the middle), the agent can't easily revise a plan made 15 steps ago, and the context fills up with intermediate observations. Plan-and-Execute is better for long-horizon tasks because the plan is made upfront and the executor has a clear goal at each step. Reflexion is better when you need the agent to learn from failures within the same session. For production, choose architecture based on task horizon, not just familiarity with ReAct.

---

## Trap 2: "Parallel tool calls make agents faster — always use them"

**What most people say:** "OpenAI and Anthropic both support parallel tool calls, so you should always enable parallelism to reduce latency."

**Correct answer:** Parallel tool calls are only safe when the tools are truly independent — when the output of one doesn't affect the input or validity of another. If tools have dependencies (e.g., first look up customer_id, then use customer_id to fetch orders), parallelizing them causes the agent to hallucinate the customer_id for the second call. The correct pattern: (1) Identify which tools can run in parallel (no data dependencies); (2) Group independent tools into parallel batches; (3) Sequence dependent tools. The LLM itself will determine parallelism from the schema if tools are well-described, but you must verify it's making correct groupings. Also: parallel calls multiply your rate limit consumption — 10 parallel API calls = 10x token consumption simultaneously.

```python
# SAFE to parallelize: these are truly independent
parallel_batch_1 = [
    tool_call("get_weather", {"city": "New York"}),
    tool_call("get_weather", {"city": "London"}),
    tool_call("get_exchange_rate", {"from": "USD", "to": "GBP"}),
]

# UNSAFE to parallelize: second depends on first
sequential = [
    tool_call("lookup_customer", {"email": "user@example.com"}),
    # THEN use the returned customer_id:
    tool_call("get_orders", {"customer_id": "<result from above>"}),
]
```

---

## Trap 3: "Agent memory = RAG over past conversations"

**What most people say:** "For long-term agent memory, just embed all past conversations and retrieve relevant ones."

**Correct answer:** RAG over past conversations is only one type of agent memory, and often not the most useful. The full memory taxonomy: (1) In-context/working memory — the current conversation, managed by summarization/compression; (2) Episodic memory — retrievable past interactions (this IS where RAG is used); (3) Semantic memory — structured facts about entities (e.g., "User prefers Python, works at Acme, last project was X") — better stored in a key-value or graph store than as raw embedding; (4) Procedural memory — learned tool usage patterns and success strategies — encoded in fine-tuned weights or in-context few-shot examples. The trap: people implement RAG over raw conversation transcripts and wonder why retrieval is poor. The fix: extract structured facts from conversations before storing (semantic memory), and store full interaction records separately for episodic retrieval.

---

## Trap 4: "Agents should always explain their reasoning in thoughts"

**What most people say:** "In ReAct, the Thought step makes the agent more reliable because it reasons before acting."

**Correct answer:** The Thought step in ReAct is a form of chain-of-thought prompting and does improve reasoning quality — but it also has costs and failure modes. (1) Thoughts cost tokens — for high-frequency agent tasks, the cost can be significant; (2) Thoughts are not always accurate self-reflections — the LLM can generate a convincing-sounding Thought and then take an action that contradicts it; (3) Thoughts can lead the agent astray via overthinking — for simple tasks, forcing a Thought before each action can cause the agent to question obvious next steps. In production, many successful agent systems don't use the Thought step for all actions — they use it selectively for complex reasoning and skip it for simple tool calls. The Thought step is a training wheel, not a requirement.

---

## Trap 5: "LangGraph is just LangChain with graph visualization"

**What most people say:** "LangGraph adds a visual graph editor to LangChain workflows — it's basically LangChain LCEL with better visualization."

**Correct answer:** LangGraph is a fundamentally different execution model from LangChain's chain-based LCEL. LCEL (LangChain Expression Language) is a *linear pipe* — data flows from left to right through a sequence of runnables. LangGraph is a *state machine* — nodes can be visited multiple times (cycles), edges can be conditional, and the state persists across multiple node invocations within the same graph run. This enables agent loops: the agent node runs, decides to call a tool, the tool node runs, and control returns to the agent node — this cycle repeats indefinitely until the agent decides to end. LCEL cannot represent this cycle natively. LangGraph's `StateGraph` with `MessageState`, conditional edges, and `END` termination is specifically designed for agent loops with shared mutable state. The key insight: if your workflow has a cycle (any form of iteration or retry), you need LangGraph, not LCEL.

```python
# LangGraph state machine vs LCEL chain:

# LCEL: linear, no cycles
chain = prompt | llm | tool | output_parser  # A → B → C → D, fixed

# LangGraph: cycles allowed, conditional routing
from langgraph.graph import StateGraph, END
graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_executor_node)
graph.add_conditional_edges(
    "agent",
    should_continue,  # returns "tools" or END
    {"tools": "tools", "end": END}
)
graph.add_edge("tools", "agent")  # CYCLE: tools → agent → tools → ...
```

---

## Trap 6: "Human-in-the-loop means the human approves every action"

**What most people say:** "HITL means adding an approval step before each tool call."

**Correct answer:** HITL is a spectrum, and approving every action is the most restrictive (and most unusable) implementation. Effective HITL is risk-stratified: (1) Low-risk actions (read-only, reversible) run automatically; (2) Medium-risk actions (writes to non-critical systems) get confirmation with a brief description; (3) High-risk actions (deletes, financial transactions, permission changes) require explicit approval with full context shown to the user; (4) Catastrophic-risk actions (mass delete, production deployment) require out-of-band approval (a separate confirmation email/SMS). The implementation pattern: each tool is tagged with a risk level, and the agent loop checks risk level before execution. Never interrupt for low-risk operations — users will disable HITL entirely if they get too many prompts.

```python
from enum import Enum

class RiskLevel(Enum):
    READ = "read"          # auto-execute
    WRITE = "write"        # show description, 3s window to cancel
    DESTRUCTIVE = "destructive"  # require explicit "yes" from user
    CRITICAL = "critical"  # require out-of-band MFA confirmation

TOOL_RISK_MAP = {
    "search_docs": RiskLevel.READ,
    "read_file": RiskLevel.READ,
    "create_ticket": RiskLevel.WRITE,
    "update_customer": RiskLevel.WRITE,
    "delete_file": RiskLevel.DESTRUCTIVE,
    "process_refund": RiskLevel.DESTRUCTIVE,
    "deploy_to_production": RiskLevel.CRITICAL,
}
```

---

## Trap 7: "More tools = more capable agent"

**What most people say:** "Give the agent access to as many tools as possible to handle any situation."

**Correct answer:** More tools does NOT equal more capable — it often means *less* capable. Problems with tool overload: (1) The tool selection problem becomes harder — with 50 tools, the agent spends more tokens reasoning about which tool to use; (2) Tool description tokens consume context — 50 tools with good descriptions can consume 3,000-5,000 tokens before the agent generates a single thought; (3) The agent may call obscure tools incorrectly because it knows they exist; (4) Testing and debugging becomes exponentially harder. Best practice: specialized agents with 5-8 well-designed tools are more reliable than a single generalist agent with 30 tools. If you need broad capability, use a routing layer that selects the right specialized agent for the task, rather than giving one agent every tool.

---

## Trap 8: "Agent observability is just logging tool calls"

**What most people say:** "For agent observability, log every tool call with inputs and outputs to a database."

**Correct answer:** Tool call logging is necessary but not sufficient. True agent observability requires: (1) **Trace structure**: every agent run is a parent trace with child spans for each reasoning step and tool call — you need to see the full tree, not just a flat log; (2) **LLM call logging**: the LLM's inputs (full prompt) and outputs (full response) for every reasoning step, not just the final answer; (3) **Token counting and cost tracking**: per-step token usage to diagnose cost anomalies; (4) **Latency breakdown**: which steps are slow (LLM calls vs. tool execution); (5) **Error classification**: distinguish LLM errors, tool errors, timeout errors, and validation errors; (6) **Evaluation hooks**: attach HITL feedback and automated evaluation scores to traces. LangSmith and Arize Phoenix implement all of these. A flat log table misses the hierarchical structure needed to debug agent failures.

---

## Trap 9: "CrewAI agents are truly autonomous — they run independently"

**What most people say:** "In CrewAI, you define agents with roles and they autonomously collaborate to complete tasks."

**Correct answer:** CrewAI agents are LLM calls with tool access — they are not truly autonomous in the sense of running independently in parallel by default. In sequential process mode, tasks run one after another (sequential LLM calls), not concurrently. Hierarchical process mode introduces a manager LLM that assigns tasks to agents (more LLM calls). The "autonomy" is that the LLM decides what tools to call within each task — but the task sequence and agent assignments are fixed at Crew definition time. CrewAI is excellent for structured workflows with well-defined roles but is not suitable for dynamic task allocation or truly parallel multi-agent collaboration. For dynamic parallelism, LangGraph or custom asyncio-based multi-agent systems are better choices.

---

## Trap 10: "Subagent sandboxing is only needed for code execution"

**What most people say:** "You only need sandboxing if the agent can run arbitrary code."

**Correct answer:** Sandboxing is needed for any agent that can: (1) access the filesystem (can read SSH keys, API credentials, .env files); (2) make network requests (can exfiltrate data, call external services); (3) spawn subprocesses; (4) access environment variables (often contain secrets). Even a "simple" agent that reads Markdown files from a directory could be compromised by a malicious file that tricks it into reading `~/.ssh/id_rsa` and exfiltrating it via a benign-looking "search" tool. Sandboxing layers for agents: network isolation (allow only whitelisted domains), filesystem isolation (chroot or container, read-only except specific directories), process isolation (containerized execution), resource limits (CPU, memory, time), capability restrictions (no raw socket access). For agents that DON'T run code but do use tools that touch external systems, the "sandbox" is the tool permission model — least-privilege service accounts, read-only modes, domain allowlists.
