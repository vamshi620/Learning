# Generative AI & Agents
## File 11: Prompt Engineering — A Developer's Complete Guide

---

## What You'll Learn
- The 6 core prompt engineering techniques used in production AI systems.
- How to write System Prompts that consistently produce structured output.
- Chain-of-Thought, Few-Shot, and Role prompting patterns with real examples.
- How to apply Prompt Engineering inside .NET and Python applications.
- Common mistakes and how to avoid them.

**Time Required:** 45 minutes

---

## 1. Why Prompt Engineering is a Core Dev Skill

Prompt Engineering is not about "magic words." It is about understanding how a neural network interprets input and structuring your input to reliably produce the output your application needs.

Think of it like writing a SQL query. A sloppy query returns garbage data. A well-structured query returns exactly what you need. The LLM is your engine, and your Prompt is your query.

**The cost of bad prompts:**
- Inconsistent output formats that break your JSON parsers.
- Hallucinated facts that damage user trust.
- AI that ignores safety constraints and says something harmful.
- Wasted tokens (= wasted money) generating irrelevant text before reaching the answer.

---

## 2. The 6 Core Techniques

### Technique 1: Role Prompting

Tell the model **who it is**. This fundamentally shifts its vocabulary, tone, and knowledge emphasis.

```
❌ "Explain JWT authentication."

✅ "You are a senior .NET security architect with 15 years of experience securing 
   enterprise APIs. Explain JWT authentication to a junior C# developer who 
   understands C# but has never worked with auth systems before."
```

---

### Technique 2: Chain-of-Thought (CoT)

Instruct the model to **think step-by-step before giving the final answer**. This dramatically improves accuracy on reasoning tasks (math, logic, debugging).

```
❌ "Is this C# code thread-safe?"
[Code Snippet]

✅ "Look at the following C# code. Before answering, reason through it step-by-step:
   1. Identify all shared state (static fields, instance variables modified by methods).
   2. Identify which methods could be called concurrently.
   3. Identify if any of those methods modify shared state without a lock.
   Only after completing all 3 steps, give your verdict: Is this code thread-safe?
   
   [Code Snippet]"
```

By forcing the model to spell out the reasoning chain, you get more tokens of "thinking," which leads to significantly better answers.

---

### Technique 3: Few-Shot Examples

Give the model 2-4 examples of the exact transformation you want. This is the most reliable way to enforce a specific output format.

```
You are a bug classifier. Classify bugs into: CRITICAL, HIGH, MEDIUM, LOW.

EXAMPLES:
Input: "App crashes on startup with a NullReferenceException"
Output: {"severity": "CRITICAL", "reason": "Prevents app from running"}

Input: "The hover color on the nav bar is slightly off brand"
Output: {"severity": "LOW", "reason": "Cosmetic, does not affect functionality"}

Now classify this bug:
Input: "The checkout flow fails silently for users in Australia due to a currency conversion bug"
Output:
```

The model will replicate the exact JSON format you demonstrated.

---

### Technique 4: Constraint Prompting

Explicitly tell the model what it **must NOT do**. This is critical for preventing hallucinations and scope creep.

```
You are a customer support bot for Contoso Electronics.

STRICT RULES — Do not violate these under any circumstances:
1. Do NOT answer questions about products not sold by Contoso. Say: "I can only help with Contoso products."
2. Do NOT make up product specifications. If you don't know, say "I don't have that information."
3. Do NOT discuss competitor products.
4. Do NOT engage in political, religious, or personal discussions.
5. Always respond in the same language the user writes in.
```

---

### Technique 5: Output Format Specification

Never let the model choose its output format. Always specify exactly what you want.

```
Answer the following question. Your response MUST be valid JSON and nothing else.
Do NOT include markdown code fences, explanations, or preamble. Just raw JSON.

Required JSON schema:
{
  "answer": "string (your answer here)",
  "confidence": "HIGH | MEDIUM | LOW",
  "sources": ["string", "string"] // max 3 sources
}

Question: What is the capital of Australia?
```

---

### Technique 6: The RSCEF Framework

A powerful structure for building enterprise-grade system prompts:

- **R**ole — Who is the AI?
- **S**ituation — What context does it operate in?
- **C**onstraints — What must it never do?
- **E**xpected Output — What format should it respond in?
- **F**allback — What should it say when it doesn't know?

```
ROLE:
You are "CodeReviewer AI," an expert .NET code reviewer with deep expertise in C# 12, 
performance optimization, and the SOLID principles.

SITUATION:
You are integrated into a VS Code extension. Developers paste code into the chat and 
you review it. The team uses .NET 8, Entity Framework Core, and Azure.

CONSTRAINTS:
- Never rewrite the entire code unless explicitly asked.
- Never suggest changes that would break existing API contracts.
- Always explain WHY a change is needed, not just that it should be changed.
- If the code is correct and well-written, say so. Do not invent issues.

EXPECTED OUTPUT:
Respond in Markdown with these sections:
1. **Summary** (1-2 sentences on overall quality)
2. **Issues Found** (bulleted list with severity: 🔴 CRITICAL | 🟡 MEDIUM | 🟢 MINOR)
3. **Suggested Improvements** (code snippets when relevant)

FALLBACK:
If the code is in a language other than C# or if you cannot parse it, reply:
"I can only review C# code. Please paste a valid C# snippet."
```

---

