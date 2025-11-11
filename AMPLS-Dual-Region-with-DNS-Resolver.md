# Private, Dual-Region Log Analytics Architecture using AMPLS, Private Endpoints, and DNS Resolver Hub

This document explains the architecture, the purpose of each component, and why it is used. The goal is to help customers understand how Azure Monitor, AMPLS, Private Endpoints, DNS, and Log Analytics work together across two regions.

---

## Architecture

<img width="1133" height="554" alt="image" src="https://github.com/user-attachments/assets/5398ab69-f742-4b71-aa90-33a5fd582e7b" />


## 1. High-Level Objective
The solution provides:
- Reliable and private ingestion of monitoring data.
- Dual-region Log Analytics Workspaces (LAW) for disaster recovery.
- Secure DNS resolution using the customer's hub-and-spoke model.
- Private endpoints for Azure Monitor services.

This design ensures **no public network paths**, **regional high availability**, and **cross-region failover**.

---

## 2. Hub-and-Spoke Layout
### **Hub VNet (Customer's Hub)**
This is where shared services live. We deploy:
- **DNS Private Resolver (Inbound + Outbound)**
- **DNS Forwarding Ruleset**
- **Firewall/NVA** that inspects traffic

Why here? Because all spokes can use the same DNS resolver without duplicating services.

---

## 3. DNS Private Resolver
### **Inbound Endpoint**
- All VMs and agents in spokes send DNS queries to this IP.
- Provides a single control point for DNS.

### **Outbound Endpoint**
- Used for forwarding unresolved queries.
- Allows forwarding to on-prem or external DNS systems.

### **Resolver Ruleset**
Controls DNS behavior:
- `privatelink.*` zones → resolved in Azure Private DNS.
- `*.azure.com` → resolved by Azure DNS.
- `corp.local`, `internal.company.com` → forwarded to on-prem DNS.

This creates a **bridge** between Azure and corporate DNS.

---

## 4. Private DNS Zones
Azure Monitor private endpoints require private DNS zones such as:
- `privatelink.monitor.azure.com`
- `privatelink.ods.opinsights.azure.com`
- `privatelink.oms.opinsights.azure.com`
- `privatelink.agentservice.azure-automation.net`

These zones are linked to all VNets so DNS resolver can read their records.

Why needed?
- When agents connect to Azure Monitor privately, DNS must resolve the private endpoint IPs.

---

## 5. Spoke VNets (Central & South)
Each spoke hosts:
- VM/AMA agent
- Private Endpoints created by AMPLS
- AMPLS (Azure Monitor Private Link Scope)
- LAW (Log Analytics Workspace)
- Firewall/NVA

Why separate regions?
- Regional redundancy
- Local ingestion for lower latency

---

## 6. AMPLS (Azure Monitor Private Link Scope)
AMPLS maps Azure Monitor resources to private endpoints.
Each region has its own AMPLS.

Why?
- Forces Azure Monitor traffic to stay on private network.
- Creates private endpoints for ingestion.

---

## 7. Private Endpoints per Region
Each AMPLS creates private endpoints:
- `Monitor`
- `OMS`
- `ODS`
- `AgentService`

Why?
- These endpoints replace *public* endpoints.
- Ensures traffic stays inside your VNet.

---

## 8. Log Analytics Workspace (LAW)
Two LAW instances exist:
- One in Central
- One in South

Each AMPLS points to its local LAW.

Why separate LAW?
- Regional HA/DR.
- If one region fails, logs are still available.

---

## 9. Dual Ingestion
Azure Monitor Agent (AMA) can send data to **two LAW workspaces simultaneously**.

Why?
- To ensure logs exist even if one region is completely unavailable.
- Preventing data loss during disaster.

---

## 10. Firewalls Between VNets
Each spoke has its own firewall/NVA.

Why?
- Customers often enforce security per VNet.
- DNS needs to pass through firewall to reach the Hub resolver.
- Supports inspection and compliance policies.

---

## 11. Summary of Why This Architecture Works
This solution ensures:

- Full private connectivity for Azure Monitor services
- No public internet exposure
- Hub-and-spoke friendly (shared resolver)
- Centralized DNS policy enforcement
- Dual ingestion for HA/DR
- Regional private endpoints for local failover
- Corporate DNS compatibility

You now have a fully private, secure, and DR-ready Azure Monitor deployment.

