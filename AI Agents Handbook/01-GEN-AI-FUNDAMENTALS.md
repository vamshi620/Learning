# Generative AI Fundamentals
## File 01: LLMs, Tokens, Context Windows, and Prompt Engineering

---

## What You'll Learn
- How Large Language Models (LLMs) actually work under the hood.
- Understanding Tokens vs. Words (and why it matters for cost).
- What a "Context Window" is and how it limits your applications.
- Core principles of Prompt Engineering for developers.

**Time Required:** 20-30 minutes

---

## 1. What is a Large Language Model (LLM)?

At its core, a Large Language Model (like GPT-4.1, Claude 3.7 Sonnet, Gemini 2.5 Pro, or open-source Llama 3.3) is a massive mathematical function. It does not "think" or "know facts" like a database.

> **2025 Model Landscape:**
>
> | Model | Provider | Context Window | Strength |
> |-------|----------|---------------|----------|
> | GPT-4.1 | Azure/OpenAI | 1M tokens | Long docs, coding, instruction following |
> | GPT-4o / 4o-mini | Azure/OpenAI | 128K tokens | General purpose, balanced speed & cost |
> | o1 / o3 / o4-mini | Azure/OpenAI | 200K tokens | Deep reasoning, math, complex code |
> | Claude 3.7 Sonnet | Anthropic | 200K tokens | Extended thinking, coding, agents |
> | Gemini 2.5 Pro | Google | 1M tokens | Multimodal, long-form reasoning |
> | Llama 3.3 / Phi-4 | Meta/Microsoft | 128K/16K | Open-source, self-hosted via Ollama | 

Instead, it predicts the **next most likely token** based on the sequence of tokens that came before it.

Think of it like the autocomplete on your phone's keyboard, but scaled up by billions of parameters and trained on a massive portion of the public internet.

### The Training Process
1. **Pre-training:** The model reads terabytes of text and learns grammar, logic, coding patterns, and facts. It learns to predict the next word.
2. **Fine-tuning (Instruction Tuning):** The model is trained on Q&A pairs so it learns to be a helpful assistant rather than just a text-completer.
3. **RLHF (Reinforcement Learning from Human Feedback):** Humans rate the model's answers, teaching it to be safe, polite, and accurate.

---

## 2. Tokens: The Currency of AI

Models do not read words; they read **tokens**.

A token is a chunk of characters. A good rule of thumb for English text is:
- **1 Token ≈ 4 characters**
- **100 Tokens ≈ 75 words**

### Why Tokens Matter to Developers
1. **Cost:** API providers (like Azure OpenAI or Anthropic) charge you per token. You are billed for both **Input Tokens** (your prompt) and **Output Tokens** (the model's response).
2. **Compute Time:** The more output tokens a model generates, the longer the user has to wait. Time-to-First-Token (TTFT) and Tokens-Per-Second (TPS) are key metrics for AI application performance.

### Example Tokenization
The word "Hamburger" might be one token.
The word "un-hamburger-like" might be broken into three tokens: `un`, `-hamburger`, `-like`.

---

## 3. The Context Window

The **Context Window** is the maximum number of tokens an LLM can process in a single interaction (Input + Output).

If an LLM has a context window of 8,000 tokens (like GPT-4-8k):
- If your prompt is 6,000 tokens long...
- The model can only generate a maximum of 2,000 tokens in its response before it hits the limit.

### "Stateless" Nature of LLMs
When you chat with ChatGPT, it feels like it remembers what you said 10 minutes ago. But the AI itself has **no memory**. 

Every time you send a message, the application (e.g., the ChatGPT website) sends your *entire conversation history* back to the model. This eats up the Context Window rapidly. Once the conversation exceeds the context window, older messages are dropped, and the model "forgets" them.

*This is why building AI Apps requires careful state and memory management!*

---

## 4. Prompt Engineering for Developers

When building AI applications, you aren't just typing questions into a chat box. You are writing **System Prompts** that define the application's behavior.

### The System Prompt
The System Prompt is hidden from the user. It tells the AI who it is and how to behave.

**Bad System Prompt:**
> "You are a helpful assistant. Help the user."

**Good System Prompt (Enterprise Grade):**
> "You are an expert customer support agent for Contoso Electronics. 
> 
> YOUR GOAL:
> Assist the user with troubleshooting devices. 
>
> RULES:
> 1. If the user asks about a product not sold by Contoso, politely decline to answer.
> 2. Always output your final answer in JSON format with 'status' and 'message' keys.
> 3. Do not make up technical specifications. If you don't know, say 'I don't know'."

### Best Practices for Prompts
1. **Be Specific:** Tell the model exactly what format you want (e.g., JSON, XML, Bullet points).
2. **Give it time to think:** Add instructions like "Think step-by-step before answering" or "Explain your reasoning." This forces the model to generate more tokens, giving it more compute power to arrive at the right answer.
3. **Provide Few-Shot Examples:** Give the model 2-3 examples of the input and the expected output inside the prompt.

---

## 🧪 Exercise: Token Counting

1. Go to the [OpenAI Tokenizer tool](https://platform.openai.com/tokenizer).
2. Paste in a paragraph of code (e.g., a C# class or Python script).
3. Notice how spaces, braces `{ }`, and camelCase variables are split into tokens. Code often consumes more tokens than plain English text!

---

**Next:** [02-RAG-CONCEPTS.md](./02-RAG-CONCEPTS.md) — Learn how to make an LLM read your company's private documents using Retrieval-Augmented Generation.
