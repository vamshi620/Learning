# Generative AI & Agents
## File 03: Azure OpenAI Setup & Management

---

## What You'll Learn
- The difference between OpenAI (Public) and Azure OpenAI (Enterprise).
- How to provision an Azure OpenAI resource.
- Deploying models vs. using base models.
- Authentication mechanisms (API Keys vs. Managed Identity).

**Time Required:** 30 minutes

---

## 1. OpenAI vs. Azure OpenAI

OpenAI created GPT-4, but if you build an enterprise application sending sensitive customer data to the public `api.openai.com`, you might violate data compliance laws (GDPR, HIPAA) and risk your data being used to train future public models.

**Azure OpenAI** takes OpenAI's exact models and hosts them inside your private Azure tenant.
- **Privacy:** Microsoft guarantees your prompts and data are NEVER used to train foundational models.
- **Security:** You can place Azure OpenAI behind a VNet and use Entra ID (Azure AD) for authentication.
- **Compliance:** Inherits Azure's massive compliance certifications.

---

## 2. Provisioning the Resource

To use Azure OpenAI, you must first create a resource in the Azure Portal.

1. Go to the Azure Portal.
2. Search for **Azure OpenAI**.
3. Click **Create**.
4. Choose your Subscription, Resource Group, and Region (e.g., `East US`).
5. Choose a Pricing Tier (`Standard S0`).

*Note: Azure OpenAI access is currently gated by Microsoft. Your Azure Subscription must be approved via a Microsoft application form before you can create this resource.*

---

## 3. Deployments vs. Models

In the public OpenAI API, you simply specify `"model": "gpt-4"` in your code.
In Azure OpenAI, you must create a **Deployment**.

A Deployment is an instance of a specific model version assigned a name of your choosing.

### How to Create a Deployment:
1. Open the **Azure AI Studio** (from your Azure OpenAI resource overview page).
2. Go to **Deployments** -> **Create new deployment**.
3. Select a model (e.g., `gpt-4o`).
4. Select a version (e.g., `2024-05-13`).
5. Give it a **Deployment Name** (e.g., `my-gpt4-deployment`).

*Crucial Concept:* In your code, you will reference the **Deployment Name**, not the model name. If you name your deployment `my-custom-ai`, you will pass `my-custom-ai` in your code.

---

## 4. Required Configuration Values

To connect any application (C# or Python) to Azure OpenAI, you need three pieces of information:

1. **Endpoint:** `https://<your-resource-name>.openai.azure.com/`
2. **API Version:** Azure APIs require a date-based version string (e.g., `2024-02-15-preview`).
3. **Authentication:** 
   - *Option A:* An API Key (Found under "Keys and Endpoint" in the portal).
   - *Option B:* Managed Identity (Recommended for production).

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
using Azure.AI.OpenAI;

var endpoint = new Uri("https://my-resource.openai.azure.com/");
var credentials = new DefaultAzureCredential();

var client = new OpenAIClient(endpoint, credentials);
```

---

## 🧪 Exercise: Chat Playground

1. Go to Azure AI Studio.
2. Navigate to the **Chat Playground**.
3. Under the **Setup** pane, find the **System Message** box.
4. Type: *"You are an AI that only speaks in pirate slang. You must end every sentence with 'Arrr!'."*
5. Chat with the model in the main window and observe how powerfully the System Prompt dictates its behavior.

---

**Next:** [04-AI-APPS-DOTNET.md](./04-AI-APPS-DOTNET.md) — Let's write some C# code to build an AI app using Semantic Kernel.
