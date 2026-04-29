# Generative AI & Agents
## File 13: AI Memory Patterns — Short-Term, Long-Term, and Episodic Memory

---

## What You'll Learn
- Why AI agents need explicit memory management.
- The 4 types of AI memory and when to use each.
- How to implement memory in .NET (Semantic Kernel) and Python (LangChain).
- Building a persistent memory system with Azure AI Search.

**Time Required:** 45 minutes

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

## 7. Memory in .NET (Semantic Kernel)

Semantic Kernel has a built-in `ISemanticTextMemory` abstraction.

```csharp
using Microsoft.SemanticKernel.Memory;
using Microsoft.SemanticKernel.Connectors.AzureAISearch;

// 1. Configure the memory store with Azure AI Search
var memoryStore = new AzureAISearchMemoryStore(
    endpoint: "https://your-search.search.windows.net",
    apiKey: "your-search-admin-key"
);

// 2. Build the semantic memory with an embedding model
var memory = new MemoryBuilder()
    .WithAzureOpenAITextEmbeddingGeneration(
        deploymentName: "text-embedding-ada-002",
        endpoint: endpoint,
        credential: new DefaultAzureCredential())
    .WithMemoryStore(memoryStore)
    .Build();

// 3. Save a memory
await memory.SaveInformationAsync(
    collection: "user-preferences",
    id: "vamshi_001",
    text: "Vamshi prefers Kubernetes deployments and uses .NET 8 with Entity Framework Core.",
    description: "User profile for Vamshi"
);

// 4. Search for relevant memories
var results = memory.SearchAsync(
    collection: "user-preferences",
    query: "How does this user like to deploy applications?",
    limit: 3,
    minRelevanceScore: 0.7
);

await foreach (var result in results)
{
    Console.WriteLine($"Memory: {result.Metadata.Text} (Relevance: {result.Relevance:P0})");
}
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