## 3. Prompt Templates in .NET (Semantic Kernel)

Semantic Kernel has first-class support for parameterized prompt templates using `{{$variableName}}` syntax.

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.PromptTemplates.Handlebars;

// 1. Define a prompt template as a string
string promptTemplate = """
    You are an expert code reviewer for {{language}} code.
    Review the following code and identify up to 3 issues:
    
    ```{{language}}
    {{code}}
    ```
    
    Output your review as a JSON array of issues with fields: 
    "severity", "line", and "description".
    """;

// 2. Create a KernelFunction from the template
var reviewFunction = kernel.CreateFunctionFromPrompt(promptTemplate);

// 3. Invoke with variables
var result = await kernel.InvokeAsync(reviewFunction, new KernelArguments
{
    ["language"] = "C#",
    ["code"] = File.ReadAllText("MyClass.cs")
});

Console.WriteLine(result);
```

### Loading Prompts from Files (Best Practice)

For production apps, never embed prompts as hardcoded strings in C# code. Use the Semantic Kernel prompt directory convention:

```
/Prompts
  /CodeReview
    skprompt.txt     ← The prompt template
    config.json      ← Model settings (temperature, max_tokens, etc.)
```

**`/Prompts/CodeReview/skprompt.txt`:**
```
You are an expert {{language}} code reviewer.
Review: {{code}}
Output: JSON array of issues.
```

**`/Prompts/CodeReview/config.json`:**
```json
{
  "schema": 1,
  "description": "Reviews code for issues",
  "execution_settings": {
    "default": {
      "max_tokens": 1000,
      "temperature": 0.0
    }
  }
}
```

**Loading in C#:**
```csharp
// Load all prompts from the /Prompts directory
var pluginFromDirectory = kernel.ImportPluginFromPromptDirectory("./Prompts");

// Invoke the "CodeReview" prompt
var result = await kernel.InvokeAsync(pluginFromDirectory["CodeReview"], new KernelArguments
{
    ["language"] = "C#",
    ["code"] = myCodeContent
});
```

---

## 4. Prompt Templates in Python (LangChain)

```python
from langchain_core.prompts import ChatPromptTemplate, SystemMessagePromptTemplate, HumanMessagePromptTemplate

# 1. Define the system and user templates
system_template = """
You are an expert {language} code reviewer.
RULES:
- Respond ONLY in JSON.
- Never invent issues that don't exist.
- Severity: CRITICAL, HIGH, MEDIUM, LOW
"""

human_template = """
Review this code:
```{language}
{code}
```
"""

# 2. Build the ChatPromptTemplate
prompt = ChatPromptTemplate.from_messages([
    SystemMessagePromptTemplate.from_template(system_template),
    HumanMessagePromptTemplate.from_template(human_template)
])

# 3. Create a chain (LCEL)
from langchain_core.output_parsers import JsonOutputParser

chain = prompt | llm | JsonOutputParser()

# 4. Invoke
result = chain.invoke({
    "language": "Python",
    "code": open("my_script.py").read()
})
print(result)  # Will be a parsed Python dict
```

---

## 5. Temperature and Parameters

These settings drastically affect output quality:

| Parameter | Range | Effect |
|-----------|-------|--------|
| `temperature` | 0.0 – 2.0 | 0.0 = deterministic, 2.0 = very random |
| `max_tokens` | 1 – model max | Hard cap on output length |
| `top_p` | 0.0 – 1.0 | Controls token sampling diversity |
| `frequency_penalty` | 0.0 – 2.0 | Penalizes repeated words |
| `presence_penalty` | 0.0 – 2.0 | Encourages new topics |

**Guidelines for enterprise use:**

| Use Case | Temperature | Notes |
|----------|-------------|-------|
| Code generation | 0.0–0.2 | Deterministic, reproducible code |
| Code review | 0.0 | Zero randomness for consistent feedback |
| Chatbots | 0.5–0.7 | Natural but not unpredictable |
| Creative writing | 0.8–1.2 | Encourages variety |
| Brainstorming | 1.0–1.5 | Maximum divergent thinking |

---

## 6. Common Prompt Engineering Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| "Tell me about Azure." | Too vague, generates essays | "List 5 Azure services relevant to .NET APIs in one sentence each." |
| Asking multiple questions at once | Model averages answers | Ask one question per prompt. Chain prompts if needed. |
| No output format specified | Model picks unpredictable format | Always specify JSON, Markdown, numbered list, etc. |
| Too many constraints | Model can't find a valid answer | Keep constraints to <10 rules |
| Embedding secrets in prompts | Prompt injection attacks | Never put credentials or PII in prompts |

---

## 🧪 Exercise: Prompt vs. Prompt Comparison

1. Ask Azure OpenAI: "Write a C# method that sends an email."
2. Note the output (probably uses `SmtpClient` which is deprecated).
3. Now ask: "Write a C# 8 async method that sends an email using `MailKit`. 
   The method should accept `to`, `subject`, and `body` as parameters. 
   Include proper exception handling and log errors with `ILogger<T>`. 
   Do NOT use `System.Net.Mail.SmtpClient` as it is deprecated."
4. Compare the quality difference!

---

**Next:** [12-GITHUB-COPILOT-AGENTS-GUIDE.md](./12-GITHUB-COPILOT-AGENTS-GUIDE.md) — Step-by-step guide to implementing AI Agents inside GitHub Copilot.
