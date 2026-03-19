# Chapter 07: MCP Protocol — Theory Questions

## Foundations & Architecture

1. What is the Model Context Protocol (MCP), when was it released, and what specific problem does it solve that function calling alone does not? Why did Anthropic open-source it rather than keep it proprietary?

2. Describe the MCP three-tier architecture: Host, Client, and Server. What are the responsibilities of each tier? Can a single process be both a host and a server?

3. MCP uses JSON-RPC 2.0 as its messaging layer. Explain what JSON-RPC 2.0 is, what the request/response structure looks like, and how it handles notifications (one-way messages with no response expected).

4. MCP supports three transport mechanisms: stdio, SSE (Server-Sent Events), and HTTP+SSE. For each, describe: (a) how the process topology looks, (b) when you would use it, and (c) what its limitations are.

5. Explain the MCP lifecycle: initialize → capabilities negotiation → operation → shutdown. What information is exchanged during the initialize handshake? What happens if a client requests a capability the server doesn't support?

## The Three Primitives

6. Describe MCP Tools in detail: What does a tool definition contain? What is the JSON Schema `inputSchema` used for? How does a tool call differ from a tool definition? How are tool results returned?

7. What are MCP Resources? Explain the URI scheme used for resources, resource templates (with URI template syntax like `{param}`), and the difference between `text` and `blob` resource content types.

8. What are MCP Prompts? How do they differ from Tools and Resources? Describe prompt templates with arguments. When would a server expose a Prompt instead of a Tool?

9. Explain MCP Sampling — the mechanism by which an MCP server can request that the host perform an LLM completion. What is the security model for sampling? Why is this considered a server-initiated LLM call?

10. What are MCP Roots? How does a client expose roots to a server? What purpose do roots serve for file-system-based servers?

## Implementation

11. Walk through the Python `mcp` SDK (FastMCP) — how do you define a tool, a resource, and a prompt using the decorator API? What does the server's main entry point look like?

12. Describe how MCP tool input validation works. If a tool's `inputSchema` declares a parameter as `required` and the client omits it, what happens at the protocol level?

13. How does Claude Desktop discover and connect to MCP servers? What does the `claude_desktop_config.json` configuration file contain? How do you pass environment variables to an MCP server?

14. What is the difference between `tools/call` and `tools/list` in the MCP protocol? What JSON-RPC method is used for each? What does the response schema look like for each?

15. Explain MCP security considerations: What is "tool poisoning"? What is the "confused deputy" problem in the context of MCP? How does the MCP spec recommend mitigating these threats?

16. Compare MCP Tools to OpenAI function calling and to OpenAI Plugins. What can MCP do that function calling cannot? What does MCP solve that the Plugins system failed to solve?

17. What is the MCP registry? Is there an official registry? How does MCP server discovery work in practice, and what are the limitations of the current discovery mechanism?
