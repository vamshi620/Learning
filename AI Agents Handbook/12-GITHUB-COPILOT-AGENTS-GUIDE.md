# Generative AI & Agents
## File 12: Step-by-Step Guide — Implementing AI Agents in GitHub Copilot

---

## What You'll Learn
- The GitHub Copilot Extensions ecosystem explained.
- A complete, working step-by-step guide to build and register a Copilot Agent.
- How to add RAG, tool calling, and streaming to your agent.
- Best practices for authentication, error handling, and deployment.

**Time Required:** 60 minutes

---

## 1. Understanding the Ecosystem

GitHub Copilot Extensions (also known as Copilot Agents) are available in two forms:

| Type | Description | Best For |
|------|-------------|----------|
| **Copilot Extension (Skillset)** | A set of REST API "skills" that Copilot decides when to call automatically | Simple, stateless API lookups (e.g., get Jira ticket, query DB) |
| **Copilot Extension (Agent)** | A full AI backend that handles the entire conversation and streams back custom responses | Complex workflows, custom AI models, RAG over internal docs |

In this guide, we will build the more powerful **Agent** type.

---

## 2. Prerequisites

Before you start, ensure you have:
- A GitHub account with Copilot access (Individual, Business, or Enterprise).
- VS Code with the GitHub Copilot extension installed.
- Python 3.10+ installed.
- An Azure subscription with an Azure OpenAI resource provisioned (see [File 03](./03-AZURE-OPENAI-SETUP.md)).
- `ngrok` installed for local testing: `choco install ngrok` or `winget install ngrok`.

---

## 3. Step-by-Step: Building Your First Agent

### Step 1 — Create the Project

```powershell
# Create a folder for your agent
mkdir contoso-copilot-agent
cd contoso-copilot-agent

# Create a Python virtual environment
python -m venv .venv
.venv\Scripts\Activate.ps1

# Install dependencies
pip install fastapi uvicorn sse-starlette openai python-dotenv pydantic
```

Create a `.env` file for your secrets:
```env
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_API_KEY=your_key_here
AZURE_OPENAI_API_VERSION=2024-02-15-preview
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o
```

---

### Step 2 — Understand the GitHub Copilot Request/Response Contract

When a user types `@your-agent Hello!`, GitHub sends a **POST request** to your API with this structure:

```json
{
  "copilot_thread_id": "thread_abc123",
  "messages": [
    {
      "role": "user",
      "content": "Hello!",
      "copilot_references": []
    }
  ]
}
```

Your API must respond with a **Server-Sent Events (SSE) stream** of chat completion chunks in OpenAI format.

---

### Step 3 — Build the FastAPI Agent (`main.py`)

```python
import os
import json
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from openai import AsyncAzureOpenAI
from dotenv import load_dotenv

load_dotenv()
app = FastAPI(title="Contoso Copilot Agent")

client = AsyncAzureOpenAI(
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    api_version=os.environ["AZURE_OPENAI_API_VERSION"],
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"]
)

DEPLOYMENT = os.environ["AZURE_OPENAI_CHAT_DEPLOYMENT"]

SYSTEM_PROMPT = """
You are the Contoso DevOps Assistant, an expert in Contoso's internal deployment processes.

ROLE: Help developers with Azure deployments, GitHub Actions pipelines, and Kubernetes manifests.

RULES:
1. Always reference Contoso's internal documentation at docs.contoso.internal.
2. If asked about infrastructure outside Azure (AWS, GCP), politely redirect to Azure equivalents.
3. Always provide working PowerShell commands, never bash (Contoso uses Windows dev machines).
4. When sharing code, always include proper error handling.

FALLBACK: If you cannot answer, say: "Please raise a ticket in the #devops-help Slack channel."
"""

@app.get("/")
async def health_check():
    return {"status": "ok", "agent": "Contoso Copilot Agent"}

@app.post("/")
async def handle_copilot_request(request: Request):
    """Main endpoint — GitHub Copilot routes all @contoso-agent messages here."""
    body = await request.json()

    # 1. Extract the conversation messages from Copilot's request
    messages = body.get("messages", [])

    # 2. Prepend our system prompt
    full_messages = [{"role": "system", "content": SYSTEM_PROMPT}] + messages

    # 3. Stream generator
    async def generate_sse():
        try:
            stream = await client.chat.completions.create(
                model=DEPLOYMENT,
                messages=full_messages,
                stream=True,
                max_tokens=2048,
                temperature=0.3  # Lower = more consistent for a dev assistant
            )

            async for chunk in stream:
                if chunk.choices and chunk.choices[0].delta.content:
                    content = chunk.choices[0].delta.content
                    # GitHub Copilot expects OpenAI-compatible SSE chunks
                    data = json.dumps({
                        "choices": [{"delta": {"content": content}}]
                    })
                    yield f"data: {data}\n\n"

            yield "data: [DONE]\n\n"

        except Exception as e:
            # Graceful error handling — stream back a user-friendly error
            error_msg = json.dumps({
                "choices": [{"delta": {"content": f"\n\n❌ Agent error: {str(e)}"}}]
            })
            yield f"data: {error_msg}\n\n"
            yield "data: [DONE]\n\n"

    return StreamingResponse(
        generate_sse(),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no"  # Important: disables nginx buffering for SSE
        }
    )
```

