# Securing Azure Dev/Test Resources Against Public Access

**Summary:** Protecting Critical and High priority Azure Dev/Test resources from public exposure requires a combination of network design, service configuration, and governance. The objective is simple to state and harder to deliver: **no direct connectivity from the public internet** to in-scope resources. Achieve it by (1) eliminating internet-facing endpoints, (2) routing all access through private networking (Private Endpoints + private DNS) or secure proxies (Azure Bastion / VPN), and (3) enforcing and proving the control with Azure Policy and continuous monitoring.

This document covers risk context, an architectural strategy, resource-specific step-by-step controls, a governance/enforcement model (including how to handle environments where Critical/High resources are *mixed* with lower-tier resources), and the trade-offs to plan for.

---

## Guiding Principle

Do not rely on people configuring resources correctly. **Enforce desired state with Azure Policy** so non-compliant resources cannot be created in the first place — but only after the *private path* exists, otherwise you will break connectivity.

The work breaks into three layers:

1. **Eliminate public network exposure** on the resources themselves (VMs, App Services, Storage, Key Vault, SQL, etc.).
2. **Provide a private path** (Private Endpoints + Private DNS) so workloads keep functioning.
3. **Enforce and monitor** with Azure Policy initiatives and Microsoft Defender for Cloud.

---

## 1. Public Access Risks in Dev/Test Environments

**What is "public access"?** Many Azure IaaS and PaaS resources are reachable directly from the public internet by default — e.g., a VM with a public IP and a permissive NSG rule, or a storage account / Key Vault / SQL server with "public network access" enabled. Public endpoints are discoverable and can be scanned, brute-forced, or exploited. Non-production environments are frequently targeted as easier footholds, so Critical/High Dev/Test assets (proprietary code, test data, sensitive config) should **not** be openly accessible.

**Risks by resource type:**

- **Virtual Machines:** A public IP with open RDP (3389) or SSH (22) invites brute-force login attempts and exploitation of unpatched OS vulnerabilities.
- **App Services (Web Apps / Functions):** Get a public `<app>.azurewebsites.net` endpoint by default; sensitive logic or admin endpoints can be exposed if app-level auth is weak.
- **Storage, Databases, Key Vaults:** Allow public connectivity by default. Combined with weak keys or misconfigured ACLs, public reachability can lead to data exposure and credential-based attacks.

**Goal:** Restrict public access for all Critical and High Dev/Test resources so access occurs **only** through private networks or secure, authenticated proxies.

---

## 2. Architectural Strategy: Isolate Resources & Use Private Endpoints

The foundation is **isolation**, aligned with a Zero Trust model (no implicit trust based on network location).

- **VNet segmentation:** Place Critical/High resources in private VNets, reachable only from internal networks or VPN. Use subnets to segregate by role/sensitivity and apply NSGs between them.
- **Private Endpoints for PaaS:** Use Azure Private Link to give each PaaS resource (Storage, SQL, Key Vault, App Service) a private IP inside your VNet, keeping traffic on the Azure backbone. Then **disable public network access** on the resource.
- **Private DNS:** Wire up Private DNS Zones (e.g., `privatelink.blob.core.windows.net`, `privatelink.vaultcore.azure.net`, `privatelink.database.windows.net`, `privatelink.azurewebsites.net`) and link them to the VNet so FQDNs resolve to private IPs. **This step is the one most often missed** — without it, services become unreachable even from inside the VNet.
- **Bastion or VPN for management:** Replace open RDP/SSH and public IPs with **Azure Bastion** (browser-based RDP/SSH over TLS via the portal) or a **point-to-site/site-to-site VPN**.

**Outcome:** All inbound traffic is funneled through controlled, authenticated channels. The only way to reach a resource is to be on the private network or to arrive via an approved path (authenticated Bastion session or VPN).

---

## 3. Implementation: Securing Each Resource Type (Step-by-Step)

> Plan these changes during a maintenance window, and **always add the private path/allow rules before disabling public access**.

### 3.1 Virtual Machines — Remove Public IPs & Secure Remote Access

**Best practice:** No Critical VM should have a public IP or permanently open management ports. Use a NAT gateway / Azure Firewall for controlled *outbound* access; use Bastion or JIT for *inbound* management.

1. **Audit for public IPs:** Use Azure Resource Graph or the portal (Network Interfaces) to find NICs with public IPs and NSGs allowing `Any`/`Internet` inbound. Detach public IPs from Critical VMs (IP configuration → set Public IP to **None**) — *ensure Bastion/VPN access exists first.*
2. **Close inbound ports:** Remove NSG rules allowing broad inbound internet (`*`, `Any`, `0.0.0.0/0`). If a service genuinely must be reachable, restrict to specific source ranges (office/VPN egress).
3. **Deploy Azure Bastion:** Create a Bastion host in the VNet with a dedicated subnet named **`AzureBastionSubnet` (minimum /26)**, then use **Connect → Bastion** on the VM for in-browser RDP/SSH with no VM public IP.
   - *Note on SKUs:* The **Developer SKU** is a low/no-cost, Microsoft-managed shared option that requires **no dedicated subnet and no public IP**, but it supports **one VM/session at a time and no VNet peering** — suitable for light dev/test only. For multiple concurrent sessions, peered VNets, or advanced features, use **Basic/Standard/Premium** (which *do* require the `AzureBastionSubnet`).
