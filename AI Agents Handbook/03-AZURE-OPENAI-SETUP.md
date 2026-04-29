# Generative AI & Agents
## File 03: Azure OpenAI Setup & Management

---

## What You'll Learn
- The difference between OpenAI (Public) and Azure AI Foundry (Enterprise).
- How to provision via the Azure AI Foundry portal (ai.azure.com).
- Deploying models vs. using the model catalog.
- Authentication mechanisms (API Keys vs. Managed Identity).
- New April 2026: Foundry Local, BYO AI Gateway.

**Time Required:** 30 minutes

> **📅 Updated: April 2026** — Azure AI Studio is now Azure AI Foundry. Navigate to [ai.azure.com](https://ai.azure.com).

---

## 1. OpenAI vs. Azure OpenAI

OpenAI created GPT-4, but if you build an enterprise application sending sensitive customer data to the public `api.openai.com`, you might violate data compliance laws (GDPR, HIPAA) and risk your data being used to train future public models.

**Azure OpenAI** takes OpenAI's exact models and hosts them inside your private Azure tenant.
- **Privacy:** Microsoft guarantees your prompts and data are NEVER used to train foundational models.
- **Security:** You can place Azure OpenAI behind a VNet and use Entra ID (Azure AD) for authentication.
- **Compliance:** Inherits Azure's massive compliance certifications.

---

## 2. Provisioning via Azure AI Foundry (April 2026)

> **Important:** Azure AI Studio has been renamed to **Azure AI Foundry**. Go to [ai.azure.com](https://ai.azure.com).

Provisioning now uses the **AI Foundry Hub + Project** model:

1. Go to [ai.azure.com](https://ai.azure.com).
2. Click **+ Create** → **Hub** (your organization-level container).
3. Inside the Hub, create a **Project** (your application-level container).
4. Access is no longer gated by a Microsoft approval form for most Azure subscriptions.
5. Inside your Project, go to **Model catalog** → find your model → **Deploy**.

*Note: PTU (Provisioned Throughput Units) deployments are now recommended for production workloads with predictable traffic.*

---

## 3. Deployments vs. Model Catalog

In the public OpenAI API, you simply specify `"model": "gpt-4"` in your code.
In Azure AI Foundry, you must create a **Deployment**.

### How to Create a Deployment:
1. Open [ai.azure.com](https://ai.azure.com) and open your Project.
2. Go to **Models + endpoints** → **Deploy model**.
3. Browse the model catalog — includes OpenAI models (GPT-5.4, o4-mini), Microsoft models (Phi-4-Reasoning-Vision), and 3rd-party models (Claude Sonnet 4.6, Gemini 3.1 Pro via MaaS).
4. Select your model and choose **Deployment type:**
   - **Standard** — pay per token, best for variable workloads
   - **Global Standard** — higher throughput via global routing
   - **Provisioned (PTU)** — reserved capacity, best for production SLAs
5. Give it a **Deployment Name** (e.g., `gpt-5-4-prod`).

> **April 2026:** You can now deploy **Claude Sonnet 4.6** and **Gemini 3.1 Pro** directly inside Azure AI Foundry as first-class Models-as-a-Service (MaaS), billed to your Azure subscription — no separate Anthropic or Google accounts needed.

---

## 4. Required Configuration Values (April 2026)

To connect any application (C# or Python) to Azure AI Foundry, you need:

1. **Endpoint:** `https://<hub-name>.services.ai.azure.com/` *(new format for AI Foundry Hub endpoints)*
2. **API Version:** Use `2025-01-01-preview` or later for access to latest models.
3. **Deployment Name:** The name you gave your model deployment.
4. **Authentication:**
   - *Option A:* API Key (from **Keys and Endpoint** tab in your Project settings).
   - *Option B:* `DefaultAzureCredential` via Managed Identity (required for production).

---

## 5. Security Best Practices

### The Danger of API Keys
If you accidentally commit your Azure OpenAI API Key to GitHub, attackers can use it to generate thousands of dollars of AI responses at your expense within minutes.

### Using Managed Identity
Instead of keys, use Entra ID.

**1. Assign the Role:**
In the Azure Portal, go to your Azure OpenAI resource -> Access Control (IAM) -> Add Role Assignment. Assign the **"Cognitive Services OpenAI User"** role to your App Service's Managed Identity (or your own developer account for local testing).

**2. In .NET Code:**
Use `DefaultAzureCredential` from the `Azure.Identity` package. This completely eliminates the need for an API Key in your `appsettings.json`.

```csharp
using Azure.Identity;
using Azure.AI.Inference;  // New package for Azure AI Foundry (April 2026)

var endpoint = new Uri("https://my-hub.services.ai.azure.com/");
var credentials = new DefaultAzureCredential();

// Use the new ChatCompletionsClient (wraps as IChatClient via Microsoft.Extensions.AI)
var client = new ChatCompletionsClient(endpoint, credentials);
var chatClient = client.AsChatClient(modelId: "gpt-5-4-prod");
```

---

## 6. New in April 2026: BYO AI Gateway

You can now connect the Foundry Agent Service to models hosted behind your own **Azure API Management** or custom AI gateway. This gives enterprises:
- Full control over model routing and rate limits.
- Ability to use fine-tuned or on-premises models with the Foundry agent infrastructure.
- Centralized billing and governance across multiple model providers.

---

## 🧪 Exercise: Chat Playground

1. Go to [ai.azure.com](https://ai.azure.com) (Azure AI Foundry).
2. Open your Project → **Playgrounds** → **Chat Playground**.
3. Under the **Setup** pane, find the **System message** box.
4. Type: *"You are an AI that only speaks in pirate slang. You must end every sentence with 'Arrr!'."*
5. Try switching between different deployed models (GPT-5.4 vs. o4-mini) and notice the difference in reasoning quality!

---

**Next:** [04-AI-APPS-DOTNET.md](./04-AI-APPS-DOTNET.md) — Let's write some C# code to build an AI app using Semantic Kernel.
