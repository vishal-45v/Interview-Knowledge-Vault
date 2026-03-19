# Chapter 08: AI Agents — Diagram Explanations

## Diagram 1: ReAct Loop vs. Plan-and-Execute Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│              REACT LOOP vs. PLAN-AND-EXECUTE                           │
└─────────────────────────────────────────────────────────────────────────┘

REACT (Reasoning + Acting):
─────────────────────────────

  User Query
      │
      ▼
  ┌─────────────────────────────────────────────────────────────┐
  │                    SINGLE LLM                               │
  │                                                             │
  │  Thought: "I need to find X first"                         │
  │      │                                                      │
  │      ▼                                                      │
  │  Action: tool_call("search", {"q": "X"})   ◄───────────┐   │
  │      │                                                  │   │
  │      ▼                                                  │   │
  │  Observation: "X = 42"                                  │   │
  │      │                                                  │   │
  │      ▼                                                  │   │
  │  Thought: "Now I need Y, which depends on X"            │   │
  │      │                                                  │   │
  │      ▼                                                  │   │
  │  Action: tool_call("calculate", {"x": 42}) ─────────────┘   │
  │      │                                                      │
  │      ▼                                                      │
  │  Final Answer (when LLM stops calling tools)               │
  └─────────────────────────────────────────────────────────────┘

  ✓ Simple to implement   ✗ Fails on long tasks (lost in middle)
  ✓ Adaptive             ✗ Can't revise strategy midway
  ✓ Works well <8 steps  ✗ Expensive (many LLM calls)

─────────────────────────────────────────────────────────────────────────

PLAN-AND-EXECUTE:
──────────────────

  User Query
      │
      ▼
  ┌────────────────────────┐
  │    PLANNER LLM         │  ← Strong model (GPT-4o / Claude 3.5)
  │                        │
  │  Output: [Step 1,      │
  │           Step 2,      │
  │           Step 3,      │
  │           Step 4]      │
  └─────────────┬──────────┘
                │ Plan
                ▼
  ┌────────────────────────────────────────────────────────────┐
  │                    EXECUTOR LLM                            │
  │              (can be a cheaper/smaller model)              │
  │                                                            │
  │  For each step in plan:                                    │
  │    [Step 1] → tool calls → result₁                       │
  │    [Step 2] + result₁ → tool calls → result₂             │
  │    [Step 3] + result₁,₂ → tool calls → result₃           │
  │    [Step 4] → synthesize all results → Final Answer        │
  └────────────────────────────────────────────────────────────┘

  ✓ Better for 10+ step tasks   ✗ Requires re-planning on failure
  ✓ Plan = built-in progress tracking  ✗ Initial plan may be wrong
  ✓ Can use cheaper executor model     ✗ More complex implementation
```

---

## Diagram 2: LangGraph State Machine — Agent with Tool Loop

```
┌─────────────────────────────────────────────────────────────────────────┐
│                LANGGRAPH AGENT STATE MACHINE                           │
└─────────────────────────────────────────────────────────────────────────┘

STATE:
┌────────────────────────────────────────────────────────────────────┐
│  AgentState = {                                                    │
│    "messages": [...],        # accumulated conversation history    │
│    "iteration_count": 0,     # safety counter                     │
│    "tool_results": {},       # keyed by tool call ID              │
│    "final_answer": None      # set when agent is done            │
│  }                                                                 │
└────────────────────────────────────────────────────────────────────┘

GRAPH STRUCTURE:
                    START
                      │
                      ▼
               ┌─────────────┐
               │  AGENT NODE │ ← LLM call with tools bound
               │             │   reads messages from state
               │             │   appends LLM response to state
               └──────┬──────┘
                      │
            ┌─────────▼─────────┐
            │  CONDITIONAL EDGE │
            │  should_continue()?│
            └─────────┬─────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
     "tools"      "end"       "max_iter"
          │            │            │
          ▼            ▼            ▼
   ┌────────────┐   [END]    [FORCE END]
   │ TOOLS NODE │            (safety limit)
   │            │
   │ Executes   │
   │ all tool   │
   │ calls in   │
   │ parallel   │
   │            │
   │ Appends    │
   │ results to │
   │ state      │
   └─────┬──────┘
         │
         └────────────► AGENT NODE (loop back)

