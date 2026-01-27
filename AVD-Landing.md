# CAF / Azure Landing Zone (ALZ) Accelerator for Azure Virtual Desktop (AVD) — Design

**Primary region:** Southeast Asia (Singapore)  
**Scope:** CAF-aligned landing zone foundation + AVD workload placement for **600 users** (pooled RemoteApp via Web Client) with same-region (zone-level) resiliency.

---

## 1) Document purpose
This document describes the **enterprise-grade Cloud Adoption Framework (CAF) / Azure Landing Zone (ALZ)** target architecture and an implementation approach for an **Azure Virtual Desktop (AVD)** platform deployment.

The design aligns with Microsoft’s landing zone principles and supports:
- Security & compliance (policy guardrails, identity, logging)
- Scalability & operational consistency
- Centralized governance and cost management
- Hub-and-spoke networking for enterprise connectivity
- Same-region resiliency using **Availability Zones** (Singapore-only)

---

## 2) Key requirements and assumptions (used for this design)

### 2.1 Requirements captured
- **Delivery:** **RemoteApp** (not full desktop)
- **Client access:** **Web Client only** (no thick client)
- **Authentication:** **MFA only** (via customer Entra ID Conditional Access)
- **Workload:** web-based applications; “medium” user workload assumptions
- **Data residency:** data must **not flow outside Singapore**
- **High availability / DR:** **within Singapore only** (zone-level resiliency)
- **Security controls:** enterprise-grade controls (Firewall + Bastion included in baseline)
- **Connectivity:** VPN Gateway shown as **optional/precautionary** for future integrations

### 2.2 Capacity and cost model assumptions (the “25 + 5” model)
This is a **pooled AVD architecture** (not 1 VM per user). Multiple users share each session host.

#### Peak capacity design
Based on Microsoft guidance and common real-world deployments for medium workloads:
- A `D8as v5` (8 vCPU, 32 GB RAM) typically supports **~20–25 users**
- Using **25 × D8as v5** session hosts during peak provides approximately:
  - **25 hosts × ~24 users/host ≈ 600 concurrent users**

#### Autoscaling to control cost
To avoid paying for peak capacity 24×7, the design uses autoscaling:

- **Business hours (peak window):**  
  - **25 × D8as v5**  
  - Runs for approximately **220 hours/month**  
  - Assumption: ~22 business days/month × ~10 hours/day ≈ 220 hours

- **Off-peak (nights/weekends/low-usage):**  
  - **5 × D4as v5**  
  - Runs for approximately **510 hours/month**  
  - 730 hours/month (full month) − 220 hours ≈ 510 hours

**Result:** You pay for **~25 larger servers during office hours** and **~5 smaller servers during non-office hours**, not 30 servers all the time.

#### How load is distributed
- AVD load balances user sessions across available session hosts.
- Autoscaling starts additional VMs as load increases and stops/deallocates hosts as load decreases.
- Users are spread across the running hosts based on the configured load balancing policy.

---

## 3) Target CAF/ALZ management group structure

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Architecture" 
     src="https://github.com/user-attachments/assets/d515ffd9-591d-4da3-89cc-9b818ca16e33"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>

---

## 4) Subscription layout (recommended)

| Subscription | Purpose | Typical contents |
|---|---|---|
| **Identity** | Identity services & supporting components | Entra ID integrations, optional AAD DS/AD DS (if required), identity tooling |
| **Management** | Central operations/monitoring/security | Log Analytics, Automation (if used), Sentinel (optional), dashboards |
| **Connectivity (Hub)** | Shared network connectivity and control | Hub VNet, **Firewall**, **Bastion**, optional VPN Gateway, shared routing/DNS |
| **AVD-Prod (Workload)** | AVD resources for production | Host pools, session hosts, FSLogix storage, SIG/images, scaling plans |

> If customer prefers fewer subscriptions, **Identity + Management** can be combined, but the above is the standard ALZ separation for enterprise operations.

---

## 5) Network topology (Hub-Spoke)

### 5.1 Conceptual topology (Singapore)

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Architecture" 
     src="https://github.com/user-attachments/assets/a1c03c31-7292-4acf-b0cc-cf3223832ac4"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


### 5.2 VPN Gateway (optional / precautionary)
AVD does not strictly require VPN connectivity if all apps are public SaaS. However, **VPN Gateway is included as an optional, future-proofing item** in case private connectivity is required later for:
- VNet peering requirements
- Private integrations to another Azure environment
- Secure private back-end dependencies

If the business confirms **no private connectivity** is required, this item can be removed to reduce cost.

---

## 6) Where each service lives (ALZ placement)

| Service | Location (Subscription) |
|---|---|
| Hub VNet | Connectivity (Hub) |
| Azure Firewall | Connectivity (Hub) |
| Azure Bastion | Connectivity (Hub) |
| VPN Gateway *(optional)* | Connectivity (Hub) |
| DDoS Network Protection *(optional)* | Connectivity (Hub) |
| Log Analytics Workspace | Management |
| Microsoft Sentinel *(optional)* | Management |
| AVD host pools / app groups / workspaces | AVD-Prod |
| Session host VMs + NICs | AVD-Prod |
| Managed disks | AVD-Prod |
| FSLogix storage (Azure Files Premium **ZRS**) | AVD-Prod |
| Shared Image Gallery / Images | AVD-Prod *(or Management, per preference)* |

