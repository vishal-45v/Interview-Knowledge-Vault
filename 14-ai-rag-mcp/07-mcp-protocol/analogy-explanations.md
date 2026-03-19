# Chapter 07: MCP Protocol — Analogy Explanations

## Analogy 1: MCP is Like a Universal Remote Control Standard

**The Story:**
Before universal remote controls existed, every TV, DVD player, and cable box used a different proprietary remote. If you bought a Sony TV, LG DVD player, and Comcast cable box, you needed three different remotes. Manufacturers were forced to build their own remote for every device. Consumers were frustrated. Then came universal remote standards (like HDMI-CEC) — a shared protocol that allowed any compliant remote to control any compliant device.

MCP is the universal remote standard for AI models and external tools. Before MCP, every AI application (Claude, ChatGPT, Copilot, etc.) implemented its own proprietary integration format. A company building a "search our database" tool had to build it four times — once for each AI platform. With MCP, they build one MCP server and every compliant AI host (Claude Desktop, Claude Code, future AI apps) can use it.

**Connection to MCP:**
```json
// Before MCP: OpenAI-specific function definition
{
  "type": "function",
  "function": {
    "name": "search_db",
    "description": "...",
    "parameters": { ... }
  }
}
// Only works with OpenAI API

// With MCP: one server, multiple clients
// The same MCP server works with:
// - Claude Desktop
// - Claude Code
// - Any future MCP-compatible host
// Build once, run everywhere.
```

---

## Analogy 2: Host/Client/Server is Like a Restaurant

**The Story:**
In a restaurant, there are three distinct roles. The customer (Host) decides what they want and has the final say on the order. The waiter (Client) takes the order, communicates with the kitchen, and brings back the food — the waiter doesn't cook and doesn't decide what's on the menu. The kitchen (Server) prepares the food — it knows its capabilities (what it can make), maintains its own equipment, and responds to orders but doesn't interact with customers directly.

**Connection to MCP:**
- **Customer (Host):** Claude Desktop — manages the user experience, sends messages to Claude, decides which tool calls to authorize, handles the UI.
- **Waiter (Client):** The MCP client embedded in Claude Desktop — maintains the protocol connection, translates tool call requests into JSON-RPC messages, handles transport details.
- **Kitchen (Server):** Your MCP server — exposes tools/resources/prompts, executes the actual logic, reports back results, never directly touches the user.

```
User → [Claude Desktop (Host)] → [MCP Client] → [Your MCP Server]
                                                     ↓
                                              executes tool
                                                     ↓
User ← [Claude generates response] ← [MCP Client] ← result
```

The waiter (client) never makes decisions about what to cook — that's the customer (host) and kitchen (server). Similarly, the MCP client is just a transport layer.

---

## Analogy 3: Tools vs. Resources vs. Prompts — A Library

**The Story:**
Imagine a specialized research library. It has three types of services:

1. **Librarians (Tools):** They perform *actions* — they can order new books, send interlibrary loan requests, scan a document and email it to you, reserve a study room. Librarians do things that change the state of the world.

2. **The Book Collection (Resources):** Books and journals are *read-only data sources* identified by call numbers (like URIs). You can request any book by its call number. The collection doesn't change from the act of reading it. Resource templates are like call number patterns: "PR[4500-4999] refers to Victorian English fiction."

3. **Research Guides (Prompts):** Pre-structured templates that help you ask better research questions. "To write a literature review, start by searching these three databases, then compare findings on these dimensions..." Prompts don't do anything — they structure how you approach a task.

**Connection to MCP:**
```python
# TOOL: Does something, has side effects
@mcp.tool()
def reserve_conference_room(room_id: str, date: str, duration_hours: int) -> str:
    """Reserve a conference room. MODIFIES the room booking system."""
    booking_system.reserve(room_id, date, duration_hours)
    return f"Room {room_id} reserved for {date}"

# RESOURCE: Read-only, identified by URI
@mcp.resource("rooms://{room_id}/availability")
def room_availability(room_id: str) -> str:
    """Current availability schedule for a room. READ-ONLY."""
    return booking_system.get_schedule(room_id)

# PROMPT: Template that structures a task
@mcp.prompt()
def meeting_planning_prompt(attendees: str, duration: str) -> list:
    """A structured prompt for planning a meeting."""
    return [{"role": "user", "content":
             f"Help me find a meeting time for {attendees} that lasts {duration}. "
             "Check everyone's calendar availability first, then suggest 3 time slots."}]
```

---

## Analogy 4: JSON-RPC over stdio is Like Passing Notes Through a Shared Pipe

**The Story:**
Imagine two offices separated by a wall, connected only by a pneumatic tube system (like old-school bank drive-throughs). Office A (the MCP client) puts a note in the tube with a request ID and a question. The note shoots through the tube to Office B (the MCP server). Office B writes the answer on a new note with the matching request ID and shoots it back. For announcements that don't need a reply (notifications), Office A just shoots a note labeled "no response needed."

