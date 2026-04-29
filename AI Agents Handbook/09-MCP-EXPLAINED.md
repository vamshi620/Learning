# Generative AI & Agents
## File 09: Model Context Protocol (MCP) Explained

---

## What You'll Learn
- What the Model Context Protocol (MCP) is.
- Why Anthropic created it.
- How it standardizes tool calling across different AI assistants (Claude Desktop, GitHub Copilot, Cursor).

**Time Required:** 30 minutes

---

## 1. The "N to M" Integration Problem

In [File 08](./08-CLAUDE-AGENTS-SKILLS.md), we wrote a custom Python script to define a tool (`get_stock_price`) and passed it to the Anthropic API. 

But what if you want to use that exact same tool inside **Claude Desktop**? What if you want to use it in **Cursor**? What if you want to use it in a **Semantic Kernel .NET app**?

Historically, you would have to rewrite the tool schema and integration logic for every single AI application you use. This is the "N to M" integration problem.

---

## 2. Enter MCP

The **Model Context Protocol (MCP)** is an open standard created by Anthropic (but adopted by Microsoft, GitHub, and others) to solve this problem.

Think of MCP like **USB-C for AI Agents**.

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
The AI application the user interacts with. It maintains the connection to the LLM (like GPT-4 or Claude 3.5).
*Examples of MCP Clients:*
- Claude Desktop App
- Cursor IDE
- GitHub Copilot (via extensions)

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

## 🧪 Exercise: Explore the MCP Registry

The open-source community is building hundreds of MCP servers.
1. Go to [https://github.com/modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) (or search for the official MCP registry).
2. Look at the code for the `fetch` or `github` MCP servers. Notice how they standardize tool definitions so any AI client can use them.

---

**Next:** [10-AZURE-AI-AGENTS.md](./10-AZURE-AI-AGENTS.md) — Let's bring everything together and build a fully autonomous AI Agent using the ReAct pattern in Azure.
