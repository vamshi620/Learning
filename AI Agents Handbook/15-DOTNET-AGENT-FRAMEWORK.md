# Generative AI & Agents
## File 15: Microsoft Agent Framework — Semantic Kernel Agents, AutoGen.NET & Microsoft.Extensions.AI

---

## What You'll Learn
- The 2025 .NET AI stack: where each library fits.
- `Microsoft.Extensions.AI` — the new unified abstraction layer.
- `Microsoft.Extensions.VectorData` — the vector DB abstraction.
- Semantic Kernel Agent Framework — building single and multi-agent systems.
- AutoGen.NET — Microsoft's multi-agent conversation framework.
- Azure AI Foundry Agent Service — fully managed agents in the cloud.

**Time Required:** 60 minutes

---

## 1. The 2025 .NET AI Landscape — Know Your Layers

Microsoft reorganized the entire .NET AI ecosystem in 2024–2025. Understanding the layers is critical to choosing the right tool:

```
┌────────────────────────────────────────────────────────────────┐
│                    Your Application Code                       │
├────────────────────────────────────────────────────────────────┤
│      Semantic Kernel / AutoGen.NET  (Agent Orchestration)      │
├────────────────────────────────────────────────────────────────┤
│   Microsoft.Extensions.AI  (IChatClient / IEmbeddingGenerator) │
├────────────────────────────────────────────────────────────────┤
│  Azure OpenAI SDK  |  OpenAI SDK  |  Ollama  |  GitHub Models │
└────────────────────────────────────────────────────────────────┘
```

| Library | Role | When to Use |
|---------|------|------------|
| `Microsoft.Extensions.AI` | Low-level abstraction (`IChatClient`) | All .NET apps — replaces vendor-specific SDKs |
| `Microsoft.Extensions.VectorData` | Vector DB abstraction | RAG pipelines — swap vector DBs without code changes |
| `Microsoft.SemanticKernel` | Agent orchestration, Plugins, Process Framework | Complex AI apps, multi-step workflows |
| `Microsoft.AutoGen` | Multi-agent conversations (Actor model) | When agents need to talk to each other |
| Azure AI Foundry Agent Service | Fully managed agents in the cloud | Production agents without infrastructure management |

---

## 2. Microsoft.Extensions.AI — The Unified Abstraction

This is the **most important addition to .NET in 2025**. It provides a single `IChatClient` interface so your code is not tied to any specific AI vendor.

### Why it matters:
- Write code against `IChatClient`.
- Switch from Azure OpenAI → OpenAI → Ollama → GitHub Models without changing a single line of your application code.
- Built-in middleware pipeline for logging, caching, retries, and telemetry.

### NuGet Packages

```powershell
# Core abstraction (always install this)
dotnet add package Microsoft.Extensions.AI

# Provider implementation — choose one or more:
dotnet add package Microsoft.Extensions.AI.AzureAIInference   # Azure AI Foundry / Azure OpenAI
dotnet add package Microsoft.Extensions.AI.OpenAI             # OpenAI direct
dotnet add package Microsoft.Extensions.AI.Ollama             # Local models via Ollama
```

### IChatClient — The Universal Interface

```csharp
using Microsoft.Extensions.AI;
using Azure.AI.Inference;
using Azure.Identity;

// 1. Create a provider-specific client
var azureClient = new ChatCompletionsClient(
    new Uri("https://your-resource.services.ai.azure.com"),
    new DefaultAzureCredential());

// 2. Wrap it in the universal IChatClient interface
IChatClient chatClient = azureClient.AsChatClient("gpt-4o");

// 3. Your app code only uses IChatClient — zero coupling to Azure
var response = await chatClient.GetResponseAsync(
    "Explain Dependency Injection in .NET in 2 sentences."
);
Console.WriteLine(response.Text);
```

### Registering in ASP.NET Core DI

```csharp
// In Program.cs
builder.Services.AddAzureAIInferenceChatClient(
    new Uri(builder.Configuration["AzureOpenAI:Endpoint"]!),
    new DefaultAzureCredential(),
    builder.Configuration["AzureOpenAI:DeploymentName"]!)
    // Add middleware pipeline
    .UseOpenTelemetry()       // Automatic tracing to Application Insights
    .UseLogging()             // Log all requests/responses
    .UseFunctionInvocation(); // Auto-invoke tool calls

// Any service can now inject IChatClient
```

