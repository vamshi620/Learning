# Generative AI & Agents
## File 05: Building AI Apps in Python (LangChain)

---

## What You'll Learn
- Why Python is the dominant language for AI.
- What LangChain is and how it simplifies LLM orchestration.
- How to connect Python to Azure OpenAI.
- Building a full RAG pipeline in Python.

**Time Required:** 45 minutes

---

## 1. The Python AI Ecosystem (2025 State)

While .NET has Semantic Kernel and Microsoft.Extensions.AI (see [File 15](./15-DOTNET-AGENT-FRAMEWORK.md)), Python is the undisputed king of the AI ecosystem. Almost all new AI frameworks, research papers, and vector database SDKs release in Python first.

**2025 Python AI Stack:**

| Tool | Role |
|------|------|
| `langchain` / `langchain-community` | LLM orchestration and RAG pipelines |
| `langgraph` | Stateful multi-agent orchestration (successor to LangChain agents) |
| `llama-index` | RAG-specialized framework |
| `openai` / `anthropic` | Direct SDK access to models |
| `fastapi` | Serving AI endpoints as REST APIs |
| `chromadb` / `qdrant-client` | Local and cloud vector databases |

> **2025 Update:** For complex multi-agent systems in Python, **LangGraph** has largely replaced older LangChain agent patterns. It provides a graph-based state machine that gives much more control over the agent loop than the simple `AgentExecutor`.

### Setup
Ensure you have Python 3.10+ installed, then install the required libraries:

```bash
pip install langchain langchain-openai python-dotenv
```

---

## 2. Connecting to Azure OpenAI

Create a `.env` file in your directory to store your connection details:

```env
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
AZURE_OPENAI_API_KEY=your_api_key_here
AZURE_OPENAI_API_VERSION=2024-02-15-preview
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o
```

Now, initialize the model in Python:

```python
import os
from dotenv import load_dotenv
from langchain_openai import AzureChatOpenAI
from langchain_core.messages import SystemMessage, HumanMessage

load_dotenv()

# Initialize the Azure OpenAI client
llm = AzureChatOpenAI(
    azure_deployment=os.environ["AZURE_OPENAI_CHAT_DEPLOYMENT"],
    api_version=os.environ["AZURE_OPENAI_API_VERSION"],
    temperature=0.7 # 0.0 is deterministic, 1.0 is highly creative
)

# Test a simple message
messages = [
    SystemMessage(content="You are a helpful assistant."),
    HumanMessage(content="Explain what an API is in one sentence.")
]

response = llm.invoke(messages)
print(response.content)
```

---

## 3. LangChain Expression Language (LCEL)

LangChain's defining feature is LCEL, which allows you to chain components together using the pipe `|` operator, similar to Unix command lines.

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# 1. Create a prompt template
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are an expert on {topic}."),
    ("user", "Tell me a joke about {topic}.")
])

# 2. Build the chain: Prompt -> LLM -> String Output Parser
chain = prompt | llm | StrOutputParser()

# 3. Invoke the chain
result = chain.invoke({"topic": "programming"})
print(result)
```

---

## 4. Building a RAG Pipeline in Python

Python makes building RAG incredibly fast. We will use `FAISS` (an in-memory vector database from Meta) to simulate our Vector DB.

```bash
pip install faiss-cpu tiktoken
```

Here is a complete, working RAG implementation in ~20 lines of code:

```python
from langchain_community.document_loaders import TextLoader
from langchain_openai import AzureOpenAIEmbeddings
from langchain_community.vectorstores import FAISS
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. LOAD: Read the document
# Assume you have a file 'company_policy.txt'
loader = TextLoader("company_policy.txt")
docs = loader.load()

# 2. CHUNK: Split into 500-token chunks with 50-token overlap
text_splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = text_splitter.split_documents(docs)

# 3. EMBED & STORE: Create embeddings and store in FAISS Vector DB
embeddings = AzureOpenAIEmbeddings(azure_deployment="text-embedding-ada-002")
vector_store = FAISS.from_documents(chunks, embeddings)

# 4. RETRIEVE: Create a retriever to fetch the top 3 closest chunks
retriever = vector_store.as_retriever(search_kwargs={"k": 3})

# 5. GENERATE: Create a RAG Chain
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain

prompt = ChatPromptTemplate.from_template("""
Answer the question based only on the provided context:
<context>
{context}
</context>
Question: {input}
""")

document_chain = create_stuff_documents_chain(llm, prompt)
retrieval_chain = create_retrieval_chain(retriever, document_chain)

# Execute!
response = retrieval_chain.invoke({"input": "What is the travel policy?"})
print(response["answer"])
```

---

## 🧪 Exercise: Swap the Vector DB
The example above uses `FAISS` which disappears when the script ends. 
1. Research **ChromaDB** or **Qdrant**.
2. Modify the script to save the vector store to a persistent local folder on your hard drive using ChromaDB, so you don't have to re-embed the document every time you run the script!

---

**Next:** [06-GITHUB-COPILOT-BASICS.md](./06-GITHUB-COPILOT-BASICS.md) — Let's pivot from building apps to boosting your own productivity with GitHub Copilot.
