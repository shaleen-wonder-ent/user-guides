# AMPLS Multi-Region Component Placement Guide
## Where to Place What - Detailed Deployment Strategy


---

## 📚 Table of Contents

- [Overview](#overview)
- [Hub vs Spoke Decision Matrix](#hub-vs-spoke-decision-matrix)
- [Regional Placement Strategy](#regional-placement-strategy)
- [Component-by-Component Placement](#component-by-component-placement)
- [Subnet Design and Allocation](#subnet-design-and-allocation)
- [Resource Group Strategy](#resource-group-strategy)
- [Naming Conventions](#naming-conventions)
- [Step-by-Step Deployment Order](#step-by-step-deployment-order)
- [Common Mistakes to Avoid](#common-mistakes-to-avoid)

---

## Overview

### The Central Question: Hub or Spoke?

```yaml
Decision Criteria:

Place in HUB if:
   Shared across multiple workloads
   Provides connectivity services (VPN/ExpressRoute)
   Centralized security control point
   Network routing/firewall function
  
Place in SPOKE if:
   Workload-specific service
   Isolated security boundary needed
   Independent lifecycle from other services
   Requires separate RBAC/governance
```

### Architecture Philosophy

<img width="800" height="389" alt="image" src="https://github.com/user-attachments/assets/12db3c23-b532-4078-a9d5-bb2a574d89ba" />


---

## Hub vs Spoke Decision Matrix

### Quick Reference Table

| Component | Placement | Reasoning | VNet/Subnet |
|-----------|-----------|-----------|-------------|
| **ExpressRoute Gateway** |  Hub | Shared connectivity | vnet-hub-XX / GatewaySubnet |
| **VPN Gateway** |  Hub | Shared connectivity | vnet-hub-XX / GatewaySubnet |
| **Azure Firewall** |  Hub | Centralized security | vnet-hub-XX / AzureFirewallSubnet |
| **DNS Forwarder VMs** |  Spoke (Monitor) | Workload-specific DNS | vnet-spoke-monitor-XX / snet-dns |
| **Private Endpoints (AMPLS)** |  Spoke (Monitor) | Service-specific isolation | vnet-spoke-monitor-XX / snet-pe |
| **Private DNS Zones** |  Spoke (Monitor) | Linked to spoke VNets | Global resource, linked to spoke |
| **AMPLS** |  Spoke (Monitor) | Monitoring service boundary | Global resource, RG in region |
| **Log Analytics Workspace** |  Spoke (Monitor) | Data residency per region | Regional resource |
| **Application Workloads** |  Spoke (App) | Separate from monitoring | vnet-spoke-app-XX |

### Detailed Decision Flow

<img width="2165" height="1560" alt="mermaid-diagram-2025-11-19-212856" src="https://github.com/user-attachments/assets/0f00ef0b-2f10-4057-b5cb-c373033b8df7" />


---

## Regional Placement Strategy

### Multi-Region Design Principles

```yaml
Regional Placement Rules:

1. Data Residency:
   - LAW: Must be in region where data is collected
   - Example: CI VMs → LAW-CI (Central India)
   
2. Latency Optimization:
   - Private Endpoints: Same region as LAW
   - DNS VMs: Same region as Private Endpoints
   
3. Disaster Recovery:
   - Each region: Independent monitoring infrastructure
   - No cross-dependencies for critical path
   
4. Cost Optimization:
   - Avoid cross-region data transfer
   - Use regional Private Endpoints (no egress)
   
5. Compliance:
   - Data doesn't leave region (if required)
   - Private Link ensures Azure backbone only
```

### Regional Architecture Layout

```
┌───────────────────────────────────────────────────────────────┐
│                        CENTRAL INDIA REGION                   │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   HUB VNET (10.0.0.0/16)                │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │  GatewaySubnet: 10.0.0.0/24                        │ │  │
│  │  │  - ExpressRoute Gateway                            │ │  │
│  │  │  - VPN Gateway (optional)                          │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │  AzureFirewallSubnet: 10.0.1.0/24                  │ │  │
│  │  │  - Azure Firewall / NVA                            │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │  Management Subnet: 10.0.10.0/24                   │ │  │
│  │  │  - Bastion, Jump Boxes                             │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────┘  │
│                              ↕ VNet Peering                   │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │           SPOKE VNET - MONITORING (10.1.0.0/16)         │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │  DNS Subnet: 10.1.0.0/24                           │ │  │
│  │  │  - vm-dns-ci-01: 10.1.0.4                          │ │  │
│  │  │  - vm-dns-ci-02: 10.1.0.5                          │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────┐ │  │
│  │  │  Private Endpoint Subnet: 10.1.100.0/24            │ │  │
│  │  │  - pe-ampls-ci: 10.1.100.50                        │ │  │
│  │  └────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              MONITORING RESOURCES (Global/Regional)     │  │
│  │  - AMPLS-CI (Global resource, RG in CI)                 │  │
│  │  - LAW-CI (Central India)                               │  │
│  │  - Private DNS Zones (Global, linked to spoke-CI)       │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                        SOUTH INDIA REGION                       │
├─────────────────────────────────────────────────────────────────┤
│  [Same structure as Central India, with 10.2.x.x addressing]    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component-by-Component Placement

### 1. ExpressRoute / VPN Gateway

**Placement: HUB VNet**

```yaml
Component: ExpressRoute Gateway
Location: GatewaySubnet in Hub VNet
Subnet: Must be named "GatewaySubnet" (Azure requirement)

Why Hub?:
   Shared by all spokes via peering
   Centralized connectivity management
   Single point for on-premises routing
   Cost-effective (one gateway serves all)

Deployment Details:
  VNet: vnet-hub-ci (10.0.0.0/16)
  Subnet: GatewaySubnet (10.0.0.0/24)
  SKU: ErGw1Az (or higher for production)
  Zone Redundancy: Enabled (recommended)
  
Configuration:
  - Create in Hub VNet first (before spokes)
  - Configure BGP peering with on-premises
  - Enable FastPath for lower latency (optional)
  
Peering Settings:
  - In spoke peering: "Use remote gateway" = Enabled
  - In hub peering: "Allow gateway transit" = Enabled
```

**Terraform Example:**
```hcl
# Hub VNet
resource "azurerm_virtual_network" "hub_ci" {
  name                = "vnet-hub-ci"
  location            = "centralindia"
  resource_group_name = azurerm_resource_group.hub_ci.name
  address_space       = ["10.0.0.0/16"]
}

# Gateway Subnet (required name)
resource "azurerm_subnet" "gateway_ci" {
  name                 = "GatewaySubnet"
  resource_group_name  = azurerm_resource_group.hub_ci.name
  virtual_network_name = azurerm_virtual_network.hub_ci.name
  address_prefixes     = ["10.0.0.0/24"]
}

# ExpressRoute Gateway
resource "azurerm_virtual_network_gateway" "ergw_ci" {
  name                = "ergw-hub-ci"
  location            = "centralindia"
  resource_group_name = azurerm_resource_group.hub_ci.name
  type                = "ExpressRoute"
  sku                 = "ErGw1Az"
  
  ip_configuration {
    public_ip_address_id          = azurerm_public_ip.ergw_ci.id
    private_ip_address_allocation = "Dynamic"
    subnet_id                     = azurerm_subnet.gateway_ci.id
  }
}
```

### 2. Azure Firewall / Network Virtual Appliance

**Placement: HUB VNet**

```yaml
Component: Azure Firewall
Location: AzureFirewallSubnet in Hub VNet
Subnet: Must be named "AzureFirewallSubnet" (if using Azure Firewall)

Why Hub?:
   Centralized traffic inspection
   East-West and North-South filtering
   Unified logging and monitoring
   Consistent security policies

Deployment Details:
  VNet: vnet-hub-ci (10.0.0.0/16)
  Subnet: AzureFirewallSubnet (10.0.1.0/24)
  SKU: Standard or Premium
  
Use Cases:
  - Inspect spoke-to-spoke traffic (optional)
  - Filter outbound internet (if needed)
  - Log all traffic flows
  
Route Tables:
  Spoke Subnets → 0.0.0.0/0 → Azure Firewall IP
  (Except GatewaySubnet and AzureFirewallSubnet)
```

**Note:** For AMPLS architecture, Azure Firewall is **optional** since:
- Private Link traffic stays on Azure backbone
- No internet breakout needed for monitoring
- Can simplify by removing firewall (cost savings)

### 3. Custom DNS VMs

**Placement: SPOKE VNet (Monitoring)**

```yaml
Component: Custom DNS Virtual Machines
Location: Dedicated DNS subnet in Monitoring Spoke
Subnet: snet-dns-ci (10.1.0.0/24)

Why Spoke (not Hub)?:
   Workload-specific (monitoring DNS resolution)
   Isolated from general connectivity services
   Easier to manage lifecycle independently
   Can be scaled per monitoring requirements
   Note: Some architectures place DNS in Hub (also valid)

Deployment Details:
  Region: Central India
  VNet: vnet-spoke-monitor-ci (10.1.0.0/16)
  Subnet: snet-dns-ci (10.1.0.0/24)
  
  VM1: vm-dns-ci-01
    - IP: 10.1.0.4 (Static)
    - Size: Standard_B2s
    - OS: Windows Server 2022
    - Role: DNS Server
    
  VM2: vm-dns-ci-02
    - IP: 10.1.0.5 (Static)
    - Size: Standard_B2s
    - OS: Windows Server 2022
    - Role: DNS Server (replica)

Configuration:
  - Install DNS Server role
  - Configure forwarder: 168.63.129.16 (Azure DNS)
  - NO conditional forwarders to other regions (precise forwarding)
  - High availability via 2 VMs

VNet DNS Settings:
  - vnet-spoke-monitor-ci → DNS Servers: 10.1.0.4, 10.1.0.5
```

**Why NOT in Hub?**
```yaml
Considered but rejected:
   DNS VMs in hub mixes connectivity with workload services
   Harder to apply monitoring-specific RBAC
   Lifecycle tied to hub changes (harder to update)
   Spoke placement: Clear separation of concerns
```

**Terraform Example:**
```hcl
# DNS Subnet
resource "azurerm_subnet" "dns_ci" {
  name                 = "snet-dns-ci"
  resource_group_name  = azurerm_resource_group.monitor_ci.name
  virtual_network_name = azurerm_virtual_network.spoke_monitor_ci.name
  address_prefixes     = ["10.1.0.0/24"]
}

# DNS VM 1
resource "azurerm_windows_virtual_machine" "dns_ci_01" {
  name                = "vm-dns-ci-01"
  location            = "centralindia"
  resource_group_name = azurerm_resource_group.monitor_ci.name
  size                = "Standard_B2s"
  admin_username      = "azureadmin"
  
  network_interface_ids = [azurerm_network_interface.dns_ci_01.id]
  
  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
  }
  
  source_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2022-Datacenter"
    version   = "latest"
  }
}

# Network Interface with Static IP
resource "azurerm_network_interface" "dns_ci_01" {
  name                = "nic-dns-ci-01"
  location            = "centralindia"
  resource_group_name = azurerm_resource_group.monitor_ci.name
  
  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.dns_ci.id
    private_ip_address_allocation = "Static"
    private_ip_address            = "10.1.0.4"
  }
}
```

### 4. Private Endpoints (for AMPLS)

**Placement: SPOKE VNet (Monitoring)**

```yaml
Component: Private Endpoint for AMPLS
Location: Dedicated PE subnet in Monitoring Spoke
Subnet: snet-pe-monitor-ci (10.1.100.0/24)

Why Spoke?:
   Service-specific isolation
   Clear security boundary for monitoring
   Independent NSG rules for AMPLS access
   Scales with monitoring workload, not hub

Deployment Details:
  Region: Central India
  VNet: vnet-spoke-monitor-ci (10.1.0.0/16)
  Subnet: snet-pe-monitor-ci (10.1.100.0/24)
  
  Private Endpoint:
    Name: pe-ampls-ci
    IP: 10.1.100.50 (auto-assigned)
    Connected to: AMPLS-CI
    Subresource: azuremonitor

Subnet Configuration:
  CRITICAL: privateEndpointNetworkPolicies = "Disabled"
  Reason: Required for Private Endpoints to function

Private DNS Zone Group:
  Automatically creates A records in:
    - privatelink.monitor.azure.com
    - privatelink.oms.opinsights.azure.com
    - privatelink.ods.opinsights.azure.com
    - privatelink.agentsvc.azure-automation.net
```

**Terraform Example:**
```hcl
# Private Endpoint Subnet
resource "azurerm_subnet" "pe_monitor_ci" {
  name                 = "snet-pe-monitor-ci"
  resource_group_name  = azurerm_resource_group.monitor_ci.name
  virtual_network_name = azurerm_virtual_network.spoke_monitor_ci.name
  address_prefixes     = ["10.1.100.0/24"]
  
  # CRITICAL: Disable network policies for PE
  private_endpoint_network_policies_enabled = false
}

# Private Endpoint
resource "azurerm_private_endpoint" "ampls_ci" {
  name                = "pe-ampls-ci"
  location            = "centralindia"
  resource_group_name = azurerm_resource_group.monitor_ci.name
  subnet_id           = azurerm_subnet.pe_monitor_ci.id
  
  private_service_connection {
    name                           = "psc-ampls-ci"
    private_connection_resource_id = azurerm_monitor_private_link_scope.ci.id
    subresource_names              = ["azuremonitor"]
    is_manual_connection           = false
  }
  
  private_dns_zone_group {
    name                 = "pdzg-ampls-ci"
    private_dns_zone_ids = [
      azurerm_private_dns_zone.monitor.id,
      azurerm_private_dns_zone.oms.id,
      azurerm_private_dns_zone.ods.id,
      azurerm_private_dns_zone.agentsvc.id
    ]
  }
}
```

### 5. Private DNS Zones

**Placement: Global resource, linked to SPOKE VNet**

```yaml
Component: Private DNS Zones
Scope: Global resource (not region-specific)
Resource Group: rg-monitor-ci (for management)

Why Linked to Spoke?:
   Zones resolve for resources in spoke
   Auto-registration works with PE in spoke
   Spoke VNet DNS VMs can query zones
   Could link to hub (also valid), but spoke is cleaner

Deployment Details:
  Zones to Create (per region):
    1. privatelink.monitor.azure.com
    2. privatelink.oms.opinsights.azure.com
    3. privatelink.ods.opinsights.azure.com
    4. privatelink.agentsvc.azure-automation.net
  
  VNet Links (CI example):
    - vnet-spoke-monitor-ci
    - Auto-registration: Enabled
    
  DNS Records (auto-created by PE):
    - <workspace-id>.oms.opinsights.azure.com → 10.1.100.50
    - <workspace-id>.ods.opinsights.azure.com → 10.1.100.50
    - api.monitor.azure.com → 10.1.100.50
```

**Important Notes:**
```yaml
Common Questions:

Q: Should I link zones to Hub or Spoke?
A: Link to Spoke (Monitoring)
   - Reason: DNS VMs are in spoke
   - Reason: Private Endpoints are in spoke
   - Result: Direct resolution path

Q: What if I link to both Hub and Spoke?
A: It works, but unnecessary
   - Adds complexity
   - No benefit for this architecture
   
Q: Do I need separate zones per region?
A: YES - Each region has its own zones
   - Reason: Different Private Endpoint IPs
   - CI zones: resolve to 10.1.100.50
   - SI zones: resolve to 10.2.100.50
```

**Terraform Example:**
```hcl
# Private DNS Zone
resource "azurerm_private_dns_zone" "monitor_ci" {
  name                = "privatelink.monitor.azure.com"
  resource_group_name = azurerm_resource_group.monitor_ci.name
}

# Link to Spoke VNet
resource "azurerm_private_dns_zone_virtual_network_link" "monitor_ci" {
  name                  = "link-monitor-ci-spoke"
  resource_group_name   = azurerm_resource_group.monitor_ci.name
  private_dns_zone_name = azurerm_private_dns_zone.monitor_ci.name
  virtual_network_id    = azurerm_virtual_network.spoke_monitor_ci.id
  registration_enabled  = true
}
```

### 6. Azure Monitor Private Link Scope (AMPLS)

**Placement: Global resource, Resource Group in region**

```yaml
Component: AMPLS
Scope: Global (not tied to specific region)
Resource Group: rg-monitor-ci (regional RG for management)

Why Regional RG?:
   Easier to manage regionally grouped resources
   RBAC per region if needed
   Cost tracking per region
   AMPLS itself is global, RG is just for organization

Deployment Details:
  Name: ampls-ci
  Resource Group: rg-monitor-ci
  Access Mode: Private Only (recommended)
  
  Linked Resources:
    - LAW-CI (Central India)
    - Application Insights CI (if applicable)
  
  Private Endpoint:
    - pe-ampls-ci (10.1.100.50 in Spoke CI)

Configuration:
  Query Access Mode:
    - Open: Allows both private and public (NOT recommended)
    - Private Only: Forces all access via PE (recommended)
  
  Ingestion Access Mode:
    - Open: Allows both
    - Private Only: Forces via PE
```

**Terraform Example:**
```hcl
# AMPLS
resource "azurerm_monitor_private_link_scope" "ci" {
  name                = "ampls-ci"
  resource_group_name = azurerm_resource_group.monitor_ci.name
}

# Link LAW to AMPLS
resource "azurerm_monitor_private_link_scoped_service" "law_ci" {
  name                = "law-ci-scoped"
  resource_group_name = azurerm_resource_group.monitor_ci.name
  scope_name          = azurerm_monitor_private_link_scope.ci.name
  linked_resource_id  = azurerm_log_analytics_workspace.ci.id
}
```

### 7. Log Analytics Workspace (LAW)

**Placement: Regional resource in monitoring Resource Group**

```yaml
Component: Log Analytics Workspace
Location: MUST be in the region where data is collected
Resource Group: rg-monitor-ci

Why Regional?:
   Data residency requirements
   Lower latency for ingestion
   Compliance (data doesn't leave region)
   Cost optimization (no cross-region egress)

Deployment Details:
  Name: law-ci
  Location: Central India
  Resource Group: rg-monitor-ci
  SKU: PerGB2018
  Retention: 30 days (default)
  
  Network Isolation:
    - Public Network Access: Disabled
    - Access via: AMPLS only
  
  Connected to:
    - AMPLS-CI (via Private Link Scoped Service)

Data Sources (examples):
  - Azure VMs in Central India
  - AKS clusters in Central India
  - Application Insights in Central India
```

**Terraform Example:**
```hcl
resource "azurerm_log_analytics_workspace" "ci" {
  name                = "law-ci-${random_string.suffix.result}"
  location            = "centralindia"
  resource_group_name = azurerm_resource_group.monitor_ci.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  
  # Disable public access (force via AMPLS)
  internet_ingestion_enabled = false
  internet_query_enabled     = false
}
```

### 8. Application Workloads (Optional)

**Placement: Separate SPOKE VNet**

```yaml
Component: Application VMs, AKS, etc.
Location: Dedicated application spoke
VNet: vnet-spoke-app-ci (10.1.10.0/16)

Why Separate Spoke?:
   Isolation from monitoring infrastructure
   Independent security policies
   Separate RBAC for app teams
   Can delete/recreate without affecting monitoring

Deployment Example:
  VNet: vnet-spoke-app-ci (10.1.10.0/16)
  Subnets:
    - snet-web: 10.1.10.0/24 (web tier)
    - snet-app: 10.1.11.0/24 (app tier)
    - snet-data: 10.1.12.0/24 (data tier)
  
  Monitoring Agent Configuration:
    - Workspace: LAW-CI
    - Endpoint: via Private Endpoint (10.1.100.50)
    - DNS: Uses VNet DNS (10.1.0.4, 10.1.0.5)
```

---

## Subnet Design and Allocation

### Subnet Sizing Guide

```yaml
Subnet Sizing Principles:

1. Gateway Subnet (/24):
   - Azure requirement: /27 minimum
   - Recommendation: /24 for future growth
   - Reason: Multiple gateway instances, updates

2. Firewall Subnet (/24 or /26):
   - Azure Firewall: /26 minimum
   - Recommendation: /24 for scale
   - Reason: Multiple firewall instances

3. DNS Subnet (/27 or /28):
   - 2 DNS VMs = 2 IPs
   - Recommendation: /28 (14 usable IPs)
   - Room for: 10+ DNS VMs if needed

4. Private Endpoint Subnet (/27 or /26):
   - 1 PE per service = 1 IP
   - AMPLS: 1 IP
   - Storage: 1 IP each
   - Recommendation: /27 (30 usable IPs)
   - Room for: 20+ different services

5. Application Subnets (varies):
   - Small: /27 (30 IPs)
   - Medium: /24 (254 IPs)
   - Large: /22 (1022 IPs)
```

### Complete IP Allocation Plan - Central India

```
┌──────────────────────────────────────────────────────────────┐
│                  CENTRAL INDIA - IP PLAN                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  HUB VNET: 10.0.0.0/16                                       │
│  ├─ GatewaySubnet:         10.0.0.0/24   (256 IPs)           │
│  ├─ AzureFirewallSubnet:   10.0.1.0/24   (256 IPs)           │
│  ├─ ManagementSubnet:      10.0.10.0/24  (256 IPs)           │
│  └─ Reserved:              10.0.11.0/24 - 10.0.255.0/24      │
│                                                              │
│  SPOKE VNET (Monitoring): 10.1.0.0/16                        │
│  ├─ DNS Subnet:            10.1.0.0/24   (254 usable)        │
│  │  ├─ vm-dns-ci-01:       10.1.0.4                          │
│  │  ├─ vm-dns-ci-02:       10.1.0.5                          │
│  │  └─ Available:          10.1.0.6 - 10.1.0.254             │
│  │                                                           │
│  ├─ Private Endpoint:      10.1.100.0/24 (254 usable)        │
│  │  ├─ pe-ampls-ci:        10.1.100.50 (auto-assigned)       │
│  │  ├─ pe-storage-ci:      10.1.100.51 (if needed)           │
│  │  └─ Available:          10.1.100.52 - 10.1.100.254        │
│  │                                                           │
│  └─ Reserved:              10.1.1.0/24 - 10.1.99.0/24        │
│                                                              │
│  SPOKE VNET (Apps): 10.1.10.0/16 (if separate)               │
│  ├─ Web Tier:              10.1.10.0/24                      │
│  ├─ App Tier:              10.1.11.0/24                      │
│  └─ Data Tier:             10.1.12.0/24                      │
└──────────────────────────────────────────────────────────────┘
```

### Complete IP Allocation Plan - South India

```
┌──────────────────────────────────────────────────────────────┐
│                  SOUTH INDIA - IP PLAN                       │
├──────────────────────────────────────────────────────────────┤
│  (Same structure as CI, with 10.2.x.x and 10.10.x.x ranges)  │
│                                                              │
│  HUB VNET: 10.10.0.0/16                                      │
│  SPOKE VNET (Monitoring): 10.2.0.0/16                        │
│  SPOKE VNET (Apps): 10.2.10.0/16                             │
└──────────────────────────────────────────────────────────────┘
```

---

## Resource Group Strategy

### Resource Group Design

```yaml
Resource Group Philosophy:

1. Group by Lifecycle:
   - Resources that are created/deleted together
   - Example: All monitoring resources in one RG

2. Group by RBAC Boundary:
   - Different teams need different access
   - Example: Network team (hub), App team (spoke)

3. Group by Region:
   - Easier management per region
   - Disaster recovery planning

4. Group by Environment:
   - Prod, Dev, Test in separate RGs
```

### Recommended Resource Group Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                      SUBSCRIPTION STRUCTURE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  CENTRAL INDIA                                                  │
│  ├─ rg-network-hub-ci                                           │
│  │  ├─ vnet-hub-ci                                              │
│  │  ├─ ExpressRoute Gateway                                     │
│  │  ├─ Azure Firewall                                           │
│  │  └─ Route Tables                                             │
│  │                                                              │
│  ├─ rg-monitor-ci                                               │
│  │  ├─ vnet-spoke-monitor-ci                                    │
│  │  ├─ vm-dns-ci-01, vm-dns-ci-02                               │
│  │  ├─ pe-ampls-ci                                              │
│  │  ├─ ampls-ci                                                 │
│  │  ├─ law-ci                                                   │
│  │  └─ Private DNS Zones (4 zones)                              │
│  │                                                              │
│  └─ rg-apps-ci                                                  │
│     ├─ vnet-spoke-app-ci                                        │
│     └─ Application resources                                    │
│                                                                 │
│  SOUTH INDIA                                                    │
│  ├─ rg-network-hub-si                                           │
│  ├─ rg-monitor-si                                               │
│  └─ rg-apps-si                                                  │
│                                                                 │
│  SHARED (if needed)                                             │
│  └─ rg-shared-governance                                        │
│     ├─ Azure Policy assignments                                 │
│     └─ Management groups                                        │
└─────────────────────────────────────────────────────────────────┘
```

### Resource Group Details

```yaml
rg-network-hub-ci:
  Purpose: Hub networking infrastructure
  Owner: Network Team
  RBAC: Network Contributor (limited team)
  Resources:
    - Hub VNet
    - ExpressRoute Gateway
    - Azure Firewall (optional)
    - Route Tables
    - NSGs for hub subnets

rg-monitor-ci:
  Purpose: Monitoring infrastructure (AMPLS)
  Owner: Platform/Monitoring Team
  RBAC: Contributor for monitoring team, Reader for app teams
  Resources:
    - Spoke VNet (Monitoring)
    - DNS VMs
    - Private Endpoints
    - AMPLS
    - LAW
    - Private DNS Zones
    - NSGs for monitoring subnets

rg-apps-ci:
  Purpose: Application workloads
  Owner: Application Team
  RBAC: Contributor for app team
  Resources:
    - Spoke VNet (Apps)
    - Application VMs, AKS, etc.
    - App-specific resources
```

---

## Naming Conventions

### Naming Standard (Microsoft CAF Aligned)

```yaml
Format: <resource-type>-<workload>-<region>-<instance>

Examples:
  VNets:
    - vnet-hub-ci
    - vnet-spoke-monitor-ci
    - vnet-spoke-app-si
  
  Subnets:
    - snet-dns-ci
    - snet-pe-monitor-ci
    - GatewaySubnet (Azure required name)
    - AzureFirewallSubnet (Azure required name)
  
  VMs:
    - vm-dns-ci-01
    - vm-dns-si-02
  
  Private Endpoints:
    - pe-ampls-ci
    - pe-storage-si
  
  Resource Groups:
    - rg-network-hub-ci
    - rg-monitor-si
  
  AMPLS:
    - ampls-ci
    - ampls-si
  
  LAW:
    - law-ci
    - law-si
```

### Naming Convention Table

| Resource Type | Prefix | Example | Notes |
|--------------|--------|---------|-------|
| Resource Group | rg- | rg-monitor-ci | Lowercase |
| Virtual Network | vnet- | vnet-hub-ci | Lowercase |
| Subnet | snet- | snet-dns-ci | Lowercase, except Azure-required names |
| Virtual Machine | vm- | vm-dns-ci-01 | Lowercase + instance number |
| Network Interface | nic- | nic-dns-ci-01 | Matches VM name |
| Private Endpoint | pe- | pe-ampls-ci | Lowercase |
| Private DNS Zone | (full FQDN) | privatelink.monitor.azure.com | Exact Azure service name |
| AMPLS | ampls- | ampls-ci | Lowercase |
| LAW | law- | law-ci | Lowercase |
| NSG | nsg- | nsg-dns-ci | Lowercase |
| Route Table | rt- | rt-spoke-ci | Lowercase |

---

## Step-by-Step Deployment Order

### Phase-Based Deployment Sequence

```yaml
PHASE 1: Foundation (Networking) - Week 1
   Step 1.1: Create Resource Groups
     - rg-network-hub-ci
     - rg-network-hub-si
     - rg-monitor-ci
     - rg-monitor-si
  
   Step 1.2: Deploy Hub VNets
     - vnet-hub-ci (10.0.0.0/16)
     - vnet-hub-si (10.10.0.0/16)
     - Create: GatewaySubnet, AzureFirewallSubnet, ManagementSubnet
  
   Step 1.3: Deploy Spoke VNets (Monitoring)
     - vnet-spoke-monitor-ci (10.1.0.0/16)
     - vnet-spoke-monitor-si (10.2.0.0/16)
     - Create: snet-dns, snet-pe-monitor
  
   Step 1.4: Configure VNet Peering
     - Hub CI ↔ Spoke Monitor CI
     - Hub SI ↔ Spoke Monitor SI
     - Hub CI ↔ Hub SI (optional, for inter-region)
  
   Step 1.5: Deploy Gateways
     - ExpressRoute Gateway in Hub CI
     - ExpressRoute Gateway in Hub SI (or share via peering)
     - Configure BGP peering

PHASE 2: DNS Infrastructure - Week 2
   Step 2.1: Deploy DNS VMs
     - vm-dns-ci-01 (10.1.0.4)
     - vm-dns-ci-02 (10.1.0.5)
     - vm-dns-si-01 (10.2.0.4)
     - vm-dns-si-02 (10.2.0.5)
  
   Step 2.2: Configure DNS VMs
     - Install DNS Server role
     - Configure forwarder: 168.63.129.16
     - NO cross-region forwarders (precise forwarding model)
  
   Step 2.3: Create Private DNS Zones
     - CI: 4 zones (monitor, oms, ods, agentsvc)
     - SI: 4 zones (same)
  
   Step 2.4: Link Private DNS Zones to Spoke VNets
     - Link CI zones → vnet-spoke-monitor-ci
     - Link SI zones → vnet-spoke-monitor-si
     - Enable auto-registration
  
   Step 2.5: Update VNet DNS Settings
     - vnet-spoke-monitor-ci → DNS: 10.1.0.4, 10.1.0.5
     - vnet-spoke-monitor-si → DNS: 10.2.0.4, 10.2.0.5

PHASE 3: Azure Monitor Setup - Week 3
   Step 3.1: Create Log Analytics Workspaces
     - law-ci (Central India)
     - law-si (South India)
     - Disable public network access
  
   Step 3.2: Create AMPLS
     - ampls-ci
     - ampls-si
  
   Step 3.3: Link LAW to AMPLS
     - law-ci → ampls-ci
     - law-si → ampls-si
  
   Step 3.4: Create Private Endpoints
     - pe-ampls-ci in snet-pe-monitor-ci
     - pe-ampls-si in snet-pe-monitor-si
     - Configure Private DNS Zone Groups
  
   Step 3.5: Verify DNS Records
     - Check A records auto-created in Private DNS Zones
     - Verify: <workspace-id>.oms.opinsights.azure.com → PE IP

PHASE 4: On-Premises Integration - Week 4
   Step 4.1: Document Workspace IDs
     - Get LAW-CI workspace ID
     - Get LAW-SI workspace ID
  
   Step 4.2: Configure On-Premises DNS
     - Add conditional forwarders:
       * CI workspace → 10.1.0.4, 10.1.0.5
       * SI workspace → 10.2.0.4, 10.2.0.5
  
   Step 4.3: Test DNS Resolution
     - From on-prem: nslookup <ci-workspace-id>.oms.opinsights.azure.com
     - Expected: 10.1.100.50 (CI PE IP)
     - From on-prem: nslookup <si-workspace-id>.oms.opinsights.azure.com
     - Expected: 10.2.100.50 (SI PE IP)
  
   Step 4.4: Test Connectivity
     - From on-prem: Test-NetConnection -ComputerName 10.1.100.50 -Port 443
     - Should succeed via ExpressRoute

PHASE 5: Validation & Monitoring - Week 5
   Step 5.1: Deploy Test VMs in Spoke
     - vm-test-ci-01 in Spoke Monitor CI
     - vm-test-si-01 in Spoke Monitor SI
  
   Step 5.2: Install Monitoring Agents
     - Install Log Analytics Agent on test VMs
     - Configure to use LAW-CI and LAW-SI
  
   Step 5.3: Validate Data Ingestion
     - Check data appears in LAW-CI
     - Check data appears in LAW-SI
  
   Step 5.4: User Access Testing
     - User logs into Azure Portal from on-premises
     - Queries LAW-CI via portal
     - Queries LAW-SI via portal
     - Verify all traffic uses private IPs
  
   Step 5.5: Performance Baseline
     - Measure DNS resolution time
     - Measure query execution time
     - Document for future comparison
```

---

## Common Mistakes to Avoid

### Critical Configuration Errors

```yaml
 MISTAKE 1: Placing Private Endpoints in Hub
   Problem: Mixes connectivity with workload services
   Solution: Place PEs in dedicated spoke (monitoring)
   Impact: Confusion, harder to manage RBAC

 MISTAKE 2: Not Disabling privateEndpointNetworkPolicies
   Problem: PE subnet requires this setting
   Solution: Set to "Disabled" on PE subnet
   Impact: PE creation will fail

 MISTAKE 3: Linking Private DNS Zones to Wrong VNet
   Problem: Zones linked to Hub instead of Spoke
   Solution: Link to vnet-spoke-monitor-XX (where DNS VMs are)
   Impact: DNS VMs can't resolve private endpoints

 MISTAKE 4: Forgetting to Update VNet DNS Settings
   Problem: VNet still uses Azure default DNS (168.63.129.16)
   Solution: Set VNet DNS to custom: 10.1.0.4, 10.1.0.5
   Impact: VMs use default DNS, can't resolve private IPs

 MISTAKE 5: Using Same IP Range Across Regions
   Problem: Both CI and SI use 10.1.0.0/16
   Solution: Use unique ranges (CI: 10.1.x.x, SI: 10.2.x.x)
   Impact: Routing conflicts if hub-to-hub peering

 MISTAKE 6: Not Configuring Precise Forwarding Rules
   Problem: On-prem DNS forwards to wrong regional DNS
   Solution: Configure per-workspace conditional forwarders
   Impact: Queries fail or go to wrong region

 MISTAKE 7: Forgetting to Disable LAW Public Access
   Problem: LAW still accessible from internet
   Solution: Disable "Public network access for ingestion" and "query"
   Impact: Security gap, AMPLS not enforced

 MISTAKE 8: Creating AMPLS with Wrong Access Mode
   Problem: AMPLS set to "Open" instead of "Private Only"
   Solution: Set to "Private Only" mode
   Impact: Public access still allowed

 MISTAKE 9: Not Planning for Subnet Growth
   Problem: /28 subnet for PEs, no room for more services
   Solution: Use /27 or /26 for PE subnets
   Impact: Can't add new PEs without subnet expansion

 MISTAKE 10: Mixing Application Workloads with Monitoring
   Problem: App VMs in same spoke as DNS/PE
   Solution: Separate spoke for apps (vnet-spoke-app-XX)
   Impact: Security boundary unclear, harder to manage
```

### Troubleshooting Checklist

```yaml
If DNS Resolution Fails:
  □ Check: On-prem DNS has correct conditional forwarders?
  □ Check: Azure DNS VMs are running?
  □ Check: Private DNS Zones linked to spoke VNet?
  □ Check: VNet DNS settings point to custom DNS (10.1.0.4)?
  □ Check: NSG allows DNS traffic (port 53)?

If Connectivity Fails:
  □ Check: ExpressRoute/VPN is up?
  □ Check: Can ping Azure DNS VM from on-prem?
  □ Check: Can ping PE IP (10.1.100.50) from on-prem?
  □ Check: NSG allows HTTPS (port 443) to PE subnet?
  □ Check: Azure Firewall (if present) allows traffic?

If Agents Can't Send Data:
  □ Check: VM DNS resolves workspace URL to private IP?
  □ Check: VM can reach PE IP on port 443?
  □ Check: Workspace is linked to AMPLS?
  □ Check: PE is connected to AMPLS?
  □ Check: LAW public access is disabled?

If Portal Queries Fail:
  □ Check: User has RBAC permissions on LAW?
  □ Check: User's network can route to PE IP?
  □ Check: AMPLS access mode is correct?
  □ Check: Private DNS Zone Group configured on PE?
```

---

## Summary Placement Reference Card

```
┌────────────────────────────────────────────────────────────────┐
│                   QUICK PLACEMENT REFERENCE                    │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  HUB VNET (10.0.0.0/16 or 10.10.0.0/16)                        │
│  ├─ GatewaySubnet (10.0.0.0/24)                                │
│  │  └─ ExpressRoute Gateway                                    │
│  ├─ AzureFirewallSubnet (10.0.1.0/24)                          │
│  │  └─ Azure Firewall  (optional)                              │
│  └─ ManagementSubnet (10.0.10.0/24)                            │
│     └─ Bastion, Jump Boxes                                     │
│                                                                │
│  SPOKE VNET - MONITORING (10.1.0.0/16 or 10.2.0.0/16)          │
│  ├─ snet-dns (10.1.0.0/24)                                     │
│  │  └─ DNS VMs (10.1.0.4, 10.1.0.5)                            │
│  ├─ snet-pe-monitor (10.1.100.0/24)                            │
│  │  └─ Private Endpoint for AMPLS                              │
│  └─ Private DNS Zones (Global, linked to this VNet)            │
│                                                                │
│  MONITORING RESOURCES (Regional RG)                            │
│  ├─ AMPLS  (Global resource, RG in region)                     │
│  └─ LAW  (Regional resource)                                   │
│                                                                │
│  SPOKE VNET - APPLICATIONS (separate, if needed)               │
│  └─ Application workloads                                      │
└────────────────────────────────────────────────────────────────┘
```

---

***End of document***