```csharp
// In any service class:
public class MyAiService(IChatClient chatClient)
{
    public async Task<string> ReviewCodeAsync(string code)
    {
        var messages = new List<ChatMessage>
        {
            new(ChatRole.System, "You are a .NET 8 code reviewer."),
            new(ChatRole.User, $"Review this code:\n```csharp\n{code}\n```")
        };
        
        var response = await chatClient.GetResponseAsync(messages);
        return response.Text;
    }
}
```

### The Middleware Pipeline

```csharp
// Build a sophisticated middleware pipeline for production
IChatClient chatClient = new AzureOpenAIClient(...)
    .AsChatClient("gpt-4o")
    .AsBuilder()
    .UseDistributedCache(cache)          // Cache identical requests (saves cost)
    .UseRateLimiting(rateLimiter)        // Protect against runaway token usage
    .UseRetryOnError(maxRetries: 3)      // Auto-retry on transient failures
    .UseOpenTelemetry(logContents: true) // Full tracing + token counting
    .Build();
```

---

## 3. Microsoft.Extensions.VectorData — Vector DB Abstraction

The same portability principle, applied to vector databases.

```powershell
dotnet add package Microsoft.Extensions.VectorData
dotnet add package Microsoft.SemanticKernel.Connectors.AzureAISearch  # Or Qdrant, Pinecone, etc.
```

```csharp
using Microsoft.Extensions.VectorData;
using Microsoft.SemanticKernel.Connectors.AzureAISearch;

// Your data model
public class CompanyDocument
{
    [VectorStoreKey]
    public string Id { get; set; } = "";

    [VectorStoreData]
    public string Content { get; set; } = "";

    [VectorStoreData]
    public string Department { get; set; } = "";

    [VectorStoreVector(Dimensions: 1536)]  // text-embedding-3-small dimensions
    public ReadOnlyMemory<float> Embedding { get; set; }
}

// Register in DI
builder.Services.AddAzureAISearchVectorStore(
    new Uri(builder.Configuration["AzureSearch:Endpoint"]!),
    new DefaultAzureCredential());

// Use IVectorStore anywhere
public class DocumentSearchService(IVectorStore vectorStore, IEmbeddingGenerator<string, Embedding<float>> embeddingGenerator)
{
    public async Task<IList<CompanyDocument>> SearchAsync(string query)
    {
        var collection = vectorStore.GetCollection<string, CompanyDocument>("company-docs");
        
        var queryEmbedding = await embeddingGenerator.GenerateEmbeddingVectorAsync(query);
        
        var results = collection.SearchAsync(queryEmbedding, top: 5);
        
        var docs = new List<CompanyDocument>();
        await foreach (var result in results)
        {
            docs.Add(result.Record);
        }
        return docs;
    }
}
```

---

## 4. Semantic Kernel Agent Framework (2025)

Semantic Kernel v1.x introduced a first-class **Agent Framework**. Agents are SK entities that can plan, use Plugins (tools), and collaborate with other agents.

```powershell
dotnet add package Microsoft.SemanticKernel
dotnet add package Microsoft.SemanticKernel.Agents.Core
dotnet add package Microsoft.SemanticKernel.Agents.AzureAI   # For Azure AI Foundry agents
```

### Creating a ChatCompletionAgent

```csharp
using Microsoft.SemanticKernel;
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.ChatCompletion;

var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion(
        deploymentName: "gpt-4o",
        endpoint: "https://your-resource.openai.azure.com/",
        credential: new DefaultAzureCredential())
    .Build();

// Create an agent with a specific persona and tools
var agent = new ChatCompletionAgent
{
    Name = "DevOpsAssistant",
    Instructions = """
        You are an expert Azure DevOps assistant for Contoso.
        Help developers with CI/CD pipelines, Kubernetes manifests, and Azure deployments.
        Always provide working PowerShell commands.
        When you don't know, recommend the #devops-help Slack channel.
        """,
    Kernel = kernel,
};

// Create a conversation thread
var thread = new ChatHistoryAgentThread();
await thread.AddSystemMessageAsync("Session started for user: Vamshi");

// Chat with the agent
await foreach (var message in agent.InvokeAsync("How do I deploy a .NET 8 app to AKS?", thread))
{
    Console.Write(message.Content);
}
```

### Adding Plugins (Tools) to an Agent

