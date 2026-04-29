# Generative AI & Agents
## File 07: Copilot Custom Agents with Python

---

## What You'll Learn
- What GitHub Copilot Extensions / Custom Agents are.
- How to create an agent that responds to `@my-company`.
- Setting up a Python backend for your agent.

**Time Required:** 45 minutes

---

## 1. What is a Custom Copilot Agent?

While GitHub Copilot is great at writing general code, it doesn't know your company's specific internal APIs, proprietary deployment processes, or highly custom frameworks.

A **Custom Copilot Agent** allows you to create your own bot that integrates directly into the GitHub Copilot Chat window. Users can type `@contoso-bot deploy this app` and your custom Python backend will process the request.

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

## 3. Building the Agent Backend (Python + FastAPI)

Here is the foundational code to build a Custom Agent using Python and FastAPI.

### Prerequisites
```bash
pip install fastapi uvicorn sse-starlette openai pydantic
```

### The API Implementation (`main.py`)

```python
from fastapi import FastAPI, Request
from sse_starlette.sse import EventSourceResponse
from openai import AsyncAzureOpenAI
import json
import os

app = FastAPI()

# Initialize Azure OpenAI Client
client = AsyncAzureOpenAI(
    api_key=os.environ.get("AZURE_OPENAI_API_KEY"),
    api_version="2024-02-15-preview",
    azure_endpoint=os.environ.get("AZURE_OPENAI_ENDPOINT")
)
DEPLOYMENT_NAME = "gpt-4o"

@app.post("/")
async def handle_copilot_request(request: Request):
    """
    GitHub Copilot sends a POST request here when a user types @your-agent
    """
    body = await request.json()
    messages = body.get("messages", [])

    # 1. Inject your custom company context!
    system_prompt = {
        "role": "system",
        "content": "You are the Contoso internal developer assistant. "
                   "Always remind the user to check the internal wiki at wiki.contoso.com"
    }
    messages.insert(0, system_prompt)

    # 2. Generator function to stream the response back via SSE
    async def generate():
        stream = await client.chat.completions.create(
            model=DEPLOYMENT_NAME,
            messages=messages,
            stream=True
        )
        
        async for chunk in stream:
            if len(chunk.choices) > 0 and chunk.choices[0].delta.content:
                # Copilot expects SSE events in this specific JSON format
                yield {
                    "event": "copilot_response",
                    "data": json.dumps({"choices": [{"delta": {"content": chunk.choices[0].delta.content}}]})
                }
        
        # Signal completion
        yield {"event": "copilot_response", "data": "[DONE]"}

    # 3. Return the streaming response
    return EventSourceResponse(generate())

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
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

## 🧪 Exercise: Add Tool Calling to the Agent

Modify the Python code above to intercept specific questions. 
If the user asks `@contoso-dev-bot what is my ticket status?`, have your Python code extract the ticket ID, make a real HTTP request to Jira/Azure DevOps, and return the live status to the Copilot window!

---

**Next:** [08-CLAUDE-AGENTS-SKILLS.md](./08-CLAUDE-AGENTS-SKILLS.md) — Let's look at Claude 3.5, Tool Calling, and Agent "Skills".