---

## 7) AVD workload placement and host pool model (zones / resiliency)

### 7.1 AVD placement model

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Architecture" 
     src="https://github.com/user-attachments/assets/981b91b8-6a26-4eec-96cd-14035a1b714c"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


### 7.2 High availability & DR strategy (same-region / zone-level resilience)
The solution is designed for enterprise-grade availability **inside the Singapore region** using Availability Zones.

**What we are doing**
- **Two host pools** deployed across different **Availability Zones**
- **Active/Passive model**
  - Primary pool handles user load
  - Secondary pool provides standby capacity

**Profiles**
- FSLogix profiles stored on **Azure Files Premium with ZRS**
- Profiles are replicated across zones inside Singapore to support zone failure scenarios.

**What this protects against**
- Single VM failure
- Rack failure
- Datacenter **zone** failure
- Host pool capacity issues

**In case of a zone failure**
- Autoscaling brings up capacity in the surviving zone
- Users reconnect and continue working
- User profiles remain available due to ZRS storage

> This is **intra-region DR** (zone-level resiliency), which aligns to the current “Singapore-only” requirement.

---

## 8) DR / baseline management VM (why it exists)
A small always-on VM is included for operational safety:

- **1 × D4as v5 running 24×7**
- Purpose:
  - Management access point
  - Break-glass / emergency access VM
  - Recovery and maintenance access point
- This VM is **not** part of user load capacity.

---

## 9) FSLogix storage sizing (8 TB)
We assumed approximately **10–12 GB per user profile** (e.g., Outlook/Teams/OneDrive caches + user settings). For 600 users:

- 600 × ~12 GB ≈ **7.2 TB**
- Rounded up to **8 TB** for:
  - growth
  - safety margin
  - performance headroom

Storage characteristics:
- **Azure Files Premium**
- **ZRS** (zone resilient)
- Central to user experience, therefore sized conservatively.

---

## 10) Security stack (included vs optional)

### 10.1 Included in baseline design
- **Azure Firewall** (egress control, inspection, compliance)
- **Azure Bastion** (secure admin access without public IPs)
- **NSGs + UDRs** (segmentation and routing control)
- **Azure Policy** (governance, compliance guardrails)
- Central logging via **Log Analytics** (in-region)

### 10.2 Optional (depending on customer's standards)
- **Microsoft Defender for Cloud** (recommended if not enabled centrally)
- **Microsoft Sentinel** (only if SOC/SIEM requirement exists)
- **Azure DDoS Protection** (only if mandated)

---

## 11) AVD “control plane” cost clarification (BOM guidance)
The **AVD service control plane** has no fixed infrastructure cost. Billing is driven by:
- Compute (VMs)
- Storage (profiles/disks)
- Network egress and connectivity services (Firewall, VPN, etc.)
- Monitoring and security services (Log Analytics, Defender, Sentinel if enabled)

**BOM note:**  
We model costs using the underlying billable resources (VMs, storage, network) and **do not** add the “Azure Virtual Desktop” pricing-calculator tile into totals, because it can lead to confusion/double counting when the same compute/storage items are already explicitly listed.

---

## 12) Mandatory governance policies (examples)
The following are typical ALZ guardrails; final set to be confirmed with cstomer's standards:
- Enforce allowed regions = **Southeast Asia (Singapore)** only
- Enforce required **tags** (CostCenter, Owner, AppName, Env)
- Enforce diagnostic settings to central Log Analytics
- Restrict public IP usage (where appropriate)
- Enforce security baselines (Defender for Cloud) if mandated

---

## 13) Execution plan (high level)

### Phase 1 — Foundation (1–2 weeks)
- Deploy ALZ/CAF landing zone accelerator
- Establish management groups, RBAC baseline, policy guardrails
- Deploy hub networking including Firewall + Bastion
- Deploy central Log Analytics baseline (Singapore)
- Add VPN Gateway if confirmed required (optional)

### Phase 2 — AVD Platform (1 week)
- Deploy AVD spoke VNet and peer to hub
- Deploy AVD control plane objects (host pools/app groups/workspaces)
- Deploy session hosts with autoscaling:
  - Peak: **25 × D8as v5** for ~220 hours/month
  - Off-peak: **5 × D4as v5** for ~510 hours/month
- Deploy FSLogix (Azure Files Premium **ZRS**, sized to **8 TB**)

### Phase 3 — Hardening (as required)
- Enable Defender for Cloud / Sentinel if mandated
- Refine policies (region restriction, logging, tagging)
- Document operational runbooks and support model

---

## 14) Summary (simple terms)
You are paying for:
- **~25 larger servers during office hours**
- **~5 smaller servers during non-office hours**
- **1 small always-on management VM (break-glass)**
- **~8 TB** of premium, zone-resilient profile storage (FSLogix)
- Enterprise-grade security and monitoring controls (Firewall, Bastion, logging)

This elastic capacity model is designed to support **~600 concurrent users** under the stated medium workload assumptions, while controlling cost through autoscaling.


---
Azure Virtual Desktop Landing Zone architecture

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Architecture" 
     src="https://github.com/user-attachments/assets/7426df0a-a678-48f6-9ae7-c793789652ae"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>

