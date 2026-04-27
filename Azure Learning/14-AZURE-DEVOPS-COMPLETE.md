# 14 — Azure DevOps: Complete A-Z Reference

> **Zero-to-Hero Azure DevOps Handbook**
> Covers Azure Boards, Repos, Pipelines, Test Plans, Artifacts, GitHub Actions comparison, and real-world pipeline patterns.

---

## Table of Contents

1. [Azure DevOps Introduction](#1-azure-devops-introduction)
2. [Azure Boards](#2-azure-boards)
3. [Azure Repos](#3-azure-repos)
4. [Azure Pipelines (YAML) — Very Detailed](#4-azure-pipelines-yaml)
5. [Azure Test Plans](#5-azure-test-plans)
6. [Azure Artifacts](#6-azure-artifacts)
7. [GitHub Actions vs Azure Pipelines](#7-github-actions-vs-azure-pipelines)
8. [Common Pipeline Patterns](#8-common-pipeline-patterns)

---

## 1. Azure DevOps Introduction

### What is Azure DevOps?

Azure DevOps is Microsoft's end-to-end DevOps platform that provides developer services for teams to plan work, collaborate on code development, and build and deploy applications. It is a cloud-based SaaS platform (also available as on-premises **Azure DevOps Server**, formerly TFS).

Azure DevOps covers the full software development lifecycle:

```
Plan → Code → Build → Test → Release → Deploy → Operate → Monitor
 ↑                                                              ↓
 └──────────────────── Continuous Feedback ────────────────────┘
```

Key facts:
- Accessible at `https://dev.azure.com/{yourOrganization}`
- Supports any language, any platform (Windows, Linux, macOS, cloud, on-prem)
- Integrates natively with GitHub, Jira, Slack, Teams, Jenkins, SonarQube, and more
- Each service can be used independently — you don't have to use all five

---

### Azure DevOps vs GitHub vs Jenkins

| Feature                        | Azure DevOps            | GitHub                     | Jenkins                       |
|-------------------------------|-------------------------|----------------------------|-------------------------------|
| **CI/CD**                     | Azure Pipelines (YAML)  | GitHub Actions (YAML)      | Jenkinsfile (Groovy DSP)      |
| **Source Control**            | Azure Repos (Git/TFVC)  | GitHub Repos               | External (GitHub, Bitbucket)  |
| **Work Tracking**             | Azure Boards (rich)     | GitHub Issues/Projects     | None (use Jira)               |
| **Package Registry**          | Azure Artifacts         | GitHub Packages            | None (Nexus/Artifactory)      |
| **Test Management**           | Azure Test Plans        | None native                | None native                   |
| **Hosted Runners/Agents**     | Yes (Windows/Linux/Mac) | Yes (GitHub-hosted)        | No (self-hosted only)         |
| **Self-Hosted Agents**        | Yes (agent pools)       | Yes (self-hosted runners)  | Yes (nodes)                   |
| **Marketplace Extensions**    | Azure DevOps Marketplace| GitHub Marketplace         | Jenkins Plugin Directory      |
| **Private repos free tier**   | 5 users free            | Unlimited                  | Free (self-hosted)            |
| **Enterprise on-prem option** | Azure DevOps Server     | GitHub Enterprise Server   | Jenkins (self-hosted)         |
| **Best for**                  | Enterprise, .NET, Azure | Open Source, GitHub native | Legacy, highly customized     |

**Decision Guide:**
- Use **Azure DevOps** when: Enterprise team on Azure, need Test Plans, migrating from TFS, want full integration across boards/repos/pipelines
- Use **GitHub + Actions** when: open source project, greenfield startup, GitHub-first culture
- Use **Jenkins** when: existing Jenkins investment, complex custom pipeline logic, on-prem only requirement

---

### Components: Boards, Repos, Pipelines, Test Plans, Artifacts

```
┌──────────────────────────────────────────────────────────────────┐
│                        Azure DevOps                              │
│                                                                  │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  Boards   │  │   Repos   │  │ Pipelines │  │Test Plans │    │
│  │           │  │           │  │           │  │           │    │
│  │ Plan work │  │ Host code │  │ Build &   │  │ Manual &  │    │
│  │ track bugs│  │ Git/TFVC  │  │ Deploy    │  │ automated │    │
│  │ sprints   │  │ PRs       │  │ CI/CD     │  │ tests     │    │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                      Artifacts                            │  │
│  │       Host NuGet, npm, Maven, Python, Universal packages  │  │
│  └───────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

### Organizations, Projects, Teams

**Hierarchy:**

```
Azure DevOps Organization (e.g., https://dev.azure.com/contoso)
│
├── Project: ECommerceApp
│   ├── Team: Backend Team
│   ├── Team: Frontend Team
│   ├── Team: DevOps Team
│   └── Repositories, Pipelines, Boards, Artifacts
│
├── Project: MobileApp
│   ├── Team: iOS Team
│   └── Team: Android Team
│
└── Project: SharedInfra
    └── Team: Platform Team
```

- **Organization**: Top-level container. Billing is per-organization. Maps to a company or a department.
- **Project**: Contains repos, pipelines, boards, artifacts. Can be public or private.
- **Team**: A group of people within a project. Each team can have its own backlog, sprints, and dashboards.
- **Area Path**: Hierarchical classification for work items (e.g., `ECommerceApp\Backend\API`)
- **Iteration Path**: Sprints / time-boxed periods (e.g., `ECommerceApp\Sprint 1`)

---

### Pricing (Free Tier, Basic, Basic + Test Plans)

| Plan                   | Price                   | What's Included                                                  |
|-----------------------|-------------------------|------------------------------------------------------------------|
| **Free Tier**          | $0                      | 5 free Basic users, unlimited Stakeholders, 1 Microsoft-hosted CI/CD parallel job (1800 min/month), unlimited private Git repos |
| **Basic**              | $6/user/month           | Full access to Boards, Repos, Pipelines (beyond free tier)      |
| **Basic + Test Plans** | $52/user/month          | Everything in Basic + Azure Test Plans (manual testing)         |
| **Visual Studio subscribers** | Included in VS license | Basic plan included at no extra cost                        |
| **Additional parallel jobs** | $40/job/month (hosted) | Extra pipeline parallel jobs                                 |
| **Self-hosted agents** | Free                    | Unlimited parallel jobs on self-hosted agents                   |

**Stakeholder access** (free, unlimited): Can view boards, create/edit work items, approve releases — but **cannot** access Repos or Pipelines.

---

### az devops CLI Setup and Authentication

The Azure DevOps CLI is an extension of the Azure CLI (`az`).

```bash
# ── Step 1: Install Azure CLI ──────────────────────────────────────────────────
# On Ubuntu/Debian:
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# On macOS:
brew update && brew install azure-cli

# On Windows (PowerShell):
winget install Microsoft.AzureCLI

# ── Step 2: Install Azure DevOps extension ────────────────────────────────────
az extension add --name azure-devops

# ── Step 3: Login to Azure ────────────────────────────────────────────────────
az login
# Or use device code flow (headless):
az login --use-device-code

# ── Step 4: Set your organization and project defaults ────────────────────────
az devops configure --defaults \
  organization=https://dev.azure.com/contoso \
  project=ECommerceApp

# Verify configuration:
az devops configure --list

# ── Step 5: Use a Personal Access Token (PAT) instead of interactive login ────
# Create a PAT in Azure DevOps:
# User Settings → Personal Access Tokens → New Token
# Scopes: select what you need (e.g., Full access for dev, targeted for CI)

export AZURE_DEVOPS_EXT_PAT=<your-pat-token>

# Or store in a file and reference:
echo "your-pat" | az devops login --organization https://dev.azure.com/contoso

# ── Useful diagnostic commands ────────────────────────────────────────────────
az devops project list --output table
az devops team list --project ECommerceApp --output table
az devops extension list --output table
```

---

## 2. Azure Boards

Azure Boards provides Agile planning tools: work item tracking, sprint planning, backlogs, boards, and reporting.

### Work Item Types: Epic, Feature, User Story, Task, Bug

**Default hierarchy for Scrum/Agile:**

```
Epic
└── Feature
    └── User Story (or Product Backlog Item in Scrum)
        ├── Task
        └── Bug
```

| Work Item Type   | Description                                                           | Who Creates It    |
|-----------------|-----------------------------------------------------------------------|-------------------|
| **Epic**         | Large body of work spanning multiple sprints/releases (e.g., "Payment System Overhaul") | Product Owner / Manager |
| **Feature**      | A chunk of an Epic (e.g., "Credit Card Payment Integration")         | Product Owner     |
| **User Story**   | A user-facing requirement (e.g., "As a user, I can pay with Visa")  | Product Owner / BA |
| **Task**         | Technical work to complete a User Story (e.g., "Implement Stripe API call") | Developer |
| **Bug**          | A defect found in the system (e.g., "Checkout button broken on Safari") | QA / Dev        |
| **Test Case**    | (With Test Plans) A specific test to verify behavior                 | QA Engineer       |
| **Impediment**   | (Scrum) A blocker preventing work (e.g., "Waiting for DB access")   | Scrum Master      |

**Work Item fields:**
- **Title**: Short description
- **State**: New → Active → Resolved → Closed (customizable per type)
- **Assigned To**: The person responsible
- **Area Path**: Classification (e.g., `ECommerceApp\Backend`)
- **Iteration Path**: Sprint (e.g., `ECommerceApp\Sprint 5`)
- **Priority**: 1 (critical) to 4 (low)
- **Story Points / Effort**: Estimation in points
- **Tags**: Free-form labels for filtering
- **Description**: Rich text with images, attachments
- **Discussion**: Comments thread

---

### Scrum vs Kanban vs CMMI Process Templates

When creating a project, you choose a **process template**. This determines work item types, states, and fields.

| Aspect                | Scrum                          | Agile                          | CMMI                           | Kanban              |
|-----------------------|-------------------------------|-------------------------------|-------------------------------|---------------------|
| **Work item types**   | PBI, Bug, Task, Epic, Feature  | User Story, Bug, Task, Epic   | Requirement, Change, Issue    | User Story (+ Kanban board view) |
| **Time-boxing**       | Sprints (fixed-length)         | Iterations                    | Iterations                    | Continuous flow     |
| **Story estimation**  | Story Points                   | Story Points                  | Size, Original Estimate       | None required       |
| **Velocity tracking** | Yes                            | Yes                           | Yes                           | Throughput-based    |
| **Best for**          | Scrum teams with fixed sprints | General Agile teams           | Regulated industries, CMMI compliance | Flow-based ops, support |
| **Complexity**        | Medium                         | Medium                        | High                          | Low                 |

**Which to choose:**
- Most teams → **Agile** (flexible, familiar terminology)
- Pure Scrum practitioner teams → **Scrum**
- Defense/government/highly regulated → **CMMI**
- Operations/support/continuous flow → use **Kanban board** view on any process

---

### Sprints and Iterations

Sprints are time-boxed iterations, typically 1-4 weeks.

```bash
# ── Creating and managing iterations with CLI ─────────────────────────────────

# List all iterations/sprints:
az boards iteration project list --project ECommerceApp --output table

# Create a new iteration (sprint):
az boards iteration project create \
  --name "Sprint 10" \
  --project ECommerceApp \
  --start-date "2024-07-01" \
  --finish-date "2024-07-14"

# Add iteration to a team (so the team "owns" this sprint):
az boards iteration team add \
  --team "Backend Team" \
  --project ECommerceApp \
  --id <iteration-id>

# Set current sprint for a team:
az boards iteration team set-backlog-iteration \
  --team "Backend Team" \
  --project ECommerceApp \
  --id <iteration-id>
```

**Sprint Ceremonies:**
1. **Sprint Planning**: Select User Stories from Product Backlog, define Sprint Goal
2. **Daily Standup**: 15-min sync (what did I do, what will I do, any blockers)
3. **Sprint Review**: Demo completed work to stakeholders
4. **Sprint Retrospective**: Team reflects on process improvements

---

### Boards, Backlogs, Queries

**Kanban Board:**
- Visual card-based view of work in a column-per-state layout
- Columns map to work item states (e.g., New → Active → Resolved → Done)
- Supports WIP (Work In Progress) limits per column
- Swimlanes for priority (e.g., Expedite lane for critical items)
- Color coding by tag, type, or priority

**Backlog:**
- Hierarchical list view of all work items
- Product Backlog: unordered list of all work to be done
- Sprint Backlog: items committed for the current sprint
- Can drag-and-drop to reorder/prioritize
- Parent-child relationships visible

**Queries:**
Query items across the project using WIQL (Work Item Query Language):

```
# Example WIQL query — find all active bugs assigned to me:
SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo]
FROM WorkItems
WHERE [System.WorkItemType] = 'Bug'
  AND [System.State] = 'Active'
  AND [System.AssignedTo] = @Me
ORDER BY [System.CreatedDate] DESC
```

```bash
# ── Work item CLI commands ────────────────────────────────────────────────────

# Create a User Story:
az boards work-item create \
  --type "User Story" \
  --title "As a user, I can reset my password via email" \
  --project ECommerceApp \
  --area "ECommerceApp\Authentication" \
  --iteration "ECommerceApp\Sprint 10" \
  --assigned-to "jane@contoso.com" \
  --fields "Microsoft.VSTS.Scheduling.StoryPoints=5" "System.Tags=authentication;security"

# Show a work item:
az boards work-item show --id 1234 --output table

# Update a work item:
az boards work-item update \
  --id 1234 \
  --state "Active" \
  --assigned-to "john@contoso.com"

# Create a Task under a User Story:
az boards work-item create \
  --type "Task" \
  --title "Implement password reset email service" \
  --project ECommerceApp \
  --fields "System.Parent=1234"

# Link two work items:
az boards work-item relation add \
  --id 1234 \
  --relation-type "Child" \
  --target-id 1235

# Run a stored query:
az boards query --id <query-id> --output table

# Search work items:
az boards work-item list \
  --project ECommerceApp \
  --wiql "SELECT [System.Id],[System.Title] FROM WorkItems WHERE [System.WorkItemType]='Bug' AND [System.State]='Active'" \
  --output table
```

---

### Delivery Plans

Delivery Plans provide a calendar-style view showing work across multiple teams and their sprints.

```
Timeline: July 2024
─────────────────────────────────────────────────────────────────
Team              │ Week 1    │ Week 2    │ Week 3    │ Week 4
──────────────────┼───────────┼───────────┼───────────┼──────────
Backend Team      │ Sprint 10 │ Sprint 10 │ Sprint 11 │ Sprint 11
  - Payment API   │ ████████  │           │           │
  - Auth Refactor │           │ ████████  │           │
Frontend Team     │ Sprint 10 │ Sprint 10 │ Sprint 11 │ Sprint 11
  - Checkout UI   │ ████████  │           │           │
  - Dark Mode     │           │           │ ████████  │
Mobile Team       │ Sprint 9  │ Sprint 9  │ Sprint 10 │ Sprint 10
  - iOS Payments  │           │ ████████  │           │
─────────────────────────────────────────────────────────────────
```

Create via: **Boards → Delivery Plans → New Plan**

---

### Custom Fields and Workflows

Customize work item types via the **process template inheritance** model:

1. Go to **Organization Settings → Process**
2. Create an inherited process based on Agile/Scrum/CMMI
3. Add custom fields:
   - Text, Integer, DateTime, Boolean, Picklist
   - e.g., "Customer Name" (text), "Risk Level" (picklist: Low/Medium/High)
4. Add custom states:
   - e.g., Add "In Review" between "Active" and "Resolved"
5. Apply custom process to projects:
   - Project Settings → Project Details → Change process

**Custom workflow example for User Stories:**
```
New → Active → In Code Review → Testing → Done
         ↓
       Blocked (custom state with red color)
```

---

## 3. Azure Repos

Azure Repos provides unlimited, free private Git repositories (and TFVC for legacy teams).

### Git vs TFVC

| Aspect               | Git (recommended)                    | TFVC (Team Foundation Version Control)  |
|---------------------|--------------------------------------|----------------------------------------|
| **Model**            | Distributed — full copy locally      | Centralized — server is authoritative  |
| **Branching**        | Lightweight, cheap, fast             | Expensive, heavyweight                 |
| **Offline work**     | Full capability offline              | Limited offline capability             |
| **History**          | Each commit has full snapshot        | Changesets stored on server            |
| **Migration**        | Easy to migrate to GitHub            | Migration complex                      |
| **Recommended for**  | All new projects                     | Legacy teams migrating from TFS        |
| **Lock files**       | Not supported natively               | Supports exclusive checkouts           |

**Use Git.** TFVC is legacy. Only use TFVC if you're migrating an existing TFS system that requires it.

---

### Repository Management

```bash
# ── Repository CLI commands ───────────────────────────────────────────────────

# List all repos in a project:
az repos list --project ECommerceApp --output table

# Create a new repository:
az repos create \
  --name "payment-service" \
  --project ECommerceApp

# Clone a repo (standard Git):
git clone https://contoso@dev.azure.com/contoso/ECommerceApp/_git/payment-service
# With PAT embedded (for automation — use credential manager in dev):
git clone https://<PAT>@dev.azure.com/contoso/ECommerceApp/_git/payment-service

# Show repo details:
az repos show --repository payment-service --project ECommerceApp

# Delete a repo:
az repos delete --id <repo-id> --project ECommerceApp --yes

# List branches:
az repos ref list --repository payment-service --project ECommerceApp --output table

# Import a repo from GitHub/external:
az repos import create \
  --git-url https://github.com/myorg/myrepo \
  --repository payment-service \
  --project ECommerceApp
```

**Branch Policies (protect important branches):**

```bash
# Enable minimum reviewer count on main branch:
az repos policy approver-count create \
  --allow-downvotes false \
  --blocking true \
  --branch main \
  --creator-vote-counts false \
  --enabled true \
  --minimum-approver-count 2 \
  --repository-id <repo-id> \
  --reset-on-source-push true \
  --project ECommerceApp

# Require linked work items:
az repos policy work-item-linking create \
  --blocking true \
  --branch main \
  --enabled true \
  --repository-id <repo-id> \
  --project ECommerceApp

# Require a successful build before merging:
az repos policy build create \
  --blocking true \
  --branch main \
  --build-definition-id <pipeline-id> \
  --display-name "PR Build Validation" \
  --enabled true \
  --manual-queue-only false \
  --queue-on-source-update-only true \
  --repository-id <repo-id> \
  --valid-duration 720 \
  --project ECommerceApp

# Enforce comment resolution before merge:
az repos policy comment-required create \
  --blocking true \
  --branch main \
  --enabled true \
  --repository-id <repo-id> \
  --project ECommerceApp
```

---

### Branch Strategies

#### GitFlow

Best for: teams with scheduled releases, versioned software (libraries, mobile apps)

```
main ─────────────────●──────────────────────●──────────
                      ↑ release/1.0          ↑ release/2.0
develop ──●──●──●──●──●──────●──●──●──────●──●──────────
          ↑        ↑              ↑        ↑
     feature/A  feature/B    feature/C  feature/D

hotfix branches come off main:
main ──────────●──────────────────────
               └─ hotfix/1.0.1 ──→ merge to main AND develop
```

Branches:
- `main`: Production code only, always deployable
- `develop`: Integration branch, next release
- `feature/xxx`: New features, branch from develop
- `release/x.x`: Release stabilization, branch from develop
- `hotfix/xxx`: Emergency fixes, branch from main

```bash
# GitFlow example:
git checkout develop
git checkout -b feature/payment-integration
# ... work ...
git push origin feature/payment-integration
# Create PR: feature/payment-integration → develop

# Start a release:
git checkout develop
git checkout -b release/1.2
# bump version, fix release bugs
git checkout main && git merge --no-ff release/1.2 && git tag v1.2
git checkout develop && git merge --no-ff release/1.2
```

#### GitHub Flow

Best for: web apps with continuous deployment, small teams

```
main ──●────────────────●────────────────●──────────────
       └─ feature/A ──→─┘                └─ feature/B ─→
          (PR + deploy)                    (PR + deploy)
```

Rules:
1. `main` is always deployable
2. Branch from `main` for every change
3. Open a PR early (draft PR for visibility)
4. Deploy from the branch to staging before merging
5. Merge to `main` = deploys to production

```bash
# GitHub Flow:
git checkout main && git pull
git checkout -b feature/add-apple-pay
# ... work and commit ...
git push origin feature/add-apple-pay
# Open PR → review → CI passes → merge to main → auto-deploy
```

#### Trunk-Based Development

Best for: mature teams with strong CI/CD, high deployment frequency (10+ deploys/day)

```
main (trunk) ──●──●──●──●──●──●──●──●──●──●──
               ↑  ↑  ↑  ↑
               │  │  │  └─ short-lived feature branch (< 1 day)
               │  │  └──── direct commit by senior dev
               │  └─────── direct commit
               └────────── direct commit
```

Key practices:
- All developers commit to `main` (trunk) at least once per day
- Feature flags hide incomplete features in production
- Short-lived branches (< 1 day) if needed
- Very strong CI — every commit triggers full test suite

---

### Pull Request Policies

```bash
# ── Pull request CLI commands ─────────────────────────────────────────────────

# Create a PR:
az repos pr create \
  --repository payment-service \
  --source-branch feature/stripe-integration \
  --target-branch main \
  --title "Add Stripe payment integration" \
  --description "Implements Stripe checkout flow. Fixes #1234." \
  --reviewers "jane@contoso.com" "john@contoso.com" \
  --work-items 1234 1235 \
  --project ECommerceApp \
  --draft false \
  --auto-complete true \
  --squash true \
  --delete-source-branch true

# List PRs:
az repos pr list \
  --repository payment-service \
  --project ECommerceApp \
  --status active \
  --output table

# Show PR details:
az repos pr show --id 42 --output json

# Approve a PR:
az repos pr set-vote --id 42 --vote approve

# Complete (merge) a PR:
az repos pr update \
  --id 42 \
  --status completed \
  --merge-commit-message "Merge: Add Stripe payment integration (#42)"

# Abandon a PR:
az repos pr update --id 42 --status abandoned

# Add a reviewer:
az repos pr reviewer add --id 42 --reviewers "newreviewer@contoso.com"
```

**Best PR practices:**
- Keep PRs small (< 400 lines changed) — easier to review
- Use draft PRs for work in progress
- Write meaningful PR descriptions with "why", not just "what"
- Reference work items in PR description (`#1234` or `AB#1234`)
- Respond to all review comments before requesting re-review

---

### Code Search

Azure Repos includes full-text code search across all repositories:

```bash
# Search for code via CLI:
az repos code search \
  --search-text "StripePaymentService" \
  --project ECommerceApp \
  --output table

# Search with filters:
az repos code search \
  --search-text "connectionString ext:cs" \
  --project ECommerceApp \
  --output table
```

In the web UI: use the search box in the top bar, select "Code" from the dropdown. Supports:
- File name search: `fileName:appsettings.json`
- Extension filter: `ext:cs`
- Repository filter: `repo:payment-service`
- Path filter: `path:src/Controllers`
- Boolean operators: `AND`, `OR`, `NOT`

---

## 4. Azure Pipelines (YAML)

Azure Pipelines is the CI/CD engine. Pipelines are defined in YAML files stored in your repository.

### Pipeline Concepts

```
┌─────────────────── Pipeline ───────────────────────────────────┐
│                                                                 │
│  Triggers: what starts the pipeline                            │
│  (push to main, PR, schedule, manual, pipeline completion)     │
│                                                                 │
│  ┌─────────────── Stage: Build ────────────────────────────┐  │
│  │  ┌──────────── Job: Build_App ──────────────────────┐   │  │
│  │  │  Step 1: Checkout (implicit)                      │   │  │
│  │  │  Step 2: Task: DotNetCoreCLI@2 (restore)          │   │  │
│  │  │  Step 3: Task: DotNetCoreCLI@2 (build)            │   │  │
│  │  │  Step 4: Task: DotNetCoreCLI@2 (test)             │   │  │
│  │  │  Step 5: Task: PublishBuildArtifacts@1             │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────── Stage: Deploy_Staging ───────────────────┐  │
│  │  dependsOn: Build                                        │  │
│  │  ┌──────────── Deployment Job: Deploy ──────────────┐   │  │
│  │  │  Environment: staging                             │   │  │
│  │  │  Step 1: Download artifact                        │   │  │
│  │  │  Step 2: Deploy to App Service                    │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌─────────────── Stage: Deploy_Prod ──────────────────────┐  │
│  │  dependsOn: Deploy_Staging                               │  │
│  │  condition: succeeded()                                  │  │
│  │  [APPROVAL GATE: requires manual approval]               │  │
│  │  ┌──────────── Deployment Job: Deploy ──────────────┐   │  │
│  │  │  Environment: production                          │   │  │
│  │  └──────────────────────────────────────────────────┘   │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

**Key terms:**
- **Trigger**: Event that starts a pipeline run
- **Stage**: A logical group of jobs (e.g., Build, Test, Deploy). Stages run sequentially by default.
- **Job**: A set of steps that run on a single agent. Jobs within a stage can run in parallel.
- **Step**: A single action — either a **Task** (predefined action) or a **Script** (shell/PowerShell command)
- **Task**: A pre-packaged action from the marketplace (e.g., `DotNetCoreCLI@2`, `Docker@2`)
- **Agent**: The machine that runs a job (Microsoft-hosted or self-hosted)
- **Artifact**: Files produced by a pipeline and passed between stages

---

### YAML Pipeline Structure — Full Annotated Example

```yaml
# ── azure-pipelines.yml ── Full annotated example ────────────────────────────

# Name of the pipeline run (supports variables and counters):
name: $(Build.DefinitionName)-$(Build.SourceBranchName)-$(Rev:r)

# ── TRIGGERS ──────────────────────────────────────────────────────────────────
trigger:
  branches:
    include:
      - main
      - release/*
    exclude:
      - feature/experimental-*
  paths:
    include:
      - src/**
      - tests/**
    exclude:
      - docs/**
      - '**/*.md'
  tags:
    include:
      - v*.*.*

# PR trigger (builds on pull requests targeting these branches):
pr:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - src/**

# Scheduled trigger (nightly at 2 AM UTC on weekdays):
schedules:
  - cron: "0 2 * * 1-5"
    displayName: Nightly Build
    branches:
      include:
        - main
    always: true  # run even if no code changes

# ── GLOBAL VARIABLES ──────────────────────────────────────────────────────────
variables:
  # Hardcoded variable:
  buildConfiguration: 'Release'
  dotnetVersion: '8.x'
  imageRepository: 'ecommerceapp'
  containerRegistry: 'contosoacr.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'

  # Variable group (secrets and shared config stored in Azure DevOps library):
  - group: ecommerce-prod-secrets   # contains: AZURE_CLIENT_SECRET, DB_CONNECTION_STRING, etc.
  - group: ecommerce-shared-config  # contains: AZURE_SUBSCRIPTION_ID, AKS_CLUSTER_NAME, etc.

# ── AGENT POOL ────────────────────────────────────────────────────────────────
pool:
  vmImage: 'ubuntu-latest'  # Microsoft-hosted agent
  # For self-hosted, use:
  # name: 'MyAgentPool'
  # demands:
  #   - dotnet
  #   - docker

# ── STAGES ────────────────────────────────────────────────────────────────────
stages:

# ─────── STAGE 1: Build and Test ─────────────────────────────────────────────
- stage: Build
  displayName: 'Build and Test'
  jobs:

  - job: BuildAndTest
    displayName: 'Build, Test, and Package'
    timeoutInMinutes: 60  # fail if job takes more than 60 min
    variables:
      # Job-scoped variable (only available in this job):
      testResultsDir: '$(Agent.TempDirectory)/TestResults'

    steps:

    # Checkout source code (implicit, but shown for clarity):
    - checkout: self
      fetchDepth: 0        # full history (needed for GitVersion, SonarQube)
      lfs: false
      submodules: recursive

    # Use specific .NET version:
    - task: UseDotNet@2
      displayName: 'Use .NET $(dotnetVersion)'
      inputs:
        version: $(dotnetVersion)
        includePreviewVersions: false

    # Restore NuGet packages:
    - task: DotNetCoreCLI@2
      displayName: 'Restore NuGet packages'
      inputs:
        command: restore
        projects: '**/*.csproj'
        feedsToUse: 'config'
        nugetConfigPath: 'nuget.config'

    # Build the solution:
    - task: DotNetCoreCLI@2
      displayName: 'Build solution'
      inputs:
        command: build
        projects: '**/*.csproj'
        arguments: '--configuration $(buildConfiguration) --no-restore'

    # Run unit tests with code coverage:
    - task: DotNetCoreCLI@2
      displayName: 'Run unit tests'
      inputs:
        command: test
        projects: '**/*Tests.csproj'
        arguments: >
          --configuration $(buildConfiguration)
          --no-build
          --collect:"XPlat Code Coverage"
          --results-directory $(testResultsDir)
          --logger "trx;LogFileName=results.trx"
          -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura
      continueOnError: false

    # Publish test results to pipeline:
    - task: PublishTestResults@2
      displayName: 'Publish test results'
      inputs:
        testResultsFormat: 'VSTest'
        testResultsFiles: '$(testResultsDir)/**/*.trx'
        mergeTestResults: true
        failTaskOnFailedTests: true
      condition: succeededOrFailed()

    # Publish code coverage report:
    - task: PublishCodeCoverageResults@1
      displayName: 'Publish code coverage'
      inputs:
        codeCoverageTool: 'Cobertura'
        summaryFileLocation: '$(testResultsDir)/**/coverage.cobertura.xml'
      condition: succeededOrFailed()

    # Publish the app:
    - task: DotNetCoreCLI@2
      displayName: 'Publish app'
      inputs:
        command: publish
        publishWebProjects: true
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)/app'
        zipAfterPublish: true

    # Publish build artifact (passes files to next stages):
    - task: PublishPipelineArtifact@1
      displayName: 'Publish pipeline artifact'
      inputs:
        targetPath: '$(Build.ArtifactStagingDirectory)/app'
        artifact: 'drop'
        publishLocation: 'pipeline'

# ─────── STAGE 2: Docker Build and Push ─────────────────────────────────────
- stage: Containerize
  displayName: 'Build and Push Docker Image'
  dependsOn: Build
  condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'))
  variables:
    imageTag: $(Build.BuildId)

  jobs:
  - job: Docker
    displayName: 'Build and push to ACR'
    steps:

    # Download artifact from Build stage:
    - task: DownloadPipelineArtifact@2
      displayName: 'Download build artifact'
      inputs:
        artifact: 'drop'
        path: '$(Build.SourcesDirectory)/publish'

    # Login to Azure Container Registry and build+push image:
    - task: Docker@2
      displayName: 'Build and push image to ACR'
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: 'ACR-ServiceConnection'  # service connection name
        tags: |
          $(imageTag)
          latest
        arguments: '--build-arg BUILD_NUMBER=$(Build.BuildId)'

    # Scan image for vulnerabilities with Trivy:
    - script: |
        docker run --rm \
          -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy:latest image \
          --exit-code 1 \
          --severity HIGH,CRITICAL \
          $(containerRegistry)/$(imageRepository):$(imageTag)
      displayName: 'Security scan with Trivy'
      continueOnError: true  # don't block the pipeline, just warn

    # Publish image tag as variable for downstream stages:
    - script: |
        echo "##vso[task.setvariable variable=imageTag;isOutput=true]$(imageTag)"
      displayName: 'Set output variable: imageTag'
      name: setImageTag

# ─────── STAGE 3: Deploy to Staging ─────────────────────────────────────────
- stage: Deploy_Staging
  displayName: 'Deploy to Staging'
  dependsOn: Containerize
  condition: succeeded()
  variables:
    # Consume output variable from previous stage:
    imageTag: $[ stageDependencies.Containerize.Docker.outputs['setImageTag.imageTag'] ]

  jobs:
  # Deployment job (not regular job) — has environment and strategy:
  - deployment: DeployToStaging
    displayName: 'Deploy to AKS Staging'
    environment: 'staging'       # environment with approval checks
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      runOnce:
        deploy:
          steps:

          - task: KubernetesManifest@0
            displayName: 'Deploy to AKS'
            inputs:
              action: deploy
              kubernetesServiceConnection: 'AKS-Staging-ServiceConnection'
              namespace: 'staging'
              manifests: |
                $(Pipeline.Workspace)/k8s/deployment.yaml
                $(Pipeline.Workspace)/k8s/service.yaml
              containers: |
                $(containerRegistry)/$(imageRepository):$(imageTag)

          # Run smoke tests against staging:
          - script: |
              curl -f https://staging.ecommerce.contoso.com/health || exit 1
              echo "Smoke test passed!"
            displayName: 'Run smoke tests'

# ─────── STAGE 4: Deploy to Production ──────────────────────────────────────
- stage: Deploy_Production
  displayName: 'Deploy to Production'
  dependsOn: Deploy_Staging
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  variables:
    imageTag: $[ stageDependencies.Containerize.Docker.outputs['setImageTag.imageTag'] ]

  jobs:
  - deployment: DeployToProduction
    displayName: 'Deploy to AKS Production'
    environment: 'production'    # production environment has manual approval gate
    pool:
      vmImage: 'ubuntu-latest'
    strategy:
      # Blue-Green deployment strategy:
      runOnce:
        preDeploy:
          steps:
          - script: echo "Pre-deploy checks: verifying prod readiness..."
            displayName: 'Pre-deploy checks'
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy to AKS Production'
            inputs:
              action: deploy
              kubernetesServiceConnection: 'AKS-Production-ServiceConnection'
              namespace: 'production'
              manifests: |
                $(Pipeline.Workspace)/k8s/deployment.yaml
                $(Pipeline.Workspace)/k8s/service.yaml
              containers: |
                $(containerRegistry)/$(imageRepository):$(imageTag)
        routeTraffic:
          steps:
          - script: echo "Traffic routing complete"
        postRouteTraffic:
          steps:
          - script: |
              # Verify production health after routing traffic
              curl -f https://www.ecommerce.contoso.com/health || exit 1
            displayName: 'Production health check'
        on:
          failure:
            steps:
            - script: echo "Deployment failed — initiating rollback..."
              displayName: 'Rollback on failure'
            - task: KubernetesManifest@0
              inputs:
                action: deploy
                kubernetesServiceConnection: 'AKS-Production-ServiceConnection'
                namespace: 'production'
                manifests: $(Pipeline.Workspace)/k8s/rollback.yaml
          success:
            steps:
            - script: echo "Production deployment successful!"
              displayName: 'Notify success'
```

---

### Variables: Hardcoded, Pipeline Variables, Variable Groups, Secrets

**1. Hardcoded variables** (in the YAML file — fine for non-sensitive values):
```yaml
variables:
  buildConfiguration: 'Release'
  dotnetVersion: '8.x'
```

**2. Pipeline variables** (defined in Azure DevOps UI → Edit Pipeline → Variables):
- Settable at queue time (users can override when manually triggering)
- Can be marked as secret (encrypted, never shown in logs)
```yaml
variables:
  myVar: $(MyPipelineVariable)  # references variable set in UI
```

**3. Variable Groups** (Azure DevOps Library → Variable Groups):
```yaml
variables:
  - group: my-variable-group    # links entire group
  - name: localVar
    value: 'something'
```

```bash
# Create a variable group via CLI:
az pipelines variable-group create \
  --name "ecommerce-prod-secrets" \
  --variables \
    DB_SERVER="prod-db.database.windows.net" \
    APP_INSIGHTS_KEY="$(AppInsightsKey)" \
  --authorize true \
  --project ECommerceApp

# Add a secret variable to a group:
az pipelines variable-group variable create \
  --group-id <group-id> \
  --name "STRIPE_SECRET_KEY" \
  --value "sk_live_xxxxx" \
  --secret true \
  --project ECommerceApp

# Link variable group to Azure Key Vault (so secrets are pulled from Key Vault):
# In Azure DevOps UI: Library → Variable Groups → "Link secrets from Azure key vault"
```

**4. Secrets** (never echoed in logs):
```yaml
variables:
  mySecret: $(SECRET_FROM_LIBRARY)  # secret variable — masked in all logs

steps:
- script: |
    # This will print "***" in logs, not the actual value:
    echo "My secret is $(mySecret)"
    # To use in environment variables:
    export MY_SECRET=$(mySecret)
    ./deploy.sh
  env:
    # Explicitly map secret to environment variable:
    STRIPE_KEY: $(STRIPE_SECRET_KEY)
```

**5. Dynamic variables** (set during a pipeline run):
```yaml
steps:
- script: |
    # Set a pipeline variable from a script:
    echo "##vso[task.setvariable variable=myDynamicVar]hello"
    
    # Set an output variable (accessible in downstream jobs):
    echo "##vso[task.setvariable variable=imageTag;isOutput=true]$(Build.BuildId)"
  name: setVars

- script: |
    # Use the dynamic variable in a subsequent step:
    echo "Dynamic var: $(myDynamicVar)"
```

---

### Environments and Deployment Jobs

Environments are named deployment targets with history, approvals, and checks.

```bash
# Create environments via CLI:
az pipelines environment create \
  --name staging \
  --project ECommerceApp

az pipelines environment create \
  --name production \
  --project ECommerceApp

# List environments:
az pipelines environment list --project ECommerceApp --output table
```

**Adding Approval Checks (in Azure DevOps UI):**
1. Pipelines → Environments → `production`
2. Click `...` (More actions) → Approvals and Checks
3. Add → Approvals
4. Specify approvers: `jane@contoso.com`, `john@contoso.com`
5. Set timeout: 3 days
6. Instructions: "Review the staging deployment before approving production"

**Other available checks:**
- **Business hours check**: Only deploy between 09:00-17:00 Mon-Fri
- **Invoke REST API**: Call an external webhook to validate (e.g., change management system)
- **Query Work Items**: Ensure no P0 bugs are open
- **Required template**: Force use of approved pipeline templates
- **Exclusive lock**: Only one deployment to this environment at a time

---

### Templates (Step, Job, Stage Templates)

Templates allow you to reuse pipeline components and enforce governance.

**Step template** (`templates/build-steps.yml`):
```yaml
# templates/build-steps.yml
parameters:
  - name: dotnetVersion
    type: string
    default: '8.x'
  - name: buildConfiguration
    type: string
    default: 'Release'
  - name: publishPath
    type: string
    default: '$(Build.ArtifactStagingDirectory)'

steps:
- task: UseDotNet@2
  displayName: 'Use .NET ${{ parameters.dotnetVersion }}'
  inputs:
    version: ${{ parameters.dotnetVersion }}

- task: DotNetCoreCLI@2
  displayName: 'Restore'
  inputs:
    command: restore
    projects: '**/*.csproj'

- task: DotNetCoreCLI@2
  displayName: 'Build'
  inputs:
    command: build
    projects: '**/*.csproj'
    arguments: '--configuration ${{ parameters.buildConfiguration }} --no-restore'

- task: DotNetCoreCLI@2
  displayName: 'Test'
  inputs:
    command: test
    projects: '**/*Tests.csproj'
    arguments: '--configuration ${{ parameters.buildConfiguration }} --no-build'
```

**Job template** (`templates/deploy-to-aks-job.yml`):
```yaml
# templates/deploy-to-aks-job.yml
parameters:
  - name: environment
    type: string
  - name: serviceConnection
    type: string
  - name: namespace
    type: string
  - name: imageTag
    type: string

jobs:
- deployment: Deploy_${{ parameters.environment }}
  displayName: 'Deploy to ${{ parameters.environment }}'
  environment: ${{ parameters.environment }}
  pool:
    vmImage: 'ubuntu-latest'
  strategy:
    runOnce:
      deploy:
        steps:
        - task: KubernetesManifest@0
          inputs:
            action: deploy
            kubernetesServiceConnection: ${{ parameters.serviceConnection }}
            namespace: ${{ parameters.namespace }}
            manifests: $(Pipeline.Workspace)/k8s/*.yaml
            containers: $(containerRegistry)/$(imageRepository):${{ parameters.imageTag }}
```

**Stage template** (`templates/deploy-stage.yml`):
```yaml
# templates/deploy-stage.yml
parameters:
  - name: stageName
    type: string
  - name: environment
    type: string
  - name: dependsOn
    type: string
    default: ''
  - name: imageTag
    type: string

stages:
- stage: ${{ parameters.stageName }}
  displayName: 'Deploy to ${{ parameters.environment }}'
  ${{ if ne(parameters.dependsOn, '') }}:
    dependsOn: ${{ parameters.dependsOn }}
  jobs:
  - template: deploy-to-aks-job.yml
    parameters:
      environment: ${{ parameters.environment }}
      serviceConnection: 'AKS-${{ parameters.environment }}-SC'
      namespace: ${{ lower(parameters.environment) }}
      imageTag: ${{ parameters.imageTag }}
```

**Using templates in main pipeline:**
```yaml
# azure-pipelines.yml — using templates

trigger:
  - main

variables:
  - group: shared-config

stages:
- stage: Build
  jobs:
  - job: Build
    steps:
    # Include step template:
    - template: templates/build-steps.yml
      parameters:
        dotnetVersion: '8.x'
        buildConfiguration: 'Release'

- template: templates/deploy-stage.yml   # Include stage template
  parameters:
    stageName: Deploy_Staging
    environment: staging
    dependsOn: Build
    imageTag: $(Build.BuildId)

- template: templates/deploy-stage.yml
  parameters:
    stageName: Deploy_Production
    environment: production
    dependsOn: Deploy_Staging
    imageTag: $(Build.BuildId)
```

**Extending a required template** (governance/enforcement):
```yaml
# Central template: templates/required-steps-template.yml
# Mandatory steps every pipeline must include (e.g., security scan):
parameters:
  - name: stages
    type: stageList

stages:
- stage: SecurityScan
  displayName: 'Security Scan (Required)'
  jobs:
  - job: SAST
    steps:
    - script: echo "Running mandatory security scan..."
    - task: CredScan@3  # Credential scanner
- ${{ parameters.stages }}
```

```yaml
# Pipeline that extends the required template:
extends:
  template: templates/required-steps-template.yml
  parameters:
    stages:
    - stage: Build
      jobs:
      - job: BuildApp
        steps:
        - script: dotnet build
```

---

### Self-Hosted Agents vs Microsoft-Hosted Agents

| Aspect                   | Microsoft-Hosted Agents              | Self-Hosted Agents                    |
|--------------------------|--------------------------------------|---------------------------------------|
| **Setup**                | Zero — use immediately               | Install agent on your VM/container    |
| **Cost**                 | 1800 min/month free, then $40/parallel job | Free for unlimited parallelism   |
| **OS options**           | ubuntu-latest, windows-latest, macOS-latest | Any OS you can install the agent on |
| **Pre-installed tools**  | Extensive (see Microsoft docs)       | Only what you install                 |
| **Performance**          | Standard (shared compute)            | Can use powerful dedicated hardware   |
| **Network access**       | Public internet only                 | Can access private networks           |
| **Persistent disk**      | No (fresh VM per job)                | Yes (can cache Docker layers, etc.)  |
| **Security isolation**   | High (new VM per job)                | You manage security                   |
| **Custom tools**         | No (use script to install each time) | Yes — pre-install and cache           |
| **Best for**             | Public repos, simple builds, standard builds | Private networks, large builds, cost control |

**Microsoft-hosted agent images:**
- `ubuntu-latest` → Ubuntu 22.04 (fastest, most tools)
- `ubuntu-22.04` → Ubuntu 22.04 (pinned version)
- `ubuntu-20.04` → Ubuntu 20.04
- `windows-latest` → Windows Server 2022
- `windows-2022` → Windows Server 2022
- `macos-latest` → macOS 13 (Ventura)
- `macos-14` → macOS 14 (Sonoma, Apple Silicon)

**Installing a self-hosted agent:**
```bash
# ── Install Azure Pipelines agent on Linux ────────────────────────────────────

# Create a dedicated user for the agent:
sudo useradd -m azagent
sudo su - azagent

# Download the agent:
mkdir myagent && cd myagent
curl -O https://vstsagentpackage.blob.core.windows.net/agent/3.227.2/vsts-agent-linux-x64-3.227.2.tar.gz
tar zxvf vsts-agent-linux-x64-3.227.2.tar.gz

# Configure the agent:
./config.sh \
  --url https://dev.azure.com/contoso \
  --auth pat \
  --token <your-PAT-with-agent-pools-readwrite> \
  --pool "MyAgentPool" \
  --agent "MyAgent-$(hostname)" \
  --replace \
  --acceptTeeEula

# Run as a service (Linux systemd):
sudo ./svc.sh install azagent
sudo ./svc.sh start
sudo ./svc.sh status

# ── Self-hosted agent in a Docker container ───────────────────────────────────
# Create Dockerfile:
cat > Dockerfile.agent << 'EOF'
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y \
    curl git jq libicu70 libssl3 \
    dotnet-sdk-8.0 docker.io kubectl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /azp
COPY start.sh .
RUN chmod +x start.sh

ENV AZP_URL=""
ENV AZP_TOKEN=""
ENV AZP_POOL="Default"
ENV AZP_AGENT_NAME=""

ENTRYPOINT ["./start.sh"]
EOF
```

**Agent pools:**
```bash
# Create an agent pool:
az pipelines pool create \
  --name "LinuxBuildPool" \
  --authorize true \
  --pool-type private \
  --project ECommerceApp

# List pools:
az pipelines pool list --output table

# List agents in a pool:
az pipelines agent list \
  --pool-id <pool-id> \
  --output table
```

---

### Service Connections

Service connections store credentials to external services (Azure, Docker Hub, GitHub, etc.).

```bash
# ── Create service connections via CLI ────────────────────────────────────────

# Azure Resource Manager (recommended: workload identity federation):
az devops service-endpoint azurerm create \
  --azure-rm-service-principal-id <app-id> \
  --azure-rm-subscription-id <subscription-id> \
  --azure-rm-subscription-name "My Subscription" \
  --azure-rm-tenant-id <tenant-id> \
  --name "Azure-Production-SC" \
  --project ECommerceApp

# Docker Hub:
az devops service-endpoint create \
  --service-endpoint-configuration endpoint.json \
  --project ECommerceApp
# (endpoint.json defines Docker Hub credentials)

# GitHub:
az devops service-endpoint github create \
  --github-url https://github.com \
  --name "GitHub-SC" \
  --project ECommerceApp

# List service connections:
az devops service-endpoint list \
  --project ECommerceApp \
  --output table

# Authorize service connection for all pipelines:
az devops service-endpoint update \
  --id <endpoint-id> \
  --enable-for-all true \
  --project ECommerceApp
```

**Service connection types:**
| Type | Use case |
|------|----------|
| Azure Resource Manager | Deploy to Azure (VMs, App Service, AKS, etc.) |
| Docker Registry | Push/pull Docker images (ACR, Docker Hub) |
| Kubernetes | Deploy to Kubernetes clusters |
| GitHub | Trigger builds from GitHub, checkout code |
| npm | Publish/restore npm packages |
| NuGet | Publish/restore NuGet packages |
| Maven | Use Maven artifacts |
| SSH | Connect to Linux servers via SSH |
| Generic | Custom REST API connections |

---

### Build Artifacts and Pipeline Artifacts

**Build Artifacts** (older, stored in Azure DevOps): Use `PublishBuildArtifacts@1` / `DownloadBuildArtifacts@0`

**Pipeline Artifacts** (newer, faster, stored in Azure Pipelines): Use `PublishPipelineArtifact@1` / `DownloadPipelineArtifact@2`

```yaml
# ── Publish a pipeline artifact ───────────────────────────────────────────────
- task: PublishPipelineArtifact@1
  displayName: 'Publish drop artifact'
  inputs:
    targetPath: '$(Build.ArtifactStagingDirectory)'
    artifact: 'drop'           # artifact name
    publishLocation: 'pipeline' # 'pipeline' or 'filepath' (file share)

# ── Download a pipeline artifact ─────────────────────────────────────────────
- task: DownloadPipelineArtifact@2
  displayName: 'Download drop artifact'
  inputs:
    artifact: 'drop'
    path: '$(Pipeline.Workspace)/drop'

# Download from a specific pipeline run:
- task: DownloadPipelineArtifact@2
  inputs:
    source: 'specific'          # 'current' or 'specific'
    project: 'ECommerceApp'
    pipeline: <pipeline-id>
    runVersion: 'latest'        # 'latest', 'latestFromBranch', 'specific'
    runBranch: 'refs/heads/main'
    artifact: 'drop'
    path: '$(Pipeline.Workspace)/drop'
```

---

### Caching in Pipelines

Caching speeds up pipelines by reusing dependencies between runs.

```yaml
# ── Cache NuGet packages ──────────────────────────────────────────────────────
variables:
  NUGET_PACKAGES: $(Pipeline.Workspace)/.nuget/packages
  CACHE_KEY_NUGET: nuget | "$(Agent.OS)" | **/packages.lock.json

steps:
- task: Cache@2
  displayName: 'Cache NuGet packages'
  inputs:
    key: $(CACHE_KEY_NUGET)
    restoreKeys: |
      nuget | "$(Agent.OS)"
      nuget
    path: $(NUGET_PACKAGES)
    cacheHitVar: CACHE_RESTORED_NUGET

- task: DotNetCoreCLI@2
  displayName: 'Restore (skips if cache hit)'
  inputs:
    command: restore
    projects: '**/*.csproj'
  condition: ne(variables.CACHE_RESTORED_NUGET, 'true')

# ── Cache npm packages ────────────────────────────────────────────────────────
- task: Cache@2
  displayName: 'Cache npm packages'
  inputs:
    key: 'npm | "$(Agent.OS)" | package-lock.json'
    restoreKeys: |
      npm | "$(Agent.OS)"
    path: $(npm_config_cache)

# ── Cache Docker layers ───────────────────────────────────────────────────────
# Use BuildKit with inline cache:
- script: |
    docker buildx build \
      --cache-from type=registry,ref=$(containerRegistry)/$(imageRepository):cache \
      --cache-to type=registry,ref=$(containerRegistry)/$(imageRepository):cache,mode=max \
      --push \
      -t $(containerRegistry)/$(imageRepository):$(Build.BuildId) \
      .
  displayName: 'Docker build with layer cache'
```

---

### Matrix Strategy

Run the same job configuration multiple times with different parameters (useful for cross-platform or multi-version testing):

```yaml
jobs:
- job: Test
  displayName: 'Test on multiple platforms'
  strategy:
    matrix:
      # Each entry is a separate job instance:
      Linux_DotNet8:
        imageName: 'ubuntu-latest'
        dotnetVersion: '8.x'
      Linux_DotNet6:
        imageName: 'ubuntu-latest'
        dotnetVersion: '6.x'
      Windows_DotNet8:
        imageName: 'windows-latest'
        dotnetVersion: '8.x'
      macOS_DotNet8:
        imageName: 'macos-latest'
        dotnetVersion: '8.x'
    maxParallel: 4  # run all 4 in parallel
  pool:
    vmImage: $(imageName)
  steps:
  - task: UseDotNet@2
    inputs:
      version: $(dotnetVersion)
  - task: DotNetCoreCLI@2
    inputs:
      command: test
      projects: '**/*Tests.csproj'
```

---

### Conditional Expressions

```yaml
# ── Conditions on steps ───────────────────────────────────────────────────────
steps:
# Always run (even if previous steps failed):
- script: echo "Always runs"
  condition: always()

# Only run if all previous steps succeeded (default):
- script: echo "Runs on success"
  condition: succeeded()

# Only run if a previous step failed:
- script: echo "Runs on failure"
  condition: failed()

# Run if succeeded OR failed (but NOT if cancelled):
- script: echo "Cleanup"
  condition: succeededOrFailed()

# Conditional based on a variable:
- script: echo "Deploying to production"
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

# Conditional based on trigger type:
- script: echo "This is a PR build"
  condition: eq(variables['Build.Reason'], 'PullRequest')

# NOT a PR build AND branch is main:
- script: echo "Release build"
  condition: and(succeeded(), ne(variables['Build.Reason'], 'PullRequest'), eq(variables['Build.SourceBranch'], 'refs/heads/main'))

# ── If/else using template expressions (${{ }}) ───────────────────────────────
# Template expressions are evaluated at compile time (before the pipeline runs):
variables:
  ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}:
    deployEnvironment: production
  ${{ else }}:
    deployEnvironment: staging

steps:
- script: echo "Deploying to $(deployEnvironment)"

# ── Conditional task inputs ───────────────────────────────────────────────────
- task: AzureWebApp@1
  inputs:
    azureSubscription: 'Azure-SC'
    appName: ${{ if eq(variables['Build.SourceBranch'], 'refs/heads/main') }}ecommerce-prod${{ else }}ecommerce-staging${{ end }}
```

---

### Rolling, Blue-Green, Canary Deployments

#### Rolling Deployment
Gradually replaces old instances with new ones:

```yaml
jobs:
- deployment: RollingDeploy
  environment:
    name: production
    resourceType: VirtualMachine   # for VM-based rolling deployments
  strategy:
    rolling:
      maxParallel: 2   # deploy to 2 servers at a time
      preDeploy:
        steps:
        - script: echo "Pre-deploy checks on $(Agent.MachineName)"
      deploy:
        steps:
        - script: |
            sudo systemctl stop ecommerceapp
            sudo cp $(Pipeline.Workspace)/drop/ecommerceapp.zip /opt/app/
            sudo unzip -o /opt/app/ecommerceapp.zip -d /opt/app/
            sudo systemctl start ecommerceapp
          displayName: 'Deploy and restart app'
      routeTraffic:
        steps:
        - script: echo "Traffic routing (handled by load balancer)"
      postRouteTraffic:
        steps:
        - script: curl -f http://$(Agent.MachineName)/health
          displayName: 'Health check'
      on:
        failure:
          steps:
          - script: sudo systemctl stop ecommerceapp && sudo systemctl start ecommerceapp-previous
            displayName: 'Rollback'
```

#### Blue-Green Deployment
Two identical environments; switch traffic between them:

```yaml
# In azure-pipelines.yml:
stages:
- stage: Deploy_Green
  jobs:
  - deployment: DeployGreen
    environment: production-green
    strategy:
      runOnce:
        deploy:
          steps:
          # Deploy to the "green" (inactive) slot in App Service:
          - task: AzureWebApp@1
            inputs:
              azureSubscription: 'Azure-SC'
              appName: 'ecommerce-prod'
              deployToSlotOrASE: true
              resourceGroupName: 'ecommerce-rg'
              slotName: 'green'
              package: '$(Pipeline.Workspace)/drop/*.zip'

          - script: |
              # Test the green slot before swapping:
              curl -f https://ecommerce-prod-green.azurewebsites.net/health
            displayName: 'Validate green slot'

- stage: Swap_Slots
  dependsOn: Deploy_Green
  jobs:
  - deployment: SwapToGreen
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          # Swap green (staging) slot to production:
          - task: AzureAppServiceManage@0
            inputs:
              azureSubscription: 'Azure-SC'
              action: 'Swap Slots'
              webAppName: 'ecommerce-prod'
              resourceGroupName: 'ecommerce-rg'
              sourceSlot: 'green'
              swapWithProduction: true
```

#### Canary Deployment
Route small percentage of traffic to new version, gradually increase:

```yaml
jobs:
- deployment: CanaryDeploy
  environment: production
  strategy:
    canary:
      increments: [10, 25, 50, 100]  # % of traffic to new version
      preDeploy:
        steps:
        - script: echo "Preparing canary deployment at $(Strategy.CycleNumber)% traffic"
      deploy:
        steps:
        - task: KubernetesManifest@0
          inputs:
            action: deploy
            kubernetesServiceConnection: 'AKS-Production-SC'
            namespace: production
            strategy: canary
            percentage: $(Strategy.CycleNumber)
            manifests: k8s/deployment.yaml
            containers: $(containerRegistry)/$(imageRepository):$(Build.BuildId)
      routeTraffic:
        steps:
        - script: echo "$(Strategy.CycleNumber)% traffic now routed to canary"
      postRouteTraffic:
        steps:
        - script: |
            # Monitor error rate for 5 minutes:
            sleep 300
            ERROR_RATE=$(curl -s https://monitoring.contoso.com/api/error-rate)
            if [ "$ERROR_RATE" -gt "1" ]; then
              echo "Error rate too high: $ERROR_RATE%. Failing canary."
              exit 1
            fi
            echo "Error rate OK: $ERROR_RATE%"
          displayName: 'Monitor canary health'
      on:
        failure:
          steps:
          - task: KubernetesManifest@0
            inputs:
              action: reject
              strategy: canary
              manifests: k8s/deployment.yaml
```

---

### Azure CLI Commands to Manage Pipelines

```bash
# ── Pipeline management CLI commands ──────────────────────────────────────────

# Create a pipeline (from existing YAML file in repo):
az pipelines create \
  --name "ECommerce-CI-CD" \
  --repository payment-service \
  --repository-type tfsgit \
  --branch main \
  --yaml-path azure-pipelines.yml \
  --project ECommerceApp \
  --skip-first-run false

# List pipelines:
az pipelines list --project ECommerceApp --output table

# Show pipeline details:
az pipelines show --name "ECommerce-CI-CD" --project ECommerceApp

# Run (queue) a pipeline:
az pipelines run \
  --name "ECommerce-CI-CD" \
  --branch main \
  --project ECommerceApp \
  --variables buildConfiguration=Release imageTag=manual-test \
  --parameters imageTag=manual-test

# List pipeline runs:
az pipelines runs list \
  --pipeline-name "ECommerce-CI-CD" \
  --project ECommerceApp \
  --status all \
  --top 20 \
  --output table

# Show a specific run:
az pipelines runs show \
  --id <run-id> \
  --project ECommerceApp

# Show pipeline run logs (specific artifact/test results):
az pipelines runs artifact list \
  --run-id <run-id> \
  --project ECommerceApp \
  --output table

# Cancel a running pipeline:
az pipelines runs cancel \
  --id <run-id> \
  --project ECommerceApp

# Delete a pipeline:
az pipelines delete \
  --id <pipeline-id> \
  --project ECommerceApp \
  --yes

# Update pipeline (e.g., change default branch):
az pipelines update \
  --name "ECommerce-CI-CD" \
  --branch develop \
  --project ECommerceApp

# Add/update pipeline variables:
az pipelines variable create \
  --name "DEPLOYMENT_ENV" \
  --value "production" \
  --allow-override true \
  --pipeline-name "ECommerce-CI-CD" \
  --project ECommerceApp

az pipelines variable update \
  --name "DEPLOYMENT_ENV" \
  --value "staging" \
  --pipeline-name "ECommerce-CI-CD" \
  --project ECommerceApp
```

---

## 5. Azure Test Plans

Azure Test Plans provides manual and automated test management.

### Manual Test Plans and Test Cases

**Structure:**
```
Test Plan: "Sprint 10 Regression Testing"
├── Test Suite: "Authentication"
│   ├── Test Case #1: "Valid login with correct credentials"
│   ├── Test Case #2: "Invalid login with wrong password shows error"
│   └── Test Case #3: "Login with MFA"
├── Test Suite: "Payment"
│   ├── Test Case #4: "Visa credit card payment succeeds"
│   ├── Test Case #5: "Expired card payment fails with message"
│   └── Test Case #6: "Refund processes correctly"
└── Test Suite: "Checkout"
    └── ...
```

**Creating a test case:**
- Title: "Valid login with correct credentials"
- Steps:
  1. Navigate to https://app.contoso.com/login  → *Expected: Login page displays*
  2. Enter username: test@contoso.com           → *Expected: Username field populates*
  3. Enter password: ValidPass123!              → *Expected: Password field shows dots*
  4. Click "Sign In"                            → *Expected: Redirected to dashboard*
- Priority: 1 (High)
- Area: ECommerceApp\Authentication
- Automated: Link to automated test method

### Test Runs and Results

```bash
# Test plan CLI commands:
az devops test plan list --project ECommerceApp --output table

az devops test plan show \
  --plan-id <plan-id> \
  --project ECommerceApp

az devops test run list \
  --plan-id <plan-id> \
  --project ECommerceApp \
  --output table
```

**Test result states:**
- **Active**: Test not yet run
- **Passed**: Test passed on last run
- **Failed**: Test failed on last run
- **Blocked**: Cannot run (e.g., environment issue)
- **Not Applicable**: Test not relevant to this build

### Exploratory Testing

Exploratory testing (ET) is unscripted testing: testers explore the application looking for defects without predefined test cases.

**Azure Test & Feedback browser extension:**
- Install from Chrome/Edge Web Store: "Test & Feedback (Azure DevOps)"
- Connect to your Azure DevOps organization
- Capture screenshots, screen recordings, notes while testing
- Automatically creates work items (bug, task) with rich context
- Attaches screenshots and repro steps automatically

**Best practices for exploratory testing:**
- Time-box sessions (45-90 minutes)
- Define a charter: "Explore the checkout flow as a new user"
- Focus on boundary conditions and error paths
- Pair exploratory testing with new features (before sprint review)

### Integration with CI/CD

```yaml
# Publish test results from automated pipeline to Test Plans:
- task: DotNetCoreCLI@2
  inputs:
    command: test
    projects: '**/*Tests.csproj'
    arguments: '--logger "trx;LogFileName=results.trx"'
    publishTestResults: true  # automatically publishes TRX files

# Or explicitly publish:
- task: PublishTestResults@2
  inputs:
    testResultsFormat: 'VSTest'
    testResultsFiles: '**/*.trx'
    testRunTitle: 'Sprint 10 Automated Tests'
    mergeTestResults: true
    failTaskOnFailedTests: true
    buildConfiguration: Release
```

**Connecting automated tests to Test Cases:**
In Visual Studio or your test framework, associate a test method with a Test Plan Test Case:
```csharp
[TestClass]
public class LoginTests
{
    [TestMethod]
    [TestCategory("Regression")]
    // Associate with Azure DevOps Test Case ID:
    [Microsoft.VisualStudio.TestTools.UnitTesting.Description("Test Case #1")]
    public void ValidLogin_ShouldRedirectToDashboard()
    {
        // test code...
    }
}
```

---

## 6. Azure Artifacts

Azure Artifacts is a package management service for hosting and sharing packages.

### What is Azure Artifacts?

Azure Artifacts lets you:
- Host your own private npm, NuGet, Maven, Python, and Universal packages
- Use as a proxy/cache for public registries (npmjs.com, nuget.org, Maven Central, PyPI)
- Share packages between teams and projects
- Version and manage package lifecycles

**Feeds** are containers for packages. Two scopes:
- **Project-scoped feed**: Only accessible within one project. Simpler permissions.
- **Organization-scoped feed**: Accessible from all projects in the organization.

### Package Types

```bash
# ── Azure Artifacts CLI commands ──────────────────────────────────────────────

# Create a feed:
az artifacts feed create \
  --name "ECommerce-Packages" \
  --project ECommerceApp \
  --public false

# List feeds:
az artifacts feed list --project ECommerceApp --output table

# Show feed details:
az artifacts feed show \
  --name "ECommerce-Packages" \
  --project ECommerceApp
```

**NuGet:**
```bash
# Authenticate to Azure Artifacts NuGet feed:
# Add to nuget.config:
cat > nuget.config << 'EOF'
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="ECommerce-Packages" value="https://pkgs.dev.azure.com/contoso/ECommerceApp/_packaging/ECommerce-Packages/nuget/v3/index.json" />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
  </packageSources>
</configuration>
EOF

# Install Azure Artifacts Credential Provider:
dotnet tool install -g azure-artifacts-credprovider

# Restore packages:
dotnet restore --interactive

# Push a package to Azure Artifacts:
dotnet nuget push bin/Release/MyPackage.1.0.0.nupkg \
  --source "ECommerce-Packages" \
  --api-key az
```

**npm:**
```bash
# Configure npm to use Azure Artifacts feed:
# .npmrc file:
cat > .npmrc << 'EOF'
registry=https://pkgs.dev.azure.com/contoso/ECommerceApp/_packaging/ECommerce-Packages/npm/registry/
always-auth=true
EOF

# Authenticate:
az artifacts universal download --organization https://dev.azure.com/contoso --feed ECommerce-Packages --name mypackage --version 1.0.0 --path ./

# Login with vsts-npm-auth (Windows) or manual token:
npm install -g vsts-npm-auth
vsts-npm-auth -config .npmrc

# Publish:
npm publish

# Install from private feed:
npm install @contoso/shared-components
```

**Universal Packages** (arbitrary files — not specific to a language):
```bash
# Publish a universal package (e.g., deployment scripts, config files):
az artifacts universal publish \
  --organization https://dev.azure.com/contoso \
  --project ECommerceApp \
  --feed ECommerce-Packages \
  --name "deployment-scripts" \
  --version "1.2.3" \
  --description "Deployment scripts and Kubernetes manifests" \
  --path ./deploy/

# Download a universal package:
az artifacts universal download \
  --organization https://dev.azure.com/contoso \
  --project ECommerceApp \
  --feed ECommerce-Packages \
  --name "deployment-scripts" \
  --version "1.2.3" \
  --path ./deploy/
```

### Upstream Sources

Upstream sources allow your feed to proxy external registries. When a package is requested, it's first checked in your feed, then in the upstream (and a copy is saved in your feed).

**Benefits:**
- Single feed URL for all packages (private + public)
- Saved copies protect against npm left-pad incidents
- Governance: can block packages not in your feed

**Configuring upstream in the UI:**
1. Feed Settings → Upstream Sources
2. Add upstream:
   - npm: `https://registry.npmjs.org`
   - NuGet: `https://api.nuget.org/v3/index.json`
   - PyPI: `https://pypi.org/simple`
   - Maven Central: `https://repo.maven.apache.org/maven2`

### Using Artifacts in Pipelines

```yaml
# ── Authenticate to Azure Artifacts in a pipeline ────────────────────────────
steps:
# NuGet authentication (sets up credentials automatically):
- task: NuGetAuthenticate@1
  displayName: 'Authenticate to Azure Artifacts'

# npm authentication:
- task: npmAuthenticate@0
  displayName: 'Authenticate npm to Azure Artifacts'
  inputs:
    workingFile: .npmrc

# Maven authentication:
- task: MavenAuthenticate@0
  inputs:
    mavenServiceConnections: 'MavenArtifactsFeed'

# pip/Python authentication (twine for publishing):
- task: TwineAuthenticate@1
  inputs:
    artifactFeed: 'ECommerce-Packages'

# Publish NuGet package after build:
- task: DotNetCoreCLI@2
  displayName: 'Pack NuGet package'
  inputs:
    command: pack
    packagesToPack: 'src/Shared/**/*.csproj'
    versioningScheme: 'byBuildNumber'

- task: DotNetCoreCLI@2
  displayName: 'Push NuGet package'
  inputs:
    command: push
    packagesToPush: '$(Build.ArtifactStagingDirectory)/**/*.nupkg'
    nuGetFeedType: 'internal'
    publishVstsFeed: 'ECommerce-Packages'
```

---

## 7. GitHub Actions vs Azure Pipelines

### Feature Comparison Table

| Feature                          | Azure Pipelines                                   | GitHub Actions                                |
|---------------------------------|---------------------------------------------------|-----------------------------------------------|
| **Configuration language**       | YAML (azure-pipelines.yml)                        | YAML (.github/workflows/*.yml)               |
| **Trigger syntax**               | `trigger:` / `pr:` / `schedules:`                | `on:` (push, pull_request, schedule, etc.)   |
| **Stages/Environments**          | First-class stages with approvals                 | Environments with protection rules           |
| **Deployment approvals**         | Environment approval gates (rich)                 | Environment protection rules (required reviewers) |
| **Reusable components**          | Templates (step/job/stage), extends              | Reusable workflows (`workflow_call`), composite actions |
| **Hosted runners**               | Ubuntu, Windows, macOS                            | Ubuntu, Windows, macOS                       |
| **Self-hosted runners**          | Agent pools with capabilities                     | Self-hosted runners with labels              |
| **Variable groups / secrets**    | Variable groups + Key Vault integration           | GitHub Secrets + Environments secrets        |
| **Artifacts**                    | Pipeline Artifacts + Azure Artifacts              | GitHub Artifacts + GitHub Packages           |
| **Matrix strategy**              | Yes                                               | Yes                                          |
| **Conditional logic**            | `condition:` + `${{ if }}` expressions           | `if:` expressions                            |
| **Work item / project tracking** | Deep integration with Azure Boards               | GitHub Issues/Projects (lighter-weight)      |
| **Marketplace**                  | Azure DevOps Marketplace (tasks/extensions)       | GitHub Marketplace (actions)                 |
| **Container jobs**               | Yes (`container:`)                                | Yes (`container:`)                           |
| **Service containers**           | Yes (`services:`)                                 | Yes (`services:`)                            |
| **Concurrency control**          | Yes (`lockBehavior`)                              | Yes (`concurrency:`)                         |
| **OIDC / Workload Identity**     | Yes (Workload Identity Federation)                | Yes (`permissions: id-token: write`)         |
| **Cost (public repos)**          | Free unlimited minutes (public)                   | Free unlimited minutes (public)              |
| **Cost (private repos)**         | 1800 min/month free, $40/extra parallel job      | 2000 min/month free, $0.008/min after        |
| **Best for**                     | Enterprises on Azure, teams using Azure Boards    | GitHub-native teams, open source, startups  |

### Migration Guide: Azure Pipelines → GitHub Actions

```yaml
# ── Azure Pipelines (before) ──────────────────────────────────────────────────
trigger:
  branches:
    include: [main]

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

steps:
- task: UseDotNet@2
  inputs:
    version: '8.x'
- task: DotNetCoreCLI@2
  inputs:
    command: restore
    projects: '**/*.csproj'
- task: DotNetCoreCLI@2
  inputs:
    command: build
    arguments: '--configuration $(buildConfiguration)'
- task: DotNetCoreCLI@2
  inputs:
    command: test
    projects: '**/*Tests.csproj'
```

```yaml
# ── GitHub Actions (after) ────────────────────────────────────────────────────
name: Build and Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  BUILD_CONFIGURATION: Release

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.x'

    - name: Restore dependencies
      run: dotnet restore

    - name: Build
      run: dotnet build --configuration ${{ env.BUILD_CONFIGURATION }} --no-restore

    - name: Test
      run: dotnet test --no-build --verbosity normal
```

**Key migration mappings:**
| Azure Pipelines          | GitHub Actions                    |
|--------------------------|-----------------------------------|
| `trigger: branches:`     | `on: push: branches:`             |
| `pr: branches:`          | `on: pull_request: branches:`     |
| `pool: vmImage:`         | `runs-on:`                        |
| `variables:`             | `env:` / `vars:` / `secrets:`    |
| `task: UseDotNet@2`      | `uses: actions/setup-dotnet@v4`  |
| `task: Docker@2`         | `uses: docker/build-push-action` |
| `task: AzureWebApp@1`    | `uses: azure/webapps-deploy`     |
| `condition: succeeded()` | `if: success()`                   |
| `dependsOn:`             | `needs:`                          |
| `stage:`                 | `jobs:` (with `needs:`)           |
| Variable group           | Environment secrets / org secrets |
| Service connection       | GitHub Secret (AZURE_CREDENTIALS) |

### When to Use Each

**Choose Azure Pipelines when:**
- Your team uses Azure Boards for work tracking and needs deep bi-directional integration
- You need rich manual test management (Azure Test Plans)
- You're migrating from TFS/Azure DevOps Server on-premises
- You need enterprise-grade approval workflows with rich environment gates
- Your compliance requirements need audit trails that Azure DevOps provides
- Team is .NET/Microsoft-heavy and benefits from native task ecosystem

**Choose GitHub Actions when:**
- Your source code is already on GitHub
- You're building open source software
- You want a simpler, unified platform (code + CI/CD in one place)
- Your team uses GitHub Issues/Projects for tracking
- You want to leverage the vast GitHub Actions marketplace (10,000+ actions)
- You're deploying to cloud-agnostic or multi-cloud environments

**You can use both together:**
```yaml
# GitHub Actions can trigger Azure Pipelines:
- name: Trigger Azure Pipeline
  run: |
    curl -X POST \
      -H "Authorization: Bearer ${{ secrets.AZURE_DEVOPS_PAT }}" \
      -H "Content-Type: application/json" \
      "https://dev.azure.com/contoso/ECommerceApp/_apis/pipelines/1/runs?api-version=7.0" \
      -d '{"resources":{"repositories":{"self":{"refName":"refs/heads/main"}}}}'
```

---

## 8. Common Pipeline Patterns

### Pattern 1: Publish to Azure Container Registry

```yaml
# azure-pipelines-acr.yml
trigger:
  branches:
    include: [main]

variables:
  acrName: 'contosoregistry'
  acrLoginServer: 'contosoregistry.azurecr.io'
  imageRepository: 'ecommerceapp'
  imageTag: $(Build.BuildId)

pool:
  vmImage: 'ubuntu-latest'

steps:
# Login to ACR using service connection:
- task: Docker@2
  displayName: 'Login to ACR'
  inputs:
    command: login
    containerRegistry: 'ACR-ServiceConnection'

# Build image:
- task: Docker@2
  displayName: 'Build image'
  inputs:
    command: build
    repository: $(imageRepository)
    dockerfile: '**/Dockerfile'
    containerRegistry: 'ACR-ServiceConnection'
    tags: |
      $(imageTag)
      latest
    arguments: |
      --build-arg BUILD_ID=$(Build.BuildId)
      --build-arg BUILD_DATE=$(Build.SourceVersionMessage)

# Push image:
- task: Docker@2
  displayName: 'Push image to ACR'
  inputs:
    command: push
    repository: $(imageRepository)
    containerRegistry: 'ACR-ServiceConnection'
    tags: |
      $(imageTag)
      latest

# OR use buildAndPush in one task:
- task: Docker@2
  displayName: 'Build and push to ACR'
  inputs:
    command: buildAndPush
    repository: $(imageRepository)
    dockerfile: '**/Dockerfile'
    containerRegistry: 'ACR-ServiceConnection'
    tags: |
      $(imageTag)
      latest

# Scan image with Trivy (optional):
- script: |
    docker run --rm \
      -v /var/run/docker.sock:/var/run/docker.sock \
      aquasec/trivy:latest image \
      --severity HIGH,CRITICAL \
      --format table \
      $(acrLoginServer)/$(imageRepository):$(imageTag)
  displayName: 'Scan image for vulnerabilities'
  continueOnError: true
```

---

### Pattern 2: Deploy to AKS

```yaml
# Prereq: AKS service connection in Azure DevOps
# Azure DevOps → Project Settings → Service Connections → New → Kubernetes

stages:
- stage: Deploy_AKS
  jobs:
  - deployment: Deploy
    environment:
      name: production
      resourceType: Kubernetes
      tags: 'aks-production'
    strategy:
      runOnce:
        deploy:
          steps:

          # Create or update Kubernetes secrets from Azure Key Vault:
          - task: AzureKeyVault@2
            inputs:
              azureSubscription: 'Azure-SC'
              keyVaultName: 'contoso-keyvault'
              secretsFilter: 'DB-CONNECTION-STRING,REDIS-PASSWORD'
              runAsPreJob: false

          - task: KubernetesManifest@0
            displayName: 'Create image pull secret'
            inputs:
              action: createSecret
              kubernetesServiceConnection: 'AKS-Production-SC'
              namespace: production
              secretType: dockerRegistry
              secretName: acr-pull-secret
              dockerRegistryEndpoint: 'ACR-ServiceConnection'

          - task: KubernetesManifest@0
            displayName: 'Deploy to AKS'
            inputs:
              action: deploy
              kubernetesServiceConnection: 'AKS-Production-SC'
              namespace: production
              manifests: |
                k8s/namespace.yaml
                k8s/deployment.yaml
                k8s/service.yaml
                k8s/ingress.yaml
                k8s/hpa.yaml
              imagePullSecrets: acr-pull-secret
              containers: |
                $(acrLoginServer)/$(imageRepository):$(imageTag)

          # Wait for deployment to complete:
          - task: Kubernetes@1
            displayName: 'Wait for rollout'
            inputs:
              connectionType: Kubernetes Service Connection
              kubernetesServiceEndpoint: 'AKS-Production-SC'
              namespace: production
              command: rollout
              arguments: 'status deployment/ecommerceapp --timeout=5m'
```

**Example Kubernetes deployment manifest** (`k8s/deployment.yaml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerceapp
  namespace: production
  labels:
    app: ecommerceapp
    version: "$(imageTag)"   # replaced by pipeline via envsubst
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecommerceapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: ecommerceapp
    spec:
      containers:
      - name: ecommerceapp
        image: contosoregistry.azurecr.io/ecommerceapp:$(imageTag)
        ports:
        - containerPort: 8080
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: DB_CONNECTION_STRING
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-connection-string
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

---

### Pattern 3: Deploy to App Service

```yaml
# azure-pipelines-appservice.yml
stages:
- stage: Build
  jobs:
  - job: Build
    steps:
    - task: DotNetCoreCLI@2
      inputs:
        command: publish
        publishWebProjects: true
        arguments: '--configuration Release --output $(Build.ArtifactStagingDirectory)'
        zipAfterPublish: true
    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: $(Build.ArtifactStagingDirectory)
        artifact: drop

- stage: Deploy_Staging
  dependsOn: Build
  jobs:
  - deployment: DeployToStaging
    environment: staging
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: drop
              path: $(Pipeline.Workspace)/drop

          - task: AzureWebApp@1
            displayName: 'Deploy to App Service staging slot'
            inputs:
              azureSubscription: 'Azure-SC'
              appType: webAppLinux
              appName: 'ecommerce-prod'
              deployToSlotOrASE: true
              resourceGroupName: 'ecommerce-rg'
              slotName: 'staging'
              package: '$(Pipeline.Workspace)/drop/*.zip'
              appSettings: |
                -ASPNETCORE_ENVIRONMENT Staging
                -ApplicationInsights__InstrumentationKey $(APP_INSIGHTS_KEY)

          - task: AzureAppServiceManage@0
            displayName: 'Start App Service slot'
            inputs:
              azureSubscription: 'Azure-SC'
              action: 'Start Azure App Service'
              webAppName: 'ecommerce-prod'
              specifySlotOrASE: true
              resourceGroupName: 'ecommerce-rg'
              slot: 'staging'

- stage: Deploy_Production
  dependsOn: Deploy_Staging
  jobs:
  - deployment: SwapSlots
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          # Swap staging slot → production:
          - task: AzureAppServiceManage@0
            displayName: 'Swap staging → production'
            inputs:
              azureSubscription: 'Azure-SC'
              action: 'Swap Slots'
              webAppName: 'ecommerce-prod'
              resourceGroupName: 'ecommerce-rg'
              sourceSlot: 'staging'
              swapWithProduction: true

          # Verify production is healthy:
          - script: |
              for i in 1 2 3 4 5; do
                RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" https://ecommerce.contoso.com/health)
                if [ "$RESPONSE" = "200" ]; then
                  echo "Health check passed (attempt $i)"
                  exit 0
                fi
                echo "Attempt $i failed (HTTP $RESPONSE). Retrying..."
                sleep 15
              done
              echo "Health check failed after 5 attempts"
              exit 1
            displayName: 'Production health check'
```

---

### Pattern 4: Infrastructure Deployment with Bicep

```yaml
# azure-pipelines-bicep.yml
trigger:
  branches:
    include: [main]
  paths:
    include:
      - infra/**

variables:
  - group: infra-config
  resourceGroup: 'ecommerce-rg'
  location: 'eastus'
  bicepFile: 'infra/main.bicep'
  parametersFile: 'infra/main.parameters.json'

pool:
  vmImage: 'ubuntu-latest'

stages:
- stage: Validate
  displayName: 'Validate Bicep'
  jobs:
  - job: Validate
    steps:
    # Validate Bicep syntax:
    - script: |
        az bicep build --file $(bicepFile) --stdout > /dev/null
        echo "Bicep syntax valid"
      displayName: 'Validate Bicep syntax'

    # What-if to preview changes:
    - task: AzureCLI@2
      displayName: 'Bicep what-if'
      inputs:
        azureSubscription: 'Azure-SC'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          az deployment group what-if \
            --resource-group $(resourceGroup) \
            --template-file $(bicepFile) \
            --parameters @$(parametersFile) \
            --parameters environment=production \
            --result-format FullResourcePayloads

- stage: Deploy_Infra
  displayName: 'Deploy Infrastructure'
  dependsOn: Validate
  jobs:
  - deployment: DeployBicep
    environment: production
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureCLI@2
            displayName: 'Deploy Bicep to Azure'
            inputs:
              azureSubscription: 'Azure-SC'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                # Create resource group if not exists:
                az group create \
                  --name $(resourceGroup) \
                  --location $(location) \
                  --tags Environment=Production Project=ECommerce ManagedBy=Pipeline

                # Deploy Bicep:
                DEPLOYMENT_OUTPUT=$(az deployment group create \
                  --name "deployment-$(Build.BuildId)" \
                  --resource-group $(resourceGroup) \
                  --template-file $(bicepFile) \
                  --parameters @$(parametersFile) \
                  --parameters \
                    environment=production \
                    buildId=$(Build.BuildId) \
                  --output json)

                # Extract outputs for use in subsequent jobs:
                AKS_CLUSTER=$(echo $DEPLOYMENT_OUTPUT | jq -r '.properties.outputs.aksClusterName.value')
                ACR_LOGIN_SERVER=$(echo $DEPLOYMENT_OUTPUT | jq -r '.properties.outputs.acrLoginServer.value')

                echo "##vso[task.setvariable variable=aksClusterName;isOutput=true]$AKS_CLUSTER"
                echo "##vso[task.setvariable variable=acrLoginServer;isOutput=true]$ACR_LOGIN_SERVER"
                echo "AKS Cluster: $AKS_CLUSTER"
                echo "ACR: $ACR_LOGIN_SERVER"
            name: deployBicep
```

**Sample Bicep file** (`infra/main.bicep`):
```bicep
@description('Environment name (staging, production)')
param environment string

@description('Azure region for deployment')
param location string = resourceGroup().location

@description('Build ID for tagging')
param buildId string

var prefix = 'ecommerce-${environment}'

resource acr 'Microsoft.ContainerRegistry/registries@2023-01-01-preview' = {
  name: '${replace(prefix, '-', '')}acr'
  location: location
  sku: { name: environment == 'production' ? 'Premium' : 'Standard' }
  properties: {
    adminUserEnabled: false
    networkRuleBypassOptions: 'AzureServices'
  }
  tags: {
    Environment: environment
    BuildId: buildId
    ManagedBy: 'Bicep'
  }
}

resource aks 'Microsoft.ContainerService/managedClusters@2023-07-01' = {
  name: '${prefix}-aks'
  location: location
  identity: { type: 'SystemAssigned' }
  properties: {
    dnsPrefix: '${prefix}-aks'
    agentPoolProfiles: [
      {
        name: 'agentpool'
        count: environment == 'production' ? 3 : 1
        vmSize: environment == 'production' ? 'Standard_D4s_v3' : 'Standard_B2s'
        osType: 'Linux'
        mode: 'System'
        enableAutoScaling: true
        minCount: environment == 'production' ? 3 : 1
        maxCount: environment == 'production' ? 10 : 3
      }
    ]
    networkProfile: {
      networkPlugin: 'azure'
      loadBalancerSku: 'standard'
    }
  }
  tags: {
    Environment: environment
    BuildId: buildId
  }
}

output aksClusterName string = aks.name
output acrLoginServer string = acr.properties.loginServer
```

---

### Pattern 5: Database Migration Automation

```yaml
# azure-pipelines-db-migration.yml
stages:
- stage: Build
  jobs:
  - job: Build
    steps:
    - task: DotNetCoreCLI@2
      displayName: 'Build EF migration bundle'
      inputs:
        command: custom
        custom: ef
        arguments: |
          migrations bundle
          --project src/ECommerce.Data/ECommerce.Data.csproj
          --startup-project src/ECommerce.API/ECommerce.API.csproj
          --configuration Release
          --output $(Build.ArtifactStagingDirectory)/efbundle
          --self-contained

    - task: PublishPipelineArtifact@1
      inputs:
        targetPath: $(Build.ArtifactStagingDirectory)/efbundle
        artifact: db-migration

- stage: Migrate_Staging
  dependsOn: Build
  jobs:
  - deployment: MigrateDB
    environment: staging
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: db-migration
              path: $(Pipeline.Workspace)/migration

          - task: AzureCLI@2
            displayName: 'Run EF migrations on staging DB'
            inputs:
              azureSubscription: 'Azure-SC'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                # Fetch DB connection string from Key Vault:
                DB_CONN=$(az keyvault secret show \
                  --vault-name contoso-kv \
                  --name staging-db-connection-string \
                  --query value -o tsv)

                # Make migration bundle executable:
                chmod +x $(Pipeline.Workspace)/migration/efbundle

                # Run migrations (EF bundle):
                $(Pipeline.Workspace)/migration/efbundle \
                  --connection "$DB_CONN" \
                  --verbose

                echo "Database migration completed successfully"
              failOnStandardError: true

- stage: Migrate_Production
  dependsOn: Migrate_Staging
  jobs:
  - deployment: MigrateProdDB
    environment: production   # requires approval
    strategy:
      runOnce:
        deploy:
          steps:
          - task: DownloadPipelineArtifact@2
            inputs:
              artifact: db-migration
              path: $(Pipeline.Workspace)/migration

          - task: AzureCLI@2
            displayName: 'Backup production database before migration'
            inputs:
              azureSubscription: 'Azure-SC'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                az sql db export \
                  --admin-password "$(PROD_DB_ADMIN_PASSWORD)" \
                  --admin-user sqladmin \
                  --auth-type SQL \
                  --name ecommerce-prod \
                  --server contoso-sql-server \
                  --resource-group ecommerce-rg \
                  --storage-key "$(STORAGE_ACCOUNT_KEY)" \
                  --storage-key-type StorageAccessKey \
                  --storage-uri "https://contosostorage.blob.core.windows.net/db-backups/pre-migration-$(Build.BuildId)-$TIMESTAMP.bacpac"
                echo "Database backup completed: pre-migration-$(Build.BuildId)-$TIMESTAMP.bacpac"

          - task: AzureCLI@2
            displayName: 'Run EF migrations on production DB'
            inputs:
              azureSubscription: 'Azure-SC'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                DB_CONN=$(az keyvault secret show \
                  --vault-name contoso-kv \
                  --name prod-db-connection-string \
                  --query value -o tsv)

                chmod +x $(Pipeline.Workspace)/migration/efbundle

                $(Pipeline.Workspace)/migration/efbundle \
                  --connection "$DB_CONN" \
                  --verbose
```

---

## Summary: Azure DevOps Quick Reference

```
╔══════════════════════════════════════════════════════════════════╗
║             AZURE DEVOPS — QUICK REFERENCE CARD                 ║
╠══════════════════════════════════════════════════════════════════╣
║ BOARDS        │ az boards work-item create/update/show          ║
║               │ Epic → Feature → User Story → Task/Bug          ║
║               │ Process: Agile (most common), Scrum, CMMI       ║
╠══════════════════════════════════════════════════════════════════╣
║ REPOS         │ az repos create/list/show                       ║
║               │ az repos pr create/list/update                  ║
║               │ Branch strategy: GitHub Flow (simplest)         ║
║               │                  GitFlow (versioned releases)    ║
╠══════════════════════════════════════════════════════════════════╣
║ PIPELINES     │ az pipelines create/run/list                    ║
║               │ trigger → stages → jobs → steps → tasks         ║
║               │ Deployment jobs + environments + approvals       ║
║               │ Templates for governance and reuse               ║
╠══════════════════════════════════════════════════════════════════╣
║ TEST PLANS    │ Manual test cases + exploratory testing          ║
║               │ PublishTestResults@2 in pipelines               ║
╠══════════════════════════════════════════════════════════════════╣
║ ARTIFACTS     │ az artifacts universal publish/download         ║
║               │ NuGet + npm + Maven + Python + Universal         ║
║               │ Upstream sources for public registry proxy       ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Next: [15-AZURE-ENTRA-AND-SECURITY.md](./15-AZURE-ENTRA-AND-SECURITY.md) — Microsoft Entra ID, RBAC, PIM, Key Vault, and Zero Trust*