4. **Enable JIT VM access** (Microsoft Defender for Cloud): keeps management ports closed and opens them on-demand, time-bound, to approved IPs — useful for local RDP/SSH clients without permanent exposure.

**Result:** Critical VMs are reachable only via Bastion or JIT-approved paths, neutralizing RDP/SSH brute-force exposure.

### 3.2 Azure App Services — Private Endpoint / Access Restrictions

**Best practice:** Bring App Services into the VNet or restrict their public endpoint.

1. **VNet Integration (outbound):** Configure if the app must call back-end resources in a VNet (this controls *outbound*, not inbound).
2. **Create a Private Endpoint:** Web App → Networking → Private Endpoint; choose VNet/subnet. Then create/link a Private DNS Zone for `privatelink.azurewebsites.net` so `<app>.azurewebsites.net` resolves privately.
3. **Disable public access:** Networking → set **Public network access = Disabled** (ARM property `publicNetworkAccess`). Unapproved clients receive HTTP 403.
   - *Fallback if you can't fully disable:* Use **Access Restrictions** — add allow rules for specific IP ranges or VNet subnets (Service Endpoints), default action **Deny**.

**Result:** Internet users can't reach the app; authorized users connect via VPN or a VNet-resident client.

### 3.3 Azure Storage Accounts — Private Endpoints & Firewall

**Best practice:** Stop unrestricted public access. Prefer **Public network access: Disabled** (private endpoint only); use **Selected networks** only if specific ranges are genuinely required.

1. **Review current settings:** Storage Account → Networking. If set to **All networks (Enabled)**, change it.
2. **Create Private Endpoint(s):** One per needed sub-resource (blob, file, queue, table). Confirm DNS records (`privatelink.blob.core.windows.net`, etc.) resolve to the private IP.
3. **Restrict public access:** Set **Public network access = Disabled** (or **Selected networks** with explicit allow rules). Also set **Allow blob anonymous access = Disabled** for Critical data.

**Result:** The account no longer accepts connections from the internet, protecting data even if keys leak or a container ACL is mis-set.

### 3.4 Azure Databases (Azure SQL and similar) — Private Access & Firewall

**Best practice:** Disable public connectivity; use Private Endpoints + firewall rules.

1. **Create Private Endpoint:** On the logical SQL server → Networking; update DNS (`privatelink.database.windows.net`) so the server FQDN resolves privately.
2. **Disable public network access:** Networking → **Public network access = Disable** (private endpoint only). If some external access is unavoidable, use **Selected networks** with explicit IP rules, then revert to Disabled when possible.
3. **Other DB services:** Same pattern — Cosmos DB ("Enable public network" toggle off), Managed SQL Instance (no public endpoint), Key Vault ("Allow public access = No" once a private endpoint exists).

**Result:** Databases are reachable only via the private path; off-network connection attempts fail.

---

## 4. Governance & Enforcement: Making "No Public Access" Stick

Remediation fixes today; **policy prevents drift tomorrow.**

### 4.1 Azure Policy (the enforcement engine)

Bundle relevant built-in definitions into a single **Policy Initiative** (e.g., "Restrict Public Access — Critical/High") and assign it to the appropriate scope. Relevant built-ins include:

- **Storage** — "Storage accounts should disable public network access" / "Storage account public access should be disallowed"
- **Key Vault** — "Azure Key Vault should disable public network access" / "Key Vaults should use private link"
- **SQL** — "Public network access on Azure SQL Database should be disabled"
- **VMs / Network** — "Network interfaces should not have public IPs"; restrict allowed resource types to block public IP creation
- **Private Link coverage** — the "\[service\] should use private link" family across PaaS services

**Roll out Audit → Deny:** assign as **Audit** first to measure blast radius, remediate, then switch to **Deny**.

Example deny logic for storage:

```json
{
  "if": {
    "allOf": [
      { "field": "type", "equals": "Microsoft.Storage/storageAccounts" },
      {
        "field": "Microsoft.Storage/storageAccounts/publicNetworkAccess",
        "notEquals": "Disabled"
      }
    ]
  },
  "then": { "effect": "deny" }
}
```

### 4.2 Targeting Critical/High When Resources Are *Mixed* With Lower Tiers

When Critical/High resources are **not** cleanly separated by subscription or management group, scope-based assignment risks breaking lower environments. Two approaches:

