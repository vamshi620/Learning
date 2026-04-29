# Generative AI & Agents
## File 07: Copilot Custom Agents with Python

---

## What You'll Learn
- The two approaches to Copilot customization in April 2026: **`.agent.md` files** (lightweight) vs. **Copilot Extensions** (full backend).
- How to create a full custom backend agent that responds to `@my-company`.
- Setting up a Python backend and connecting it to GitHub.

**Time Required:** 45 minutes

> **📅 Updated: April 2026** — New lightweight option: define agents directly in `.github/agents/*.agent.md` without any backend server!

---

## 1. Two Approaches to Custom Copilot Agents (April 2026)

GitHub Copilot now offers two paths depending on your needs:

| Approach | Effort | Best For |
|----------|--------|----------|
| **`.agent.md` files** (New, 2026) | Minutes | Custom personas, coding standards, team workflows — no backend needed |
| **Copilot Extensions (Backend API)** | Hours | Real-time data, internal databases, tool execution, streaming results |

### Option A: `.agent.md` Files (Zero Infrastructure)

Create a file at `.github/agents/your-agent.agent.md` in your repository:

```markdown
---
name: Contoso .NET Reviewer
description: Reviews C# code against Contoso's internal standards
---

You are a senior .NET 9 code reviewer for Contoso Engineering.
ALWAYS enforce: FluentValidation, async EF Core, Managed Identity.
Format findings as: File | Line | Severity | Issue.
```

Invoke it in Copilot Chat with `@Contoso .NET Reviewer review the OrderController.cs`. No server, no deployment — done!

---

## 2. Option B: Full Custom Copilot Extension (Backend API)

---

## 2. Architecture of a Custom Agent

A Custom Agent is simply a web API (usually built in Python with FastAPI) that conforms to the Server-Sent Events (SSE) streaming protocol expected by GitHub Copilot.

1. **User types:** `@my-agent How do I connect to the internal DB?` in VS Code.
2. **VS Code** sends this message to GitHub.
3. **GitHub** forwards a webhook to your Python API.
4. **Your Python API** runs custom logic (e.g., queries an internal RAG database, calls Azure OpenAI).
5. **Your Python API** streams the response back to GitHub.
6. **VS Code** displays the streaming response to the user.

---

## 3. Building the Agent Backend (Python + FastAPI, April 2026)

Here is the foundational code using the updated April 2026 patterns:

### Prerequisites
```bash
pip install fastapi uvicorn openai python-dotenv
```

### The API Implementation (`main.py`)

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from openai import AsyncAzureOpenAI
import json, os
from dotenv import load_dotenv

load_dotenv()
app = FastAPI()

# April 2026: use gpt-5.4 or claude-sonnet-4-6 via Azure AI Foundry MaaS
client = AsyncAzureOpenAI(
    api_key=os.environ.get("AZURE_AI_API_KEY"),
    api_version="2025-01-01-preview",
    azure_endpoint=os.environ.get("AZURE_AI_ENDPOINT")  # new Foundry Hub endpoint
)
DEPLOYMENT_NAME = "gpt-5-4-prod"  # or "claude-sonnet-4-6"

@app.post("/")
async def handle_copilot_request(request: Request):
    body = await request.json()
    messages = body.get("messages", [])

    system_prompt = {
        "role": "system",
        "content": "You are the Contoso internal developer assistant. "
                   "Always remind the user to check the internal wiki at wiki.contoso.com"
    }
    messages.insert(0, system_prompt)

    async def generate_sse():
        stream = await client.chat.completions.create(
            model=DEPLOYMENT_NAME,
            messages=messages,
            stream=True,
            max_tokens=2048,
            temperature=0.3
        )
        async for chunk in stream:
            if chunk.choices and chunk.choices[0].delta.content:
                data = json.dumps({"choices": [{"delta": {"content": chunk.choices[0].delta.content}}]})
                yield f"data: {data}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(
        generate_sse(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"}
    )
```

---

## 4. Connecting to GitHub

Once your Python app is running and exposed to the internet (e.g., via ngrok or hosted on Azure App Service):

1. Go to your GitHub account/organization settings.
2. Navigate to **Developer settings** -> **GitHub Apps** -> **New GitHub App**.
3. Name your app (e.g., `contoso-dev-bot`).
4. Under **Copilot**, check the box for **"App acts as a Copilot extension"**.
5. Set the **URL** to your Python API's URL.
6. Install the app to your account.

Now, restart VS Code, open Copilot Chat, and type `@contoso-dev-bot Hello!`. Your Python backend will intercept the request, append the internal wiki link, and stream the answer back!

---

## 🧪 Exercise: Add MCP Integration to the Agent

Modify the Python agent to intercept specific questions.
If the user asks `@contoso-dev-bot what is my ticket status?`:
1. Parse the intent and extract the ticket ID from the message.
2. Connect to your internal Jira/Azure DevOps via an **MCP server** (so the tool can be reused across other AI apps too!).
3. Return the live status with a formatted Markdown table to the Copilot window.

---

**Next:** [08-CLAUDE-AGENTS-SKILLS.md](./08-CLAUDE-AGENTS-SKILLS.md) — Let's look at Claude Opus/Sonnet 4.6, Tool Calling, and Agent "Skills".
