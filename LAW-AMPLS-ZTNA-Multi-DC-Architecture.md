# Log Analytics Workspace (LAW) & AMPLS Architecture Design
## Contoso Azure Hub-Spoke Network with Custom DNS and ZTNA Access

**Purpose:** Architecture design and implementation guide for LAW and AMPLS Private Endpoints in Contoso's multi-region hub-spoke topology

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Component Placement](#component-placement)
4. [DNS Architecture](#dns-architecture)
5. [Routing Flow](#routing-flow)
6. [ZTNA Integration](#ztna-integration)
7. [Implementation Phases](#implementation-phases)
8. [Validation & Testing](#validation--testing)
9. [Security Considerations](#security-considerations)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## Executive Summary

### Business Requirements
- **Multi-Region Deployment**: Chennai and Pune Azure regions
- **Hub-Spoke Topology**: Centralized security and management
- **Custom DNS**: Organization-wide custom DNS infrastructure
- **Centralized Private DNS**: All Azure Private DNS Zones hosted in Pune
- **ZTNA Access**: Remote users need to query Log Analytics Workspace via Zero Trust Network Access
- **Private Connectivity**: All Azure Monitor traffic must remain private (no public endpoints)

### Solution Overview
This architecture implements Azure Monitor Private Link Scope (AMPLS) with private endpoints in both regional hub VNETs, connected to a centralized Log Analytics Workspace. DNS resolution is handled through custom organizational DNS with conditional forwarding to Azure DNS forwarder VMs, resolving against centralized Private DNS Zones in Pune.

### Key Design Decisions
- **AMPLS Private Endpoints**: Deployed in BOTH Chennai and Pune Hub VNETs for regional resilience
- **Log Analytics Workspace**: Single centralized workspace (recommended location: Pune)
- **Private DNS Zones**: Centralized in Pune, linked to both Hub VNETs
- **DNS Forwarders**: Azure DNS forwarder VMs in each Hub VNET to bridge custom DNS and Azure Private DNS
- **ZTNA Connectivity**: ZTNA connector deployed in Pune Hub VNET with access to AMPLS private endpoint

---

## Architecture Overview

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        ZTNA Users (Remote)                       │
│                     (Authenticated via ZTNA)                     │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    ┌────────▼─────────┐
                    │  ZTNA Solution   │
                    │ (Zscaler/Prisma) │
                    └────────┬─────────┘
                             │
                ┌────────────▼─────────────┐
                │ ZTNA Connector           │
                │ (Pune Hub VNET)          │
                └──────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                      CHENNAI REGION                              │
│  ┌───────────────────────────────────────────────────┐           │
│  │              Contoso HUB VNET (Chennai)           │           │
│  │  ┌─────────────────────┐  ┌──────────────────┐    │           │
│  │  │ DNS Forwarder VM    │  │ AMPLS Private    │    │           │
│  │  │ 10.10.254.4         │  │ Endpoint         │    │           │
│  │  │                     │  │ IP: 10.10.5.5    │    │           │
│  │  └─────────────────────┘  └──────────────────┘    │           │
│  └───────────────────────────────────────────────────┘           │
│           ↑                        ↑                             │
│           │ Peering                │ Peering                     │
│  ┌────────┴────────┐      ┌────────┴────────┐                    │
│  │  APP VNET       │      │  DB VNET        │                    │
│  │  (Chennai)      │      │  (Chennai)      │                    │
│  └─────────────────┘      └─────────────────┘                    │
└──────────────────────────────────────────────────────────────────┘
                             │
                    ┌────────▼─────────┐
                    │ ExpressRoute /   │
                    │ SDWAN            │
                    └────────┬─────────┘
                             │
┌─────────────────────────────────────────────────────────────────┐
│                        PUNE REGION                              │
│  ┌───────────────────────────────────────────────────┐          │
│  │              Contoso HUB VNET (Pune)              │          │
│  │  ┌─────────────────────┐  ┌──────────────────┐    │          │
│  │  │ DNS Forwarder VM    │  │ AMPLS Private    │    │          │
│  │  │ 10.20.254.4         │  │ Endpoint         │    │          │
│  │  │                     │  │ IP: 10.20.5.5    │    │          │
│  │  └─────────────────────┘  └──────────────────┘    │          │
│  │                                                   │          │
│  │  ┌─────────────────────────────────────────────┐  │          │
│  │  │     ZTNA Connector Subnet                   │  │          │
│  │  └─────────────────────────────────────────────┘  │          │
│  └───────────────────────────────────────────────────┘          │
│           ↑                        ↑                            │
│           │ Peering                │ Peering                    │
│  ┌────────┴────────┐      ┌────────┴────────┐                   │
│  │  APP VNET       │      │  PaaS Prod VNET │                   │
│  │  (Pune)         │      │  (Pune)         │                   │
│  └─────────────────┘      └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│         Centralized Private DNS Zones (Pune Region)             │
│                                                                 │
│  • privatelink.monitor.azure.com                                │
│  • privatelink.oms.opinsights.azure.com                         │
│  • privatelink.ods.opinsights.azure.com                         │
│  • privatelink.agentsvc.azure-automation.net                    │
│  • privatelink.blob.core.windows.net                            │
│                                                                 │
│  Linked VNETs: Chennai Hub VNET + Pune Hub VNET                 │
└─────────────────────────────────────────────────────────────────┘
                             │
                    ┌────────▼─────────┐
                    │      AMPLS       │
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │ Log Analytics    │
                    │ Workspace (LAW)  │
                    │ Location: Pune   │
                    └──────────────────┘
```

---

## Component Placement

### 1. AMPLS Private Endpoint Placement

#### Regional Deployment Strategy
Deploy AMPLS private endpoints in **BOTH** Hub VNETs to provide:
- **Regional resilience**: Each region can independently access LAW
- **Optimized routing**: Traffic stays within region before going to AMPLS
- **Fault tolerance**: If one region's AMPLS endpoint fails, the other remains available

#### Chennai Hub VNET
- **Resource Name**: `pe-ampls-chennai`
- **Subnet**: `snet-ampls-pe-chennai`
- **Subnet Address Space**: Example - `10.10.5.0/27`
- **Private IP Assignment**: Dynamic (e.g., `10.10.5.5`)
- **Purpose**: Serves all Chennai spoke VNETs (APP, DB, PaaS Prod)

#### Pune Hub VNET
- **Resource Name**: `pe-ampls-pune`
- **Subnet**: `snet-ampls-pe-pune`
- **Subnet Address Space**: Example - `10.20.5.0/27`
- **Private IP Assignment**: Dynamic (e.g., `10.20.5.5`)
- **Purpose**: Serves all Pune spoke VNETs + ZTNA access

#### Subnet Design Considerations
- **Dedicated Subnet**: Create separate subnet for AMPLS private endpoints
- **Subnet Size**: /27 (32 IPs) is sufficient for multiple private endpoints
- **NSG Rules**: Allow inbound 443 from spoke VNETs
- **No NAT Gateway**: Private endpoints don't require outbound internet
- **Service Endpoints**: Not required (using Private Link)

---

### 2. Log Analytics Workspace Deployment

#### Centralized vs. Regional LAW

**Recommended: Single Centralized LAW**
- **Location**: Pune region (co-located with centralized DNS)
- **Resource Group**: Dedicated monitoring resource group
- **Workspace Name**: Example - `law-Contoso-centralized`
- **SKU**: PerGB2018 (pay-as-you-go with commitment tiers available)

**Benefits of Centralized Approach:**
- Single pane of glass for all monitoring data
- Simplified alerting and dashboard management
- Cross-region correlation and analysis
- Cost optimization through data aggregation
- Simplified RBAC and access control

**Alternative: Regional LAW (Not Recommended)**
- Would require separate LAW in Chennai and Pune
- Increased management overhead
- Difficult cross-region analysis
- Higher costs due to separate ingestion

#### LAW Configuration
- **Data Retention**: Configure based on compliance requirements (default 30-90 days)
- **Daily Cap**: Set optional daily ingestion cap for cost control
- **Network Isolation**: 
  - Public network access for ingestion: **Disabled**
  - Public network access for query: **Disabled**
  - Access only via AMPLS private endpoints

---

### 3. DNS Forwarder VMs

#### Purpose and Function
Since the organization uses **custom DNS servers**, Azure VMs cannot directly query Azure Private DNS Zones. DNS forwarder VMs bridge this gap by:
- Receiving DNS queries from custom DNS (via conditional forwarding)
- Forwarding to Azure DNS resolver (168.63.129.16)
- Enabling resolution of Private DNS Zone records

#### Chennai Hub VNET DNS Forwarder
- **VM Name**: `vm-dns-forwarder-chennai`
- **Subnet**: `snet-dns-forwarder-chennai`
- **Private IP**: Example - `10.10.254.4` (static assignment recommended)
- **OS**: Ubuntu 22.04 LTS or Windows Server 2022
- **VM Size**: Standard_B2s (2 vCPU, 4 GB RAM) - sufficient for DNS
- **Software**: dnsmasq (Linux) or DNS Server role (Windows)
- **High Availability**: Deploy 2 VMs in Availability Set for production

#### Pune Hub VNET DNS Forwarder
- **VM Name**: `vm-dns-forwarder-pune`
- **Subnet**: `snet-dns-forwarder-pune`
- **Private IP**: Example - `10.20.254.4` (static assignment recommended)
- **OS**: Ubuntu 22.04 LTS or Windows Server 2022
- **VM Size**: Standard_B2s
- **Software**: dnsmasq (Linux) or DNS Server role (Windows)
- **High Availability**: Deploy 2 VMs in Availability Set for production

#### DNS Forwarder Configuration Requirements
- **Inbound Rules**: Allow TCP/UDP 53 from custom DNS servers
- **Outbound Rules**: Allow TCP/UDP 53 to 168.63.129.16
- **Monitoring**: Enable Azure Monitor agent, track DNS query metrics
- **Backup**: Not required (stateless service)

---

### 4. Private DNS Zones

#### Centralization in Pune
All Azure Private DNS Zones are created and managed in **Pune region** resource group.

#### Required Private DNS Zones for Azure Monitor

| DNS Zone | Purpose |
|----------|---------|
| `privatelink.monitor.azure.com` | Azure Monitor global endpoint |
| `privatelink.oms.opinsights.azure.com` | Log Analytics workspace management |
| `privatelink.ods.opinsights.azure.com` | Log Analytics data ingestion |
| `privatelink.agentsvc.azure-automation.net` | Azure Automation agent service |
| `privatelink.blob.core.windows.net` | Blob storage for diagnostics (if needed) |

#### VNET Links
Each Private DNS Zone must be linked to:
- ✅ **Chennai Hub VNET**: For Chennai AMPLS endpoint DNS registration
- ✅ **Pune Hub VNET**: For Pune AMPLS endpoint DNS registration
- ❌ **Spoke VNETs**: NOT linked (they use custom DNS, not Azure DNS)

#### Auto-Registration
- **Setting**: Disabled for all VNET links
- **Reason**: Private endpoint DNS records are auto-created by Azure, not by VM registration

#### DNS Record Structure
When AMPLS private endpoints are created, Azure automatically creates A records:

```
# Chennai AMPLS endpoint
<workspace-id>.ods.opinsights.azure.com → 10.10.5.5

# Pune AMPLS endpoint  
<workspace-id>.ods.opinsights.azure.com → 10.20.5.5
```

**Note**: Both records can coexist. Clients will resolve to their nearest endpoint based on DNS forwarder location.

---

## DNS Architecture

### DNS Resolution Flow

#### For Applications in Chennai Spoke VNETs

```
Step 1: Application queries workspace-id.ods.opinsights.azure.com
          ↓
Step 2: VNET DNS settings point to Custom Corporate DNS
          ↓
Step 3: Corporate DNS has conditional forwarder:
        *.ods.opinsights.azure.com → 10.10.254.4 (Chennai DNS Forwarder)
          ↓
Step 4: Chennai DNS Forwarder VM receives query
          ↓
Step 5: DNS Forwarder forwards to Azure DNS: 168.63.129.16
          ↓
Step 6: Azure DNS queries Private DNS Zone (linked to Chennai Hub VNET)
          ↓
Step 7: Private DNS Zone returns: 10.10.5.5 (Chennai AMPLS PE)
          ↓
Step 8: Resolution returned to application
          ↓
Step 9: Application connects to 10.10.5.5:443
          ↓
Step 10: Traffic routed via VNET peering to Hub
          ↓
Step 11: AMPLS Private Endpoint receives connection
          ↓
Step 12: AMPLS forwards to Log Analytics Workspace
```

#### For Applications in Pune Spoke VNETs

Same flow as Chennai, but:
- Corporate DNS forwards to `10.20.254.4` (Pune DNS Forwarder)
- Resolves to `10.20.5.5` (Pune AMPLS PE)
- Traffic stays within Pune region

#### For ZTNA Users

```
Step 1: ZTNA user initiates query via ZTNA client
          ↓
Step 2: ZTNA solution routes DNS query per policy
          ↓
Step 3: ZTNA Connector in Pune Hub VNET receives query
          ↓
Step 4: ZTNA Connector DNS configured to use 10.20.254.4
          ↓
Step 5: Pune DNS Forwarder forwards to Azure DNS
          ↓
Step 6: Resolves to 10.20.5.5 (Pune AMPLS PE)
          ↓
Step 7: ZTNA user connects to 10.20.5.5:443 via ZTNA tunnel
          ↓
Step 8: Traffic reaches AMPLS PE in Pune Hub
          ↓
Step 9: AMPLS forwards to Log Analytics Workspace
```

---

### Custom DNS Configuration

#### On Corporate DNS Servers

**Create Conditional Forwarders** for all Azure Monitor Private DNS zones:

**For Chennai Zone/Region:**
- Zone: `privatelink.monitor.azure.com`
  - Forward to: `10.10.254.4`
- Zone: `privatelink.oms.opinsights.azure.com`
  - Forward to: `10.10.254.4`
- Zone: `privatelink.ods.opinsights.azure.com`
  - Forward to: `10.10.254.4`
- Zone: `privatelink.agentsvc.azure-automation.net`
  - Forward to: `10.10.254.4`

**For Pune Zone/Region:**
- Same zones, forward to: `10.20.254.4`

**Regional Routing Strategy:**
- Corporate DNS should route Chennai users to Chennai DNS Forwarder
- Corporate DNS should route Pune users to Pune DNS Forwarder
- This can be based on source IP, Active Directory site, or DNS policies

---

### DNS Forwarder VM Setup

#### Linux (Ubuntu) with dnsmasq

**Installation Steps:**
1. Deploy Ubuntu 22.04 LTS VM in Hub VNET
2. Assign static private IP (e.g., 10.10.254.4)
3. Install dnsmasq package
4. Configure dnsmasq to forward Azure Private DNS queries to 168.63.129.16
5. Configure dnsmasq to forward all other queries to upstream DNS
6. Enable and start dnsmasq service
7. Configure VM NSG to allow inbound UDP/TCP 53
8. Test DNS resolution from another VM

**Configuration Requirements:**
- Listen on all interfaces or specific private IP
- Forward queries for `privatelink.*.azure.com` to 168.63.129.16
- Forward other queries to corporate DNS or internet DNS
- Enable logging for troubleshooting
- Set cache size appropriately (e.g., 1000 entries)

#### Windows Server with DNS Role

**Installation Steps:**
1. Deploy Windows Server 2022 VM in Hub VNET
2. Assign static private IP (e.g., 10.10.254.4)
3. Install DNS Server role
4. Create conditional forwarders for each Private DNS zone
5. Point each zone to 168.63.129.16
6. Configure root hints for non-Azure queries
7. Enable and configure DNS logging
8. Test resolution from another VM

**Configuration Requirements:**
- Disable recursion for non-forwarded zones (optional)
- Set up forwarders for each `privatelink.*.azure.com` zone
- Configure firewall to allow DNS traffic
- Enable DNS event logging
- Regular monitoring of DNS query logs

---

## Routing Flow

### Network Traffic Path

#### Spoke VNET to LAW (Data Ingestion)

**Chennai APP VNET Example:**

```
┌───────────────────────────────────────────────────────────┐
│ Step 1: Application in APP VNET                           │
│         Sends logs to LAW API endpoint                    │
│         Destination: workspace-id.ods.opinsights.azure.com│
└────────────────────┬──────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 2: DNS Resolution                                   │
│         Resolves to: 10.10.5.5 (Chennai AMPLS PE)        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 3: Routing Decision                                 │
│         Destination 10.10.5.5 is in Hub VNET             │
│         Route via VNET peering                           │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 4: VNET Peering                                     │
│         Traffic traverses peering to Chennai Hub VNET    │
│         No charge for ingress to Hub                     │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 5: NSG Processing (if enabled)                      │
│         Hub VNET NSG allows inbound 443 to AMPLS subnet  │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 6: Private Endpoint                                 │
│         AMPLS PE receives connection on 10.10.5.5:443    │
│         Terminates TLS connection                        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 7: AMPLS Processing                                 │
│         Validates request against AMPLS scope            │
│         Checks if LAW is in AMPLS scope                  │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 8: Azure Backbone                                   │
│         Forwards to LAW over Azure private network       │
│         No internet traversal                            │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 9: Log Analytics Workspace                          │
│         Receives and ingests log data                    │
│         Returns 200 OK response                          │
└──────────────────────────────────────────────────────────┘
```

---

### User-Defined Routes (UDR) Considerations

#### When Azure Firewall is in Hub

If you have Azure Firewall in the Hub VNET (as shown in your diagram as "Internal FW"), you need to consider routing:

**Option 1: Default Route via Firewall**
- Spoke VNETs have UDR with default route (0.0.0.0/0) pointing to Azure Firewall
- Add exception route for AMPLS subnet to bypass firewall
- Route AMPLS traffic directly via VNET local

**Option 2: Specific Routes Only**
- Don't use default route via firewall
- Only route internet-bound traffic via firewall
- AMPLS traffic stays local within VNET

**Recommended Approach:**
Add specific route in spoke VNET route tables:
- Destination: AMPLS PE subnet (e.g., 10.10.5.0/27)
- Next hop: Virtual Network (VnetLocal)
- This ensures AMPLS traffic bypasses firewall inspection

**Firewall Rules (if inspection required):**
- Source: Spoke VNET subnets
- Destination: AMPLS PE subnet
- Port: 443
- Action: Allow
- Protocol: HTTPS

---

### VNET Peering Configuration

#### Required Settings

**From Spoke to Hub:**
- ✅ **Allow virtual network access**: Enabled
- ✅ **Allow forwarded traffic**: Enabled (for internet via firewall)
- ✅ **Allow gateway transit**: Disabled (unless using VPN/ER gateway in hub)
- ✅ **Use remote virtual network's gateway**: Enabled (if using ER/VPN in hub)

**From Hub to Spoke:**
- ✅ **Allow virtual network access**: Enabled
- ✅ **Allow forwarded traffic**: Enabled
- ✅ **Allow gateway transit**: Enabled (if hub has ER/VPN gateway)
- ✅ **Use remote virtual network's gateway**: Disabled

#### DNS Settings in VNET Peering
- **DNS Servers**: Spokes must use Custom DNS (corporate DNS IPs)
- **DNS Forwarding**: Handled via conditional forwarders, not VNET peering settings

---

## ZTNA Integration

### Zero Trust Network Access Architecture

#### ZTNA Solution Integration

**Supported ZTNA Solutions:**
- Zscaler Private Access (ZPA)
- Palo Alto Prisma Access
- Cloudflare Zero Trust
- Microsoft Entra Private Access
- Other ZTNA providers with Azure connector capability

#### ZTNA Connector Deployment

**Location**: Pune Hub VNET (co-located with centralized resources)

**Deployment Options:**

**Option A: VM-Based Connector**
- Deploy ZTNA connector as VM in dedicated subnet
- Subnet: `snet-ztna-connector`
- Size: Per ZTNA vendor recommendation (typically 4-8 vCPU)
- High Availability: Deploy 2+ connectors
- Connectivity: Direct access to Pune Hub VNET resources

**Option B: Container-Based Connector**
- Deploy in Azure Container Instances or AKS
- More lightweight than VM
- Easier scaling and updates
- Same network connectivity requirements

**Option C: ZTNA Service Edge**
- Some ZTNA solutions offer direct Azure VNET integration
- No connector VM required
- Typically uses VNET injection or Private Link

---

### ZTNA Access Flow

#### Remote User Access to LAW

```
┌──────────────────────────────────────────────────────────┐
│ Step 1: Remote User Authentication                       │
│         User authenticates to ZTNA solution (MFA, etc.)  │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 2: ZTNA Policy Evaluation                           │
│         ZTNA evaluates user identity, device posture     │
│         Checks if user authorized for LAW access         │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 3: Application Segment Match                        │
│         User accesses *.ods.opinsights.azure.com         │
│         Matches ZTNA application segment for LAW         │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 4: Traffic Routing                                  │
│         ZTNA client encrypts and sends to ZTNA cloud     │
│         ZTNA cloud routes to Pune Hub connector          │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 5: ZTNA Connector in Pune Hub                       │
│         Connector receives encrypted traffic             │
│         Decrypts and initiates connection to destination │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 6: DNS Resolution via DNS Forwarder                 │
│         Connector DNS: 10.20.254.4 (Pune DNS Forwarder)  │
│         Resolves to: 10.20.5.5 (Pune AMPLS PE)           │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 7: Connection to AMPLS PE                           │
│         ZTNA connector connects to 10.20.5.5:443         │
│         Traffic stays within Pune Hub VNET               │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 8: AMPLS to LAW                                     │
│         Same flow as internal users                      │
│         Data forwarded to Log Analytics Workspace        │
└────────────────────┬─────────────────────────────────────┘
                     │
┌────────────────────▼─────────────────────────────────────┐
│ Step 9: Response Flow                                    │
│         LAW response → AMPLS PE → Connector → ZTNA →     │
│         User's browser/application                       │
└──────────────────────────────────────────────────────────┘
```

---

### ZTNA Configuration Requirements

#### Application Segment Configuration

**For Zscaler Private Access Example:**

**Application Segment Settings:**
- **Name**: Azure Monitor - Log Analytics
- **Domain Names**: 
  - `*.ods.opinsights.azure.com`
  - `*.oms.opinsights.azure.com`
  - `*.monitor.azure.com`
- **TCP Ports**: 443
- **Health Check**: Enabled, probe 10.20.5.5:443
- **Server Groups**: Pune-Hub-ZTNA-Connectors
- **Bypass Type**: Never (always via ZTNA)
- **CNAME Enabled**: No (using direct resolution)

**Access Policy:**
- **Users/Groups**: IT-Operations, SecOps, Monitoring-Team
- **Posture Checks**: Device trust, OS patch level
- **MFA**: Required
- **Session Duration**: 12 hours
- **Logging**: Enabled

#### DNS Configuration for ZTNA

**Option 1: ZTNA DNS Forwarding (Recommended)**
- Configure ZTNA connector to use Pune DNS Forwarder: 10.20.254.4
- All Azure Private DNS queries resolved via forwarder
- Most seamless for users

**Option 2: Split DNS in ZTNA**
- Configure ZTNA cloud to override DNS for specific domains
- `*.ods.opinsights.azure.com` → 10.20.5.5
- Less flexible, requires manual IP management

**Option 3: Local DNS Override**
- ZTNA client overrides local DNS for Azure Monitor domains
- Points to Azure Private DNS via tunnel
- Can cause conflicts with other DNS requirements

---

### ZTNA Network Requirements

#### Connector Subnet Configuration
- **Subnet**: Dedicated for ZTNA connectors
- **Address Space**: /27 or larger (32+ IPs for HA)
- **NSG Inbound**: Allow ZTNA cloud IPs (vendor-specific)
- **NSG Outbound**: 
  - Allow 443 to AMPLS PE subnet
  - Allow 443 to ZTNA cloud
  - Allow DNS (53) to DNS Forwarder

#### High Availability
- Deploy minimum 2 ZTNA connectors
- Use Availability Zones if available in region
- Monitor connector health via ZTNA dashboard
- Set up alerts for connector failures

#### Security Considerations
- Connectors should NOT have public IPs
- All traffic via ZTNA cloud is encrypted
- Implement network segmentation (dedicated subnet)
- Regular patching of connector VMs
- Enable Azure Defender for servers on connector VMs

---

## Implementation Phases

### Phase 1: Foundation Setup (Week 1)

#### Tasks
1. **Create AMPLS Resources**
   - Create AMPLS in Pune region
   - Create AMPLS in Chennai region
   - Document resource IDs

2. **Create Subnets in Hub VNETs**
   - Chennai Hub: Create `snet-ampls-pe-chennai`
   - Pune Hub: Create `snet-ampls-pe-pune`
   - Chennai Hub: Create `snet-dns-forwarder-chennai`
   - Pune Hub: Create `snet-dns-forwarder-pune`
   - Pune Hub: Create `snet-ztna-connector` (if using ZTNA)

3. **Deploy Log Analytics Workspace**
   - Create LAW in Pune region
   - Configure retention period
   - Disable public network access
   - Document workspace ID and keys

#### Success Criteria
- All Azure resources created
- Subnets properly configured with address spaces
- LAW accessible via Azure Portal (public access still enabled for setup)

---

### Phase 2: DNS Infrastructure (Week 1-2)

#### Tasks
1. **Create Centralized Private DNS Zones**
   - Create all 5 required Private DNS Zones in Pune
   - Document zone resource IDs

2. **Deploy DNS Forwarder VMs**
   - Deploy VMs in Chennai Hub and Pune Hub
   - Assign static private IPs
   - Install DNS forwarding software
   - Configure forwarding to 168.63.129.16

3. **Link Private DNS Zones to Hub VNETs**
   - Link each zone to Chennai Hub VNET
   - Link each zone to Pune Hub VNET
   - Disable auto-registration

4. **Configure Corporate DNS**
   - Create conditional forwarders on corporate DNS servers
   - Forward Azure Private DNS zones to DNS Forwarder VMs
   - Test resolution from corporate network

#### Success Criteria
- DNS Forwarder VMs operational
- Private DNS Zones linked to Hub VNETs
- Corporate DNS forwarding configured
- Basic DNS resolution working (test with nslookup)

---

### Phase 3: AMPLS & Private Endpoints (Week 2)

#### Tasks
1. **Create AMPLS Private Endpoints**
   - Create PE in Chennai Hub (snet-ampls-pe-chennai)
   - Create PE in Pune Hub (snet-ampls-pe-pune)
   - Configure DNS zone groups for each PE
   - Verify automatic A record creation

2. **Add LAW to AMPLS Scopes**
   - Add LAW to Chennai AMPLS
   - Add LAW to Pune AMPLS
   - Verify AMPLS scope configuration

3. **Disable LAW Public Access**
   - Disable public network access for ingestion
   - Disable public network access for query
   - Test that public access is blocked

#### Success Criteria
- Private endpoints created with private IPs
- DNS resolution returns private IPs (not public)
- LAW accessible only via private endpoints
- Public access blocked (verified with curl from internet)

---

### Phase 4: Network Configuration (Week 2-3)

#### Tasks
1. **Configure NSG Rules**
   - Allow inbound 443 to AMPLS subnets from spoke VNETs
   - Allow DNS traffic to DNS Forwarder VMs
   - Document all NSG rules

2. **Configure VNET Peering**
   - Verify existing peering between Hub and Spokes
   - Enable required peering options
   - Test connectivity across peering

3. **Configure User-Defined Routes (if needed)**
   - Create routes for AMPLS traffic if using Azure Firewall
   - Ensure AMPLS traffic bypasses firewall or firewall allows it
   - Test routing with traceroute/tracert

#### Success Criteria
- NSG rules properly configured
- VNET peering working correctly
- Routing verified for AMPLS traffic
- No connectivity issues between spoke and Hub

---

### Phase 5: ZTNA Integration (Week 3-4)

#### Tasks
1. **Deploy ZTNA Connector**
   - Deploy connector VM/container in Pune Hub
   - Configure connector to connect to ZTNA cloud
   - Verify connector status in ZTNA dashboard

2. **Configure ZTNA Application Segment**
   - Create application segment for Azure Monitor
   - Configure domain patterns
   - Set health checks

3. **Configure ZTNA DNS**
   - Point ZTNA connector DNS to 10.20.254.4
   - Test DNS resolution from connector
   - Verify resolution returns private IP

4. **Create ZTNA Access Policies**
   - Define user groups for LAW access
   - Configure MFA and posture checks
   - Set session policies

#### Success Criteria
- ZTNA connector operational
- Application segment created
- DNS resolution working via ZTNA
- Access policies functional

---

### Phase 6: Testing & Validation (Week 4)

#### Tasks
1. **Test from Spoke VNETs**
   - Deploy test VMs in each spoke VNET
   - Install Log Analytics agent
   - Verify agent connects via private endpoint
   - Check data ingestion in LAW

2. **Test DNS Resolution**
   - Test from each spoke VNET
   - Verify resolution to correct AMPLS PE IP
   - Test from both Chennai and Pune spokes

3. **Test ZTNA Access**
   - Test from remote user with ZTNA client
   - Access Azure Portal and query LAW
   - Verify traffic goes through ZTNA connector
   - Check ZTNA logs

4. **Performance Testing**
   - Measure query latency
   - Measure data ingestion rates
   - Compare to baseline metrics

#### Success Criteria
- All spoke VNETs can ingest logs
- DNS resolution working from all locations
- ZTNA users can access LAW
- Performance meets SLA requirements

---

### Phase 7: Monitoring & Documentation (Week 5)

#### Tasks
1. **Enable Monitoring**
   - Configure diagnostic settings on AMPLS
   - Set up alerts for AMPLS issues
   - Monitor DNS Forwarder VM health
   - Monitor ZTNA connector health

2. **Create Documentation**
   - Network diagrams with actual IPs
   - DNS resolution flowcharts
   - Troubleshooting runbooks
   - Disaster recovery procedures

3. **Knowledge Transfer**
   - Train operations team
   - Document support procedures
   - Create escalation paths

4. **Production Cutover**
   - Migrate production workloads
   - Update monitoring agents
   - Verify all data sources working

#### Success Criteria
- Comprehensive monitoring in place
- Documentation complete and approved
- Team trained
- Production workloads migrated successfully

---

## Validation & Testing

### DNS Resolution Testing

#### From Spoke VNET VMs

**Test 1: Basic Resolution**
- Open command prompt/terminal on VM in spoke VNET
- Run: `nslookup <workspace-id>.ods.opinsights.azure.com`
- **Expected Result**: Should resolve to private IP (10.10.5.5 or 10.20.5.5)
- **Failure**: If resolves to public IP, DNS forwarding not working

**Test 2: DNS Resolution Path**
- On Linux VM: `dig <workspace-id>.ods.opinsights.azure.com +trace`
- **Expected Result**: Should show resolution through custom DNS → DNS forwarder
- **Failure**: If shows public DNS servers, VNET DNS settings incorrect

**Test 3: All Required Domains**
- Test each Azure Monitor domain:
  - `<workspace-id>.ods.opinsights.azure.com`
  - `<workspace-id>.oms.opinsights.azure.com`
  - `<region>.monitoring.azure.com`
- **Expected Result**: All should resolve to AMPLS PE private IP
- **Failure**: Missing Private DNS Zone or incorrect DNS forwarding

#### From DNS Forwarder VM

**Test 4: Azure DNS Resolution**
- On DNS Forwarder VM
- Query: `nslookup <workspace-id>.ods.opinsights.azure.com 168.63.129.16`
- **Expected Result**: Should resolve to AMPLS PE IP
- **Failure**: VNET link to Private DNS Zone missing

**Test 5: Forwarder Configuration**
- Check DNS forwarder config
- Verify it forwards Azure Private DNS queries to 168.63.129.16
- **Expected Result**: Configuration files show correct forwarding
- **Failure**: Misconfigured DNS forwarder

---

### Connectivity Testing

#### From Spoke VNET

**Test 6: HTTPS Connectivity**
- From spoke VM: `curl -v https://<workspace-id>.ods.opinsights.azure.com`
- **Expected Result**: Connection succeeds, returns HTTP response
- **Failure Codes**:
  - Connection timeout: Routing or NSG issue
  - Certificate error: DNS resolution to wrong IP
  - 403 Forbidden: AMPLS scope misconfiguration

**Test 7: Log Analytics Agent**
- Install Microsoft Monitoring Agent on test VM
- Configure with workspace ID and key
- Check agent status
- **Expected Result**: Agent shows "Connected" status
- **Failure**: Check agent logs, verify connectivity to all required endpoints

**Test 8: Network Trace**
- Use Network Watcher Connection Monitor or tcpdump
- Trace connection to LAW endpoint
- **Expected Result**: Traffic goes to AMPLS PE IP (10.x.x.x), not public IP
- **Failure**: Routing issue or DNS resolution to public IP

#### From ZTNA Client

**Test 9: ZTNA User Access**
- Connect via ZTNA client as authorized user
- Access Azure Portal → Log Analytics Workspace
- Run KQL query
- **Expected Result**: Query executes successfully
- **Failure**: Check ZTNA logs, verify access policy

**Test 10: ZTNA DNS**
- From ZTNA client: `nslookup <workspace-id>.ods.opinsights.azure.com`
- **Expected Result**: Resolves to AMPLS PE IP via ZTNA tunnel
- **Failure**: ZTNA DNS configuration incorrect

---

### Performance Testing

**Test 11: Query Latency**
- Run KQL queries from different locations:
  - Chennai spoke VNET
  - Pune spoke VNET
  - ZTNA remote user
- Measure query execution time
- **Expected Result**: Latency < 500ms for simple queries
- **Baseline**: Compare to direct Azure Portal access

**Test 12: Ingestion Rate**
- Send test log data at high volume
- Monitor ingestion lag in LAW
- **Expected Result**: Data appears within 2-5 minutes
- **Failure**: May indicate bandwidth or AMPLS throttling issues

---

### Security Testing

**Test 13: Public Access Blocked**
- From internet-connected machine (not via ZTNA)
- Try to access: `https://<workspace-id>.ods.opinsights.azure.com`
- **Expected Result**: Connection should fail or be rejected
- **Failure**: Public access not properly disabled

**Test 14: Unauthorized ZTNA Access**
- Attempt access with non-authorized ZTNA user
- **Expected Result**: Access denied by ZTNA policy
- **Failure**: ZTNA access policy too permissive

**Test 15: NSG Validation**
- Test connections from unauthorized subnets
- **Expected Result**: Blocked by NSG
- **Failure**: NSG rules too permissive

---

## Security Considerations

### Network Security

#### Network Segmentation
- **AMPLS Subnet**: Dedicated subnet, no other resources
- **DNS Forwarder Subnet**: Isolated from general workloads
- **ZTNA Connector Subnet**: Strict access controls
- **Spoke VNETs**: Logically separated by function (APP, DB, etc.)

#### NSG Best Practices
- **Principle of Least Privilege**: Only allow required ports
- **Source IP Restriction**: Limit to known spoke VNET ranges
- **Service Tags**: Use where applicable
- **Logging**: Enable NSG flow logs for audit
- **Regular Review**: Quarterly review of NSG rules

#### Azure Firewall Integration
If using Azure Firewall in Hub:
- **Option 1**: Allow direct AMPLS traffic (bypass firewall)
- **Option 2**: Inspect AMPLS traffic (requires proper certificate handling)
- **TLS Inspection**: Be cautious, can break Private Link
- **Application Rules**: Create specific rules for Azure Monitor FQDNs

---

### Identity & Access Management

#### LAW Access Control
- **RBAC Roles**: Use built-in roles (Log Analytics Reader, Contributor)
- **Custom Roles**: Create for specific needs
- **Resource-Level**: Assign at LAW level, not subscription
- **Table-Level**: Use table-level RBAC for sensitive data
- **Audit**: Enable activity logs on LAW

#### ZTNA Access Policies
- **Multi-Factor Authentication**: Required for all users
- **Device Posture**: Check device compliance
- **Conditional Access**: Based on location, time, risk
- **Session Controls**: Limit session duration
- **Just-In-Time Access**: For privileged operations

---

### Data Security

#### Encryption in Transit
- **TLS 1.2+**: All connections use modern TLS
- **Private Endpoints**: Traffic never exposed to internet
- **Azure Backbone**: Traffic over Microsoft's private network

#### Encryption at Rest
- **LAW Data**: Encrypted with Microsoft-managed keys by default
- **Customer-Managed Keys (CMK)**: Optional, for compliance requirements
- **Key Vault Integration**: If using CMK, key in Azure Key Vault

#### Data Retention & Compliance
- **Retention Policies**: Configure based on compliance requirements
- **Data Purge**: Ability to purge data for GDPR compliance
- **Audit Logging**: All access logged
- **Export**: Ability to export for long-term retention

---

### Monitoring & Alerting

#### What to Monitor

**AMPLS Health**
- Private endpoint connection state
- Request volume to AMPLS
- Error rates

**DNS Forwarder VMs**
- VM availability (uptime)
- DNS query volume
- Query failures
- CPU/Memory utilization

**Log Analytics Workspace**
- Data ingestion rate
- Query performance
- Storage usage
- Anomalous query patterns

**ZTNA Connector**
- Connector status (online/offline)
- Connection count
- Latency metrics
- Failed authentication attempts

**Network Connectivity**
- VNET peering status
- NSG flow logs analysis
- Route table changes

#### Alerting Strategy

**Critical Alerts (Immediate Response)**
- AMPLS private endpoint disconnected
- DNS Forwarder VM down
- LAW ingestion stopped
- ZTNA connector offline

**Warning Alerts (Review Within Hours)**
- High DNS query failure rate
- Elevated LAW query latency
- AMPLS request throttling
- Abnormal data ingestion patterns

**Informational (Daily Review)**
- Capacity approaching thresholds
- New connection attempts
- Configuration changes

---

## Troubleshooting Guide

### Common Issue 1: DNS Not Resolving to Private IP

**Symptoms:**
- `nslookup` returns public IP instead of private IP (10.x.x.x)
- Connections fail or time out

**Possible Causes:**
1. Private DNS Zone not linked to Hub VNET
2. Conditional forwarder not configured on corporate DNS
3. DNS Forwarder VM not working
4. VNET DNS settings pointing to wrong DNS servers

**Troubleshooting Steps:**
1. Check VNET DNS settings in spoke VNET (should be corporate DNS)
2. Test DNS resolution from DNS Forwarder VM directly
3. Verify Private DNS Zone VNET links
4. Check conditional forwarder configuration on corporate DNS
5. Check DNS Forwarder VM service status
6. Review DNS Forwarder VM logs

**Resolution:**
- Fix the specific component that's failing in the DNS chain
- Test resolution at each hop (spoke VM → corporate DNS → DNS forwarder → Azure DNS)

---

### Common Issue 2: Connection Timeout to AMPLS

**Symptoms:**
- DNS resolves correctly to private IP
- But connection to 10.x.x.x:443 times out

**Possible Causes:**
1. NSG blocking traffic
2. VNET peering misconfigured
3. User-Defined Route sending traffic to wrong next hop
4. Private endpoint not approved or misconfigured

**Troubleshooting Steps:**
1. Check NSG rules on AMPLS subnet
2. Check NSG rules on source VM subnet
3. Verify VNET peering status (should be "Connected")
4. Check route table (use "Effective Routes" feature)
5. Use Network Watcher "Connection Troubleshoot" tool
6. Check private endpoint status in Azure Portal

**Resolution:**
- Add NSG allow rule if needed
- Fix routing if traffic being sent to firewall incorrectly
- Approve private endpoint connection if pending

---

### Common Issue 3: LAW Not in AMPLS Scope

**Symptoms:**
- Connection succeeds
- But returns 403 Forbidden error

**Possible Causes:**
1. LAW not added to AMPLS as scoped resource
2. Wrong AMPLS scope being used
3. Public network access required but disabled

**Troubleshooting Steps:**
1. Check AMPLS scoped resources list
2. Verify LAW resource ID matches
3. Check if connecting to correct AMPLS endpoint
4. Review LAW firewall settings

**Resolution:**
- Add LAW to AMPLS scoped resources
- Verify network isolation settings match architecture

---

### Common Issue 4: ZTNA Users Cannot Access

**Symptoms:**
- Internal users can access LAW fine
- ZTNA users get errors or timeouts

**Possible Causes:**
1. ZTNA connector not connected to Pune Hub VNET
2. ZTNA DNS not configured correctly
3. ZTNA access policy blocking user
4. NSG blocking ZTNA connector subnet

**Troubleshooting Steps:**
1. Check ZTNA connector status in ZTNA dashboard
2. Test DNS resolution from ZTNA connector VM
3. Review ZTNA access logs
4. Verify user in correct ZTNA access group
5. Check NSG rules for ZTNA connector subnet
6. Test connectivity from ZTNA connector VM to AMPLS PE

**Resolution:**
- Fix ZTNA connector configuration
- Update ZTNA DNS settings
- Add user to appropriate access policy
- Update NSG rules

---

### Common Issue 5: Slow Query Performance

**Symptoms:**
- Queries taking significantly longer than expected
- Timeouts on large queries

**Possible Causes:**
1. Network latency through AMPLS
2. LAW performance tier insufficient
3. Query not optimized
4. High concurrent query volume

**Troubleshooting Steps:**
1. Measure latency at each hop (client → AMPLS → LAW)
2. Check LAW performance metrics (query duration, throttling)
3. Review query complexity
4. Check concurrent query count
5. Compare to baseline when using public endpoints (if available)

**Resolution:**
- Optimize queries
- Consider LAW capacity reservations for better performance
- If network latency high, investigate routing or VNET peering issues

---

### Common Issue 6: Agent Not Connecting

**Symptoms:**
- Log Analytics agent shows disconnected
- No data ingestion from agent

**Possible Causes:**
1. Agent configured with wrong workspace ID/key
2. Agent cannot resolve LAW endpoints
3. Firewall blocking agent traffic
4. Agent using public endpoints instead of private

**Troubleshooting Steps:**
1. Verify workspace ID and key in agent config
2. Test DNS resolution from agent VM
3. Check agent logs for connection errors
4. Verify agent using correct endpoints (private, not public)
5. Test network connectivity from agent VM to AMPLS PE
6. Check if agent VM has internet access (for initial registration)

**Resolution:**
- Reconfigure agent with correct workspace ID
- Fix DNS resolution
- Update firewall/NSG rules
- May need to allow limited internet for agent registration

---

## Appendix

### Required Private DNS Zones - Complete List

| Zone Name | Purpose | Record Type |
|-----------|---------|-------------|
| `privatelink.monitor.azure.com` | Global Azure Monitor endpoint | A record for AMPLS |
| `privatelink.oms.opinsights.azure.com` | Workspace management and configuration | A record for each workspace |
| `privatelink.ods.opinsights.azure.com` | Data ingestion endpoint | A record for each workspace |
| `privatelink.agentsvc.azure-automation.net` | Azure Automation agent service | A record for automation accounts |
| `privatelink.blob.core.windows.net` | Diagnostic storage (if needed) | A record for storage accounts |

---

### Network Architecture Summary Table

| Component | Chennai | Pune | Purpose |
|-----------|---------|------|---------|
| **AMPLS** | `ampls-chennai` | `ampls-pune` | Private Link Scope |
| **AMPLS PE** | 10.10.5.5 | 10.20.5.5 | Private endpoint IP |
| **DNS Forwarder** | 10.10.254.4 | 10.20.254.4 | Azure DNS bridge |
| **ZTNA Connector** | - | In Pune Hub | Remote user access |
| **LAW** | - | Pune (centralized) | Log workspace |
| **Private DNS** | - | Pune (centralized) | DNS zones |

---

### Port & Protocol Requirements

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| Spoke VNETs | DNS Forwarder | 53 | UDP/TCP | DNS queries |
| DNS Forwarder | 168.63.129.16 | 53 | UDP/TCP | Azure DNS queries |
| Spoke VNETs | AMPLS PE | 443 | TCP | HTTPS to LAW |
| ZTNA Connector | DNS Forwarder | 53 | UDP/TCP | DNS queries |
| ZTNA Connector | AMPLS PE | 443 | TCP | HTTPS to LAW |
| Corporate DNS | DNS Forwarder | 53 | UDP/TCP | Conditional forwarding |

---

### Useful Azure CLI Commands (Reference Only)

**Check Private Endpoint Status:**
```
az network private-endpoint show \
  --name <pe-name> \
  --resource-group <rg-name> \
  --query "privateLinkServiceConnections[0].privateLinkServiceConnectionState"
```

**List AMPLS Scoped Resources:**
```
az monitor private-link-scope scoped-resource list \
  --scope-name <ampls-name> \
  --resource-group <rg-name>
```

**Check VNET DNS Settings:**
```
az network vnet show \
  --name <vnet-name> \
  --resource-group <rg-name> \
  --query "dhcpOptions.dnsServers"
```

**Verify Private DNS Zone Links:**
```
az network private-dns link vnet list \
  --zone-name <zone-name> \
  --resource-group <rg-name>
```

---

### Best Practices Checklist

- [ ] AMPLS private endpoints in dedicated subnets
- [ ] DNS forwarder VMs deployed with HA (2+ instances)
- [ ] All Private DNS Zones linked to Hub VNETs only (not spokes)
- [ ] Corporate DNS conditional forwarders configured
- [ ] Public network access disabled on LAW
- [ ] NSG rules follow least privilege principle
- [ ] VNET peering configured with correct settings
- [ ] ZTNA access policies with MFA enabled
- [ ] Monitoring and alerting configured
- [ ] Documentation complete and accessible
- [ ] Disaster recovery procedures documented
- [ ] Regular testing of failover scenarios
- [ ] Periodic review of access policies
- [ ] Backup of DNS forwarder configurations

---

### Support & Escalation

**For DNS Issues:**
1. Check DNS forwarder VM health
2. Verify corporate DNS conditional forwarders
3. Escalate to network team if unresolved

**For AMPLS/Private Endpoint Issues:**
1. Verify private endpoint connection state
2. Check NSG and routing
3. Open Azure support ticket with "Private Link" category

**For LAW Issues:**
1. Check LAW health in Azure Portal
2. Verify data ingestion metrics
3. Open Azure support ticket with "Log Analytics" category

**For ZTNA Issues:**
1. Check connector health in ZTNA dashboard
2. Review access policies
3. Escalate to ZTNA vendor support

---

### Document Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-10-28 | shaleent_microsoft | Initial architecture design |

---

**End of Document**
