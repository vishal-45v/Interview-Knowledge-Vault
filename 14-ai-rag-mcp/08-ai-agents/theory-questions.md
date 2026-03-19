# Chapter 08: AI Agents — Theory Questions

## Agent Architectures

1. Explain the ReAct (Reasoning + Acting) loop. Walk through the Thought → Action → Observation → Thought cycle. What are the LLM prompting patterns that make ReAct work? What are its failure modes under what conditions?

2. Compare Plan-and-Execute architecture to ReAct. In Plan-and-Execute, a planner LLM generates a step-by-step plan, and an executor LLM executes each step. What are the advantages over ReAct? When does the planning step fail, and how do you handle plan invalidation mid-execution?

3. What is Reflexion? How does it differ from standard ReAct? Describe the three memory stores Reflexion uses (in-context trajectory, episodic memory, semantic memory) and how verbal reinforcement replaces gradient-based learning.

4. How does OpenAI's parallel tool calling work? What is the JSON format for requesting multiple tool calls simultaneously? What is the correct way to handle parallel tool call results when passing them back to the model?

5. Compare Anthropic's `tool_use` content block format to OpenAI's `function` calling format. What are the structural differences? How does each handle the tool result in the conversation history?

## Memory Types

6. Define the four agent memory types: in-context (working memory), external/vector (long-term retrieval), episodic (past interaction recall), semantic (fact storage), and procedural (learned behaviors). For each, give a concrete implementation example.

7. What is the "lost in the middle" problem for agent memory, and how does it affect agents with large tool call histories? What architectural patterns address this: context compression, summarization, memory distillation?

8. Describe how you would implement episodic memory for an agent — the ability to recall past conversations with a specific user. What data structure would you use, how would you index it, and how would you decide when to retrieve past episodes?

## Multi-Agent Systems

9. Explain the orchestrator-subagent pattern in multi-agent systems. How does the orchestrator decide which subagent to invoke? What information does the subagent receive, and how do results flow back to the orchestrator?

10. What is the difference between supervisor patterns and swarm patterns in multi-agent architectures? When is each appropriate? What are the coordination failure modes unique to multi-agent systems (deadlock, conflicting actions, redundant work)?

11. Describe LangGraph's state machine approach to agent orchestration. What are nodes, edges, and state? How do conditional edges work? How does LangGraph handle cycles (loops) compared to linear chains?

12. What is CrewAI and how does it model agents? Describe the Agent, Task, Tool, and Crew primitives. How does CrewAI handle sequential vs. hierarchical process types?

## Tool Design & Failure Modes

13. What is idempotency in the context of agent tools, and why is it critical? Provide a concrete example of a non-idempotent tool that causes problems when the agent retries, and show how you would make it idempotent.

14. Describe the five most common agent failure modes: infinite loops, hallucinated tool calls, context overflow, tool result misinterpretation, and compounding errors. For each, provide a detection strategy and a mitigation pattern.

15. What is schema precision in tool design? Give an example of a poorly-specified tool schema that leads to incorrect LLM behavior and show how tightening the schema fixes the problem.

16. What are human-in-the-loop (HITL) patterns in agent systems? Describe three distinct HITL integration points (pre-execution approval, mid-execution pause, post-execution review) and when each is appropriate.

17. Explain agent observability. What is a trace in LangSmith? What is a span? How do you instrument a custom agent to emit traces? What metrics do you monitor in production agent systems?

18. How do you evaluate agent performance? Contrast: (a) step-level evaluation (individual tool call correctness), (b) trajectory evaluation (complete reasoning path quality), and (c) outcome evaluation (did the agent achieve the goal). What are the tradeoffs of each?
