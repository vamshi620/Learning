# .NET Azure Project — Team Quick Start Guide
## Start Here → Your Team's Roadmap to Azure Deployment

**Created:** April 27, 2026  
**Audience:** .NET developers new to Azure  
**Goal:** Deploy your .NET applications to App Service and AKS with confidence

---

## 👋 Welcome

This folder contains **practical, project-focused guides** for your .NET team. Unlike reference documentation, these files are designed to be followed step-by-step as you work on real projects. Each file explains the **what**, **why**, and **how** — so your team learns while building.

---

## 📂 File Map — Read in This Order

```
PHASE 1: SETUP (Day 1)
├── 01-AZURE-SETUP-FOR-DOTNET-TEAMS.md     ← Set up Azure accounts, CLI, and permissions
│
PHASE 2: PREPARE YOUR APP (Day 2-3)
├── 02-PREPARE-DOTNET-APP-FOR-CLOUD.md     ← Make your .NET app cloud-ready
│
PHASE 3: DEPLOY (Day 4-7)
├── 03-DEPLOY-TO-APP-SERVICE.md            ← Deploy to Azure App Service (simpler)
├── 04-DEPLOY-TO-AKS-STEP-BY-STEP.md      ← Deploy to AKS with Docker (advanced)
│
PHASE 4: AUTOMATE (Week 2)
├── 05-CI-CD-PIPELINE-SETUP.md             ← Automate builds and deployments
│
PHASE 5: OPERATE (Week 3+)
├── 06-MONITORING-AND-DEBUGGING.md         ← Monitor, alert, and debug in production
├── 07-SECURITY-CHECKLIST.md               ← Secure your apps and infrastructure
├── 08-COST-MANAGEMENT-GUIDE.md            ← Control and optimize Azure spending
│
REFERENCE (Anytime)
├── 09-TROUBLESHOOTING-PLAYBOOK.md         ← Fix common issues fast
└── 10-COMMANDS-CHEATSHEET.md              ← All commands on one page
```

---

## 🎯 Which Deployment Option Is Right for You?

```
Question: How many microservices do you have?

   1-3 services, simple web app/API?
   └── START WITH APP SERVICE (File 03)
       ✅ Simplest to deploy and manage
       ✅ No Docker knowledge needed (optional)
       ✅ Built-in SSL, scaling, deployment slots
       ✅ Best for: APIs, web apps, simple backends

   4+ services, or need containers?
   └── GO WITH AKS (File 04)
       ✅ Full container orchestration
       ✅ Microservices architecture
       ✅ Advanced scaling and networking
       ⚠️ Requires Docker + Kubernetes knowledge
```

> **Recommendation for beginners:** Start with **App Service** (File 03) to get your first app running in Azure today. Move to AKS later when your architecture demands it.

---

## 🛠️ Prerequisites Checklist

Before your team starts, everyone needs these installed:

### Software (Every Team Member)

| Tool | What It Does | Install |
|------|-------------|---------|
| **.NET 8 SDK** | Build and run your app | `winget install Microsoft.DotNet.SDK.8` |
| **Visual Studio 2022** or **VS Code** | Code editor | `winget install Microsoft.VisualStudio.2022.Community` |
| **Azure CLI** | Manage Azure from terminal | `winget install Microsoft.AzureCLI` |
| **Git** | Source control | `winget install Git.Git` |
| **Docker Desktop** | Build containers (for AKS path) | `winget install Docker.DockerDesktop` |
| **kubectl** | Manage Kubernetes (for AKS path) | `az aks install-cli` |

### Azure Access (Team Lead / Admin)

- [ ] Azure subscription created (Free tier: $200 credit for 30 days)
- [ ] Resource Group created for the project
- [ ] Each team member has Azure access (Contributor role on the Resource Group)
- [ ] Azure DevOps or GitHub repository set up for the project

### Verify Everything Works

```powershell
# Run these commands in PowerShell to verify installation
dotnet --version          # Should show 8.x.x
az --version              # Should show 2.x.x
git --version             # Should show 2.x.x
docker --version          # Should show 2x.x.x (only for AKS path)

# Login to Azure
az login                  # Opens browser for login
az account show           # Shows your subscription
```

---

## 📋 Team Roles and Responsibilities

| Role | Person | Responsibility | Key Files |
|------|--------|---------------|-----------|
| **Tech Lead** | _________ | Architecture decisions, code reviews | All files |
| **Developer 1** | _________ | App preparation, API development | 02, 03/04 |
| **Developer 2** | _________ | App preparation, frontend/services | 02, 03/04 |
| **DevOps Lead** | _________ | CI/CD, infrastructure, monitoring | 01, 05, 06 |
| **Security** | _________ | Security review, Key Vault setup | 07 |

---

## 🗓️ Suggested Timeline

| Week | Activity | Files |
|------|----------|-------|
| **Week 1** | Azure setup + prepare .NET app + first deployment | 01 → 02 → 03 |
| **Week 2** | Set up CI/CD pipeline + AKS deployment (if needed) | 04 → 05 |
| **Week 3** | Monitoring + security hardening | 06 → 07 |
| **Week 4** | Cost optimization + production readiness | 08 → 09 → 10 |

---

## ❓ Quick FAQ

**Q: Do I need Docker for App Service?**  
A: No! App Service can deploy directly from your .NET code. Docker is optional for App Service but required for AKS.

**Q: How much will Azure cost?**  
A: See File 08 for detailed estimates. Quick answer: Dev/test for a small team starts around $50-100/month with App Service, $200-400/month with AKS.

**Q: Can we start with App Service and move to AKS later?**  
A: Absolutely! This is the recommended approach. File 02 prepares your app for both.

**Q: What if something goes wrong?**  
A: File 09 has a troubleshooting playbook for every common issue.

---

> **Next Step:** Open [01-AZURE-SETUP-FOR-DOTNET-TEAMS.md](01-AZURE-SETUP-FOR-DOTNET-TEAMS.md) and follow the setup instructions.
