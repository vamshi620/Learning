# Generative AI & Agents
## File 13: AI Memory Patterns — Short-Term, Long-Term, and Episodic Memory

---

## What You'll Learn
- Why AI agents need explicit memory management.
- The 4 types of AI memory and when to use each.
- How to implement memory in .NET (Microsoft Agent Framework 1.0) and Python (LangGraph).
- Building a persistent memory system with Azure AI Search or Azure AI Foundry Managed Memory.

**Time Required:** 45 minutes

> **📅 Updated: April 2026** — Added Azure AI Foundry **Managed Memory** (Public Preview) which eliminates the need to provision Redis or Azure AI Search for agent memory.

---

## 1. The Memory Problem

Every time you call an LLM, it starts with **zero memory**. Your application is responsible for feeding the model all the context it needs.

A naive implementation sends the **entire conversation history** every time. This works for short conversations, but breaks in production:
- A 100-message customer support thread might exceed the context window.
- Sending 50,000 tokens of history costs money on every single message.
- Important facts mentioned early in the conversation get "forgotten" when dropped.

Production AI applications need sophisticated **Memory Management**.

---

## 2. The 4 Types of AI Memory

| Type | What It Stores | Lifetime | Analogy |
|------|---------------|---------|---------|
| **In-Context (Working)** | Recent conversation history | Current session only | RAM |
| **External (Long-Term)** | User preferences, past summaries | Persistent (database) | Hard Drive |
| **Episodic** | Summaries of past conversations | Persistent, retrievable by relevance | Diary |
| **Semantic (Entity)** | Named entities extracted from conversations (people, products, settings) | Persistent | Address Book |

---

## 3. In-Context Memory — The Sliding Window

The simplest strategy: keep only the last N messages in the context window.

### In Python (LangChain)

```python
from langchain.memory import ConversationBufferWindowMemory
from langchain.chains import ConversationChain

# Keep only the last 10 messages (5 turns)
memory = ConversationBufferWindowMemory(k=10, return_messages=True)

conversation = ConversationChain(
    llm=llm,
    memory=memory,
    verbose=True
)

# Chat
conversation.predict(input="My name is Vamshi and I work in Azure DevOps.")
conversation.predict(input="What technologies do I use?")
# ✅ Will correctly answer because the name is within the last 10 messages
```

---

## 4. Summary Memory — Compress, Don't Discard

Instead of dropping old messages, **summarize** them. This preserves the essential information while dramatically reducing token count.

### Conversation Summary Memory (Python)

```python
from langchain.memory import ConversationSummaryBufferMemory

# Summarize conversations older than 500 tokens
memory = ConversationSummaryBufferMemory(
    llm=llm,
    max_token_limit=500,
    return_messages=True
)
```

Internally, this will maintain a rolling summary of the conversation that looks like:
> *"The human introduced themselves as Vamshi, a .NET developer working on Azure Kubernetes deployments. They mentioned their team uses Bicep for IaC and Azure DevOps for CI/CD."*

This summary uses far fewer tokens than the raw message history, but preserves all important facts.

---

## 5. Long-Term Memory — Persisting Across Sessions

This is where most tutorials stop, but production apps need more. If a user closes the app and comes back tomorrow, all `ConversationBufferMemory` is lost.

**Solution:** Save memory to a database after each turn, and load it when the user returns.

### Example: Redis-Backed Memory (Python)

```python
from langchain.memory import RedisChatMessageHistory
from langchain.memory import ConversationBufferMemory

# Each user gets a unique session ID (e.g., their user ID from your auth system)
user_id = "vamshi@contoso.com"

# Redis stores the message history; loads on startup, saves on each message
message_history = RedisChatMessageHistory(
    session_id=user_id,
    url="redis://your-redis-cache.redis.cache.windows.net:6380",
    password=os.environ["REDIS_PASSWORD"],
    ssl=True
)

memory = ConversationBufferWindowMemory(
    chat_memory=message_history,
    k=10,
    return_messages=True
)

# Now the conversation history persists across app restarts!
```

---

## 6. Semantic Memory with Azure AI Search

For the most powerful memory, use a **Vector Store** (like Azure AI Search) as your memory bank. When the agent needs to remember something, it searches for semantically relevant past interactions.

### The Pattern

**On every conversation turn:**
1. Extract key facts from the user's message (e.g., "User prefers TypeScript over JavaScript").
2. Embed those facts and store them in Azure AI Search.

**Before responding to a new message:**
1. Search the memory store for facts related to the current question.
2. Inject those facts into the system prompt.

### Implementation (Python)

```python
from langchain_community.vectorstores import AzureSearch
from langchain.memory import VectorStoreRetrieverMemory

# Initialize Azure AI Search as the memory backend
vector_store = AzureSearch(
    azure_search_endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
    azure_search_key=os.environ["AZURE_SEARCH_KEY"],
    index_name="ai-agent-memory",
    embedding_function=embeddings.embed_query
)

# Wrap it as a LangChain retriever memory
retriever = vector_store.as_retriever(search_kwargs={"k": 5})
memory = VectorStoreRetrieverMemory(retriever=retriever)

# Save a fact to memory
memory.save_context(
    inputs={"input": "My name is Vamshi and I prefer Kubernetes over App Service."},
    outputs={"output": "Noted! I'll keep that in mind."}
)

# Later, retrieve relevant memories for a new question
relevant_memories = memory.load_memory_variables(
    {"prompt": "How should I deploy this .NET API?"}
)
# Returns: "The user prefers Kubernetes over App Service."
```

