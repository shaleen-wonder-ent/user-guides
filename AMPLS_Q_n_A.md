## Azure Log Analytics Workspace (LAW) with AMPLS - Q&A ##

## Executive Summary

This document explains the architecture, cost implications, operational behavior, and troubleshooting guidance for deploying Azure Log Analytics Workspace (LAW) using Azure Monitor Private Link Scope (AMPLS).

The implementation ensures:
- Private network connectivity
- Elimination of public exposure
- Predictable cost model
- Multi-region resilience
- Secure DNS resolution

---

## Table of Contents

1. [Egress Costing](#1-egress-costing)
2. [Controls and Features of LAW for Log Customization](#2-controls-and-features-of-law-for-log-customization)
3. [Which Types of Logs are in LAW](#3-which-types-of-logs-are-in-law)
4. [How Logs Will Further Move from LAW](#4-how-logs-will-further-move-from-law)
5. [Existing Logs Flow vs. After Introducing AMPLS](#5-existing-logs-flow-vs-after-introducing-ampls)
6. [Bandwidth Consumption for LAW](#6-bandwidth-consumption-for-law)
7. [DNS Resolution Issues with AMPLS](#7-dns-resolution-issues-with-ampls)

---

## 1. Egress Costing

### With AMPLS (Proposed Architecture)

| Cost Component | Rate | Description | Remarks |
|----------------|------|-------------|---------|
| **Private Endpoint** | ~$0.01/hour (~$7.30/month) | Per Private Endpoint, per region | [Azure Private Link pricing](https://azure.microsoft.com/en-us/pricing/details/private-link)
| **Data Processing** | ~$0.01/GB | Data processed through Private Endpoint |
| **Public Egress** | $0 | No public internet egress charges |

**Total Example Cost:**
- 2 regions (Central India + South India)
- 2 Private Endpoints: ~$14.60/month
- 100 GB/month data: ~$1.00/month
- **Total: ~$15.60/month infrastructure cost**

### Without AMPLS (Traditional Public Endpoint)

| Cost Component | Rate | Description |
|----------------|------|-------------|
| **Public Egress** | $0.087/GB (varies by region) | Data transfer out to internet |
| **Cross-region transfer** | $0.02-0.05/GB | Between Azure regions over public endpoint |

**Total Example Cost:**
- 100 GB/month data: ~$8.70/month (egress only)
- Less predictable, scales linearly with volume

### Cost Benefits of AMPLS

✅ **Eliminates public egress charges** for log ingestion  
✅ **Reduced cross-region costs** via Azure backbone  
✅ **Predictable pricing** with flat PE rate  
✅ **Better for high-volume scenarios** (break-even typically at 500GB+/month)  
✅ **Enhanced security** at minimal additional cost

---

## 2. Controls and Features of LAW for Log Customization

### 2.1 Data Collection Rules (DCRs)

**Capabilities:**
- **Filtering**: Filter logs at source before ingestion (reduce costs)
- **Transformation**: Use KQL to transform data during ingestion
- **Routing**: Send different log types to different destinations
- **Granular Control**: Collect specific event IDs, counters, or syslog facilities

**Documentation:**
- [Create Data Collection Rules](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-rule-create-edit)
- [DCR Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-rule-overview)

**Example DCR Transformation:**
```kql
source
| where Severity in ("Error", "Critical")
| extend Environment_CF = "Production"
| project-away RawData
```

---

### 2.2 Table Plans

| Plan | Retention | Query Capability | Cost | Use Case |
|------|-----------|-----------------|------|----------|
| **Analytics** | 4-730 days (default) | Full KQL, Alerts | Higher | Real-time monitoring, dashboards |
| **Basic** | 8 days | Limited KQL, no alerts | 50% lower | High-volume troubleshooting logs |
| **Archive** | Up to 12 years | Search jobs only | Lowest | Compliance, long-term retention |

**Documentation:**
- [Table Plans Guide](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/logs-table-plans)
- [Query Basic Logs](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/basic-logs-query)
- [Manage Tables](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-logs-tables?tabs=azure-portal)

---

### 2.3 Data Retention Configuration

**Workspace-Level Retention:**
- Default: 30 days (included in ingestion cost)
- Configurable: 4 days to 730 days
- Extended: Up to 12 years with archive tier

**Table-Level Retention:**
- Override workspace default per table
- Separate analytics and archive periods
- Example: 90 days analytics + 7 years archive

**Configuration Paths:**
1. Azure Portal: `Workspace → Usage and estimated costs → Data Retention`
2. Per Table: `Workspace → Tables → Manage table → Retention`
3. API/CLI: Automated configuration at scale

**Documentation:**
- [Manage Data Retention](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/data-retention-configure)
- [Configure Retention (Troubleshooting)](https://learn.microsoft.com/en-us/troubleshoot/azure/azure-monitor/log-analytics/billing/configure-data-retention)

---

### 2.4 Data Transformations (KQL at Ingestion)

**Transformation Features:**
- Apply KQL queries during ingestion
- Filter unwanted data before storage
- Parse and enrich log data
- Remove sensitive information
- Cost optimization through data reduction

**Supported KQL Operations:**
- `where`, `extend`, `project`, `project-away`
- `parse`, `parse_json`, `extract`
- **Not supported**: `summarize`, `join`, `union` (single-row operations only)

**Documentation:**
- [Transformations Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-transformations)
- [Supported KQL Features](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-transformations-kql)
- [Sample Transformations](https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-rule-samples)

**Example: Remove PII and Filter Errors**
```kql
source
| where Level == "Error" or Level == "Critical"
| extend RequestContext = parse_json(Context)
| project-away EmailAddress, PhoneNumber
| project TimeGenerated, Level, Message, RequestContext
```

---

### 2.5 RBAC and Access Control

**Access Control Modes:**
- **Workspace-context**: Access to all logs in workspace
- **Resource-context**: Access limited to specific resource logs
- **Table-level**: Control access per table
- **Row-level**: Granular access based on conditions

**Built-in Roles:**
- `Log Analytics Reader`: Read-only access
- `Log Analytics Contributor`: Read/write access
- `Log Analytics Data Reader`: Fine-grained data access
- Custom roles with specific permissions

**Granular RBAC Features:**
- Column-level security (hide sensitive fields)
- Row-level filtering (by department, location, etc.)
- Query limits (CPU/memory caps per user)
- Conditional access policies

**Documentation:**
- [Manage Access to Workspaces](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-access)
- [Granular RBAC](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/granular-rbac-log-analytics)
- [Table-Level Access](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/manage-table-access)
- [Roles and Permissions](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/roles-permissions-security)

---

## 3. Which Types of Logs are in LAW

### 3.1 Azure Resource Logs

| Log Type | Description | Example Tables |
|----------|-------------|----------------|
| VM Diagnostics | Performance counters, metrics | `Perf`, `Heartbeat` |
| Activity Logs | Control plane operations | `AzureActivity` |
| Resource Diagnostics | Service-specific logs | `AzureDiagnostics` |
| Network Logs | NSG flows, firewall logs | `AzureNetworkAnalytics_CL` |

---

### 3.2 Application Logs

| Log Type | Description | Example Tables |
|----------|-------------|----------------|
| Application Insights | Telemetry, traces, exceptions | `AppTraces`, `AppExceptions` |
| Custom Application Logs | API/SDK ingestion | Custom tables (`*_CL`) |
| IIS Logs | Web server access logs | `W3CIISLog` |
| Dependency Tracking | External service calls | `AppDependencies` |

---

### 3.3 Security Logs

| Log Type | Description | Example Tables |
|----------|-------------|----------------|
| Microsoft Defender | Security alerts, vulnerabilities | `SecurityAlert`, `SecurityEvent` |
| Azure AD Logs | Sign-ins, audit logs | `SigninLogs`, `AuditLogs` |
| Sentinel Events | Security information events | `SecurityEvent` |
| Key Vault Audits | Access and operation logs | `AzureDiagnostics` (KeyVault) |

---

### 3.4 Infrastructure Logs

| Log Type | Description | Example Tables |
|----------|-------------|----------------|
| Container Logs | AKS, Container Instances | `ContainerLog`, `KubePodInventory` |
| Windows Event Logs | System, Application, Security | `Event` |
| Linux Syslog | System logs | `Syslog` |
| Update Management | Patch compliance | `Update`, `UpdateSummary` |

---

### 3.5 Platform Logs

| Log Type | Description | Example Tables |
|----------|-------------|----------------|
| Heartbeat | Agent connectivity | `Heartbeat` |
| Performance Counters | CPU, memory, disk | `Perf` |
| Change Tracking | File/registry changes | `ConfigurationChange` |
| Service Map | Application dependencies | `VMConnection` |

---

## 4. How Logs Will Further Move from LAW

### 4.1 Export Options

```
┌─────────────────────────────────────────────────────────┐
│          Log Analytics Workspace (LAW)                  │
└─────────────────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
   Storage Acc.     Event Hub          ADX
   (Archive)        (Stream)      (Long-term)
        │                │                │
        ▼                ▼                ▼
   Compliance      SIEM/Splunk     Analytics
                   ServiceNow      Power BI
```

---

### 4.2 Data Export Methods

| Method | Destination | Use Case | Real-time |
|--------|-------------|----------|-----------|
| **Data Export Rules** | Storage, Event Hub | Continuous export | Yes |
| **Logic Apps** | Any API endpoint | Workflow automation | Yes |
| **Azure Data Explorer** | ADX clusters | Long-term analytics | Scheduled |
| **Power BI** | Power BI workspaces | Visualization | Scheduled |
| **REST API** | Custom applications | Programmatic access | On-demand |
| **Archive Tier** | Low-cost storage | Compliance retention | No (restore) |

---

### 4.3 Data Export Rules (Current Method)

**Configuration:**
1. Navigate to: `Workspace → Settings → Data Export`
2. Create rule with:
   - **Tables**: Select which tables to export
   - **Destination**: Storage Account or Event Hub
   - **Filter**: Optional time-based filter

**Example Destinations:**
- **Storage Account**: Archive for compliance (cheaper than LAW)
- **Event Hub**: Stream to Splunk, Sentinel, or custom apps
- **Both**: Dual destination for archive + streaming

---

### 4.4 Integration Scenarios

**To SIEM (Splunk, QRadar):**
```
LAW → Event Hub → SIEM Connector → SIEM Platform
```

**To ServiceNow:**
```
LAW → Logic App (scheduled query) → ServiceNow API → Incident
```

**To Power BI:**
```
LAW → Power BI Connector → Direct Query or Import
```

**To Azure Data Explorer:**
```
LAW → Continuous Export → ADX → Cross-cluster queries
```

---

## 5. Existing Logs Flow vs. After Introducing AMPLS

### 5.1 Before AMPLS (Public Endpoint)

```
┌─────────────────────────────────────────────────────────┐
│               On-Premises Environment                   │
│                                                         │
│  ┌──────────┐         ┌──────────────┐                  │
│  │   VM     │────────>│  Firewall    │                  │
│  │ (Agent)  │         │ (Allow 443)  │                  │
│  └──────────┘         └──────────────┘                  │
└─────────────────────────────┬───────────────────────────┘
                              │ Public Internet
                              │ (Egress charges apply)
                              ▼
          ┌───────────────────────────────────┐
          │  *.ods.opinsights.azure.com       │
          │  (Public IP: 13.x.x.x)            │
          └───────────────────────────────────┘
                              │
                              ▼
          ┌───────────────────────────────────┐
          │   Log Analytics Workspace (LAW)   │
          │   (Public Endpoint)               │
          └───────────────────────────────────┘
```

**Characteristics:**
- ❌ DNS resolves to **public IP**
- ❌ Uses public endpoint (even though traffic may stay on Microsoft backbone)
- ❌ No network isolation
- ❌ Requires outbound internet (NSG/firewall rules)
- ❌ Subject to public egress charges
- ❌ Exposed to internet threats

---

### 5.2 After AMPLS (Private Endpoint)

```
┌─────────────────────────────────────────────────────────────────┐
│                  On-Premises Environment                        │
│                                                                 │
│  ┌──────────┐         ┌──────────────┐                          │
│  │ User PC  │────────>│   On-Prem    │                          │
│  │          │         │  DNS Server  │                          │
│  └──────────┘         │ 192.168.1.10 │                          │
│                       └──────────────┘                          │
└─────────────────────────────┬───────────────────────────────────┘
                              │ ExpressRoute / VPN
                              │ (Private connectivity)
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                    Azure VNet - Hub                            │
│                                                                │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │  CI DNS VMs         │    │  SI DNS VMs         │            │
│  │  10.1.0.4/10.1.0.5  │    │  10.2.0.4/10.2.0.5  │            │
│  └──────────┬──────────┘    └──────────┬──────────┘            │
│             │                          │                       │
│             ▼                          ▼                       │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │  Private DNS Zone   │    │  Private DNS Zone   │            │
│  │  privatelink.*.     │    │  privatelink.*.     │            │
│  │  opinsights...      │    │  opinsights...      │            │
│  └──────────┬──────────┘    └──────────┬──────────┘            │
│             │                          │                       │
│             ▼                          ▼                       │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │  Private Endpoint   │    │  Private Endpoint   │            │
│  │   10.1.100.50       │    │   10.2.100.50       │            │
│  └──────────┬──────────┘    └──────────┬──────────┘            │
│             │                          │                       │
│             ▼                          ▼                       │
│  ┌─────────────────────┐    ┌─────────────────────┐            │
│  │    AMPLS-CI         │    │    AMPLS-SI         │            │
│  └──────────┬──────────┘    └──────────┬──────────┘            │
└─────────────┼──────────────────────────┼───────────────────────┘
              │                          │
              ▼                           ▼
   ┌──────────────────┐       ┌──────────────────┐
   │     LAW-CI       │       │     LAW-SI       │
   │ (Central India)  │       │ (South India)    │
   └──────────────────┘       └──────────────────┘
```

**Characteristics:**
- ✅ DNS resolves to **private IP** (10.1.100.50, 10.2.100.50)
- ✅ Traffic stays within **Azure backbone / VPN**
- ✅ Full network isolation and security
- ✅ No internet breakout required
- ✅ On-prem access via ExpressRoute/VPN
- ✅ No public egress charges
- ✅ Supports multi-region with regional Private Endpoints

---

### 5.3 DNS Resolution Flow (Your Architecture)

**Query Path:**
```
On-prem PC (192.168.1.10)
    │
    │ Query: workspace-id.ods.opinsights.azure.com
    ▼
On-prem DNS Server (192.168.1.10)
    │
    │ Conditional Forwarder for *.ods.opinsights.azure.com
    ▼
CI DNS VMs (10.1.0.4, 10.1.0.5) OR SI DNS VMs (10.2.0.4, 10.2.0.5)
    │
    │ Query Private DNS Zone
    ▼
Private DNS Zone (privatelink.ods.opinsights.azure.com)
    │
    │ Returns A Record
    ▼
Private Endpoint IP (10.1.100.50 or 10.2.100.50)
```

---

### 5.4 Key Differences Summary

| Aspect | Before AMPLS (Public) | After AMPLS (Private) |
|--------|----------------------|----------------------|
| **DNS Resolution** | Public IP (13.x.x.x) | Private IP (10.x.x.x) |
| **Traffic Path** | Public internet | Azure backbone/VPN |
| **Network Isolation** | None | Full isolation |
| **Internet Requirement** | Required | Not required |
| **Egress Charges** | Yes | No |
| **Security** | Exposed to internet | Private connectivity |
| **Latency** | Variable | Lower and consistent |
| **Access Control** | Public endpoint policies | AMPLS scope control |
| **Compliance** | Limited | Enhanced (data stays private) |

---

## 6. Bandwidth Consumption for LAW

### 6.1 Typical Bandwidth by Workload

| Workload Type | Average Volume | Bandwidth (sustained) |
|---------------|----------------|----------------------|
| **Basic VM** | 1-5 GB/month | 0.01-0.05 Mbps |
| **Application Server** | 10-50 GB/month | 0.1-0.5 Mbps |
| **Container Cluster (AKS)** | 50-500 GB/month | 0.5-5 Mbps |
| **High-volume Application** | 500+ GB/month | 5+ Mbps |

---

### 6.2 Bandwidth by Log Type

| Log Type | Typical Rate | Bandwidth Impact |
|----------|-------------|------------------|
| **Performance Counters** | 1 KB/min | Very Low |
| **Windows Event Logs** | Varies (depends on verbosity) | Low-Medium |
| **IIS Logs** | 1-5 KB per request | Medium-High |
| **Custom App Logs** | Highly variable | Low-High |
| **Container Logs** | 10-100 MB/hour per pod | High |
| **Security Logs** | 5-20 KB per event | Medium |

---

### 6.3 Bandwidth Estimation Formula

```
Daily Bandwidth (MB/s) = (Total GB per day × 1024 MB) / 86400 seconds
```

**Example Calculation:**
- 100 GB/month = ~3.33 GB/day
- 3.33 GB/day = 3,407 MB/day
- 3,407 MB / 86,400 seconds = **0.039 MB/s** = **39 KB/s sustained**

**With compression** (typical 3-5x reduction):
- Actual bandwidth: ~**10-13 KB/s**

---

### 6.4 Bandwidth with AMPLS Architecture

**Characteristics:**
- Bandwidth flows through Private Endpoints (not public internet)
- No impact on corporate internet bandwidth
- Utilizes Azure ExpressRoute/VPN capacity
- Monitor using Private Endpoint metrics:
  - `Bytes In`
  - `Bytes Out`
  - `Connections`

**Monitoring:**
```kql
AzureMetrics
| where ResourceType == "MICROSOFT.NETWORK/PRIVATEENDPOINTS"
| where MetricName in ("BytesIn", "BytesOut")
| summarize avg(Average), max(Maximum) by bin(TimeGenerated, 5m)
```

---

### 6.5 Optimization Strategies

| Strategy | Impact | Implementation |
|----------|--------|----------------|
| **Use DCRs to filter logs** | 30-70% reduction | Filter unnecessary events at source |
| **Configure appropriate intervals** | 20-40% reduction | Adjust collection frequency |
| **Use Basic tier for high-volume logs** | 50% cost reduction | Change table plan |
| **Enable compression** | 3-5x reduction | Enabled by default |
| **Transform data at ingestion** | 20-50% reduction | Remove unnecessary fields |

---

### 6.6 Bandwidth Monitoring Query

```kql
// Monitor ingestion volume by table
Usage
| where DataType != ""
| summarize IngestedGB = sum(Quantity) / 1024 by DataType, bin(StartTime, 1d)
| order by IngestedGB desc

// Monitor bandwidth by source
Heartbeat
| summarize Count = count() by Computer, bin(TimeGenerated, 1h)
| extend EstimatedMB = Count * 0.001  // Rough estimate
```

---

## 7. DNS Resolution Issues with AMPLS

### 7.1 Common DNS Issues

#### Issue #1: DNS Resolves to Public IP Instead of Private IP

**Symptom:**
```bash
nslookup workspace-id.ods.opinsights.azure.com
# Returns: Public IP (13.x.x.x) instead of Private IP (10.x.x.x)
```

**Root Causes:**
- Private DNS Zone not linked to VNet
- Conditional forwarders not configured on on-prem DNS
- DNS cache contains old public IP records

**Resolution:**
1. Verify Private DNS Zone is linked to Hub VNet
2. Check conditional forwarders on 192.168.1.10:
   - Forward `*.ods.opinsights.azure.com` → 10.1.0.4, 10.1.0.5
   - Forward `*.oms.opinsights.azure.com` → 10.1.0.4, 10.1.0.5
3. Flush DNS cache:
   ```bash
   # Windows
   ipconfig /flushdns
   
   # Linux
   systemd-resolve --flush-caches
   ```

---

#### Issue #2: Missing Private DNS Zones

**Required DNS Zones for Full LAW + AMPLS Functionality:**

| DNS Zone | Purpose |
|----------|---------|
| `privatelink.monitor.azure.com` | Azure Monitor global endpoint |
| `privatelink.oms.opinsights.azure.com` | Log ingestion endpoint |
| `privatelink.ods.opinsights.azure.com` | Data access endpoint |
| `privatelink.agentsvc.azure-automation.net` | Azure Automation (Update Management) |
| `privatelink.blob.core.windows.net` | Diagnostics storage |

**Resolution:**
1. Create all required Private DNS Zones
2. Link each zone to your Hub VNet
3. Verify A records exist for your workspace endpoints

---

#### Issue #3: AMPLS Access Mode Configuration

**Problem:** AMPLS configured to allow public access, causing DNS confusion

**Access Modes:**
- **Open**: Allows both public and private access (bypasses PE for public clients)
- **Private Only**: Forces all access through Private Endpoint ✅ **Recommended**

**Check Configuration:**
1. Navigate to: `AMPLS → Network Isolation`
2. Verify: `Accept access from public networks not connected through a Private Link Scope` = **No**

**If set to Yes:**
- Clients may randomly use public endpoint
- DNS may return public IP
- Defeats purpose of AMPLS

---

#### Issue #4: Conditional Forwarder Not Working

**Verification Steps:**

```bash
# From on-prem PC, test DNS resolution through on-prem DNS
nslookup workspace-id.ods.opinsights.azure.com 192.168.1.10

# Expected: Private IP (10.1.100.50 or 10.2.100.50)
# If returns public IP: Conditional forwarder issue
```

**Resolution:**
1. Verify conditional forwarder exists on 192.168.1.10:
   - Zone: `ods.opinsights.azure.com`
   - Forward to: `10.1.0.4`, `10.1.0.5` (CI) or `10.2.0.4`, `10.2.0.5` (SI)
2. Test from on-prem DNS server directly:
   ```bash
   nslookup workspace-id.ods.opinsights.azure.com 10.1.0.4
   ```
3. Check firewall rules allow UDP 53 from on-prem to Azure DNS VMs

---

#### Issue #5: Azure DNS VM Cannot Resolve Private DNS Zone

**Symptoms:**
- On-prem to Azure DNS VM works
- Azure DNS VM cannot resolve private endpoint

**Root Cause:**
- Private DNS Zone not linked to VNet where DNS VMs reside
- DNS VMs using custom DNS instead of Azure DNS (168.63.129.16)

**Resolution:**
1. Ensure Private DNS Zones are linked to the VNet containing DNS VMs
2. Verify DNS VMs use Azure-provided DNS:
   - VNet DNS Servers: `Default (Azure-provided)`
   - OR explicitly set to: `168.63.129.16`

---

#### Issue #6: NSG Blocking Traffic

**Required NSG Rules:**

| Source | Destination | Port | Protocol | Purpose |
|--------|-------------|------|----------|---------|
| On-prem DNS | Azure DNS VMs | 53 | UDP | DNS forwarding |
| Azure DNS VMs | Private Endpoint Subnet | 443 | TCP | LAW connectivity |
| VM Subnet | Private Endpoint Subnet | 443 | TCP | Agent → LAW |

**Verification:**
```bash
# Test connectivity from Azure DNS VM to Private Endpoint
Test-NetConnection -ComputerName 10.1.100.50 -Port 443

# From on-prem or VM to workspace endpoint
Test-NetConnection -ComputerName workspace-id.ods.opinsights.azure.com -Port 443
```

---

### 7.2 DNS Troubleshooting Workflow

```
┌─────────────────────────────────────────────────────────┐
│ 1. Test DNS Resolution                                  │
│    nslookup workspace-id.ods.opinsights.azure.com       │
│    Expected: Private IP (10.x.x.x)                      │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ Returns Public IP?
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Check Private DNS Zone                               │
│    - Verify zone exists                                 │
│    - Check A record for workspace                       │
│    - Confirm VNet link                                  │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ Zone OK?
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Check Conditional Forwarders                         │
│    - On-prem DNS → Azure DNS VMs                        │
│    - Test: nslookup workspace-id... 10.1.0.4            │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ Forwarders OK?
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Check AMPLS Configuration                            │
│    - Network Isolation = Private Only                   │
│    - Workspace linked to AMPLS                          │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ AMPLS OK?
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Check NSG Rules                                      │
│    - Allow DNS (53)                                     │
│    - Allow HTTPS (443)                                  │
│    - Test: Test-NetConnection -Port 443                 │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ NSG OK?
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Flush DNS Cache                                      │
│    - ipconfig /flushdns (Windows)                       │
│    - systemd-resolve --flush-caches (Linux)             │
└─────────────────────────────────────────────────────────┘
```

---

### 7.3 Verification Commands

**From On-Premises PC:**
```bash
# Test DNS resolution through on-prem DNS
nslookup workspace-id.ods.opinsights.azure.com 192.168.1.10

# Expected output:
# Name:    workspace-id.ods.opinsights.azure.com
# Address: 10.1.100.50  (or 10.2.100.50)
```

**From Azure DNS VM:**
```bash
# Test direct resolution
nslookup workspace-id.ods.opinsights.azure.com

# Test connectivity
Test-NetConnection -ComputerName workspace-id.ods.opinsights.azure.com -Port 443

# Expected: TcpTestSucceeded : True
```

**From Azure VM with Agent:**
```bash
# Test agent connectivity
Test-NetConnection -ComputerName workspace-id.ods.opinsights.azure.com -Port 443

# Check agent logs (Windows)
Get-Content "C:\WindowsAzure\Logs\TransparentInstaller.log"

# Check agent logs (Linux)
tail -f /var/opt/microsoft/omsagent/log/omsagent.log
```

---

### 7.4 DNS Configuration Checklist

- [ ] Private DNS Zones created for all required endpoints
- [ ] Private DNS Zones linked to Hub VNet
- [ ] A records exist in Private DNS Zones for workspace endpoints
- [ ] Azure DNS VMs use Azure-provided DNS (168.63.129.16)
- [ ] On-prem DNS has conditional forwarders to Azure DNS VMs
- [ ] AMPLS configured for "Private Only" access
- [ ] Workspace linked to AMPLS
- [ ] NSG allows UDP 53 (DNS) from on-prem to Azure DNS VMs
- [ ] NSG allows TCP 443 from VMs to Private Endpoint subnet
- [ ] DNS cache flushed on client machines
- [ ] Connectivity tested with `Test-NetConnection` or `nslookup`

---

## Additional Resources

### Official Microsoft Documentation

- [Azure Monitor Private Link Scope Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/private-link-security)
- [Configure AMPLS](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/private-link-configure)
- [Private Link Design](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/private-link-design)
- [Troubleshoot AMPLS](https://learn.microsoft.com/en-us/azure/private-link/troubleshoot-private-link-connectivity)

### Cost Optimization

- [Log Analytics Pricing](https://azure.microsoft.com/en-us/pricing/details/monitor/)
- [Private Endpoint Pricing](https://azure.microsoft.com/en-us/pricing/details/private-link/)

### Architecture Guides

- [AMPLS Complete Architecture with Precise Forwarding](https://shaleen-wonder-ent.github.io/user-guides/AMPLS-Complete-Architecture-with-Precise-Forwarding.html)
- [AMPLS Component Placement Guide](https://shaleen-wonder-ent.github.io/user-guides/AMPLS-Component-Placement-Guide.html)
- [AMPLS Architecture: Hub PE vs Spoke PE](https://shaleen-wonder-ent.github.io/user-guides/AMPLS_Arch_HUB-PE_vs_Spoke-PE.html)

---