```csharp
using Microsoft.SemanticKernel;

// 1. Define a plugin
public class AzureDevOpsPlugin
{
    [KernelFunction("get_pipeline_status")]
    [Description("Gets the current status of an Azure DevOps pipeline by its name")]
    public async Task<string> GetPipelineStatusAsync(
        [Description("The pipeline name to check")] string pipelineName)
    {
        // Call the real Azure DevOps REST API
        // https://dev.azure.com/{org}/{project}/_apis/build/builds?...
        return $"Pipeline '{pipelineName}' is currently: Running (Build #1234)";
    }

    [KernelFunction("trigger_deployment")]
    [Description("Triggers a deployment pipeline for the specified environment")]
    public async Task<string> TriggerDeploymentAsync(
        string applicationName, 
        string environment)
    {
        return $"Deployment of '{applicationName}' to '{environment}' triggered. Build ID: 5678";
    }
}

// 2. Register the plugin with the agent's kernel
kernel.Plugins.AddFromType<AzureDevOpsPlugin>();

// Now when the user asks "What's the status of the release-prod pipeline?",
// the agent will automatically call GetPipelineStatusAsync("release-prod")
```

---

## 5. Multi-Agent Systems with Semantic Kernel — AgentGroupChat

```csharp
using Microsoft.SemanticKernel.Agents;
using Microsoft.SemanticKernel.Agents.Chat;

// Define two specialized agents
var reviewerAgent = new ChatCompletionAgent
{
    Name = "CodeReviewer",
    Instructions = "Review code for bugs, security issues, and SOLID violations. Be critical.",
    Kernel = kernel
};

var refactorAgent = new ChatCompletionAgent
{
    Name = "Refactorer",
    Instructions = "Take the reviewer's feedback and rewrite the code to address all issues.",
    Kernel = kernel
};

// Create a group chat that orchestrates them
var groupChat = new AgentGroupChat(reviewerAgent, refactorAgent)
{
    ExecutionSettings = new AgentGroupChatSettings
    {
        // Agents take turns until the termination condition is met
        TerminationStrategy = new KernelFunctionTerminationStrategy(
            kernel.CreateFunctionFromPrompt(
                "Return TERMINATE if the refactored code has no more issues. Otherwise CONTINUE."),
            kernel)
        {
            MaximumIterations = 6
        },
        // Round-robin selection between agents
        SelectionStrategy = new SequentialSelectionStrategy()
    }
};

// Add the code to review
groupChat.AddChatMessage(new ChatMessageContent(
    AuthorRole.User, 
    $"Review and fix this code:\n```csharp\n{codeToReview}\n```"
));

// Run the multi-agent conversation
await foreach (var message in groupChat.InvokeAsync())
{
    Console.WriteLine($"[{message.AuthorName}]: {message.Content}\n");
}
```

---

## 6. Microsoft AutoGen.NET

**AutoGen** is Microsoft Research's multi-agent framework that treats agents as **actors** — autonomous units that communicate via messages. Now available for .NET.

```powershell
dotnet add package Microsoft.AutoGen.Agents
dotnet add package Microsoft.AutoGen.Runtime.Grpc
```

```csharp
using Microsoft.AutoGen.Agents;
using Microsoft.AutoGen.Contracts;

// Define an agent that handles a specific task
[TopicSubscription("code-review-requests")]
public class CodeReviewAgent : ConsoleAgent,
    IHandle<CodeReviewRequest>,
    IHandle<RefactoredCode>
{
    private readonly IChatClient _chatClient;

    public CodeReviewAgent(IChatClient chatClient) : base(/* ... */)
    {
        _chatClient = chatClient;
    }

    public async Task HandleAsync(CodeReviewRequest request, MessageContext ctx)
    {
        var review = await _chatClient.GetResponseAsync(
            $"Review this C# code for bugs: {request.Code}");

        // Publish result to another topic for other agents to consume
        await PublishMessageAsync(
            new ReviewCompleted { Issues = review.Text, OriginalCode = request.Code },
            new TopicId("review-results"));
    }
}

// Register agents in the host
builder.Services
    .AddSingleton<IChatClient>(/* Azure OpenAI client */)
    .AddAgent<CodeReviewAgent>();
```

---

## 7. Azure AI Foundry Agent Service (Managed Agents)

