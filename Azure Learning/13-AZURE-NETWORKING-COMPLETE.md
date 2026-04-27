# 13 — Azure Networking Complete: Zero-to-Hero Reference

> **Handbook Style** | Updated for 2024 | Covers VNet, NSG, Firewall, Load Balancer, App Gateway, Front Door, VPN, ExpressRoute, DNS, Private Link, and more

---

## Table of Contents

1. [Azure Virtual Network (VNet) Fundamentals](#1-azure-virtual-network-vnet-fundamentals)
2. [Network Security Groups (NSG)](#2-network-security-groups-nsg)
3. [Azure Firewall](#3-azure-firewall)
4. [Azure Load Balancer](#4-azure-load-balancer)
5. [Azure Application Gateway](#5-azure-application-gateway)
6. [Azure Front Door](#6-azure-front-door)
7. [Azure CDN](#7-azure-cdn)
8. [Azure VPN Gateway](#8-azure-vpn-gateway)
9. [Azure ExpressRoute](#9-azure-expressroute)
10. [Azure DNS](#10-azure-dns)
11. [Azure Private Link and Private Endpoints](#11-azure-private-link-and-private-endpoints)
12. [Network Watcher](#12-network-watcher)
13. [Networking Architecture Patterns](#13-networking-architecture-patterns)

---

## 1. Azure Virtual Network (VNet) Fundamentals

### 1.1 What Is a VNet?

An Azure Virtual Network (VNet) is the fundamental building block of your private network in Azure. It enables Azure resources (VMs, App Services, databases, etc.) to securely communicate with each other, the internet, and on-premises networks.

Key properties:
- **Isolated by default** — traffic between VNets is not permitted unless explicitly configured via peering or gateways
- **Region-scoped** — a VNet lives in a single Azure region (cross-region is handled via Global VNet Peering or Virtual WAN)
- **No charge for the VNet itself** — you pay for compute resources, VPN gateways, and traffic egress

```
Azure Virtual Network (10.0.0.0/16)
┌───────────────────────────────────────────────────────────────┐
│                                                               │
│  Web Subnet (10.0.1.0/24)       App Subnet (10.0.2.0/24)    │
│  ┌─────────────────────────┐    ┌─────────────────────────┐  │
│  │  VM: 10.0.1.4           │    │  VM: 10.0.2.4           │  │
│  │  VM: 10.0.1.5           │    │  VM: 10.0.2.5           │  │
│  └─────────────────────────┘    └─────────────────────────┘  │
│                                                               │
│  Data Subnet (10.0.3.0/24)      Gateway Subnet               │
│  ┌─────────────────────────┐    (10.0.255.0/27)              │
│  │  SQL MI: 10.0.3.4       │    ┌─────────────────────────┐  │
│  │  Redis: 10.0.3.5        │    │  VPN Gateway            │  │
│  └─────────────────────────┘    └─────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

---

### 1.2 Address Spaces and Subnets

**VNet address space** is a range of private IP addresses (RFC 1918):
- `10.0.0.0/8` (10.x.x.x) — up to 16.7M addresses
- `172.16.0.0/12` (172.16.x.x–172.31.x.x)
- `192.168.0.0/16` (192.168.x.x)

**Subnets** divide the VNet address space. Azure reserves **5 IP addresses** per subnet:
- `.0` — Network address
- `.1` — Default gateway
- `.2` and `.3` — Azure DNS
- `.255` — Broadcast

So a /27 subnet (32 addresses) gives you **27 usable IPs**.

**Subnet sizing guidance:**

| Subnet Purpose | Minimum Size | Recommended |
|----------------|-------------|-------------|
| GatewaySubnet (VPN/ER) | /29 | /27 |
| AzureFirewallSubnet | /26 (required) | /26 |
| AzureBastionSubnet | /26 (required) | /26 |
| Application | /29 | /24+ |
| Database | /29 | /24+ |

```bash
# ============================================================
# VNET CLI COMMANDS
# ============================================================

# Create VNet with address space
az network vnet create \
  --resource-group myRG \
  --name myVnet \
  --location eastus \
  --address-prefixes 10.0.0.0/16

# Add subnet
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVnet \
  --name WebSubnet \
  --address-prefixes 10.0.1.0/24

az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVnet \
  --name AppSubnet \
  --address-prefixes 10.0.2.0/24

az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVnet \
  --name DataSubnet \
  --address-prefixes 10.0.3.0/24

az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVnet \
  --name GatewaySubnet \          # exact name required for VPN gateway
  --address-prefixes 10.0.255.0/27

# List subnets
az network vnet subnet list \
  --resource-group myRG \
  --vnet-name myVnet \
  --output table

# Show VNet details
az network vnet show \
  --resource-group myRG \
  --name myVnet \
  --output jsonc
```

---

### 1.3 VNet Peering

VNet Peering connects two VNets using Microsoft's backbone network (not the public internet). Traffic stays within Azure's network.

**Regional Peering** — same region (no gateway required, lowest cost)
**Global Peering** — across Azure regions (slightly higher cost per GB)

```
  VNet A (10.0.0.0/16)          VNet B (10.1.0.0/16)
  East US                        East US
  ┌──────────────────┐           ┌──────────────────┐
  │                  │◄─────────►│                  │
  │  Web: 10.0.1.0   │  Peering  │  DB: 10.1.1.0    │
  │                  │           │                  │
  └──────────────────┘           └──────────────────┘
  VM: 10.0.1.4 → ping 10.1.1.4 ✓ (via peering)
```

```bash
# Create peering from VNet A to VNet B
az network vnet peering create \
  --resource-group myRG \
  --name VnetA-to-VnetB \
  --vnet-name VnetA \
  --remote-vnet VnetB \            # or use resource ID for different subscriptions
  --allow-vnet-access              # enable traffic between the VNets

# Create peering from VNet B to VNet A (peering is NOT symmetric by default)
az network vnet peering create \
  --resource-group myRG \
  --name VnetB-to-VnetA \
  --vnet-name VnetB \
  --remote-vnet VnetA \
  --allow-vnet-access

# Enable gateway transit (hub-spoke: allow spoke to use hub's VPN gateway)
az network vnet peering update \
  --resource-group myRG \
  --name VnetA-to-VnetB \
  --vnet-name VnetA \
  --set allowGatewayTransit=true

# On the spoke side — use remote gateway
az network vnet peering update \
  --resource-group myRG \
  --name VnetB-to-VnetA \
  --vnet-name VnetB \
  --set useRemoteGateways=true

# Verify peering state (must be "Connected" on both sides)
az network vnet peering show \
  --resource-group myRG \
  --name VnetA-to-VnetB \
  --vnet-name VnetA \
  --query "peeringState"
```

**Peering limitations:**
- Non-transitive: if A↔B and B↔C, then A cannot reach C through B without additional routing configuration
- CIDR ranges cannot overlap between peered VNets
- Peering cannot be created between VNets with overlapping address spaces

---

### 1.4 Service Endpoints vs Private Endpoints

| Feature | Service Endpoint | Private Endpoint |
|---------|-----------------|-----------------|
| Traffic path | Goes through Azure backbone; source IP is VNet IP | Stays entirely within VNet; service gets a private IP |
| DNS | Public FQDN resolves to public IP; firewall allows VNet | Private DNS zone resolves public FQDN to private IP |
| Security | Service firewall restricts to VNet | No public endpoint needed at all |
| Cost | Free | ~$7/mo per endpoint + data processing |
| Exfiltration risk | Data can still be exfiltrated to other storage accounts | Full lockdown possible |
| Recommended for | Dev/test, simple scenarios | Production, compliance-sensitive workloads |

```bash
# Enable service endpoint on subnet (Storage example)
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVnet \
  --name DataSubnet \
  --service-endpoints Microsoft.Storage Microsoft.Sql

# Configure Storage account firewall to allow only from VNet
az storage account network-rule add \
  --resource-group myRG \
  --account-name mystorageaccount \
  --vnet-name myVnet \
  --subnet DataSubnet

az storage account update \
  --resource-group myRG \
  --name mystorageaccount \
  --default-action Deny
```

---

### 1.5 DNS Settings

**Options for name resolution within a VNet:**

| Type | Description | Use Case |
|------|-------------|----------|
| Azure-provided DNS | 168.63.129.16 — resolves Azure resource names | Default; works for most scenarios |
| Custom DNS server | Point VNet to your own DNS (e.g., on-prem Active Directory) | Hybrid environments with AD DNS |
| Private DNS Zones | Azure-managed private DNS; auto-registers VM names | Custom domain names within VNet |

```bash
# Set custom DNS servers on VNet (e.g., on-prem DNS)
az network vnet update \
  --resource-group myRG \
  --name myVnet \
  --dns-servers 192.168.1.10 192.168.1.11

# Reset to Azure-provided DNS
az network vnet update \
  --resource-group myRG \
  --name myVnet \
  --dns-servers ""

# Create private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name "internal.mycompany.com"

# Link private DNS zone to VNet (enables auto-registration)
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "internal.mycompany.com" \
  --name myVnetLink \
  --virtual-network myVnet \
  --registration-enabled true    # auto-registers VM hostnames

# Add a DNS record manually
az network private-dns record-set a create \
  --resource-group myRG \
  --zone-name "internal.mycompany.com" \
  --name "api"

az network private-dns record-set a add-record \
  --resource-group myRG \
  --zone-name "internal.mycompany.com" \
  --record-set-name "api" \
  --ipv4-address 10.0.2.10
# api.internal.mycompany.com → 10.0.2.10
```

---

### 1.6 Bicep Template

```bicep
// vnet-with-subnets.bicep
param location string = resourceGroup().location
param vnetName string = 'myVnet'
param vnetAddressPrefix string = '10.0.0.0/16'

var subnets = [
  { name: 'WebSubnet',  prefix: '10.0.1.0/24' }
  { name: 'AppSubnet',  prefix: '10.0.2.0/24' }
  { name: 'DataSubnet', prefix: '10.0.3.0/24' }
  { name: 'GatewaySubnet', prefix: '10.0.255.0/27' }
]

resource vnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [ vnetAddressPrefix ]
    }
    subnets: [for subnet in subnets: {
      name: subnet.name
      properties: {
        addressPrefix: subnet.prefix
        serviceEndpoints: subnet.name == 'DataSubnet' ? [
          { service: 'Microsoft.Storage' }
          { service: 'Microsoft.Sql' }
        ] : []
      }
    }]
    enableDdosProtection: false
  }
}

// VNet Peering to a hub VNet
resource peering 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2023-09-01' = {
  parent: vnet
  name: 'spoke-to-hub'
  properties: {
    remoteVirtualNetwork: {
      id: resourceId('myHubRG', 'Microsoft.Network/virtualNetworks', 'HubVnet')
    }
    allowVirtualNetworkAccess: true
    allowForwardedTraffic: true
    useRemoteGateways: true       // use hub VPN gateway
    allowGatewayTransit: false
  }
}

output vnetId string = vnet.id
output subnetIds object = {
  web:  vnet.properties.subnets[0].id
  app:  vnet.properties.subnets[1].id
  data: vnet.properties.subnets[2].id
}
```

---

## 2. Network Security Groups (NSG)

### 2.1 What Is an NSG?

A Network Security Group (NSG) is a Layer 4 firewall that filters traffic by 5-tuple: source IP, source port, destination IP, destination port, and protocol (TCP/UDP/ICMP).

An NSG can be associated with:
- A **subnet** — applies to all resources in the subnet
- A **network interface (NIC)** — applies to a specific VM

When both are associated, **both NSGs are evaluated**: subnet NSG first for inbound traffic; NIC NSG first for outbound traffic.

---

### 2.2 Inbound and Outbound Security Rules

Each rule has these fields:

| Field | Description |
|-------|-------------|
| Priority | 100–4096; lower number = higher priority |
| Source | IP, CIDR, Service Tag, or ASG |
| Source port range | Single, range, or `*` |
| Destination | IP, CIDR, Service Tag, or ASG |
| Destination port range | Single, range, or `*` |
| Protocol | TCP, UDP, ICMP, or Any |
| Action | Allow or Deny |

**Default rules** (cannot be deleted, priority 65000–65500):
- `AllowVnetInBound` — allows intra-VNet traffic
- `AllowAzureLoadBalancerInBound` — allows ALB health probes
- `DenyAllInBound` — denies everything else
- `AllowVnetOutBound` — allows outbound to VNet
- `AllowInternetOutBound` — allows outbound to internet
- `DenyAllOutBound` — denies everything else

```bash
# Create NSG
az network nsg create \
  --resource-group myRG \
  --name WebSubnetNSG \
  --location eastus

# Allow HTTP inbound from internet
az network nsg rule create \
  --resource-group myRG \
  --nsg-name WebSubnetNSG \
  --name Allow-HTTP \
  --priority 100 \
  --direction Inbound \
  --source-address-prefixes Internet \
  --source-port-ranges "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --access Allow

# Allow HTTPS inbound from internet
az network nsg rule create \
  --resource-group myRG \
  --nsg-name WebSubnetNSG \
  --name Allow-HTTPS \
  --priority 110 \
  --direction Inbound \
  --source-address-prefixes Internet \
  --source-port-ranges "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges 443 \
  --protocol Tcp \
  --access Allow

# Allow RDP only from a specific management IP
az network nsg rule create \
  --resource-group myRG \
  --nsg-name WebSubnetNSG \
  --name Allow-RDP-Management \
  --priority 200 \
  --direction Inbound \
  --source-address-prefixes 203.0.113.5/32 \
  --source-port-ranges "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges 3389 \
  --protocol Tcp \
  --access Allow

# Deny all other inbound (explicit, lower than default DenyAll)
az network nsg rule create \
  --resource-group myRG \
  --nsg-name WebSubnetNSG \
  --name Deny-All-Inbound \
  --priority 4000 \
  --direction Inbound \
  --source-address-prefixes "*" \
  --source-port-ranges "*" \
  --destination-address-prefixes "*" \
  --destination-port-ranges "*" \
  --protocol "*" \
  --access Deny

# Associate NSG with subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVnet \
  --name WebSubnet \
  --network-security-group WebSubnetNSG

# List rules
az network nsg rule list \
  --resource-group myRG \
  --nsg-name WebSubnetNSG \
  --output table

# Delete rule
az network nsg rule delete \
  --resource-group myRG \
  --nsg-name WebSubnetNSG \
  --name Allow-RDP-Management
```

---

### 2.3 Application Security Groups (ASG)

ASGs let you group VMs by role and reference these groups in NSG rules — no need to maintain IP lists.

```
Without ASG:                 With ASG:
NSG Rule: Allow 10.0.1.4,   NSG Rule: Allow ASG:WebServers
         10.0.1.5, 10.0.1.6           → ASG:AppServers:8080
         → 10.0.2.7, 10.0.2.8
         port 8080
```

```bash
# Create ASGs
az network asg create \
  --resource-group myRG \
  --name WebServersASG \
  --location eastus

az network asg create \
  --resource-group myRG \
  --name AppServersASG \
  --location eastus

# Associate VM's NIC with ASG
az network nic update \
  --resource-group myRG \
  --name WebVM1-nic \
  --application-security-groups WebServersASG

az network nic update \
  --resource-group myRG \
  --name WebVM2-nic \
  --application-security-groups WebServersASG

# NSG rule using ASG as source/destination
az network nsg rule create \
  --resource-group myRG \
  --nsg-name AppSubnetNSG \
  --name Allow-Web-To-App \
  --priority 100 \
  --direction Inbound \
  --source-asgs WebServersASG \    # ASGs instead of IPs
  --source-port-ranges "*" \
  --destination-asgs AppServersASG \
  --destination-port-ranges 8080 \
  --protocol Tcp \
  --access Allow
```

---

### 2.4 NSG Flow Logs

Flow logs record information about IP traffic through NSGs. They are stored in Azure Blob Storage and can be analyzed in Azure Traffic Analytics.

```bash
# Register the Insights provider (required)
az provider register --namespace Microsoft.Insights

# Enable NSG flow logs (v2 — includes byte/packet counts)
az network watcher flow-log create \
  --resource-group myRG \
  --name myNSGFlowLog \
  --nsg WebSubnetNSG \
  --storage-account /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/myflowlogsstorage \
  --enabled true \
  --format JSON \
  --log-version 2 \
  --retention 30 \
  --traffic-analytics true \
  --workspace /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace \
  --interval 10    # traffic analytics processing interval in minutes
```

---

## 3. Azure Firewall

### 3.1 What Is Azure Firewall vs NSG?

| Feature | NSG | Azure Firewall |
|---------|-----|----------------|
| Layer | L4 (transport) | L4 + L7 (application) |
| FQDN filtering | No | Yes |
| Threat intelligence | No | Yes |
| Centralized management | Per-NSG | Firewall Manager (multi-subscription) |
| Cost | Free | ~$1.25/hr + data processing |
| Use case | Micro-segmentation within VNet | North-south traffic control, egress filtering |

**Best practice:** Use NSGs for micro-segmentation (east-west traffic within VNet) + Azure Firewall for centralized north-south control (internet-bound and on-prem-bound traffic).

---

### 3.2 Azure Firewall Tiers

| Tier | Features | Use Case |
|------|----------|----------|
| **Standard** | L3-L7 filtering, FQDN, threat intel, DNAT/SNAT | Most enterprise workloads |
| **Premium** | All Standard + TLS inspection, IDPS (intrusion detection), URL filtering, web categories | High-security, PCI DSS, compliance |
| **Basic** | Simplified, limited rules | Small businesses, dev/test |

---

### 3.3 Rule Collections

Azure Firewall evaluates rules in order: **DNAT → Network → Application**. Within each type, lower priority number = higher precedence.

```
Rule Evaluation Order:
1. DNAT Rules    — translate inbound traffic (e.g., expose internal RDP via public IP)
2. Network Rules — L4 filtering by IP/port (allow/deny)
3. Application Rules — L7 filtering by FQDN/URL (allow/deny)

Note: If a Network rule explicitly denies traffic, Application rules are NOT evaluated.
```

```bash
# Create Azure Firewall

# First: create dedicated subnet (must be named AzureFirewallSubnet, /26 minimum)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVnet \
  --name AzureFirewallSubnet \
  --address-prefixes 10.0.100.0/26

# Create public IP for firewall
az network public-ip create \
  --resource-group myRG \
  --name FirewallPublicIP \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

# Create Azure Firewall
az network firewall create \
  --resource-group myRG \
  --name myFirewall \
  --location eastus \
  --sku-tier Standard \
  --vnet-name myVnet \
  --public-ip FirewallPublicIP \
  --enable-dns-proxy true        # required for FQDN-based network rules

# Create Firewall Policy (recommended over classic rules)
az network firewall policy create \
  --resource-group myRG \
  --name myFirewallPolicy \
  --sku Standard \
  --threat-intel-mode Alert      # Alert or Deny

# Associate policy with firewall
az network firewall update \
  --resource-group myRG \
  --name myFirewall \
  --firewall-policy myFirewallPolicy

# ---- APPLICATION RULES ----
# Allow outbound HTTPS to Microsoft Update
az network firewall policy rule-collection-group create \
  --resource-group myRG \
  --policy-name myFirewallPolicy \
  --name ApplicationRuleGroup \
  --priority 200

az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group myRG \
  --policy-name myFirewallPolicy \
  --rule-collection-group-name ApplicationRuleGroup \
  --name AllowWindowsUpdate \
  --collection-priority 100 \
  --action Allow \
  --rule-name AllowUpdate \
  --rule-type ApplicationRule \
  --source-addresses "10.0.0.0/16" \
  --protocols Https=443 \
  --fqdn-tags WindowsUpdate

# Allow outbound to specific FQDNs
az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group myRG \
  --policy-name myFirewallPolicy \
  --rule-collection-group-name ApplicationRuleGroup \
  --name AllowAzureServices \
  --collection-priority 200 \
  --action Allow \
  --rule-name AllowStorage \
  --rule-type ApplicationRule \
  --source-addresses "10.0.0.0/16" \
  --protocols Https=443 \
  --target-fqdns "*.blob.core.windows.net" "*.table.core.windows.net"

# ---- NETWORK RULES ----
az network firewall policy rule-collection-group collection add-filter-collection \
  --resource-group myRG \
  --policy-name myFirewallPolicy \
  --rule-collection-group-name ApplicationRuleGroup \
  --name AllowDNS \
  --collection-priority 100 \
  --action Allow \
  --rule-name AllowDNSQuery \
  --rule-type NetworkRule \
  --source-addresses "10.0.0.0/16" \
  --destination-addresses "8.8.8.8" "8.8.4.4" \
  --destination-ports 53 \
  --ip-protocols UDP

# ---- DNAT RULES (inbound) ----
# Forward public port 2222 to internal RDP on 10.0.1.4:3389
az network firewall policy rule-collection-group collection add-nat-collection \
  --resource-group myRG \
  --policy-name myFirewallPolicy \
  --rule-collection-group-name ApplicationRuleGroup \
  --name DNAT-RDP \
  --collection-priority 50 \
  --action DNAT \
  --rule-name RDP-VM1 \
  --rule-type NatRule \
  --source-addresses "*" \
  --destination-addresses "<FirewallPublicIP>" \
  --destination-ports 2222 \
  --ip-protocols TCP \
  --translated-address "10.0.1.4" \
  --translated-port 3389
```

---

### 3.4 Route Table for Forced Tunneling

To route all VNet traffic through the Azure Firewall, create a User Defined Route (UDR):

```bash
# Get the firewall's private IP
FIREWALL_PRIVATE_IP=$(az network firewall show \
  --resource-group myRG \
  --name myFirewall \
  --query "ipConfigurations[0].privateIPAddress" \
  --output tsv)

# Create route table
az network route-table create \
  --resource-group myRG \
  --name ForcedTunnelingRT \
  --location eastus \
  --disable-bgp-route-propagation true   # prevent BGP routes from overriding

# Add default route to firewall
az network route-table route create \
  --resource-group myRG \
  --route-table-name ForcedTunnelingRT \
  --name DefaultToFirewall \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $FIREWALL_PRIVATE_IP

# Associate route table with subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVnet \
  --name WebSubnet \
  --route-table ForcedTunnelingRT
```

---

## 4. Azure Load Balancer

### 4.1 Basic vs Standard SKU

| Feature | Basic SKU | Standard SKU |
|---------|-----------|--------------|
| Backend pool size | 300 VMs | 1000 VMs |
| Health probes | HTTP, TCP | HTTP, HTTPS, TCP |
| Availability Zones | Not supported | Zone-redundant |
| SLA | None | 99.99% |
| HA Ports | Not supported | Supported |
| Outbound rules | Not supported | Supported |
| Secure by default | No (open) | Yes (NSG required) |
| Cost | Free | Charged per rule + data |
| Recommendation | Dev/test only | All production workloads |

---

### 4.2 Public vs Internal Load Balancer

```
Public Load Balancer                Internal Load Balancer
────────────────────                ──────────────────────

Internet → Public IP                VNet Internal traffic
       ↓                                    ↓
  Frontend IP (Public)              Frontend IP (Private: 10.0.1.10)
       ↓                                    ↓
  Backend Pool (VMs)               Backend Pool (VMs/App Services)
  10.0.1.4, 10.0.1.5              10.0.2.4, 10.0.2.5

Use: External-facing apps          Use: Internal microservices,
     Web servers, APIs                  multi-tier architectures,
                                        private APIs
```

---

### 4.3 CLI Commands

```bash
# ============================================================
# AZURE LOAD BALANCER — STANDARD PUBLIC
# ============================================================

# Create public IP
az network public-ip create \
  --resource-group myRG \
  --name LBPublicIP \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

# Create load balancer with frontend
az network lb create \
  --resource-group myRG \
  --name myLoadBalancer \
  --sku Standard \
  --public-ip-address LBPublicIP \
  --frontend-ip-name FrontendIP \
  --backend-pool-name BackendPool

# Create health probe (HTTP on port 80, path /health)
az network lb probe create \
  --resource-group myRG \
  --lb-name myLoadBalancer \
  --name HttpHealthProbe \
  --protocol Http \
  --port 80 \
  --path "/health" \
  --interval 15 \              # probe every 15 seconds
  --threshold 2                # 2 failures = unhealthy

# Create load balancing rule (TCP 80 → backend 80)
az network lb rule create \
  --resource-group myRG \
  --lb-name myLoadBalancer \
  --name HttpRule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name FrontendIP \
  --backend-pool-name BackendPool \
  --probe-name HttpHealthProbe \
  --load-distribution SourceIPProtocol \   # session affinity by source IP + protocol
  --idle-timeout 4             # TCP idle timeout in minutes

# Add VM to backend pool
az network nic ip-config update \
  --resource-group myRG \
  --nic-name WebVM1-nic \
  --name ipconfig1 \
  --lb-name myLoadBalancer \
  --lb-address-pools BackendPool

# Create outbound rule (SNAT for VMs)
az network lb outbound-rule create \
  --resource-group myRG \
  --lb-name myLoadBalancer \
  --name OutboundRule \
  --frontend-ip-configs FrontendIP \
  --backend-address-pool BackendPool \
  --protocol All \
  --outbound-ports 10000 \     # number of SNAT ports per instance
  --idle-timeout 4

# Create inbound NAT rule (direct SSH to specific VM)
az network lb inbound-nat-rule create \
  --resource-group myRG \
  --lb-name myLoadBalancer \
  --name SSH-VM1 \
  --protocol Tcp \
  --frontend-port 50001 \
  --backend-port 22 \
  --frontend-ip-name FrontendIP

# HA Ports rule (internal LB — routes ALL ports to backend)
az network lb rule create \
  --resource-group myRG \
  --lb-name myInternalLB \
  --name HAPortsRule \
  --protocol All \
  --frontend-port 0 \          # 0 = all ports (HA Ports)
  --backend-port 0 \
  --frontend-ip-name FrontendIP \
  --backend-pool-name BackendPool \
  --probe-name HttpHealthProbe
```

---

## 5. Azure Application Gateway

### 5.1 What Is Application Gateway?

Application Gateway is a Layer 7 (HTTP/HTTPS) load balancer with SSL offloading, cookie-based session affinity, URL-based routing, and Web Application Firewall (WAF) capabilities.

```
Application Gateway (Layer 7)
                                  ┌──────────────┐
                          ┌──────►│ /api/*       │→ Backend Pool: API VMs
                          │       └──────────────┘
Internet → HTTPS:443 ────►│
(SSL terminated here)     │       ┌──────────────┐
                          └──────►│ /images/*    │→ Backend Pool: Storage/CDN
                                  └──────────────┘
                          (URL-path routing)
```

---

### 5.2 SKUs

| SKU | Features | Use Case |
|-----|----------|----------|
| **Standard_v2** | L7 LB, SSL, URL routing, autoscaling | Standard web apps |
| **WAF_v2** | All Standard_v2 + OWASP WAF rules | Internet-facing apps needing security |

---

### 5.3 CLI Commands

```bash
# Create Application Gateway subnet (separate subnet required)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVnet \
  --name AppGwSubnet \
  --address-prefixes 10.0.10.0/24

# Create public IP
az network public-ip create \
  --resource-group myRG \
  --name AppGwPublicIP \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

# Create Application Gateway (WAF_v2)
az network application-gateway create \
  --resource-group myRG \
  --name myAppGateway \
  --location eastus \
  --sku WAF_v2 \
  --capacity 2 \               # 2 instances; use --min-capacity/--max-capacity for autoscale
  --vnet-name myVnet \
  --subnet AppGwSubnet \
  --public-ip-address AppGwPublicIP \
  --frontend-port 443 \
  --http-settings-cookie-based-affinity Enabled \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --routing-rule-type Basic \
  --priority 100 \
  --servers "10.0.2.4" "10.0.2.5"   # initial backend pool

# Enable autoscaling
az network application-gateway update \
  --resource-group myRG \
  --name myAppGateway \
  --set autoscaleConfiguration.minCapacity=1 \
  --set autoscaleConfiguration.maxCapacity=10

# Add SSL certificate (PFX file)
az network application-gateway ssl-cert create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name mySSLCert \
  --cert-file ./mycert.pfx \
  --cert-password "CertPassword!"

# Update frontend listener to use SSL
az network application-gateway http-listener update \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name appGatewayHttpListener \
  --ssl-cert mySSLCert \
  --frontend-port 443

# Add URL path map for routing
az network application-gateway url-path-map create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name UrlPathMap \
  --paths "/api/*" \
  --address-pool APIBackendPool \
  --http-settings AppGwBackendHttpSettings \
  --rule-name ApiRule \
  --default-address-pool DefaultBackendPool \
  --default-http-settings AppGwBackendHttpSettings

# Enable WAF and set to Prevention mode
az network application-gateway waf-config set \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --enabled true \
  --firewall-mode Prevention \   # Detection (log only) or Prevention (block)
  --rule-set-type OWASP \
  --rule-set-version 3.2

# HTTP to HTTPS redirect rule
az network application-gateway redirect-config create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name HTTPtoHTTPS \
  --type Permanent \
  --target-listener appGatewayHttpsListener \
  --include-path true \
  --include-query-string true
```

---

### 5.4 Multi-Site Hosting

Application Gateway can host multiple websites (different domain names) on the same gateway by routing based on the HTTP Host header.

```bash
# Add backend pool for second site
az network application-gateway address-pool create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name Site2BackendPool \
  --servers "10.0.2.6" "10.0.2.7"

# Add listener for site2.com
az network application-gateway http-listener create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name Site2Listener \
  --frontend-ip appGatewayFrontendIP \
  --frontend-port appGatewayFrontendPort443 \
  --ssl-cert Site2SSLCert \
  --host-name "site2.example.com"   # route by host header

# Add routing rule for site2
az network application-gateway rule create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name Site2Rule \
  --http-listener Site2Listener \
  --address-pool Site2BackendPool \
  --http-settings AppGwBackendHttpSettings \
  --rule-type Basic \
  --priority 200
```

---

## 6. Azure Front Door

### 6.1 What Is Azure Front Door?

Azure Front Door (Standard/Premium) is a global, scalable entry point for web applications. It combines:
- **Global CDN** — serve cached content from Microsoft's edge PoPs worldwide
- **Global load balancing** — route traffic to the closest/healthiest backend (origin)
- **WAF** — globally applied at the edge
- **SSL offloading** — terminate TLS at the edge
- **URL rewriting and redirects**

```
                         Azure Front Door (Global Edge)
                              ┌─────────────────┐
User (Asia) ─────────────────►│  Tokyo Edge PoP │
                              └────────┬────────┘
                                       │  route to origin
                                       ▼
User (Europe) ────────────────►┌───────────────┐   Origin Group
                               │ London Edge   │──────────────────────────────────┐
                               └───────────────┘                                  │
                                                                           ┌──────▼──────┐
User (US) ─────────────────────►┌──────────────┐                          │  App Service │
                                │ Dallas Edge  │──────────────────────────► (East US)    │
                                └──────────────┘                          └─────────────┘
```

---

### 6.2 Front Door vs Traffic Manager vs Application Gateway

| Feature | Front Door | Traffic Manager | Application Gateway |
|---------|-----------|----------------|---------------------|
| Scope | Global (HTTP/S) | Global (DNS-based) | Regional (HTTP/S) |
| Layer | L7 (HTTP) | L4 (DNS) | L7 (HTTP) |
| Caching/CDN | Yes | No | No |
| WAF | Yes (global) | No | Yes (regional) |
| SSL Offload | Yes | No | Yes |
| WebSockets | Yes | No | Yes |
| Health probes | HTTP | HTTP, TCP, HTTPS | HTTP, HTTPS, TCP |
| Use case | Global web apps | Global non-HTTP, multi-protocol | Regional web apps |

---

### 6.3 CLI Commands

```bash
# Create Front Door profile (Standard tier)
az afd profile create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --sku Standard_AzureFrontDoor

# Add endpoint
az afd endpoint create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name myapp \
  --enabled-state Enabled
# Result: myapp.z01.azurefd.net

# Add origin group (load balancing and health probe config)
az afd origin-group create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --probe-request-type GET \
  --probe-protocol Https \
  --probe-interval-in-seconds 100 \
  --probe-path "/health" \
  --sample-size 4 \
  --successful-samples-required 3 \
  --additional-latency-in-milliseconds 50   # route to origins within 50ms of fastest

# Add origin (backend — can be App Service, Storage, Public IP, etc.)
az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --origin-name EastUSOrigin \
  --host-name myapp-eastus.azurewebsites.net \
  --origin-host-header myapp-eastus.azurewebsites.net \
  --http-port 80 \
  --https-port 443 \
  --priority 1 \
  --weight 1000 \
  --enabled-state Enabled

az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --origin-name WestEuropeOrigin \
  --host-name myapp-westeurope.azurewebsites.net \
  --origin-host-header myapp-westeurope.azurewebsites.net \
  --https-port 443 \
  --priority 2 \              # lower priority = failover
  --weight 1000 \
  --enabled-state Enabled

# Add custom domain
az afd custom-domain create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --custom-domain-name myCustomDomain \
  --host-name "www.myapp.com" \
  --certificate-type ManagedCertificate   # Azure manages the TLS cert

# Add route (connect endpoint + custom domain → origin group)
az afd route create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name myapp \
  --route-name DefaultRoute \
  --origin-group myOriginGroup \
  --supported-protocols Https \
  --https-redirect Enabled \
  --forwarding-protocol HttpsOnly \
  --link-to-default-domain Enabled \
  --custom-domains myCustomDomain \
  --patterns-to-match "/*"

# Add WAF policy (Premium tier)
az network front-door waf-policy create \
  --resource-group myRG \
  --name myWAFPolicy \
  --sku Standard_AzureFrontDoor \   # use Premium_AzureFrontDoor for Premium
  --mode Prevention

# Associate WAF with security policy
az afd security-policy create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --security-policy-name mySecurityPolicy \
  --domains "/subscriptions/.../endpoints/myapp" \
  --waf-policy "/subscriptions/.../frontDoorWebApplicationFirewallPolicies/myWAFPolicy"
```

---

## 7. Azure CDN

### 7.1 What Is Azure CDN?

Azure CDN (Content Delivery Network) caches static content (images, CSS, JS, videos) at edge nodes close to users worldwide, reducing latency and origin server load.

**CDN Profiles:**

| Provider | Features | Use Case |
|---------|----------|----------|
| **Microsoft (Standard)** | Basic CDN, rules engine, HTTPS | General static content |
| **Akamai (Standard)** | Global reach, media delivery | High-traffic media streaming |
| **Verizon (Standard/Premium)** | Advanced analytics, rules, real-time stats | Advanced routing needs |

> **Note:** For new deployments, consider **Azure Front Door Standard/Premium** which includes CDN capabilities plus global load balancing and WAF in a single product.

```bash
# Create CDN profile
az cdn profile create \
  --resource-group myRG \
  --name myCDNProfile \
  --sku Standard_Microsoft \
  --location global

# Create CDN endpoint
az cdn endpoint create \
  --resource-group myRG \
  --profile-name myCDNProfile \
  --name mycdnendpoint \           # becomes mycdnendpoint.azureedge.net
  --origin mystorage.blob.core.windows.net \
  --origin-host-header mystorage.blob.core.windows.net \
  --query-string-caching-behavior IgnoreQueryString \
  --content-types-to-compress \
    "text/html" "text/css" "application/javascript" "application/json" \
  --is-compression-enabled true

# Purge cached content (force refresh)
az cdn endpoint purge \
  --resource-group myRG \
  --profile-name myCDNProfile \
  --name mycdnendpoint \
  --content-paths "/*"             # purge all content

# Purge specific paths
az cdn endpoint purge \
  --resource-group myRG \
  --profile-name myCDNProfile \
  --name mycdnendpoint \
  --content-paths "/images/*" "/css/styles.css"

# Add custom domain
az cdn custom-domain create \
  --resource-group myRG \
  --endpoint-name mycdnendpoint \
  --profile-name myCDNProfile \
  --name myCustomDomain \
  --hostname "cdn.myapp.com"

# Enable HTTPS on custom domain (managed certificate)
az cdn custom-domain enable-https \
  --resource-group myRG \
  --endpoint-name mycdnendpoint \
  --profile-name myCDNProfile \
  --name myCustomDomain \
  --min-tls-version TLS12
```

---

## 8. Azure VPN Gateway

### 8.1 Site-to-Site VPN

Site-to-Site (S2S) VPN connects your on-premises network to an Azure VNet over an IPsec/IKE tunnel across the public internet.

```
On-Premises Network              Azure Virtual Network
192.168.0.0/16                   10.0.0.0/16
┌─────────────────┐              ┌─────────────────┐
│                 │   IPsec/IKE  │                 │
│  On-prem Router │◄────────────►│  VPN Gateway    │
│  (Local Network │   Internet   │  (GatewaySubnet)│
│   Gateway)      │              │                 │
└─────────────────┘              └─────────────────┘
```

```bash
# Step 1: Create VPN Gateway public IP
az network public-ip create \
  --resource-group myRG \
  --name VPNGatewayPublicIP \
  --sku Basic \
  --allocation-method Dynamic    # Basic SKU requires Dynamic

# Step 2: Create VPN Gateway (VpnGw1 — production, BGP-capable)
az network vnet-gateway create \
  --resource-group myRG \
  --name myVPNGateway \
  --location eastus \
  --public-ip-addresses VPNGatewayPublicIP \
  --vnet myVnet \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2 \
  --generation Generation2 \    # Gen2 has better throughput
  --no-wait                     # takes 20-45 minutes to provision

# Step 3: Create Local Network Gateway (represents on-prem VPN device)
az network local-gateway create \
  --resource-group myRG \
  --name OnPremLocalGateway \
  --gateway-ip-address 203.0.113.100 \  # on-prem public IP
  --local-address-prefixes 192.168.0.0/16 192.168.1.0/24   # on-prem CIDR ranges

# Step 4: Create VPN connection
az network vpn-connection create \
  --resource-group myRG \
  --name AzureToOnPrem \
  --vnet-gateway1 myVPNGateway \
  --local-gateway2 OnPremLocalGateway \
  --shared-key "MyVPNSharedKey2024!" \
  --connection-protocol IKEv2

# Check connection status
az network vpn-connection show \
  --resource-group myRG \
  --name AzureToOnPrem \
  --query "connectionStatus"
```

---

### 8.2 Point-to-Site VPN

Point-to-Site (P2S) VPN connects individual client computers to an Azure VNet. Used for remote workers and developers.

```bash
# Generate root certificate (PowerShell on Windows or openssl)
# Generate root cert
openssl req -x509 -newkey rsa:4096 -nodes \
  -keyout p2s-root-key.pem \
  -out p2s-root-cert.pem \
  -days 3650 \
  -subj "/CN=P2S-Root-Cert"

# Export root cert as base64 for upload
ROOT_CERT=$(openssl x509 -in p2s-root-cert.pem -outform DER | base64 -w0)

# Configure P2S on VPN Gateway
az network vnet-gateway update \
  --resource-group myRG \
  --name myVPNGateway \
  --address-prefixes 172.16.201.0/24 \  # VPN client address pool
  --client-protocol IkeV2 SSTP \
  --radius-secret "" \
  --vpn-auth-type Certificate

# Upload root certificate
az network vnet-gateway root-cert create \
  --resource-group myRG \
  --gateway-name myVPNGateway \
  --name P2SRootCert \
  --public-cert-data "$ROOT_CERT"

# Download VPN client configuration
az network vnet-gateway vpn-client generate \
  --resource-group myRG \
  --name myVPNGateway \
  --output tsv
# Returns a URL to download the VPN client package (ZIP)
```

---

### 8.3 VPN Gateway SKUs

| SKU | Max Throughput | Max S2S Tunnels | Max P2S Clients | BGP | Zone Redundant |
|-----|----------------|-----------------|-----------------|-----|----------------|
| Basic | 100 Mbps | 10 | 128 | No | No |
| VpnGw1 | 650 Mbps | 30 | 250 | Yes | No |
| VpnGw2 | 1 Gbps | 30 | 500 | Yes | No |
| VpnGw3 | 1.25 Gbps | 30 | 1000 | Yes | No |
| VpnGw1AZ | 650 Mbps | 30 | 250 | Yes | Yes |
| VpnGw2AZ | 1 Gbps | 30 | 500 | Yes | Yes |
| VpnGw3AZ | 1.25 Gbps | 30 | 1000 | Yes | Yes |

---

### 8.4 Active-Active vs Active-Passive

| Mode | Description | Failover | Throughput |
|------|-------------|---------|-----------|
| **Active-Passive** (default) | One instance active, one standby | 10–15 seconds | Single tunnel bandwidth |
| **Active-Active** | Both instances active, two BGP tunnels | Sub-second | Double tunnel bandwidth |

```bash
# Enable Active-Active mode (requires two public IPs)
az network public-ip create \
  --resource-group myRG \
  --name VPNGatewayPublicIP2 \
  --sku Basic \
  --allocation-method Dynamic

az network vnet-gateway update \
  --resource-group myRG \
  --name myVPNGateway \
  --public-ip-addresses VPNGatewayPublicIP VPNGatewayPublicIP2 \
  --set activeActive=true
```

---

### 8.5 BGP Configuration

BGP allows dynamic routing between Azure and on-premises. When the on-prem network changes, routes update automatically.

```bash
# Enable BGP on VPN Gateway
az network vnet-gateway update \
  --resource-group myRG \
  --name myVPNGateway \
  --enable-bgp true \
  --asn 65010                  # Azure BGP ASN (avoid 65515 — reserved by Azure)

# Configure Local Network Gateway with BGP
az network local-gateway update \
  --resource-group myRG \
  --name OnPremLocalGateway \
  --asn 65001 \                # on-prem ASN
  --bgp-peering-address 192.168.0.254   # on-prem BGP peer IP

# Enable BGP on connection
az network vpn-connection update \
  --resource-group myRG \
  --name AzureToOnPrem \
  --enable-bgp true

# View learned routes via BGP
az network vnet-gateway list-learned-routes \
  --resource-group myRG \
  --name myVPNGateway \
  --output table
```

---

## 9. Azure ExpressRoute

### 9.1 ExpressRoute vs VPN Gateway

| Feature | VPN Gateway | ExpressRoute |
|---------|-------------|-------------|
| Connection type | IPsec over internet | Private MPLS circuit (not internet) |
| Bandwidth | Up to 1.25 Gbps | 50 Mbps to 100 Gbps |
| Latency | Variable (internet) | Consistent, predictable |
| SLA | 99.9% | 99.95% |
| Setup time | Minutes–hours | Weeks–months (physical circuit) |
| Cost | Low | High (circuit + gateway + provider) |
| Use case | Smaller workloads, dev/test | Enterprise, compliance, high-throughput |

---

### 9.2 ExpressRoute Architecture

```
On-Premises DC                 ExpressRoute Circuit
┌──────────────────┐           ┌─────────────────┐       Azure Region
│  Your Router     │           │  Microsoft Edge  │       ┌────────────────┐
│  (CE Router)     │◄─────────►│  Router (MSEE)  │◄─────►│  ExpressRoute  │
│                  │  Provider │                  │       │  Gateway       │
│  BGP Peer        │  (MPLS)   │  BGP Peer        │       │                │
└──────────────────┘           └─────────────────┘       │  VNet          │
                                                          └────────────────┘
```

**Peering types:**
- **Private Peering**: connect to Azure VNets (VMs, internal services)
- **Microsoft Peering**: connect to Microsoft 365, Azure public services (Storage, SQL, etc.)

---

### 9.3 When to Use ExpressRoute

Use ExpressRoute when:
- You need **consistent, low-latency** connectivity (trading systems, real-time analytics)
- Transferring **large amounts of data** regularly (>1 TB/month makes ExpressRoute cheaper than VPN with egress costs)
- **Compliance** requires data to not traverse the public internet
- Connecting to **Microsoft 365** services reliably
- Your organization already has an MPLS provider relationship

---

## 10. Azure DNS

### 10.1 Public DNS Zones

Azure DNS hosts your public DNS zones. It uses Azure's globally distributed name server infrastructure (anycast).

```bash
# Create public DNS zone
az network dns zone create \
  --resource-group myRG \
  --name "myapp.com"

# Get NS records (configure these at your domain registrar)
az network dns zone show \
  --resource-group myRG \
  --name "myapp.com" \
  --query "nameServers" \
  --output table

# Add A record
az network dns record-set a add-record \
  --resource-group myRG \
  --zone-name "myapp.com" \
  --record-set-name "www" \
  --ipv4-address 203.0.113.10
# www.myapp.com → 203.0.113.10

# Add CNAME record (alias)
az network dns record-set cname set-record \
  --resource-group myRG \
  --zone-name "myapp.com" \
  --record-set-name "api" \
  --cname "myapp.azurewebsites.net"
# api.myapp.com → myapp.azurewebsites.net

# Add MX record (email)
az network dns record-set mx add-record \
  --resource-group myRG \
  --zone-name "myapp.com" \
  --record-set-name "@" \
  --exchange "mail.protection.outlook.com" \
  --preference 0

# Add TXT record (SPF, verification)
az network dns record-set txt add-record \
  --resource-group myRG \
  --zone-name "myapp.com" \
  --record-set-name "@" \
  --value "v=spf1 include:spf.protection.outlook.com -all"

# Add AAAA record (IPv6)
az network dns record-set aaaa add-record \
  --resource-group myRG \
  --zone-name "myapp.com" \
  --record-set-name "www" \
  --ipv6-address "2001:db8::1"

# Add SRV record
az network dns record-set srv add-record \
  --resource-group myRG \
  --zone-name "myapp.com" \
  --record-set-name "_sip._tls" \
  --priority 100 \
  --weight 1 \
  --port 443 \
  --target "sipdir.online.lync.com"

# Set TTL on a record set
az network dns record-set a update \
  --resource-group myRG \
  --zone-name "myapp.com" \
  --name "www" \
  --set ttl=300                # 300 seconds (5 minutes)

# List all records in zone
az network dns record-set list \
  --resource-group myRG \
  --zone-name "myapp.com" \
  --output table
```

---

### 10.2 Private DNS Zones

Private DNS zones resolve hostnames within your VNet(s) without exposing DNS to the public internet.

```bash
# Create private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name "internal.myapp.com"

# Link to VNet (with auto-registration of VM hostnames)
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "internal.myapp.com" \
  --name myVnetLink \
  --virtual-network myVnet \
  --registration-enabled true

# VMs get auto-registered: webvm1.internal.myapp.com → 10.0.1.4

# Add manual A records for services
az network private-dns record-set a create \
  --resource-group myRG \
  --zone-name "internal.myapp.com" \
  --name "db"

az network private-dns record-set a add-record \
  --resource-group myRG \
  --zone-name "internal.myapp.com" \
  --record-set-name "db" \
  --ipv4-address "10.0.3.4"
# db.internal.myapp.com → 10.0.3.4 (resolves only within linked VNets)
```

---

### 10.3 Split-Horizon DNS

Split-horizon DNS means the same domain name resolves differently from inside vs outside the network. Used for Private Endpoints.

```
From the internet:
  myapp.blob.core.windows.net → 13.107.6.175 (public IP)

From inside VNet (with Private DNS Zone):
  myapp.blob.core.windows.net → 10.0.3.5 (private endpoint IP)
```

This is achieved automatically when you create a Private Endpoint and configure the associated Private DNS Zone.

---

## 11. Azure Private Link and Private Endpoints

### 11.1 What Is Private Link?

Azure Private Link enables you to access Azure PaaS services (Storage, SQL, Cosmos DB, Key Vault, etc.) and your own services over a private endpoint within your VNet.

```
Without Private Endpoint                 With Private Endpoint
────────────────────────                 ──────────────────────

VNet → (public internet) → Azure SQL     VNet → 10.0.3.5 (Private NIC) → Azure SQL
                                                 (stays in Azure backbone)

Firewall rules needed: VNet CIDR         No public endpoint needed
DNS: mydb.database.windows.net           DNS: mydb.database.windows.net
  resolves to 52.x.x.x (public)           resolves to 10.0.3.5 (private)
```

---

### 11.2 CLI Commands

```bash
# ============================================================
# PRIVATE ENDPOINT — AZURE SQL EXAMPLE
# ============================================================

# Disable private endpoint network policies on subnet (required)
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVnet \
  --name DataSubnet \
  --disable-private-endpoint-network-policies true

# Create Private Endpoint
az network private-endpoint create \
  --resource-group myRG \
  --name SqlPrivateEndpoint \
  --vnet-name myVnet \
  --subnet DataSubnet \
  --private-connection-resource-id "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Sql/servers/myserver" \
  --group-ids sqlServer \
  --connection-name SqlPEConnection \
  --location eastus

# Get the private endpoint's IP
az network private-endpoint show \
  --resource-group myRG \
  --name SqlPrivateEndpoint \
  --query "customDnsConfigs[0].ipAddresses[0]" \
  --output tsv
# Returns: 10.0.3.5

# Create private DNS zone for SQL
az network private-dns zone create \
  --resource-group myRG \
  --name "privatelink.database.windows.net"

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.database.windows.net" \
  --name SqlDNSLink \
  --virtual-network myVnet \
  --registration-enabled false   # no auto-registration for PaaS services

# Create DNS A record for private endpoint
az network private-dns record-set a create \
  --resource-group myRG \
  --zone-name "privatelink.database.windows.net" \
  --name "myserver"

az network private-dns record-set a add-record \
  --resource-group myRG \
  --zone-name "privatelink.database.windows.net" \
  --record-set-name "myserver" \
  --ipv4-address "10.0.3.5"
# Now: myserver.database.windows.net → 10.0.3.5 (within VNet)

# Disable public network access on the SQL server
az sql server update \
  --resource-group myRG \
  --name myserver \
  --set publicNetworkAccess="Disabled"

# List private endpoints in a resource group
az network private-endpoint list \
  --resource-group myRG \
  --output table

# Private DNS zones commonly needed by service:
# Azure SQL:         privatelink.database.windows.net
# Storage Blob:      privatelink.blob.core.windows.net
# Storage File:      privatelink.file.core.windows.net
# Key Vault:         privatelink.vaultcore.azure.net
# Cosmos DB:         privatelink.documents.azure.com
# Service Bus:       privatelink.servicebus.windows.net
# App Service:       privatelink.azurewebsites.net
# ACR:               privatelink.azurecr.io
# Redis Cache:       privatelink.redis.cache.windows.net
```

---

## 12. Network Watcher

### 12.1 What Is Network Watcher?

Azure Network Watcher is a collection of network monitoring, diagnostic, and visualization tools. It is a regional service — enable it per region.

```bash
# Enable Network Watcher (auto-enabled when VNet is created in most cases)
az network watcher configure \
  --resource-group NetworkWatcherRG \   # always uses this fixed RG
  --locations eastus \
  --enabled true
```

---

### 12.2 IP Flow Verify

Checks if traffic from a specific source IP/port is allowed or denied to a VM's NIC by NSG rules.

```bash
# Test if HTTP traffic from internet reaches VM
az network watcher test-ip-flow \
  --resource-group myRG \
  --vm WebVM1 \
  --nic WebVM1-nic \
  --direction Inbound \
  --protocol TCP \
  --local-ip 10.0.1.4 \
  --local-port 80 \
  --remote-ip 203.0.113.50 \
  --remote-port 12345
# Output: "Access": "Allow" or "Access": "Deny" + which rule caused it
```

---

### 12.3 Connection Troubleshoot

Tests end-to-end connectivity between a source (VM or IP) and a destination (VM, FQDN, IP, port).

```bash
# Test connectivity from VM to an external endpoint
az network watcher test-connectivity \
  --resource-group myRG \
  --source-resource WebVM1 \
  --dest-address "mydb.database.windows.net" \
  --dest-port 1433 \
  --protocol TCP

# Test VM to VM connectivity
az network watcher test-connectivity \
  --resource-group myRG \
  --source-resource WebVM1 \
  --dest-resource AppVM1 \
  --dest-port 8080 \
  --protocol TCP

# Output includes: connectionStatus, avgLatencyInMs, hops (with NSG and routing info)
```

---

### 12.4 Packet Capture

Captures packets on a VM's NIC to a Storage Account or local file for analysis in Wireshark.

```bash
# Start packet capture on a VM
az network watcher packet-capture create \
  --resource-group myRG \
  --vm WebVM1 \
  --name MyCapture \
  --storage-account mydiagnosticsstorage \
  --storage-path "https://mydiagnosticsstorage.blob.core.windows.net/captures" \
  --time-limit 60 \            # stop after 60 seconds
  --bytes-to-capture-per-packet 0 \  # 0 = capture full packets
  --filters '[{"protocol":"TCP","remoteIPAddress":"*","localIPAddress":"*","localPort":"80","remotePort":"*"}]'

# Stop capture
az network watcher packet-capture stop \
  --resource-group myRG \
  --vm WebVM1 \
  --name MyCapture

# List captures
az network watcher packet-capture list \
  --resource-group myRG \
  --vm WebVM1
```

---

### 12.5 Topology View

Generates a visual topology of resources in a VNet/resource group.

```bash
# Get network topology (JSON representation)
az network watcher show-topology \
  --resource-group myRG \
  --location eastus \
  --output json | python3 -m json.tool
```

---

### 12.6 NSG Flow Logs (via Network Watcher)

```bash
# List all flow log configurations
az network watcher flow-log list \
  --location eastus \
  --output table

# Show specific flow log
az network watcher flow-log show \
  --resource-group myRG \
  --name myNSGFlowLog \
  --location eastus

# Query flow logs in Log Analytics
# (after enabling Traffic Analytics)
# KQL Query:
# AzureNetworkAnalytics_CL
# | where SubType_s == "FlowLog"
# | where FlowStatus_s == "D"   // Denied flows
# | project TimeGenerated, SrcIP_s, DestIP_s, DestPort_d, NSGName_s
# | order by TimeGenerated desc
```

---

## 13. Networking Architecture Patterns

### 13.1 Hub-and-Spoke Topology

The hub-and-spoke (or hub-and-spoke mesh) is the most common enterprise Azure networking pattern. A central hub VNet contains shared services (firewall, VPN gateway, DNS) and spoke VNets connect to it via VNet Peering.

```
                        ┌─────────────────────────────┐
                        │         HUB VNet              │
                        │         10.0.0.0/16           │
                        │                               │
                        │  ┌────────────┐               │
  On-premises ◄──────── │  │ VPN Gateway│               │
  (VPN/ER)              │  │ GatewaySubnet               │
                        │  └────────────┘               │
                        │  ┌────────────┐               │
                        │  │Azure       │               │
                        │  │Firewall    │               │
                        │  └────────────┘               │
                        │  ┌────────────┐               │
                        │  │Azure       │               │
                        │  │Bastion     │               │
                        │  └────────────┘               │
                        └──────────┬──────────────────--┘
                                   │  VNet Peering
              ┌────────────────────┼────────────────────┐
              │                    │                    │
   ┌──────────▼──────┐  ┌──────────▼──────┐  ┌─────────▼───────┐
   │  Spoke 1 VNet   │  │  Spoke 2 VNet   │  │  Spoke 3 VNet   │
   │  10.1.0.0/16    │  │  10.2.0.0/16    │  │  10.3.0.0/16    │
   │  (Dev)          │  │  (Production)   │  │  (Shared Svcs)  │
   └─────────────────┘  └─────────────────┘  └─────────────────┘
```

**Hub VNet contains:**
- VPN Gateway or ExpressRoute Gateway (on-premises connectivity)
- Azure Firewall (central egress and inter-spoke traffic control)
- Azure Bastion (secure VM access without public IPs)
- Azure DNS Private Resolver (conditional forwarding)
- Shared services (monitoring, Key Vault, ACR)

**Spoke VNets contain:**
- Application workloads
- Peered to hub; can optionally use hub's gateway (gateway transit)

```bash
# Hub VNet setup
az network vnet create \
  --resource-group HubRG \
  --name HubVnet \
  --address-prefixes 10.0.0.0/16 \
  --location eastus

# Create hub subnets
az network vnet subnet create \
  --resource-group HubRG \
  --vnet-name HubVnet \
  --name GatewaySubnet \
  --address-prefixes 10.0.255.0/27

az network vnet subnet create \
  --resource-group HubRG \
  --vnet-name HubVnet \
  --name AzureFirewallSubnet \
  --address-prefixes 10.0.100.0/26

az network vnet subnet create \
  --resource-group HubRG \
  --vnet-name HubVnet \
  --name AzureBastionSubnet \     # exact name required
  --address-prefixes 10.0.200.0/26

# Spoke VNet setup
az network vnet create \
  --resource-group Spoke1RG \
  --name Spoke1Vnet \
  --address-prefixes 10.1.0.0/16 \
  --location eastus

# Peer Spoke to Hub (allow gateway transit)
HUB_ID=$(az network vnet show \
  --resource-group HubRG \
  --name HubVnet \
  --query id --output tsv)

az network vnet peering create \
  --resource-group Spoke1RG \
  --name Spoke1-to-Hub \
  --vnet-name Spoke1Vnet \
  --remote-vnet $HUB_ID \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --use-remote-gateways true   # use hub gateway for on-prem connectivity

az network vnet peering create \
  --resource-group HubRG \
  --name Hub-to-Spoke1 \
  --vnet-name HubVnet \
  --remote-vnet $(az network vnet show -g Spoke1RG -n Spoke1Vnet --query id -o tsv) \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --allow-gateway-transit true   # hub shares its gateway
```

---

### 13.2 Azure Virtual WAN

Virtual WAN is a Microsoft-managed hub-and-spoke network for large enterprises with many branches and spoke VNets. It automates connectivity between branches, spokes, and Azure services.

```
                         ┌──────────────────────────────────┐
                         │      Virtual WAN                  │
                         │  ┌──────────────────────────────┐│
                         │  │    vHub East US               ││
                         │  │    (Managed by Azure)         ││
                         │  │  ┌──────┐  ┌──────────────┐  ││
Branch Office A ─────────┼──┼─►│S2S   │  │ ExpressRoute │  ││
(S2S VPN)                │  │  │VPN   │  │ Gateway      │◄─┼┼─── HQ (ER)
                         │  │  └──────┘  └──────────────┘  ││
Remote Users ────────────┼──┼─►┌──────┐                    ││
(P2S VPN)                │  │  │P2S   │                    ││
                         │  │  └──────┘                    ││
                         │  └──────────────────────────────┘│
                         └────────────┬─────────────────────┘
                                      │
               ┌──────────────────────┼────────────────────────┐
               │                      │                        │
    ┌──────────▼──────┐     ┌─────────▼───────┐    ┌──────────▼──────┐
    │  Spoke VNet 1   │     │  Spoke VNet 2   │    │  Spoke VNet 3   │
    └─────────────────┘     └─────────────────┘    └─────────────────┘
```

**When to use Virtual WAN over classic hub-spoke:**
- Many branches (>10 sites) connecting via VPN
- Automated branch-to-branch and branch-to-spoke routing needed
- Global scale with multiple Azure regions
- Want Microsoft to manage the hub infrastructure

---

### 13.3 DMZ (Perimeter Network) Pattern

A DMZ isolates internet-facing workloads from internal systems using multiple security layers.

```
Internet
    │
    ▼
┌──────────────────────────────────────────────────────────────┐
│  DMZ Subnet (10.0.0.0/24)                                    │
│  NSG: Allow 80/443 inbound, deny all else                    │
│  ┌──────────────────────────────────┐                        │
│  │  Azure Application Gateway (WAF) │                        │
│  └──────────────────┬───────────────┘                        │
└─────────────────────┼────────────────────────────────────────┘
                      │ (Inspected HTTP/S traffic)
                      ▼
┌──────────────────────────────────────────────────────────────┐
│  Web Tier Subnet (10.0.1.0/24)                               │
│  NSG: Allow 80/443 from AppGateway subnet only               │
│  ┌──────────────────────────────────┐                        │
│  │  Web Server VMs / App Service    │                        │
│  └──────────────────┬───────────────┘                        │
└─────────────────────┼────────────────────────────────────────┘
                      │ (API calls to backend)
                      ▼
┌──────────────────────────────────────────────────────────────┐
│  App Tier Subnet (10.0.2.0/24)                               │
│  NSG: Allow 8080 from Web subnet only                        │
│  ┌──────────────────────────────────┐                        │
│  │  API / Business Logic VMs        │                        │
│  └──────────────────┬───────────────┘                        │
└─────────────────────┼────────────────────────────────────────┘
                      │ (Database queries)
                      ▼
┌──────────────────────────────────────────────────────────────┐
│  Data Tier Subnet (10.0.3.0/24)                              │
│  NSG: Allow 1433/5432 from App subnet only                   │
│  ┌──────────────────────────────────┐                        │
│  │  SQL / PostgreSQL / Redis        │                        │
│  └──────────────────────────────────┘                        │
└──────────────────────────────────────────────────────────────┘
```

---

### 13.4 Zero-Trust Networking

Zero Trust assumes breach and verifies every request as though it originates from an untrusted network.

```
Zero Trust Principles Applied to Azure Networking
──────────────────────────────────────────────────

1. VERIFY EXPLICITLY
   - Azure AD authentication for all service-to-service calls
   - Managed Identity for VMs/App Services (no passwords)
   - Conditional Access policies (require MFA, compliant device)

2. USE LEAST PRIVILEGE ACCESS
   - NSGs with deny-all defaults, minimal allow rules
   - Azure RBAC for network resource management
   - Private Endpoints (no public exposure)
   - JIT VM access via Azure Defender for Servers

3. ASSUME BREACH
   - Azure Firewall + NSG for defense-in-depth
   - NSG Flow Logs + Traffic Analytics for detection
   - Microsoft Defender for Cloud for threat detection
   - Network Watcher for forensic analysis

Architecture:
┌─────────────────────────────────────────────────────┐
│  Zero Trust Network Architecture                    │
│                                                     │
│  ┌──────────────┐    Verify Identity                │
│  │  User/Device │────────────────────►Azure AD      │
│  └──────┬───────┘    (Conditional Access)           │
│         │                                           │
│         │  Inspect Traffic                          │
│         ▼                                           │
│  ┌──────────────┐    All traffic inspected          │
│  │ Azure        │◄── by Azure Firewall               │
│  │ Firewall     │    Premium (IDPS, TLS inspection) │
│  └──────┬───────┘                                   │
│         │                                           │
│         │  Micro-segmentation                       │
│         ▼                                           │
│  ┌──────────────┐    NSGs on each subnet/NIC        │
│  │  Workload    │    Private Endpoints for PaaS      │
│  │  VMs/Apps    │    No public IPs on workloads      │
│  └──────────────┘                                   │
└─────────────────────────────────────────────────────┘
```

---

### 13.5 Service Decision Table

#### Which Load Balancer to Use?

| Scenario | Service |
|----------|---------|
| Load balance HTTP/S traffic from internet, need WAF | **Application Gateway WAF_v2** |
| Global load balancing + CDN + WAF | **Azure Front Door Standard/Premium** |
| Load balance non-HTTP traffic (TCP/UDP) from internet | **Azure Load Balancer (Standard, Public)** |
| Load balance between internal services (microservices) | **Azure Load Balancer (Standard, Internal)** |
| Route traffic to different regional endpoints based on latency/geography | **Azure Front Door** or **Traffic Manager** |
| Route non-HTTP global traffic (SMTP, SQL, etc.) | **Traffic Manager** (DNS-based) |

#### Which Connectivity Option?

| Scenario | Service |
|----------|---------|
| < 10 remote users, VPN over internet acceptable | **VPN Gateway (P2S)** |
| Branch office to Azure, internet VPN is OK | **VPN Gateway (S2S)** |
| Predictable low latency, large data transfer, compliance | **ExpressRoute** |
| Many branches, global scale, auto-managed routing | **Virtual WAN** |
| VNet-to-VNet same region, low cost | **VNet Peering** |
| VNet-to-VNet across regions | **Global VNet Peering** |

#### Pricing Summary (Approximate, East US)

| Service | Approximate Cost |
|---------|-----------------|
| VNet | Free (pay for resources within) |
| VNet Peering | $0.01/GB transferred (same region); $0.035/GB (cross-region) |
| NSG | Free |
| Azure Firewall Standard | ~$1.25/hr + $0.016/GB processed |
| Azure Firewall Premium | ~$3.50/hr + $0.016/GB processed |
| Load Balancer Standard | $0.008/hr + $0.005/GB after 5 GB free |
| Application Gateway WAF_v2 | ~$0.126/hr (fixed) + $0.008/capacity unit-hr |
| Front Door Standard | $35/mo + $0.013/GB |
| Front Door Premium | $330/mo + $0.013/GB |
| VPN Gateway VpnGw1 | ~$0.19/hr (~$138/mo) |
| ExpressRoute 1 Gbps circuit | ~$1,500–$5,000/mo (depends on provider) |
| Private Endpoint | ~$0.01/hr per endpoint + $0.01/GB |
| Azure DNS | $0.50/zone/mo + $0.40/M queries |
| Network Watcher | Packet capture: $0.10/capture-hr; Flow logs: ingestion costs |

---

### 13.6 Complete Hub-and-Spoke Bicep Template

```bicep
// hub-spoke-network.bicep
param location string = resourceGroup().location

// ───── HUB VNET ─────
resource hubVnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: 'HubVnet'
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.0.0.0/16'] }
    subnets: [
      { name: 'GatewaySubnet',       properties: { addressPrefix: '10.0.255.0/27' } }
      { name: 'AzureFirewallSubnet',  properties: { addressPrefix: '10.0.100.0/26' } }
      { name: 'AzureBastionSubnet',   properties: { addressPrefix: '10.0.200.0/26' } }
      { name: 'SharedServicesSubnet', properties: { addressPrefix: '10.0.1.0/24' } }
    ]
  }
}

// ───── SPOKE VNETS ─────
resource spoke1Vnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: 'Spoke1Vnet'
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.1.0.0/16'] }
    subnets: [
      { name: 'WebSubnet', properties: { addressPrefix: '10.1.1.0/24' } }
      { name: 'AppSubnet', properties: { addressPrefix: '10.1.2.0/24' } }
    ]
  }
}

// ───── PEERINGS ─────
resource hubToSpoke1 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2023-09-01' = {
  parent: hubVnet
  name: 'Hub-to-Spoke1'
  properties: {
    remoteVirtualNetwork: { id: spoke1Vnet.id }
    allowVirtualNetworkAccess: true
    allowForwardedTraffic: true
    allowGatewayTransit: true   // hub shares gateway with spoke
    useRemoteGateways: false
  }
}

resource spoke1ToHub 'Microsoft.Network/virtualNetworks/virtualNetworkPeerings@2023-09-01' = {
  parent: spoke1Vnet
  name: 'Spoke1-to-Hub'
  properties: {
    remoteVirtualNetwork: { id: hubVnet.id }
    allowVirtualNetworkAccess: true
    allowForwardedTraffic: true
    allowGatewayTransit: false
    useRemoteGateways: true     // spoke uses hub's gateway
  }
  dependsOn: [ hubToSpoke1 ]
}

// ───── NSG FOR WEB SUBNET ─────
resource webNsg 'Microsoft.Network/networkSecurityGroups@2023-09-01' = {
  name: 'WebSubnetNSG'
  location: location
  properties: {
    securityRules: [
      {
        name: 'Allow-HTTPS-Inbound'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '443'
        }
      }
      {
        name: 'Allow-HTTP-Inbound'
        properties: {
          priority: 110
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '80'
        }
      }
      {
        name: 'Deny-All-Inbound'
        properties: {
          priority: 4000
          direction: 'Inbound'
          access: 'Deny'
          protocol: '*'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '*'
        }
      }
    ]
  }
}

// ───── PRIVATE DNS ZONE ─────
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'internal.myapp.com'
  location: 'global'
}

resource hubVnetDnsLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZone
  name: 'HubVnetLink'
  location: 'global'
  properties: {
    virtualNetwork: { id: hubVnet.id }
    registrationEnabled: true
  }
}

resource spoke1VnetDnsLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZone
  name: 'Spoke1VnetLink'
  location: 'global'
  properties: {
    virtualNetwork: { id: spoke1Vnet.id }
    registrationEnabled: true
  }
}

output hubVnetId string = hubVnet.id
output spoke1VnetId string = spoke1Vnet.id
```

---

*Last updated: 2024 | Part of the Azure Zero-to-Hero Learning Series*
*Previous: [12-AZURE-DATABASE-SERVICES.md](./12-AZURE-DATABASE-SERVICES.md)*
