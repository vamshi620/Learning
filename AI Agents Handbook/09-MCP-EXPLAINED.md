# Generative AI & Agents
## File 09: Model Context Protocol (MCP) Explained

---

## What You'll Learn
- What the Model Context Protocol (MCP) is and its April 2026 governance status.
- The new **Agent-to-Agent (A2A) Protocol** and how it pairs with MCP.
- How they standardize tool calling across different AI assistants (Claude, Copilot, Cursor, VS Code Agent Mode).
- Setting up MCP servers in Claude Desktop and GitHub Copilot.

**Time Required:** 30 minutes

---

## 1. The "N to M" Integration Problem

In [File 08](./08-CLAUDE-AGENTS-SKILLS.md), we wrote a custom Python script to define a tool (`get_stock_price`) and passed it to the Anthropic API. 

But what if you want to use that exact same tool inside **Claude Desktop**? What if you want to use it in **Cursor**? What if you want to use it in a **Semantic Kernel .NET app**?

Historically, you would have to rewrite the tool schema and integration logic for every single AI application you use. This is the "N to M" integration problem.

---

## 2. Enter MCP — Now an Industry Standard (April 2026)

The **Model Context Protocol (MCP)** is an open standard now governed by the **Linux Foundation** (not just Anthropic). As of April 2026, it is supported by OpenAI, Microsoft, Google, Anthropic, and hundreds of community contributors.

Think of MCP like **USB-C for AI Agents** — a universal connector between agents and tools.

- **MCP 2026 focus areas:** Stateless Streamable HTTP transport (for hyperscale), SSO-integrated auth, enterprise audit trails, and `.well-known` discovery endpoints so agents can find MCP servers without hardcoded URLs.

Instead of writing tools specifically for one app, you build an **MCP Server**.
- The MCP Server exposes tools (Functions) and resources (Files, Databases).
- Any **MCP Client** (like Claude Desktop, Cursor, or your own custom app) can connect to the server and instantly gain access to all those tools, without needing to know how they work under the hood.

---

## 3. Architecture of MCP

### The MCP Server
A lightweight server (often written in TypeScript or Python) that securely connects to your local machine or enterprise systems.
*Examples of MCP Servers:*
- **PostgreSQL MCP:** Allows an AI to read your database schema and run SQL queries.
- **GitHub MCP:** Allows an AI to read PRs, create issues, and push code.
- **File System MCP:** Allows an AI to read and write files on your local hard drive.

### The MCP Client
The AI application the user interacts with. It maintains the connection to the LLM.
*Examples of MCP Clients in April 2026:*
- Claude Desktop App
- Cursor IDE
- GitHub Copilot (native MCP support in Agentic Mode)
- VS Code Agent Mode
- Microsoft Agent Framework (natively supports MCP for tool discovery)

### The Handshake
1. The MCP Client connects to the MCP Server over `stdio` (standard input/output) or SSE (Server-Sent Events).
2. The Server says: *"I have these 3 tools available: `read_file`, `write_file`, and `search_files`."*
3. The Client passes these tools to the LLM.
4. When the LLM decides to use a tool, the Client sends an execution request back to the Server. The Server runs the code and returns the result.

---

## 4. Why Enterprise IT Loves MCP

Before MCP, giving an AI access to a company database meant uploading company data into the AI application's cloud. 

**With MCP, the data never leaves the boundary.** 
If an enterprise runs an internal MCP Server connected to their private HR database, the AI model (running in Azure OpenAI) only ever receives the specific records it asks for via the MCP tool call. The LLM acts as the "brain" routing requests, but the actual execution happens securely on the MCP Server inside the company firewall.

---

## 5. Setting up an MCP Server in Claude Desktop

If you have the Claude Desktop App installed on Windows or Mac, you can connect it to MCP servers right now.

1. Open your Claude Desktop config file (usually at `%APPDATA%\Claude\claude_desktop_config.json` on Windows).
2. Add an MCP Server definition:

```json
{
  "mcpServers": {
    "sqlite": {
      "command": "uvx",
      "args": ["mcp-server-sqlite", "--db-path", "C:/my-database.db"]
    }
  }
}
```
3. Restart Claude Desktop.
4. You will see a "Hammer" icon in Claude indicating it is connected.
5. You can now type: *"Look at my local sqlite database and tell me how many users signed up yesterday."* Claude will autonomously query your local database!

---

## 6. The A2A Protocol — Agent-to-Agent Communication (April 2026)

MCP solves **agent-to-tool** communication (vertical). But what about **agent-to-agent** communication (horizontal)?

The **Agent-to-Agent (A2A) Protocol** (v1.0, Linux Foundation, 2026) is the complementary standard that allows independent AI agents across different systems to collaborate as peers.

| | MCP | A2A |
|---|-----|-----|
| **Direction** | Agent → Tool | Agent → Agent |
| **Relationship** | Hierarchical (agent controls tool) | Peer-to-peer |
| **Discovery** | Tool manifests | **Agent Cards** (capability advertisement) |
| **Best For** | DB queries, file access, API calls | Task delegation, parallel agent workflows |

### How A2A Works
1. Each agent publishes an **Agent Card** — a JSON file describing its capabilities, input schema, and endpoint.
2. A **Planner Agent** discovers specialist agents via their Agent Cards.
3. The Planner delegates subtasks to specialist agents using the A2A protocol.
4. Each specialist agent uses MCP internally to access the tools it needs.

**The combined pattern (April 2026):**
```
User → Planner Agent (A2A) → DB Agent → MCP → Internal Database
                          ↘ Email Agent → MCP → Graph/Outlook API
                          ↘ Code Agent → MCP → GitHub Repo
```

### A2A in Microsoft Agent Framework 1.0
```csharp
// The Microsoft Agent Framework 1.0 natively supports A2A
// Agents can advertise capabilities and accept delegated tasks

[AgentCapability("code-review", "Reviews C# code for quality and security")]
public class CodeReviewAgent : IAgentHandler
{
    public async Task<AgentResponse> HandleAsync(AgentTask task, AgentContext ctx)
    {
        // Process the delegated task
        var code = task.GetInput<string>("code");
        var review = await _chatClient.GetResponseAsync(
            $"Review this C# code:\n{code}");
        return AgentResponse.Success(review.Text);
    }
}
```

---

## 🧪 Exercise: Explore the MCP Registry

The open-source community is building hundreds of MCP servers.
1. Go to [https://github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) (or search for the official MCP registry).
2. Look at the code for the `fetch` or `github` MCP servers. Notice how they standardize tool definitions so any AI client can use them.

---

**Next:** [10-AZURE-AI-AGENTS.md](./10-AZURE-AI-AGENTS.md) — Let's bring everything together and build a fully autonomous AI Agent using the ReAct pattern in Azure.