For production scenarios where you want Microsoft to manage the infrastructure, use the **Azure AI Foundry Agent Service** — the fully managed, cloud-native agent platform.

```powershell
dotnet add package Azure.AI.Projects
```

```csharp
using Azure.AI.Projects;
using Azure.Identity;

// 1. Connect to your Azure AI Foundry Project
var projectClient = new AIProjectClient(
    connectionString: "eastus.api.azureml.ms;00000000-0000-0000-0000-000000000000;my-rg;my-project",
    credential: new DefaultAzureCredential());

var agentsClient = projectClient.GetAgentsClient();

// 2. Create a managed agent (persisted in Azure — no local infrastructure needed)
var agent = await agentsClient.CreateAgentAsync(
    model: "gpt-4o",
    name: "ContosoDeveloperAssistant",
    instructions: """
        You are an expert .NET developer assistant for Contoso.
        Help with code reviews, architecture decisions, and Azure deployments.
        """,
    tools: new List<ToolDefinition> 
    { 
        new FileSearchToolDefinition(),  // Built-in RAG over uploaded files
        new CodeInterpreterToolDefinition() // Built-in Python code execution
    });

Console.WriteLine($"Agent created: {agent.Value.Id}");

// 3. Create a thread and run it
var thread = await agentsClient.CreateThreadAsync();

await agentsClient.CreateMessageAsync(
    thread.Value.Id,
    MessageRole.User,
    "Review this C# class for thread safety issues: [paste code here]");

var run = await agentsClient.CreateRunAsync(thread.Value.Id, agent.Value.Id);

// 4. Poll until complete
while (run.Value.Status == RunStatus.Queued || run.Value.Status == RunStatus.InProgress)
{
    await Task.Delay(1000);
    run = await agentsClient.GetRunAsync(thread.Value.Id, run.Value.Id);
}

// 5. Retrieve messages
var messages = agentsClient.GetMessagesAsync(thread.Value.Id);
await foreach (var message in messages)
{
    if (message.Role == MessageRole.Agent)
    {
        Console.WriteLine(message.ContentItems.OfType<MessageTextContent>().FirstOrDefault()?.Text);
    }
}
```

---

## 8. Which Microsoft AI Tool Should You Use?

Use this decision guide:

```
Is this a simple AI feature in an existing .NET API?
    YES → Use Microsoft.Extensions.AI (IChatClient) only
    
Do you need RAG (searching over documents)?
    YES → Add Microsoft.Extensions.VectorData + IChatClient
    
Does your AI need to use tools (call APIs, query DB)?
    YES → Use Semantic Kernel with Plugins (KernelFunction)
    
Do you need MULTIPLE agents collaborating?
    YES → 
        Agents need to share structured data/messages? → AutoGen.NET
        Agents taking turns reviewing/refining? → SK AgentGroupChat
    
Do you want Microsoft to manage everything (scaling, state, infrastructure)?
    YES → Azure AI Foundry Agent Service (Azure.AI.Projects)
```

---

## 9. The Latest Models Available in Azure AI Foundry (2025)

| Model | Context Window | Best For | Cost Tier |
|-------|---------------|---------|-----------|
| `gpt-4.1` | 1M tokens | Long-document analysis, complex reasoning | Highest |
| `gpt-4o` | 128K tokens | General purpose, balanced speed & quality | High |
| `gpt-4o-mini` | 128K tokens | High-volume, cost-sensitive apps | Low |
| `o1` / `o3` | 200K tokens | Deep reasoning, math, advanced code | Highest |
| `o4-mini` | 200K tokens | Fast reasoning at lower cost | Medium |
| `phi-4` | 16K tokens | On-prem/edge deployment via Ollama | Free (self-host) |

---

## 🧪 Exercise: Migrate from Raw SDK to IChatClient

1. Take the Semantic Kernel code in [File 04](./04-AI-APPS-DOTNET.md).
2. Replace the `IChatCompletionService` with `IChatClient` from `Microsoft.Extensions.AI`.
3. Register it in a proper ASP.NET Core DI container with `.UseOpenTelemetry()` middleware.
4. Switch the provider from Azure OpenAI to **Ollama** (local) by changing just one line in `Program.cs`.
5. Verify your application code required zero changes — only the DI registration changed!

---

**Next:** [00-START-HERE.md](./00-START-HERE.md) — You have mastered the full .NET AI stack!
