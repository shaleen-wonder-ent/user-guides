# TCS AVD Discovery Questionnaire (Customer-ready)

**Audience:** TCS DIS Business Team  
**Purpose:** Capture **assumptions used to draft the initial AVD landing zone + BOM inputs** and list the **questions TCS should confirm** so the solution becomes concrete.  
**Date:** 2026-01-22  
**Primary region:** Singapore

---

## 0) Current baseline (what we are designing)
- AVD delivery model: **RemoteApp** (not full desktop)
- Access method: **Web Client only** (no thick client)
- Authentication: **MFA only** (via customer Entra ID Conditional Access)
- Session host join: **Entra ID–joined**
- Workload: **Web-based applications only**, **no media**
- Connectivity: End user access via **S2S VPN / IPSec tunnel**
- Data residency constraint: **Data must not flow outside Singapore region**
- DR requirement: **Within Singapore only** (no out-of-country DR)

---

## 1) Assumptions used (must be validated by TCS)
These assumptions were used to shape the sizing approach and to enable a first-pass BOM model.

### 1.1 Users & concurrency
- **Total users (named):** 600
- **Peak concurrent users:** 300
- **Average business-hours prod host count (BOM assumption):** **10** (autoscale average during Mon–Fri 08:00–20:00 SGT)

### 1.2 AVD model
- **Host pools:** pooled (shared)
- **RemoteApps:** **Edge + Chrome** as separate RemoteApps
- **Client:** Web Client only

### 1.3 Session hosts & disks
- **OS:** Windows 11 Enterprise multi-session
- **Disk type:** Premium SSD
- **OS disk size assumption:** **P10/P15** (final selection depends on image size and performance validation)

### 1.4 Profiles / FSLogix
- **Profile technology:** FSLogix
- **Storage:** Azure Files Premium (Singapore)
- **Profile size assumption:** **5 GB per user**
- **Backup:** **No backup** assumed

### 1.5 Monitoring
- **Log ingestion assumption:** **1–3 GB/day** into Log Analytics (Singapore)
- Retention not yet specified (assume minimal until confirmed)

### 1.6 Administration
- **Bastion / jump box:** **Not assumed** (No)

---

## 2) Scope & success criteria (TCS to confirm)
1. Confirm measurable success criteria:
   - login time / RemoteApp launch time targets (p50/p95)
   - performance expectations for “web-only” browsing
2. Confirm business hours: **Mon–Fri 08:00–20:00 SGT** (used for autoscale modeling)
3. Confirm what is **in-scope vs out-of-scope** for TCS delivery and operations:
   - AVD platform build (yes/no)
   - image build + patching (yes/no)
   - FSLogix/profile storage operations (yes/no)
   - monitoring + alerting (yes/no)
   - L1/L2/L3 support responsibilities (yes/no)

---

## 3) Identity / access (customer Entra ID + MFA)
1. Confirm the customer Entra ID model:
   - Who creates/maintains security groups for AVD assignments?
   - How joiner/mover/leaver is handled?
2. Confirm Conditional Access posture:
   - MFA only (confirmed), any additional requirements? (compliant device, location/IP restrictions, etc.)
3. Confirm if any user experience/security restrictions are required:
   - clipboard, download/upload, printing, etc. (currently assumed not required)

---

## 4) Networking & connectivity (S2S VPN / IPSec)
1. Provide network topology:
   - Where does the S2S/IPSec terminate (Azure VPN Gateway / NVA / existing TCS platform device)?
2. Provide addressing and routing:
   - TCS hub/spoke CIDRs (Singapore)
   - Customer CIDRs (on-prem/VNet)
   - Route propagation expectations and any UDR requirements
3. DNS:
   - What DNS servers will session hosts use?
4. Egress/security controls:
   - Confirm whether TCS mandates Azure Firewall/NVA for outbound even though “no strict egress requirement” is stated.

---

## 5) DR and resiliency (within Singapore only)
Since data cannot flow outside Singapore and DR must remain in-country, cross-region DR is **out of scope**. Please confirm which **in-region resiliency** approach applies:

1. Availability Zones:
   - Are session hosts required to be spread across **AZs**?
2. Storage redundancy:
   - Is **ZRS** required/allowed for Azure Files (where supported) to improve zone resiliency?
3. Recovery approach:
   - Is the expectation “zone failure resiliency” or “rebuild in-region”?
4. Confirm target **RTO/RPO** for in-region events (if you have standard targets).

---

## 6) Image build & endpoint management (Entra-joined)
1. Confirm image tooling and ownership:
   - Azure Image Builder / Packer / manual
   - patch cadence and rollback plan
2. Confirm management plane:
   - Intune (recommended for Entra-joined) or alternative tooling
3. Confirm browser configuration requirements:
   - required/blocked extensions
   - homepage/bookmarks
   - any Chrome/Edge policy restrictions (to protect density/cost)

---

## 7) FSLogix / Azure Files Premium validation
1. Validate **5 GB per user** profile sizing assumption (and growth expectation).
2. Confirm performance/resiliency requirements for profile storage (LRS vs ZRS).
3. Confirm “**No backup**” assumption is acceptable.

---

## 8) Monitoring & operations
1. Confirm Log Analytics:
   - retention requirement
   - alerting requirements
2. Confirm support model:
   - L1/L2/L3 responsibilities
   - incident SLAs
3. Confirm operational runbooks required:
   - scaling failures
   - image rollback
   - FSLogix/profile incident handling
   - in-region resiliency event response (zone impact)

---

## 9) BOM inputs to confirm (to make costs concrete)
1. Confirm expected average prod host count during business hours:
   - we assumed **10** for BOM modeling
2. Confirm whether Azure Firewall/NVA and/or VPN Gateway SKUs are mandated (big cost drivers).
3. Confirm storage redundancy choice (LRS vs ZRS) and retention/logging requirements.
4. Confirm any constraints on VM families/SKUs in Singapore.

---
