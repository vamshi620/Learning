# Generative AI & Agents
## File 08: Claude Agents, Tool Calling, and Custom Skills

---

## What You'll Learn
- The difference between a simple Chatbot and an AI Agent.
- What "Tool Calling" (or Function Calling) is.
- How Claude 3.5 Sonnet handles tools.
- How to write custom "Skills" for an agent.

**Time Required:** 45 minutes

---

## 1. Chatbots vs. AI Agents

A **Chatbot** (like ChatGPT or basic Copilot) takes a prompt, searches its training data or a RAG vector database, and returns text. It is entirely passive. It cannot *do* anything.

An **AI Agent** is an LLM equipped with tools and the autonomy to use them. An agent can:
- Execute SQL queries against your database.
- Create Jira tickets.
- Trigger Azure DevOps pipelines.
- Read and write files.

The core mechanism that turns an LLM into an Agent is called **Tool Calling** (also known as Function Calling).

---

## 2. What is Tool Calling?

Tool Calling is a feature supported by modern models (like GPT-4o and Claude 3.5 Sonnet) where you provide the model with a JSON schema describing functions you have written in your code (e.g., C# or Python).

If the model realizes it needs external information to answer the user's prompt, it will output a JSON object asking *you* (the application) to run the function for it.

### The Flow:
1. **User:** *"What is the weather in Seattle?"*
2. **App:** Sends prompt to Claude + a JSON list of tools (e.g., `get_weather(location)`).
3. **Claude:** Replies with `{"tool_call": "get_weather", "arguments": {"location": "Seattle"}}`.
4. **App:** Pauses the LLM. Your C# or Python code executes the real weather API.
5. **App:** Sends the API result (e.g., "72 degrees") back to Claude.
6. **Claude:** Reads the result and generates the final human-readable answer: *"It is currently 72 degrees in Seattle."*

---

## 3. Tool Calling with Claude 3.5 Sonnet (Python Example)

Anthropic's Claude 3.5 Sonnet is currently considered one of the best models in the world for coding and tool execution.

Here is how you define a "Skill" (a tool) and pass it to Claude using the Anthropic Python SDK:

```bash
pip install anthropic
```

```python
import anthropic
import json
import os

client = anthropic.Anthropic(api_key=os.environ.get("ANTHROPIC_API_KEY"))

# 1. Define your tool schema (This tells Claude what the tool does)
tools = [
    {
        "name": "get_stock_price",
        "description": "Get the current stock price for a given ticker symbol.",
        "input_schema": {
            "type": "object",
            "properties": {
                "ticker": {
                    "type": "string",
                    "description": "The stock ticker symbol, e.g., MSFT or AAPL."
                }
            },
            "required": ["ticker"]
        }
    }
]

# 2. Ask Claude a question
response = client.messages.create(
    model="claude-3-5-sonnet-20240620",
    max_tokens=1024,
    tools=tools,
    messages=[
        {"role": "user", "content": "How much is Microsoft stock right now?"}
    ]
)

# 3. Check if Claude decided to use the tool
if response.stop_reason == "tool_use":
    tool_use = next(block for block in response.content if block.type == "tool_use")
    tool_name = tool_use.name
    tool_input = tool_use.input
    
    print(f"Claude wants to use tool: {tool_name} with args: {tool_input}")
    
    # OUTPUT: Claude wants to use tool: get_stock_price with args: {'ticker': 'MSFT'}
    
    # 4. Here is where YOUR code would actually call a real Stock API using 'MSFT'
    # and return the result back to Claude to finish the conversation!
```

---

## 4. Designing Good "Skills"

A "Skill" is just a tool you give to an agent. When designing skills:

1. **Be highly descriptive:** The `description` field in the JSON schema is critical. The LLM reads this to decide if it should use the tool. A bad description means the LLM will ignore it.
2. **Keep it atomic:** A tool should do one specific thing (e.g., `get_user_by_id`, `update_order_status`). Don't create massive "do everything" tools.
3. **Handle Errors Gracefully:** If your C# or Python tool fails (e.g., API timeout), return a string back to the LLM saying *"The API failed with a timeout. Please tell the user you cannot complete the action right now."* The LLM will read the error and apologize to the user organically!

---

## 🧪 Exercise: Build a File Writer Agent
Using the Python script above, create a new tool called `write_file(filename, content)`. Write the actual Python logic to create the file on your hard drive when Claude requests it. Then, ask Claude: *"Write a Python script that prints 'Hello World' and save it to hello.py"*. Watch your agent autonomously write code to your disk!

---

**Next:** [09-MCP-EXPLAINED.md](./09-MCP-EXPLAINED.md) — What if you want to share these tools across multiple different AI applications? Enter the Model Context Protocol.
