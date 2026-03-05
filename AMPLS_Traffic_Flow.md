# Azure Monitor Private Link Scope (AMPLS) - Traffic Routing & Bandwidth Guide

---

## Table of Contents

- [Executive Summary](#executive-summary)
- [1. AMPLS Traffic Routing Behavior](#1-ampls-traffic-routing-behavior)
- [2. Use Case 1: Application Ingestion](#2-use-case-1-application-ingestion)
- [3. Use Case 2: User Queries LAW](#3-use-case-2-user-queries-law)
- [4. Cost Implications](#4-cost-implications)
- [5. Forcing Traffic Through Firewall (Optional)](#5-forcing-traffic-through-firewall-optional)
- [6. Bandwidth Requirements](#6-bandwidth-requirements)
- [7. Measuring Bandwidth Usage](#7-measuring-bandwidth-usage)
- [8. Bandwidth Optimization Best Practices](#8-bandwidth-optimization-best-practices)
- [9. Official Microsoft Documentation](#9-official-microsoft-documentation)
- [10. FAQ](#10-faq)
- [11. Verification Steps](#11-verification-steps)
- [12. Summary Diagram](#12-summary-diagram)

---

## Executive Summary

This document explains the traffic routing behavior and bandwidth requirements when using **Azure Monitor Private Link Scope (AMPLS)** with Log Analytics Workspace (LAW) and Application Insights.

### Key Findings

- ✅ **AMPLS traffic bypasses Azure Firewall by design** (due to `/32` system routes)
- ✅ **Traffic stays on Azure private backbone** (never touches public internet)
- ✅ **No egress/bandwidth charges** for Private Link traffic (note: **Private Link “data processed” pricing may still apply** depending on your setup/region; see pricing links)
- ✅ **Typical query result sizes are highly workload-dependent** (dashboards often KBs; investigations can be MBs)
- ✅ **Hard caps that matter for bandwidth planning exist** (API: up to ~100 MiB raw results; 500,000 rows; 10-minute max runtime)

---

## 1. AMPLS Traffic Routing Behavior

### 1.1 Why Traffic Bypasses Firewall

With AMPLS configured, **all traffic to Application Insights and Log Analytics Workspace bypasses your Azure Firewall by design**, even with a `0.0.0.0/0` User-Defined Route (UDR) pointing to Azure Firewall.

**Root Cause:** Azure's routing behavior for Private Endpoints, where **more specific system routes take precedence** over User-Defined Routes (UDRs).

### 1.2 Azure Route Selection Logic

When you create a Private Endpoint for AMPLS, Azure automatically injects a **/32 system route** for that Private Endpoint's IP address into your subnet's route table.

#### How Azure Routing Works

Azure uses **longest prefix matching** to determine which route to use:

| Route Prefix | Specificity | Priority |
|--------------|-------------|----------|
| **/32** | Most specific | **Highest** (always wins) |
| /24 | More specific | High |
| /16 | Less specific | Medium |
| /8 | Less specific | Low |
| **/0** (0.0.0.0/0) | **Least specific** | **Lowest** |

#### Example Route Table

```text
Priority  Route         Next Hop Type        Next Hop IP
1         10.1.5.10/32  InterfaceEndpoint    (Private Endpoint - System Route)
2         0.0.0.0/0     VirtualAppliance     10.0.0.4 (Firewall)
```

**Result:** Traffic destined for `10.1.5.10` (Private Endpoint) uses the **/32 route** and never evaluates the `0.0.0.0/0` route.

**Key Principle:** A `/32` route (most specific) always wins over broader routes like `/16`, `/8`, or `/0`. System routes for Private Endpoints are `/32` routes. Your default route `0.0.0.0/0` to the firewall is the least specific route possible.

---

## 2. Use Case 1: Application Ingestion

### Traffic Flow: Application → Application Insights → Log Analytics Workspace

```text
WebApp / Application
  ↓ App Insights SDK sends telemetry
DNS Resolution: api.applicationinsights.azure.com → Private IP (e.g., 10.1.5.10)
  ↓
Private Endpoint (/32 system route - bypasses firewall)
  ↓
Azure Backbone (Private Link Service)
  ↓
Application Insights Resource
  ↓
Log Analytics Workspace (internal Azure Monitor routing)
  ↓
Data stored in LAW tables (AppTraces, AppRequests, AppDependencies, AppExceptions, etc.)
```

### Result

- ✅ Traffic flows through Private Endpoint
- ✅ Stays on Azure backbone network
- ❌ **No firewall involvement**
- ❌ **No public internet exposure**

---

## 3. Use Case 2: User Queries LAW

### Traffic Flow: User → Log Analytics Workspace (Troubleshooting)

```text
User (in VNet or connected via VPN/ExpressRoute)
  ↓ Opens Azure Portal or runs KQL query
DNS Resolution: <workspace-id>.ods.opinsights.azure.com → Private IP (e.g., 10.1.5.10)
  ↓
QUERY REQUEST flows to Private Endpoint
  ↓
QUERY REQUEST flows through Azure Backbone
  ↓
QUERY EXECUTED on Log Analytics Workspace
  ↓
QUERY RESULTS flow back through Azure Backbone
  ↓
QUERY RESULTS flow through Private Endpoint
  ↓
User receives data
```

### Result

- ✅ Traffic flows through Private Endpoint
- ✅ Stays on Azure backbone network
- ❌ **No firewall involvement**
- ❌ **No public internet exposure**
- 📌 **Bandwidth is mainly driven by query result size returned to the user** (plus protocol overhead)

---

## 4. Cost Implications (Private Link vs. LAW)

### Key point for planning

- **LAW ingestion pricing** is driven by data ingested/stored/retained.
- **User query bandwidth** is driven by **query result payloads** returned to clients.
- **Private Link pricing** may include **Private Endpoint hourly** and **data processed** premiums (varies by region/offer). See official pricing details. ([azure.microsoft.com](https://azure.microsoft.com/en-us/pricing/details/private-link))

---

## 5. Forcing Traffic Through Firewall (Optional)

Should you need to inspect AMPLS traffic through your firewall for compliance or security policies, you **can** force this, but it requires additional configuration.

### Required Steps

#### Step 1: Enable Network Policies on Private Endpoint Subnet

```bash
az network vnet subnet update \
  --name <PrivateEndpointSubnet> \
  --resource-group <ResourceGroup> \
  --vnet-name <VNetName> \
  --disable-private-endpoint-network-policies false
```

#### Step 2: Create Explicit /32 UDRs for Each Private Endpoint

```bash
az network route-table route create \
  --resource-group <ResourceGroup> \
  --route-table-name <RouteTableName> \
  --name AMPLS-PE-Route \
  --address-prefix 10.1.5.10/32 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address <FirewallIP>
```

#### Step 3: Configure Azure Firewall with SNAT

Required to maintain symmetric routing and ensure return traffic flows back through the firewall.

---

## 6. Bandwidth Requirements (User Queries to LAW) — **Reference Guidance**

This section addresses the customer ask: **“What bandwidth is required when a user executes Log Analytics queries?”**

### 6.1 What “bandwidth requirement” means for LAW queries

For Log Analytics queries (Portal or API), the network requirement is primarily a function of:

1. **Result payload size** returned to the user/client
2. **Query frequency** (queries per minute) and **concurrency**
3. **Compression** (the API returns up to **64 MB compressed**, which can expand to roughly **~100 MiB raw**) ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits))
4. **UI behavior** (dashboards/workbooks may refresh automatically)

So instead of a single fixed “Mbps requirement”, planning typically uses:
- **per-user steady-state Mbps** (typical interactive usage), and
- **burst capacity** (large investigations/exports).

### 6.2 Official Microsoft service limits that bound bandwidth

Microsoft documents these **Azure Monitor Logs Query API limits** (these are important because they cap worst-case results returned to clients):

- **Maximum records returned in a single query:** 500,000 ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits))  
- **Maximum size of data returned:** ~104 MB (~100 MiB) raw (API returns up to 64 MB compressed) ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits))  
- **Maximum query running time:** 10 minutes ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits))  
- **Maximum request rate:** 200 requests per 30 seconds per Microsoft Entra user or client IP ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits))  

**Implication for bandwidth:** even if a user runs an extremely broad query, the returned data is bounded by the “max size of data returned” and record limits, which helps you plan an upper bound for bursts.

### 6.3 Practical per-user bandwidth planning (rule-of-thumb)

Since Microsoft’s docs provide hard query result limits but do **not** give a single “Mbps per user” number for Log Analytics querying, the most defensible planning approach is:

- Plan based on **expected query result sizes** for your workload (KB for dashboards, MB for investigations),
- Validate with **LAQueryLogs** and Private Endpoint metrics (see below),
- Provide headroom for concurrency and bursts.

A simple sizing formula:

```text
Sustained Mbps ≈ (AvgResultMB × QueriesPerMinute × ConcurrentUsers × 8) / 60 × (1 + overhead%)
```

Example (interactive troubleshooting):
- Avg result = 2 MB
- 1 query/min/user
- 10 concurrent users
- 20% overhead

```text
Mbps ≈ (2 × 1 × 10 × 8) / 60 × 1.2 ≈ 3.2 Mbps sustained
```

### 6.4 Use LAQueryLogs to estimate real bandwidth (supported reference table)

To measure and trend usage, Microsoft provides the **LAQueryLogs** table reference and sample queries. ([learn.microsoft.com](https://learn.microsoft.com/en-gb/azure/azure-monitor/reference/tables/laquerylogs))

Useful fields include:
- `ResponseRowCount` (rows returned)
- `ResponseDurationMs`
- `ScannedGB` (for some query types)
- `Stats*` fields indicating processing characteristics ([learn.microsoft.com](https://learn.microsoft.com/en-gb/azure/azure-monitor/reference/tables/laquerylogs))

> Note: LAQueryLogs does not directly expose “bytes returned”, so many customers approximate bytes based on row counts or export actual response payload sizes from client-side tooling. The most accurate approach for network capacity is to combine LAQueryLogs (who/when/which query patterns) with Private Endpoint “Bytes In/Out” metrics.

---

## 7. Measuring Bandwidth Usage

### Option 1: Use Private Endpoint metrics (Bytes In / Bytes Out)

Recommended for “true bandwidth” measurements because it reflects actual network transfer.

### Option 2: Enable and query LAQueryLogs (query usage analytics)

Microsoft reference:
- LAQueryLogs table reference ([learn.microsoft.com](https://learn.microsoft.com/en-gb/azure/azure-monitor/reference/tables/laquerylogs))
- Example LAQueryLogs queries ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/queries/laquerylogs))

(Keep or adapt the KQL queries you already have in this document.)

---

## 8. Bandwidth Optimization Best Practices

(Keep existing best practices; they directly reduce result payload size and therefore bandwidth.)

---

## 9. Official Microsoft Documentation (Links)

Below are **official references** you can share with the customer for bandwidth/capacity planning.

### Service limits (directly relevant to query payload size / bandwidth)

- **Azure Monitor service limits** (includes **Query API limits** like max rows, max data returned, runtime, throttling) ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits))

### Usage analytics (to measure query patterns that drive bandwidth)

- **LAQueryLogs table reference** ([learn.microsoft.com](https://learn.microsoft.com/en-gb/azure/azure-monitor/reference/tables/laquerylogs))  
- **Example queries for LAQueryLogs** ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/queries/laquerylogs))  

### Private Link cost reference (if customer asks whether “bandwidth is free”)

- **Azure Private Link pricing** ([azure.microsoft.com](https://azure.microsoft.com/en-us/pricing/details/private-link))

---

## 10. FAQ

**Q: Is there an official “Mbps per user” requirement for Log Analytics queries?**  
A: Microsoft documents **hard limits** (max records returned, max result size, runtime, throttling), but not a single fixed Mbps-per-user requirement. Bandwidth needs depend on result size, concurrency, and refresh frequency. Use the limits for upper bounds and use LAQueryLogs + Private Endpoint metrics to measure real usage. ([learn.microsoft.com](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/service-limits))

---

## 11. Verification Steps

### Check Effective Routes

```bash
az network nic show-effective-route-table \
  --resource-group <ResourceGroup> \
  --name <NIC-Name> \
  --output table
```

Look for `/32` routes pointing to `InterfaceEndpoint`.

### Test DNS Resolution

```bash
nslookup <workspace-id>.ods.opinsights.azure.com
```

Should return private IPs (10.x.x.x), not public IPs.

### Trace Network Path

```bash
tracert <private-endpoint-ip>
```

Should show direct connection to Private Endpoint, not via firewall.

### Test Connectivity

```powershell
Test-NetConnection -ComputerName <private-endpoint-ip> -Port 443
```

---

## 12. Summary Diagram

```text
┌─────────────────────────────────────────────────────────────────┐
│ AMPLS DATA FLOW - COMPLETE PICTURE                               │
└─────────────────────────────────────────────────────────────────┘

INGESTION:
WebApp → Private Endpoint (10.1.5.x) → Azure Backbone → App Insights → LAW
         ↑ /32 route                    Private Link
         └─ Bypasses Firewall

QUERY/EGRESS (bandwidth driver = result size returned to user):
User → Private Endpoint (10.1.5.x) → Azure Backbone → LAW → Results
       ↑ /32 route                    Private Link
       └─ Bypasses Firewall

MICROSOFT QUERY LIMITS (bounds for planning):
- Max rows returned: 500,000
- Max data returned: ~64 MB compressed (~100 MiB raw)
- Max runtime: 10 minutes
- Throttling: 200 req / 30 sec / user or client IP

WHAT IS BYPASSED:
❌ Azure Firewall (via /32 system route)
❌ Public Internet (stays on Azure backbone)
❌ UDR 0.0.0.0/0 (overridden by /32 route)
```