KEY: The cycle TOOLS → AGENT → TOOLS is what makes this a graph,
     not a simple chain. LCEL cannot represent this cycle.

CONDITIONAL EDGE LOGIC:
  def should_continue(state: AgentState) -> str:
      last_msg = state["messages"][-1]
      if state["iteration_count"] > 10: return "max_iter"
      if last_msg.tool_calls: return "tools"
      return "end"
```

---

## Diagram 3: Multi-Agent Orchestrator Pattern

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  MULTI-AGENT ORCHESTRATOR-SUBAGENT                     │
└─────────────────────────────────────────────────────────────────────────┘

User: "Research Tesla's Q3 2024 earnings, analyze growth vs competitors,
       and write an executive summary"

                    ┌───────────────────────────────────┐
                    │         ORCHESTRATOR               │
                    │   (GPT-4o / Claude 3.5 Sonnet)    │
                    │                                   │
                    │  Decomposes task into subtasks:   │
                    │  1. Research → research_agent      │
                    │  2. Analyze  → analysis_agent      │
                    │  3. Write    → writing_agent       │
                    └──────────────┬────────────────────┘
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
  │  RESEARCH AGENT │  │ ANALYSIS AGENT  │  │  WRITING AGENT  │
  │                 │  │                 │  │                 │
  │ Model: gpt-4o-  │  │ Model: gpt-4o   │  │ Model: gpt-4o-  │
  │ mini (cheaper)  │  │ (strong model)  │  │ mini (faster)   │
  │                 │  │                 │  │                 │
  │ Tools:          │  │ Tools:          │  │ Tools:          │
  │ - search_web    │  │ - calculate     │  │ (no tools)      │
  │ - fetch_url     │  │ - chart_data    │  │ synthesis only  │
  │ - read_pdf      │  │                 │  │                 │
  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘
           │                    │                    │
           │ Raw data           │ Analysis results   │ (waits for
           │                    │                    │  both above)
           └────────────────────┴────────────────────┘
                                │
                                ▼
                    ┌───────────────────────────────────┐
                    │         ORCHESTRATOR               │
                    │  Synthesizes all results           │
                    │  Routes writing_agent the data     │
                    │  Receives final summary            │
                    └───────────────────────────────────┘
                                │
                                ▼
                         Final Answer to User

DATA FLOW PROTOCOL (messages between agents):
┌────────────────────────────────────────────────────────────┐
│ {                                                          │
│   "task_id": "T-001",                                     │
│   "assigned_to": "research_agent",                        │
│   "instructions": "Research Tesla Q3 2024 earnings...",   │
│   "context": {},   ← what orchestrator shares             │
│   "output_format": "structured JSON with sources"         │
│ }                                                          │
└────────────────────────────────────────────────────────────┘
```

---

## Diagram 4: Agent Failure Modes and Recovery

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  AGENT FAILURE MODES AND MITIGATIONS                   │
└─────────────────────────────────────────────────────────────────────────┘