---

## 7. Memory in .NET (Microsoft Agent Framework 1.0, April 2026)

> **April 2026:** The `ISemanticTextMemory` abstraction from Semantic Kernel is now available through the unified `Microsoft.Agents.AI` package with `Microsoft.Extensions.VectorData`.

```csharp
using Microsoft.Extensions.VectorData;
using Microsoft.Agents.AI;

// 1. Get the vector store from DI (registered in Program.cs)
public class AgentMemoryService(
    IVectorStore vectorStore,
    IEmbeddingGenerator<string, Embedding<float>> embeddingGenerator)
{
    private readonly string _collection = "user-memory";

    public async Task SaveMemoryAsync(string userId, string fact)
    {
        var collection = vectorStore.GetCollection<string, MemoryRecord>(_collection);
        await collection.EnsureCollectionExistsAsync();

        var embedding = await embeddingGenerator.GenerateEmbeddingVectorAsync(fact);
        await collection.UpsertAsync(new MemoryRecord
        {
            Id = $"{userId}_{Guid.NewGuid()}",
            Content = fact,
            UserId = userId,
            Embedding = embedding,
            Timestamp = DateTimeOffset.UtcNow
        });
    }

    public async Task<IList<string>> RecallAsync(string userId, string query)
    {
        var collection = vectorStore.GetCollection<string, MemoryRecord>(_collection);
        var queryEmbedding = await embeddingGenerator.GenerateEmbeddingVectorAsync(query);

        var results = collection.SearchAsync(queryEmbedding, top: 5,
            new VectorSearchOptions<MemoryRecord>
            {
                Filter = r => r.UserId == userId  // Per-user isolation
            });

        var memories = new List<string>();
        await foreach (var result in results)
            memories.Add(result.Record.Content);
        return memories;
    }
}

public class MemoryRecord
{
    [VectorStoreKey] public string Id { get; set; } = "";
    [VectorStoreData] public string Content { get; set; } = "";
    [VectorStoreData] public string UserId { get; set; } = "";
    [VectorStoreData] public DateTimeOffset Timestamp { get; set; }
    [VectorStoreVector(Dimensions: 1536)] public ReadOnlyMemory<float> Embedding { get; set; }
}
```

---

## 8. Azure AI Foundry Managed Memory (Public Preview, April 2026)

The biggest memory management update of 2026: **you no longer need to provision your own Redis or Azure AI Search** for agent memory. Azure AI Foundry now provides fully managed long-term memory as a first-class feature.

```csharp
// Program.cs — Enable Foundry Managed Memory when creating the agent
var agentsClient = projectClient.GetAgentsClient();

var agent = await agentsClient.CreateAgentAsync(
    model: "gpt-5-4-prod",
    name: "MemoryEnabledAssistant",
    instructions: "You are a personalized developer assistant. Remember user preferences.",
    metadata: new Dictionary<string, string>
    {
        { "memory_enabled", "true" },
        { "memory_scope", "user" }      // Options: "user", "thread", "global"
    }
);

// From now on, the agent automatically stores and retrieves memories.
// No Redis. No Azure AI Search index. No vector store configuration.
```

**Python equivalent:**
```python
from azure.ai.projects import AIProjectClient
from azure.identity import DefaultAzureCredential

client = AIProjectClient.from_connection_string(
    conn_str=os.environ["AZURE_AI_PROJECT_CONNECTION_STRING"],
    credential=DefaultAzureCredential()
)
agents = client.agents

agent = agents.create_agent(
    model="gpt-5.4",
    name="PersonalizedAssistant",
    instructions="Remember the user's preferences and past projects.",
    metadata={"memory_enabled": "true", "memory_scope": "user"}
)
```

---

## 8. Memory Architecture Decision Tree

```
Does the user start fresh each session?
    YES → Use In-Context (Sliding Window)
    NO → Needs persistence
    
Does the conversation typically get long (>20 messages)?
    YES → Add Summary Memory to compress history
    NO → Sliding Window + Redis is sufficient
    
Does the agent need to remember specific facts across many sessions?
    YES → Add Semantic/Vector Memory (Azure AI Search)
    NO → Redis + Summary is sufficient
    
Does the agent serve multiple users?
    YES → Namespace all memory by user_id
    NO → Single memory collection is fine
```

---

## 🧪 Exercise: Build a Personalized Agent

1. Start a conversation with your agent and tell it:
   - Your name and role.
   - Your technology preferences.
   - One current project you're working on.
2. Close the app completely.
3. Implement Redis or SQLite-backed memory.
4. Restart the app and ask: *"What do you know about me?"*
5. The agent should recall your name, role, and preferences from the previous session!

---

**Next:** [14-AI-SAFETY-EVALS.md](./14-AI-SAFETY-EVALS.md) — Learn how to keep your AI applications safe, accurate, and enterprise-ready with guardrails and evaluations.