**Option A (recommended for speed): Tag-driven targeting.**
1. Tag in-scope resources, e.g., `Criticality = Critical` or `High`.
2. Write/assign policies that only **Deny** when the tag is present, leaving untagged (lower-tier) resources untouched.
3. Enforce tagging discipline: add a policy that **requires** the criticality tag on new resources, and use a `Modify`/`Append` policy to inherit the tag from the resource group. Run a one-time tagging exercise on existing resources.

Example — deny public storage **only** for tagged resources:

```json
{
  "if": {
    "allOf": [
      { "field": "type", "equals": "Microsoft.Storage/storageAccounts" },
      {
        "anyOf": [
          { "field": "tags['Criticality']", "equals": "Critical" },
          { "field": "tags['Criticality']", "equals": "High" }
        ]
      },
      {
        "field": "Microsoft.Storage/storageAccounts/publicNetworkAccess",
        "notEquals": "Disabled"
      }
    ]
  },
  "then": { "effect": "deny" }
}
```

> Tag-based enforcement is only as strong as tagging discipline — always pair it with a "require tag" policy.

**Option B (target state): Separate Critical/High into dedicated subscriptions/management groups.** Cleaner long-term governance, but a migration effort. Treat Option A as "get secure now" and Option B as the roadmap.

### 4.3 Infrastructure as Code & Blueprints

Bake these rules into ARM/Bicep/Terraform and bundle the policies (e.g., via Azure Blueprints or a management-group assignment) so new subscriptions inherit "no public access" automatically.

### 4.4 Monitoring with Microsoft Defender for Cloud

Enable Defender for Cloud on Dev/Test subscriptions. It continuously flags risky exposures ("public network access should be disabled," "storage accounts should restrict network access," "management ports should be closed"), provides **Just-in-Time VM access**, and surfaces a **secure score** that doubles as compliance evidence. Treat any public-exposure recommendation on a Critical resource as high priority.

---

## 5. Auditing & Remediating an Existing Environment

1. **List all public IPs** (Azure Resource Graph or portal) and assess necessity.
2. **Find permissive NSG rules** (`Any` / `0.0.0.0/0` inbound) via Resource Graph or Network Watcher; tighten or remove.
3. **Check PaaS networking settings** on every storage account, SQL server, Key Vault, App Service; set to **Selected networks** or **Disabled**.
4. **Test connectivity after changes:**
   - Confirm authorized access still works (Bastion to a VM, internal client to a web app, VNet client to a database).
   - Confirm public endpoints now **deny** traffic from outside allowed networks (HTTP 403 for web apps, no RDP response, SQL public-interface-disabled errors, etc.).

---

## 6. Recommended Rollout Sequence

1. **Verify Defender for Cloud** is enabled (reporting + JIT). Enable if off.
2. **Tagging exercise** — apply `Criticality = Critical/High` and add a policy requiring the tag going forward.
3. **Assess** — inventory which in-scope resources currently expose public access.
4. **Confirm the private path** for the target environment (VNet peering / standalone VNet + Private DNS Zones). Build if missing — this is a **prerequisite**.
5. **Audit mode** — assign the (tag-scoped) initiative as **Audit** to measure blast radius safely.
6. **Remediate** — deploy Private Endpoints + DNS; set `publicNetworkAccess = Disabled`; replace VM public IPs with Bastion/JIT.
7. **Enforce** — switch the initiative to **Deny** for in-scope resources.
8. **Report** — use Policy compliance + Defender secure score as ongoing evidence.

---

## 7. Challenges & Trade-offs

- **Developer connectivity:** Remote devs/testers may need VPN or Bastion to reach internal-only resources. Plan onboarding (VPN profiles, Bastion training).
- **DNS correctness:** Private Endpoints require correct Private DNS Zone configuration; without it, services are unreachable even inside the VNet. **This is the #1 cause of "disable public access" outages.**
- **Cost & performance:** Private Endpoints and Bastion carry a small hourly cost; routing through VPN/Bastion can add minor latency. Usually negligible at dev/test scale — validate for your use cases.
- **Egress vs ingress:** Restricting *inbound* doesn't require blocking *outbound*. If a third party must reach the environment (e.g., a SaaS webhook), allow the narrowest possible scope (specific IP + port).
- **Phased rollout:** Restrict a non-critical subset first, gather feedback, then extend to all Critical/High resources. Communicate changes clearly (e.g., "use Bastion instead of direct RDP").

---

## Summary

Restricting public access for Critical and High Dev/Test resources is primarily a **prerequisites-and-governance** effort, not a difficult technical one. The enforcement (Azure Policy with Audit→Deny, tag-scoped when environments are mixed) is straightforward; the real work is **tagging discipline**, **standing up the private networking path (Private Endpoints + Private DNS)**, and **providing secure management access (Bastion/JIT)** before flipping resources to private-only.