This is exactly how JSON-RPC 2.0 over stdio works. stdout is the tube from server to client; stdin is the tube from client to server. Each message is a JSON blob followed by a newline.

**Connection to MCP:**
```
Claude Desktop process (MCP client)
    ↓ writes to stdin of subprocess
{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"search","arguments":{"query":"test"}}}

MCP Server subprocess
    ↓ reads from its stdin, processes, writes to its stdout
{"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"3 results found"}]}}

Claude Desktop reads server's stdout → gets the result

IMPORTANT: The server must NEVER print debug logs to stdout!
stdout is the protocol channel. Use stderr for logging:

import sys
print("DEBUG: tool called", file=sys.stderr)  # correct — goes to Claude's logs
print("DEBUG: tool called")                    # WRONG — corrupts the protocol channel
```

---

## Analogy 5: MCP Sampling is Like a Contractor Asking the Homeowner to Sign a Permit

**The Story:**
You hire a contractor to renovate your kitchen. The contractor is skilled and autonomous — but for certain things (pulling a building permit, ordering structural changes), they must come back to you (the homeowner) and get your explicit sign-off. The contractor can't just forge your signature or act unilaterally on decisions that require the owner's authority.

MCP Sampling is exactly this pattern. The MCP server (contractor) sometimes needs to make an LLM call (pull a permit) — it needs the host's LLM capabilities. But the host (homeowner/Claude Desktop) retains authority: it decides whether to approve the LLM call, applies its own safety filters, and may modify the request.

**Connection to MCP:**
```python
# MCP Server code — requesting a sampling (LLM call)
async def summarize_document(text: str, session) -> str:
    # Server asks the HOST to make an LLM call on its behalf
    result = await session.create_message(
        messages=[
            {
                "role": "user",
                "content": {"type": "text", "text": f"Summarize this in 2 sentences:\n\n{text}"}
            }
        ],
        max_tokens=200,
        # Server can suggest model preferences but HOST decides the actual model
        model_preferences={"hints": [{"name": "claude-3-haiku"}]}
    )
    return result.content.text

# The HOST (Claude Desktop) will:
# 1. Show this request to the user (if configured)
# 2. Apply its own system prompt on top
# 3. Use whatever model it deems appropriate
# 4. Return the result to the server
# The server cannot bypass the host's safety controls.
```

---

## Analogy 6: Tool Description Quality is Like a Good API README

**The Story:**
You're a new developer and need to use an internal API. You find two endpoints that seem similar:

- **Endpoint A** documentation: "Handles user data."
- **Endpoint B** documentation: "Retrieves a user's profile by user ID or email address. Returns: name, email, account_created_date, subscription_tier. Rate limited to 100 req/min. Use for single-user lookups. For bulk operations, use `/users/bulk-lookup`."

You instantly know to use Endpoint B and understand its constraints. Endpoint A requires you to explore, make test calls, and potentially make mistakes.

MCP tool descriptions are exactly this. The LLM is the "new developer" reading your API docs. Poor descriptions cause the LLM to call the wrong tool, pass wrong parameters, or miss edge cases.

**Connection to MCP:**
```python
# BAD tool description — the LLM will misuse this
@mcp.tool()
def get_data(query: str) -> str:
    """Gets data."""  # What data? From where? When should I use this?
    ...

# GOOD tool description — precise, actionable, includes guardrails
@mcp.tool()
def search_customer_orders(
    customer_id: str,
    status: str | None = None,
    days_back: int = 30
) -> str:
    """Search a customer's order history by customer ID.

    Returns a JSON list of orders including order_id, date, status, and total_amount.

    - status: filter by order status ('pending', 'shipped', 'delivered', 'cancelled')
    - days_back: how many days of history to return (max 365, default 30)

    Use this for: finding a customer's recent orders, checking order status.
    Do NOT use for: searching orders across all customers (use search_all_orders instead).
    Do NOT use for: getting order line items (use get_order_details instead).
    """
    ...
```

---

## Analogy 7: MCP vs Function Calling is Like a Network Protocol vs an In-Process Library

**The Story:**
Calling a function in your own code (function calling) is like using an in-house accountant — they sit in your office, share your systems, and you can call them any time instantly. But to access a bank's mortgage department, you don't extend your own codebase — you communicate over a standard protocol (phone call, structured forms, wire transfer standards). The bank has its own systems, state, and processes; you interact through a defined interface.

MCP is the "network protocol" approach. Your MCP server is the "bank" — it has its own process, its own state, its own credentials, and communicates through a standard protocol. Function calling is the "in-house accountant" — defined in your code, shares your context, lives in your application.

**Connection to MCP:**
```
Function Calling:                    MCP:
────────────────                     ────────────────
LLM API ←→ Your app code             LLM API
                │                        ↕ (Claude Desktop orchestrates)
           functions defined         MCP Client ←→ MCP Server (separate process)
           IN your code                               │
           share your secrets                    has own secrets
           die when app dies                     can outlive host
           one app only                          reusable across apps
           no standard discovery                 listable via tools/list
```
