# Generative AI & Agents
## File 02: Retrieval-Augmented Generation (RAG) Concepts

---

## What You'll Learn
- Why RAG is the most important architecture in Enterprise AI.
- What Embeddings are and how they represent text as math.
- How Vector Databases work.
- The mechanics of Chunking and Retrieval.

**Time Required:** 30 minutes

---

## 1. The Problem: Hallucinations and Stale Data

Large Language Models (LLMs) are trained on a fixed dataset. If a model was trained in 2023, it knows nothing about the new HR policy your company published in 2024. 

If you ask the model: *"What is the new Contoso travel policy?"*
It will either:
1. Say "I don't know."
2. **Hallucinate** a plausible-sounding but entirely fake travel policy.

To fix this, we don't re-train the model (which costs millions of dollars). Instead, we use **RAG**.

---

## 2. What is RAG?

**Retrieval-Augmented Generation (RAG)** is a pattern where you:
1. **Retrieve:** Find the relevant internal documents related to the user's question.
2. **Augment:** Inject those documents into the Context Window (System Prompt).
3. **Generate:** Ask the LLM to answer the question *strictly based on the injected documents*.

**Example RAG Prompt:**
> "You are an assistant. Answer the user's question using ONLY the provided context. If the context does not contain the answer, say 'I don't know'.
> 
> CONTEXT: {Insert internal HR document text here}
> 
> QUESTION: {User's question}"

---

## 3. The Magic of Embeddings

How do we search through millions of internal company documents to find the right one to inject into the prompt? Keyword search (like SQL `LIKE '%travel%'`) is too rigid. We need **Semantic Search**.

**Embeddings** translate human language into arrays of floating-point numbers (vectors).
- "Dog" `[0.12, -0.45, 0.88, ...]`
- "Puppy" `[0.11, -0.43, 0.87, ...]`
- "Car" `[-0.99, 0.22, -0.11, ...]`

Notice how "Dog" and "Puppy" have very similar numbers? The model maps words with similar *meanings* close together in a multi-dimensional mathematical space. 

When a user asks a question, we convert their question into a vector, and we find the document vectors that are mathematically closest to it (usually using **Cosine Similarity**).

---

## 4. Vector Databases

You cannot store arrays of 1,536 floating-point numbers efficiently in a standard SQL table. You need a **Vector Database**.

Popular Vector Databases:
- **Azure AI Search** (Enterprise standard in the Microsoft ecosystem)
- **Pinecone** (Cloud-native vector DB)
- **Qdrant / Milvus / Chroma** (Open-source)
- **pgvector** (An extension for PostgreSQL)

A Vector DB allows you to say: *"Here is my question's vector. Find the top 5 document vectors that are closest to this one in less than 50 milliseconds."*

---

## 5. The RAG Pipeline Workflow

Building a RAG application involves two distinct pipelines:

### Pipeline 1: Data Ingestion (Done in the background)
1. **Extract:** Load PDF, Word, or HTML files.
2. **Chunk:** LLMs have context limits. You cannot pass a 500-page PDF into an embedding model. You must split the document into "chunks" of ~500 tokens.
3. **Embed:** Send each chunk to an Embedding Model (e.g., `text-embedding-ada-002`) to get a vector.
4. **Store:** Save the chunk text and its vector into the Vector Database.

### Pipeline 2: Querying (Done when the user asks a question)
1. **User asks:** *"What is the remote work policy?"*
2. **Embed Query:** Send the user's question to the Embedding Model to get a vector.
3. **Search:** Query the Vector Database for the top 3 chunks closest to the question's vector.
4. **Augment:** Construct a prompt containing the text of those 3 chunks.
5. **Generate:** Send the augmented prompt to the Chat Model (e.g., GPT-4) to get the final answer.

---

## 🧪 Exercise: Conceptualize Chunking

Imagine you have a 100-page employee handbook.
If you chunk it by **Page**, a single paragraph spanning the bottom of Page 1 and the top of Page 2 gets split in half, destroying the context.
If you chunk it by **Paragraph**, the chunks might be too small and lack the surrounding context.

**Best Practice:** Use "Overlapping Chunks". Chunk the document into 500-token blocks, but have each block overlap the previous one by 50 tokens. This ensures no sentences are broken in half across chunk boundaries.

---

**Next:** [03-AZURE-OPENAI-SETUP.md](./03-AZURE-OPENAI-SETUP.md) — How to provision Azure OpenAI and secure your API keys.
