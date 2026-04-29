# Generative AI & Agents
## File 04: Building AI Apps in .NET (Semantic Kernel)

---

## What You'll Learn
- What Microsoft Semantic Kernel (SK) is and why .NET devs use it.
- How to connect a .NET Console App to Azure OpenAI.
- How to build a Chat Loop.
- How to implement a basic RAG flow in .NET.

**Time Required:** 45 minutes

---

## 1. What is Semantic Kernel?

When building AI apps, you could just send raw HTTP requests to the OpenAI REST API. However, as your app grows, you need a way to manage conversation history, execute tools (plugins), and switch between different AI providers.

**Microsoft Semantic Kernel (SK)** is an open-source SDK created by Microsoft specifically for building AI applications in C#. It is the .NET equivalent of LangChain in Python.

---

## 2. Setting Up Your First AI App

Create a new .NET 8 Console Application and install the necessary NuGet packages:

```powershell
dotnet new console -n AiDotNetDemo
cd AiDotNetDemo
dotnet add package Microsoft.SemanticKernel
dotnet add package Azure.Identity
```

### The Kernel Initialization

The `Kernel` is the central orchestrator in Semantic Kernel.

```csharp
using Microsoft.SemanticKernel;
using Azure.Identity;

// Your Azure OpenAI Deployment Name (e.g., "gpt-4o")
string deploymentName = "gpt-4o"; 
string endpoint = "https://your-resource.openai.azure.com/";

// Create the Kernel
var builder = Kernel.CreateBuilder();

// Add Azure OpenAI Chat Completion service using Managed Identity
builder.AddAzureOpenAIChatCompletion(
    deploymentName, 
    endpoint, 
    new DefaultAzureCredential());

Kernel kernel = builder.Build();

Console.WriteLine("Kernel built successfully!");
```

---

## 3. Building a Chat Loop with History

Remember that LLMs are stateless. To build a chat bot, you must maintain the `ChatHistory` object yourself and send it with every request.

```csharp
using Microsoft.SemanticKernel.ChatCompletion;

// Get the chat service from the kernel
var chatCompletionService = kernel.GetRequiredService<IChatCompletionService>();

// Create a history object and add the System Prompt
ChatHistory history = new ChatHistory();
history.AddSystemMessage("You are a helpful IT support assistant. Answer concisely.");

Console.WriteLine("Start chatting (Type 'exit' to quit):");

while (true)
{
    Console.Write("User: ");
    string userInput = Console.ReadLine()!;
    if (userInput.ToLower() == "exit") break;

    // 1. Add user message to history
    history.AddUserMessage(userInput);

    // 2. Send the ENTIRE history to the model
    var response = await chatCompletionService.GetChatMessageContentAsync(
        history, 
        kernel: kernel);

    Console.WriteLine($"AI: {response.Content}\n");

    // 3. Add the AI's response to the history so it remembers it for the next turn
    history.AddAssistantMessage(response.Content);
}
```

---

## 4. Basic RAG in .NET

To implement RAG, you need two things before calling the LLM:
1. An Embedding Model.
2. A Vector Database to search.

Here is the conceptual flow using `Microsoft.SemanticKernel.Memory` (or a dedicated vector DB client like the `Azure.Search.Documents` SDK):

```csharp
// 1. User asks a question
string question = "What is our WFH policy?";

// 2. Query the Vector Database to find chunks matching the question
var relevantChunks = await vectorDbClient.SearchAsync(question, topK: 3);
string context = string.Join("\n\n", relevantChunks.Select(c => c.Text));

// 3. Build the RAG Prompt
string prompt = $@"
Answer the question using ONLY the context provided.
If the answer is not in the context, say 'I don't know'.

CONTEXT:
{context}

QUESTION:
{question}
";

// 4. Send to LLM
var response = await kernel.InvokePromptAsync(prompt);
Console.WriteLine(response);
```

---

## 🧪 Exercise: Add a "Summarize" Plugin

Semantic Kernel allows you to create Plugins (tools). 
1. Look up the Semantic Kernel documentation for creating a `KernelFunction`.
2. Create a C# method `string SummarizeText(string input)` and decorate it with `[KernelFunction]`.
3. Add it to your Kernel. Now the AI can call your C# code when it needs to summarize something!

---

**Next:** [05-AI-APPS-PYTHON.md](./05-AI-APPS-PYTHON.md) — See how this exact same workflow looks in Python using LangChain.
