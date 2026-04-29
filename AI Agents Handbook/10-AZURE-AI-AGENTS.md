# Generative AI & Agents
## File 10: Autonomous AI Agents with Azure OpenAI (ReAct Pattern)

> **⚠️ 2025 Update:** Microsoft now offers the **Azure AI Foundry Agent Service** — a fully managed agent platform that handles state, tool execution, file search, and code interpretation without custom infrastructure. For new enterprise projects, consider starting there (fully covered in [File 15](./15-DOTNET-AGENT-FRAMEWORK.md)). This file covers the foundational ReAct pattern, which is still important to understand for custom/self-hosted agent scenarios.

---

## What You'll Learn
- What the ReAct (Reasoning and Acting) pattern is.
- How to combine RAG, Tool Calling, and Planning.
- Building a fully autonomous agent loop.

**Time Required:** 60 minutes

---

## 1. The Limitations of Basic Tool Calling

In previous files, we looked at basic tool calling: The user asks for the weather, the AI calls `get_weather`, you return the result, the AI answers.

But what if the user asks a complex, multi-step question?
> *"Check the inventory database for product XYZ. If we have less than 50 in stock, draft an email to the supplier ordering 100 more, and save that draft to my OneDrive."*

A single tool call cannot handle this. The AI must:
1. **Call Tool 1:** `query_database(XYZ)`
2. **Read the result:** (e.g., 20 items remaining)
3. **Realize it needs to order more.**
4. **Call Tool 2:** `draft_email(supplier_XYZ, 100)`
5. **Read the result:** (e.g., email text drafted)
6. **Call Tool 3:** `save_to_onedrive(file_text)`

To achieve this, we use the **ReAct Pattern**.

---

## 2. The ReAct Pattern

**ReAct (Reasoning + Acting)** is a framework that forces the LLM to "think out loud" before it takes an action.

The prompt loop looks like this:
- **Question:** What should I do?
- **Thought:** I need to check the inventory first.
- **Action:** Call `query_database(XYZ)`.
- **Observation:** (The system returns "20 items in stock").
- **Thought:** 20 is less than 50. I need to draft an email.
- **Action:** Call `draft_email`.
- **Observation:** (The system returns the drafted text).
- **Thought:** I have the draft, now I need to save it.
- **Action:** Call `save_to_onedrive`.
- **Observation:** (The system returns "Success").
- **Thought:** I am finished. I can return the final answer to the user.
- **Final Answer:** "I checked the stock, we had 20 items left. I have drafted the email and saved it to your OneDrive."

---

## 3. Implementing ReAct in Python (LangChain)

LangChain provides out-of-the-box support for ReAct agents.

```python
from langchain_openai import AzureChatOpenAI
from langchain.agents import create_openai_tools_agent, AgentExecutor
from langchain.tools import tool
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
import os

# 1. Initialize the LLM (Requires a powerful model like GPT-4o for complex reasoning)
llm = AzureChatOpenAI(
    azure_deployment=os.environ["AZURE_OPENAI_CHAT_DEPLOYMENT"],
    api_version="2024-02-15-preview"
)

# 2. Define the Tools
@tool
def query_database(product_id: str) -> str:
    """Queries the database to get the current stock level of a product."""
    # Simulate DB lookup
    return f"{product_id} has 20 items in stock."

@tool
def draft_email(supplier: str, quantity: int) -> str:
    """Drafts an email to order more stock from a supplier."""
    return f"Subject: Order {quantity} items. Dear {supplier}..."

@tool
def save_to_onedrive(content: str) -> str:
    """Saves text content to the user's OneDrive."""
    # Simulate saving
    return "Successfully saved to OneDrive."

tools = [query_database, draft_email, save_to_onedrive]

# 3. Create the ReAct Prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an autonomous supply chain agent. Use your tools to fulfill the user's request. Think step-by-step."),
    ("user", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"), # This is where the ReAct loop stores its Thoughts and Observations
])

# 4. Construct the Agent
agent = create_openai_tools_agent(llm, tools, prompt)

# 5. Create the Agent Executor (The loop that runs the ReAct pattern)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 6. Run it!
result = agent_executor.invoke({
    "input": "Check the inventory database for product XYZ. If we have less than 50 in stock, draft an email to the supplier ordering 100 more, and save that draft to my OneDrive."
})

print(result["output"])
```

---

## 4. The Future: Multi-Agent Systems

When a single agent has too many tools (e.g., 50+ tools), it gets confused. 
The modern architecture is **Multi-Agent Systems** (using frameworks like **AutoGen** or **LangGraph**).

Instead of one "Supply Chain Agent", you have:
1. **The Planner Agent:** Breaks the user's request into tasks.
2. **The DB Agent:** Only has database tools.
3. **The Communication Agent:** Only has email/OneDrive tools.

The Planner Agent delegates task 1 to the DB Agent. When the DB Agent finishes, the Planner delegates task 2 to the Communication Agent.

This is how Enterprise AI applications scale!

---

## 🧪 Exercise: Trace the ReAct Loop
1. Take the exact Python code provided above and copy it into your IDE.
2. Modify the user input to: *"Draft an email to supplier ABC asking for 500 units, then save it to OneDrive. Do NOT check the database first."*
3. Watch the terminal output (`verbose=True`). Notice how the agent skips the database tool and jumps straight to the `draft_email` tool.
4. Try to "trick" the agent by asking it a math question (which it doesn't have a tool for). Observe how the ReAct loop handles missing tools!

---

## 🎉 Conclusion

You have completed the **Generative AI & AI Agents Handbook**!
You now understand how to build RAG pipelines, how to connect Azure OpenAI to .NET and Python, how to build Custom Copilot extensions, and how to create fully autonomous ReAct agents.

**Happy Coding!**