FAILURE MODE 1: INFINITE LOOP
─────────────────────────────
  Agent: "I need more info" → search → "still need more" → search → ...

  Detection:
  ┌─────────────────────────────────────────────────────────────┐
  │ if len([m for m in messages if m.role=="tool"]) > 15:       │
  │     inject_message("You have been searching for a long      │
  │                     time. Synthesize what you have and      │
  │                     provide a best-effort answer now.")      │
  └─────────────────────────────────────────────────────────────┘

FAILURE MODE 2: HALLUCINATED TOOL CALLS
────────────────────────────────────────
  Agent calls: tool_call("get_revenue_2025", {"q": "Tesla"})
  But "get_revenue_2025" doesn't exist!

  Detection + Response:
  ┌─────────────────────────────────────────────────────────────┐
  │ if tool_name not in registered_tools:                       │
  │     return {"error": f"Tool '{tool_name}' does not exist.  │
  │                        Available tools: {tool_list}"}       │
  └─────────────────────────────────────────────────────────────┘

FAILURE MODE 3: CONTEXT OVERFLOW
──────────────────────────────────
  After 20 tool calls, context window is full.
  Agent starts ignoring early context (lost in middle).

  Mitigation: Progressive Summarization
  ┌─────────────────────────────────────────────────────────────┐
  │  [System Prompt]     ← NEVER compress                      │
  │  [Original Query]    ← NEVER compress                      │
  │  [Summary of steps 1-10] ← compressed from 8000 → 300 tok │
  │  [Steps 11-15 in full]   ← recent context preserved        │
  │  [Current step]          ← newest, always kept             │
  └─────────────────────────────────────────────────────────────┘

FAILURE MODE 4: COMPOUNDING ERRORS
────────────────────────────────────
  Step 1: Wrong customer_id retrieved (error)
  Step 2: Used wrong customer_id to fetch orders (propagates error)
  Step 3: Analyzed wrong orders (error compounds)
  Step N: Final answer is completely wrong despite no single obvious failure

  Mitigation: Validation checkpoints between steps
  ┌─────────────────────────────────────────────────────────────┐
  │  After each critical step, inject a verification prompt:    │
  │  "Before proceeding, confirm: is the customer ID 'C_42'     │
  │   correct for user 'alice@example.com'? If uncertain,       │
  │   re-run the customer lookup."                              │
  └─────────────────────────────────────────────────────────────┘

FAILURE DETECTION DASHBOARD:
┌─────────────────────────────────────────────────────────────────────────┐
│  Metric                    │ Threshold    │ Alert Action               │
├─────────────────────────────────────────────────────────────────────────┤
│  Iterations > 10           │ warn at 8    │ Inject synthesis prompt    │
│  Same tool called 3x       │ identical args│ Inject "try different approach" │
│  Tool error rate > 30%     │ in one run   │ Circuit break, notify human│
│  Total tokens > 80% limit  │ of context   │ Trigger compression        │
│  Run time > 120s           │ hard limit   │ Kill & return partial result│
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Diagram 5: Agent Memory Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE AGENT MEMORY SYSTEM                        │
└─────────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────────────────┐
                    │         AGENT (LLM CORE)            │
                    │                                     │
                    │  Current context window:            │
                    │  ┌───────────────────────────────┐  │
                    │  │ System prompt                 │  │
                    │  │ User message                  │  │◄── IN-CONTEXT
                    │  │ Tool call 1 + result          │      WORKING MEMORY
                    │  │ Tool call 2 + result          │      (~100K tokens)
                    │  │ ...                           │  │
                    │  └───────────────────────────────┘  │
                    └──────────────┬──────────────────────┘
                                   │ queries
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
           ▼                       ▼                       ▼
┌────────────────────┐  ┌─────────────────────┐  ┌──────────────────────┐
│  EPISODIC MEMORY   │  │  SEMANTIC MEMORY     │  │  PROCEDURAL MEMORY  │
│                    │  │                      │  │                     │
│  Vector DB of      │  │  Structured facts    │  │  In model weights   │
│  past sessions     │  │  Key-value store or  │  │  (fine-tuning) OR   │
│                    │  │  graph database      │  │  few-shot in        │
│  "Last week,       │  │                      │  │  system prompt      │
│  user asked about  │  │  user_prefs["alice"] │  │                     │
│  billing dispute"  │  │  = {lang: "Python",  │  │  "When user asks    │
│                    │  │    tier: "enterprise"}│  │  about billing,     │
│  Retrieval:        │  │                      │  │  always check       │
│  embed query →     │  │  Retrieval:          │  │  subscription       │
│  find similar      │  │  direct lookup by    │  │  status first"      │
│  past sessions     │  │  entity key          │  │                     │
└────────────────────┘  └─────────────────────┘  └──────────────────────┘

WRITE PATTERNS:
  End of session → extract facts → update Semantic Memory
  End of session → summarize → embed → store in Episodic Memory
  After N sessions → identify patterns → fine-tune or update prompt → Procedural Memory

READ PATTERNS:
  Start of session → query Episodic + Semantic with user context
  → inject relevant memories into system prompt or early messages
  → reduces need to re-ask for information the agent already knows
```
