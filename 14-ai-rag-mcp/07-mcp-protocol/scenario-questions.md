# Chapter 07: MCP Protocol — Scenario Questions

1. You're building an MCP server that gives Claude access to your company's internal Confluence wiki. The wiki has 50,000 pages and real-time updates. Design the full MCP server: what tools vs. resources would you expose, how would you handle search vs. direct page access, and how would you deal with authentication when Claude Desktop runs as a local process with no server-side session management?

2. Your team is considering whether to expose your internal APIs to Claude via MCP Tools or to use standard function calling in the OpenAI API. The APIs include: (a) a customer database (read/write), (b) a real-time pricing engine, and (c) a document storage system. Walk through the architectural decision for each one — MCP, function calling, or neither — and justify your choice.

3. A security audit flags your MCP deployment as having a "confused deputy" vulnerability. The MCP server has a tool called `execute_sql` that runs queries against a production database using the server's service account credentials (which have broad access). A malicious prompt injected into a document could cause Claude to call `execute_sql` with destructive queries. Design the security controls you would add at the MCP layer, the tool layer, and the infrastructure layer.

4. You need to build an MCP server in TypeScript that allows Claude to interact with a Git repository: list files, read file contents, create branches, commit changes, and open pull requests. Walk through: what tools you would define, what resources you would expose, how you would handle the GitHub API authentication, and what error messages you'd return for common failure cases.

5. Your organization has 15 internal MCP servers (database access, document systems, APIs, calendars, etc.). Claude Desktop users are loading all 15 servers simultaneously. Users report that Claude sometimes calls the wrong tool when multiple servers have similarly-named tools (e.g., two servers each have a `search` tool). How do you solve the tool naming conflict problem, and what guidance would you give to server developers to prevent this?

6. You're designing an MCP server that uses the Sampling capability — the server needs to ask Claude to summarize text before storing it. Walk through the complete message flow: how does the server initiate the sampling request, what does the host do with it, how is the result returned, and what are the security implications of allowing a server to trigger LLM calls?

7. A developer asks: "Can I build an MCP server that exposes a tool which calls another MCP server?" This is the "MCP server chaining" question. What are the architectural constraints, how would you implement this using the Python SDK, and what are the risks?

8. You need to migrate an existing LangChain-based agent with 12 custom tools to an MCP-based architecture so it can work with Claude Desktop. What is your migration strategy? Which tools map cleanly to MCP Tools, which would become MCP Resources, and which would require architectural rethinking?

9. Your MCP server is running as a stdio subprocess and users report that it occasionally crashes silently — Claude just says it can't reach the tool. Design a robust MCP server deployment strategy: how do you handle server crashes, restarts, health checking, and error propagation back to the user through Claude?

10. You're evaluating whether to use SSE transport vs. stdio transport for a new MCP server. The server needs to: handle 100 concurrent Claude Desktop sessions, maintain per-user state, integrate with OAuth for Google Drive access. Walk through your transport choice with specific technical justifications and the architecture you would build.
