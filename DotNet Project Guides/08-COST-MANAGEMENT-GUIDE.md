# Azure Cost Management Guide
## File 08: Control and Optimize Your Azure Spending

---

## What You'll Do in This File

By the end of this guide, you'll:
- ✅ Understand what costs money (and what doesn't)
- ✅ Set up budget alerts before you get a surprise bill
- ✅ Know how to reduce costs by 30-60%
- ✅ Have a cost-conscious architecture

**Time Required:** 30 minutes

---

## What Costs Money?

### App Service Costs

```
App Service Plan (the VM) runs 24/7 — you pay even when no traffic!

Tier        Monthly Cost    What You Get
────        ────────────    ────────────
F1 (Free)   $0              60 CPU min/day, no custom domain, no SSL
B1 (Basic)  ~$13            1 core, 1.75 GB, custom domain, SSL
S1 (Std)    ~$73            1 core, 1.75 GB, slots, auto-scale
P1v3 (Prm)  ~$138           2 cores, 8 GB, better performance

💡 Cost saver: Use B1 for dev/test, S1 for production
💡 Multiple apps can share ONE plan (saves money!)
```

### AKS Costs

```
AKS control plane is FREE. You pay for the VMs (nodes).

Node VM         Monthly Cost    Specs
───────         ────────────    ─────
Standard_B2s    ~$30            2 vCPU, 4 GB RAM (dev/test)
Standard_D2s_v5 ~$70            2 vCPU, 8 GB RAM (production)
Standard_D4s_v5 ~$140           4 vCPU, 16 GB RAM (high load)

× Number of nodes = Total cost
Example: 2x B2s = ~$60/month for dev
Example: 3x D2s_v5 = ~$210/month for production

💡 Use B-series (burstable) for dev/test
💡 Enable cluster autoscaler to add/remove nodes
```

### Database Costs

```
Azure SQL Database:
DTU Model       Monthly    Performance
────────        ───────    ───────────
Basic (5 DTU)   ~$5        Very light workloads
S0 (10 DTU)     ~$15       Small dev/test
S1 (20 DTU)     ~$30       Small production
S2 (50 DTU)     ~$75       Medium production
```

### Other Costs

```
Service                     Free Tier           Paid
───────                     ─────────           ────
Key Vault                   10K operations      $0.03/10K operations
Application Insights        5 GB/month          $2.30/GB after
Container Registry (Basic)  —                   ~$5/month
Load Balancer (Standard)    —                   ~$18/month + data
Public IP Address           —                   ~$4/month
Storage (Blob)              5 GB (first 12 mo)  ~$0.02/GB/month
Bandwidth (outbound)        5 GB/month free     ~$0.087/GB after
```

---

## Step 1: Set Up Budget Alerts

```powershell
# Create a budget with alerts at 50%, 80%, 100%
az consumption budget create `
  --amount 200 `
  --budget-name "monthly-budget" `
  --category Cost `
  --time-grain Monthly `
  --start-date "2026-05-01" `
  --end-date "2027-04-30" `
  --resource-group "myproject-dev-rg"

# Or set up in Azure Portal:
# Cost Management → Budgets → Add
# Set amount, alert thresholds, and email recipients
```

### Recommended Alert Thresholds

| Threshold | Action |
|-----------|--------|
| 50% of budget | Informational — review spending trend |
| 80% of budget | Warning — check for unexpected costs |
| 100% of budget | Critical — investigate immediately |
| 120% of budget | Emergency — consider scaling down |

---

## Step 2: Cost-Saving Strategies

### Quick Wins (5 Minutes Each)

```
1. ⭐ Delete unused resources
   az resource list --resource-group myproject-dev-rg --output table
   # Delete anything you're not using!

2. ⭐ Stop dev/test resources after hours
   # App Service: Scale down to Free tier at night
   # AKS: Scale nodes to 0 at night (if using cluster autoscaler)
   az aks nodepool scale --name nodepool1 --cluster-name $AKS --resource-group $RG --node-count 0

3. ⭐ Right-size your resources
   # Check actual CPU/memory usage vs what you're paying for
   # Azure Portal → App Service → Metrics → CPU Percentage
   # If consistently < 20%, downgrade to a smaller tier

4. ⭐ Use Azure Dev/Test pricing
   # 40-60% discount on VMs and other services
   # Requires Visual Studio subscription
```

### Medium Effort (1 Hour)

```
5. Share App Service Plans
   # Run multiple small apps on ONE plan
   # Instead of 3x B1 ($39/mo) → 1x S1 ($73/mo) with 3 apps

6. Use Reserved Instances (1 or 3 year commitment)
   # 30-60% discount on VMs, SQL, and more
   # Best for production workloads you'll run for at least a year
   # Azure Portal → Reservations → Purchase

7. Use Spot VMs for AKS dev/test
   # 60-90% discount, but Azure can take them back
   az aks nodepool add --name spotnodes --cluster-name $AKS --resource-group $RG \
     --priority Spot --spot-max-price -1 --node-count 2

8. Enable auto-shutdown for dev VMs
   # Azure Portal → VM → Auto-shutdown → Enable
   # Set to shutdown at 7 PM, restart at 8 AM
```

---

## Step 3: Sample Cost Comparison

### Dev Environment

| Architecture | Resources | Monthly Cost |
|-------------|-----------|-------------|
| **App Service (Basic)** | B1 plan + S0 SQL + Key Vault + AI | **~$33** |
| **AKS (Minimal)** | 2x B2s nodes + Basic ACR + S0 SQL + LB | **~$98** |

### Production Environment

| Architecture | Resources | Monthly Cost |
|-------------|-----------|-------------|
| **App Service (Standard)** | S1 plan + S2 SQL + KV + AI | **~$153** |
| **AKS (Standard)** | 3x D2s_v5 + Std ACR + S2 SQL + LB | **~$333** |
| **AKS (Reserved)** | Same + 1yr reservation | **~$220** |

---

## Step 4: Monitor Costs Daily

```powershell
# View current month spending
az consumption usage list `
  --start-date (Get-Date -Format "yyyy-MM-01") `
  --end-date (Get-Date -Format "yyyy-MM-dd") `
  --output table

# View cost by resource group
# Best done in Azure Portal → Cost Management → Cost Analysis
# Group by: Resource Group, Service Name, or Tag
```

### Azure Advisor Recommendations

```powershell
# Get cost optimization recommendations
az advisor recommendation list `
  --category Cost `
  --output table
```

---

## ✅ Cost Management Checklist

- [ ] Budget alert set up with email notifications
- [ ] Dev resources sized appropriately (B-series VMs, Basic SKUs)
- [ ] Unused resources identified and deleted
- [ ] Auto-shutdown on dev/test VMs
- [ ] Reserved Instances evaluated for production
- [ ] Cost review scheduled (weekly/monthly)

---

> **Next Step:** Learn to fix issues fast → [09-TROUBLESHOOTING-PLAYBOOK.md](09-TROUBLESHOOTING-PLAYBOOK.md)