---

### Step 4 — Run and Test Locally

```powershell
# Start the API server (keep this running)
uvicorn main:app --host 0.0.0.0 --port 8000 --reload

# In a separate terminal, expose it to the internet via ngrok
ngrok http 8000
```

`ngrok` will give you a public URL like `https://abc123.ngrok.io`. Save this — you'll need it to register the GitHub App.

---

### Step 5 — Register the GitHub App

1. Go to `https://github.com/settings/apps/new`.
2. Fill in the **GitHub App Name** (e.g., `contoso-devops-agent`).
3. Set the **Homepage URL** to your ngrok URL.
4. Set the **Callback URL** to your ngrok URL + `/auth/callback`.
5. Under **Permissions**, expand **Copilot** → set to `Read and Write`.
6. Scroll down to the **Copilot** section:
   - **App Type:** `Agent`
   - **Endpoint URL:** Your ngrok URL (e.g., `https://abc123.ngrok.io`)
   - **Description:** *"Internal Contoso DevOps assistant for Azure deployments."*
7. Click **Create GitHub App**.
8. On the next screen, click **Install App** → Install on your account.

---

### Step 6 — Test in VS Code

1. Reload VS Code (`Ctrl+Shift+P` → "Developer: Reload Window").
2. Open the Copilot Chat window.
3. Type `@contoso-devops-agent Hello, are you online?`
4. You should see a streaming response from your Python API!

---

## 4. Adding RAG to Your Agent (Internal Knowledge Base)

To give your agent access to company documentation:

```python
# Add to your requirements: pip install chromadb langchain-openai
from langchain_community.vectorstores import Chroma
from langchain_openai import AzureOpenAIEmbeddings

# Load your pre-built vector store (created once from your docs)
embeddings = AzureOpenAIEmbeddings(
    azure_deployment="text-embedding-ada-002",
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_key=os.environ["AZURE_OPENAI_API_KEY"],
    api_version="2024-02-15-preview"
)
vector_store = Chroma(persist_directory="./docs_index", embedding_function=embeddings)
retriever = vector_store.as_retriever(search_kwargs={"k": 4})

# In your handle_copilot_request endpoint, BEFORE building full_messages:
last_user_message = next(
    (m["content"] for m in reversed(messages) if m.get("role") == "user"), 
    ""
)
relevant_docs = retriever.invoke(last_user_message)
rag_context = "\n\n".join(doc.page_content for doc in relevant_docs)

# Inject context into system prompt
enhanced_system = SYSTEM_PROMPT + f"\n\nRELEVANT DOCUMENTATION:\n{rag_context}"
full_messages = [{"role": "system", "content": enhanced_system}] + messages
```

---

## 5. Best Practices Checklist

### Security
- [ ] **Validate the X-GitHub-Token header:** GitHub sends a token with every request. Verify it to ensure the request is genuinely from GitHub (not spoofed). Use the `@octokit/webhooks` library for this.
- [ ] **Never log user messages:** Treat Copilot conversation content as sensitive PII.
- [ ] **Use Managed Identity** in production instead of API keys.

### Reliability
- [ ] **Set a `max_tokens` limit** to prevent unexpectedly large (and expensive) responses.
- [ ] **Implement retry logic** in your Azure OpenAI calls (use `tenacity` library).
- [ ] **Return meaningful error messages** via the SSE stream so the user sees actionable feedback, not a silent failure.
- [ ] **Set a request timeout** (e.g., 30 seconds) to prevent the user's Copilot chat from hanging indefinitely.

### Performance
- [ ] **Pre-warm your vector store** at startup, not on every request.
- [ ] **Cache embedding results** for common queries.
- [ ] **Use streaming** (never `stream=False`) so the user sees tokens appearing immediately.

### Production Deployment
- [ ] Replace ngrok with a proper **Azure App Service** or **Azure Container App** endpoint.
- [ ] Integrate your repo with **CI/CD** (GitHub Actions) for automated deployments.
- [ ] Add **Application Insights** for monitoring token usage and latency.

---

## 6. Debugging Common Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| `@your-agent` not visible in Copilot Chat | App not installed or VS Code not reloaded | Reload VS Code window |
| Copilot shows "Agent not available" | Your endpoint is unreachable | Check ngrok is running, firewall rules |
| Response is blank / cuts off | SSE format incorrect | Ensure `data: [DONE]\n\n` at the end |
| "Rate limit exceeded" error in chat | Too many tokens generated | Add `max_tokens` limit to your API call |
| Agent ignores system prompt | Messages order wrong | Ensure system message is always at index 0 |

---

## 🧪 Exercise: Add a Slash Command Pattern

Modify the agent to handle a special command pattern. When the user types `@contoso-agent /deploy my-app to staging`, your Python code should:
1. Parse the intent `deploy` and target `staging`.
2. Call your real Azure DevOps API (or simulate it).
3. Return a nicely formatted status message with deployment ID and a link to the pipeline run!

---

**Next:** [13-AI-MEMORY-PATTERNS.md](./13-AI-MEMORY-PATTERNS.md) — Learn how AI Agents manage short-term, long-term, and episodic memory.
