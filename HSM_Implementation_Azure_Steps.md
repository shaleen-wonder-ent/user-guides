# Customer-Managed Keys on Azure Managed HSM — **Azure Portal Edition**
## Dual-Region Hub-and-Spoke Deployment Guide (click-path version of `HSM_Implementation_Steps.md`)

> **→ First time here?** Read the short companion overview [Managed HSM - Mental Image](https://shaleen-wonder-ent.github.io/user-guides/hsm-active-active-overview.html) first — it gives a one-page mental map of *what gets built and in what order*. Then come back here for the **portal-only** click-paths.
>
> **What this document is.** A 1:1 portal equivalent of the CLI-based [`HSM_Implementation_Steps.md`](./HSM_Implementation_Steps.md). Every phase, every numbered sub-step, and every design rationale is preserved — only the `az ...` command blocks are replaced with **Azure portal click-paths** (the exact blade, the exact buttons, and the exact fields to fill in). Phase numbering, section headings, and risk callouts are deliberately identical to the CLI guide so the two can be cross-referenced line-by-line.
>
> **Audience:** infrastructure, security, and platform engineers deploying Azure Managed HSM for the first time using the Azure portal (no Cloud Shell, no CLI scripting).
>
> **Example primary region:** South India · **Example DR region:** Central India *(swap for any two Azure paired regions)*
> **HSM pool name (both regions — same name, one logical HSM):** `org-hsm-prod` *(replace `org` everywhere with your organisation's short code, e.g. `acme`, `contoso`, `hcl`)*
>
> **Two unavoidable portal limitations to know up-front:**
> 1. **The Security Domain ceremony in Phase 5 is *not* fully portal-supported.** Generating the SD file (the `download` step that activates the HSM) **must** be run from `az keyvault security-domain download` on an admin workstation. The portal can show you the post-activation state but cannot generate the file. This is the *only* step in the whole guide that requires CLI; everything else is portal-driven. The CLI snippet for that one step is shown inline in [§5.2](#52-generate-encrypt-and-activate-the-hsm-via-cli---only-cli-required-step) for completeness.
> 2. **HSM full backups/restores ([§10.1](#101-hsm-full-backup-control-plane-periodic--event-triggered))** are also CLI / REST-only at present; portal exposes the backup blob list but does not initiate `az keyvault backup start`. The CLI snippets are kept inline in [§10.1](#101-hsm-full-backup-control-plane-periodic--event-triggered).
>
> Everything else — networks, peerings, Private DNS, the HSM resource itself, multi-region replication, Private Endpoints, RBAC, key creation, DES, SQL TDE, diagnostic settings, alerts, validation drills — is **100 % portal**.

---

## What this guide builds, in plain English

You are going to deploy a **highly secure, highly available cryptographic key store on Azure**, used by your applications, VMs, and databases to encrypt data with keys that **you** control — and that **no one, not even Microsoft, can read or extract**.

End state:

- **One logical Managed HSM** named `org-hsm-prod`, with **two physically separate clusters** (Primary + DR) under the same name, sharing the same keys via built-in **multi-region replication**. Same FQDN, same key URIs everywhere.
- **No public internet access** to the key store. Apps reach it only through **Azure Private Endpoints** inside your VNets, or from on-premises over **ExpressRoute**.
- **A hub-and-spoke network per region** that already exists — this deployment only adds one new PE spoke per region and peers it into the existing hub.
- **Active-Active operation** — apps in each region talk to their *local* HSM by default. Region-aware split-horizon Private DNS means the surviving region's workloads keep resolving the HSM to their **local** PE with no human DNS edit during a regional outage. The **global** public FQDN `<hsm>.managedhsm.azure.net` is fronted by an **internal, Microsoft-managed traffic routing layer** that Azure provisions automatically when you add the second region — there is **no customer-deployed Traffic Manager profile**.
- **An M-of-N quorum** (e.g. any 3 of 5 officers) controls the HSM's root key material.

## The journey, phase by phase

| Phase | What you do (portal) | Why it matters |
|---|---|---|
| **0. Subscription & identity prep** | Entra ID → Groups blade; Subscription → Resource providers blade. | Permissions baseline before any HSM resource exists. |
| **1. Primary network — extend the existing hub** | Capture existing hub/spoke IDs (read-only). Create one new HSM PE spoke VNet, peer it into the existing hub, link the central Private DNS zone. | The HSM cannot be exposed to the public internet; plug it into the existing private network without duplicating gateways/Bastion/DNS. |
| **2. DR network — extend the existing DR hub** | Same as Phase 1, in DR. Each region gets its **own** Private DNS zone (split-horizon). | Region-aware DNS makes Active-Active hands-free. |
| **3. Global wiring (DNS forwarding)** | Add one conditional-forward rule on existing on-prem DNS servers + existing forwarder VMs. | Hub-to-hub peering and DR ExR already exist. Global FQDN routing is platform-managed. |
| **4. Provision Primary HSM** | Key Vaults blade → **+ Create** → **Managed HSM**. ~20–30 min, lands in *Pending*. | The HSM itself — activated in Phase 5. |
| **5. Security Domain ceremony** | **CLI-only step** (the only one). Run `az keyvault security-domain download` from a jump VM. | The most security-critical step. Lose the SD file + M custodian keys = permanent key loss. |
| **6. DR HSM & replication** | Managed HSM resource → **Multi-Region Replication** → **+ Add Region**. ~30 min. | One logical HSM, two regions, RPO ≈ 0. |
| **7. Private Endpoints & lock-down** | One PE per region (Networking → Private endpoint connections → **+ Private endpoint**), each registered into its **own** regional Private DNS zone via the PE's DNS zone group. Then Networking → **Public network access = Disabled**. | After this, internet path is closed for good. |
| **8. Data-plane RBAC and keys** | Managed HSM → **Local RBAC** blade → assign roles; **Keys** blade → **+ Generate/Import**. | HSM-local RBAC, separate from Azure RBAC. |
| **9. Workload integration** | Disk Encryption Sets blade for VM disks; SQL MI → **Transparent Data Encryption** blade for TDE; App workload → Identity blade + SDK. | This is where encryption actually starts using your HSM keys. |
| **10. Backup, logging, alerts** | Diagnostic settings, Alerts (portal). Backup itself runs via CLI/REST. | Backups protect against accidental deletion; logs are mandatory for audit. |
| **11. DR drill & runbook** | Quarterly portal-driven validation; annual SD restore drill (CLI). | Rehearse the failover. |
| **12. Go-live checklist** | Final gates: public access disabled, soft-delete + purge protection on, backups verified, drills signed off. | Final gate before production traffic. |

---

## Concept primer (read once, refer back as needed)

| Term | Plain-English meaning |
|---|---|
| **Managed HSM** | A single-tenant, FIPS 140-3 Level 3 validated cryptographic appliance Azure runs for you. Microsoft cannot see your keys. |
| **Security Domain (SD)** | The HSM's "root of trust". Generated *inside* the HSM hardware, encrypted to N officers' public keys before it ever leaves. If lost beyond recovery, every key in the HSM dies with it. |
| **M-of-N quorum** | The "any M of N officers must cooperate" rule that protects the SD (e.g. 3-of-5). |
| **Hub-and-spoke** | Hub = shared services (gateway, firewall, Bastion, DNS). Spokes = workloads. Spokes don't talk to each other directly. |
| **Private Endpoint (PE)** | A NIC inside your VNet that gives the HSM a private IP from your address space. |
| **Private DNS zone** | An Azure-private DNS namespace. One `privatelink.managedhsm.azure.net` zone per region (split-horizon) so the HSM FQDN resolves to the **local** PE IP in each region. |
| **Platform-managed global routing** | When the second region is added, Azure provisions an **internal**, Microsoft-managed routing layer under `<hsm>.managedhsm.azure.net` that health-checks both regional pools (TTL 5 s). **Not** a customer-deployed Traffic Manager profile. |
| **Multi-region replication** | Two regional HSM clusters share the same Security Domain and continuously replicate keys and role assignments. Eventually consistent within ~6 minutes. |
| **Disk Encryption Set (DES)** | An Azure object binding a managed disk's encryption to a specific HSM key. |
| **TDE Protector** | SQL's wrap key. With CMK + Managed HSM, it lives in the HSM. |
| **Envelope encryption** | App encrypts data with a fast symmetric DEK, then encrypts the DEK with an HSM-held key. |

---

## Table of Contents

1. [Overview & Design Decisions](#1-overview--design-decisions)
2. [Prerequisites](#2-prerequisites)
3. [Naming, IP Plan, and Variables](#3-naming-ip-plan-and-variables)
4. [Phase 0 — Subscription, Identity, and RBAC Preparation](#phase-0--subscription-identity-and-rbac-preparation)
5. [Phase 1 — Network Foundation (Primary Region)](#phase-1--network-foundation-primary-region)
6. [Phase 2 — Network Foundation (DR Region)](#phase-2--network-foundation-dr-region)
7. [Phase 3 — Global wiring (DNS forwarding)](#phase-3--global-wiring-dns-forwarding)
8. [Phase 4 — Provision Managed HSM (Primary)](#phase-4--provision-managed-hsm-primary)
9. [Phase 5 — Activate HSM & Generate Security Domain (M-of-N Ceremony)](#phase-5--activate-hsm--generate-security-domain-m-of-n-ceremony)
10. [Phase 6 — Provision DR HSM & Enable Multi-Region Replication](#phase-6--provision-dr-hsm--enable-multi-region-replication)
11. [Phase 7 — Lock Down Network (Private Endpoints, NSG, Public Access OFF)](#phase-7--lock-down-network-private-endpoints-nsg-public-access-off)
12. [Phase 8 — HSM Data-Plane RBAC and Key Creation](#phase-8--hsm-data-plane-rbac-and-key-creation)
13. [Phase 9 — Workload Integration (VM Disks, SQL MI TDE, App Tier)](#phase-9--workload-integration-vm-disks-sql-mi-tde-app-tier)
14. [Phase 10 — Backup, Monitoring & Logging](#phase-10--backup-monitoring--logging)
15. [Phase 11 — DR Failover Drill and Runbook (Active-Active)](#phase-11--dr-failover-drill--runbook)
16. [Phase 12 — Go-Live Checklist & Hand-Over](#phase-12--go-live-checklist--hand-over)
17. [Appendix A — RBAC Role Reference](#appendix-a--rbac-role-reference)
18. [Appendix B — Security Domain Ceremony (Portal-Aware Detailed)](#appendix-b--security-domain-ceremony-portal-aware-detailed)
19. [Appendix C — Troubleshooting](#appendix-c--troubleshooting)

---

## 1. Overview & Design Decisions

> **First-time reader?** Treat this table as the executive summary — each row maps directly to one or more phases below.

| Area | Decision | Rationale |
|---|---|---|
| Key store | **Azure Managed HSM** (single-tenant, FIPS 140-3 Level 3) | Regulatory/audit requirement; tenant-isolated, not shared |
| Topology | **Extend the org's existing Hub-and-Spoke per region**, dual region | No duplicated gateway/Bastion/DNS; small blast radius |
| Regions | **Primary + DR** Azure paired region (example: South India + Central India) | Data residency; coordinated platform updates |
| Network | **Private only** (Public access DISABLED on HSM) | Zero internet exposure |
| Multi-region mode | **Active-Active** via Managed HSM multi-region replication | Each region's workloads hit their local HSM PE; no cross-region latency in steady state |
| Cross-region | **Existing hub-to-hub global VNet peering** + **HSM multi-region replication** | Same key URI in both regions; replication eventually consistent within ~6 min |
| DNS | **Region-specific Private DNS zones** (one `privatelink.managedhsm.azure.net` per region, **never shared**) + **platform-managed routing** under the global `<hsm>.managedhsm.azure.net` FQDN | Split-horizon: each region resolves the HSM to local PE. No customer Traffic Manager profile. |
| RTO/RPO | Key store RPO ≈ 0 (replication eventually consistent ≤ ~6 min). HSM failover automatic — in-region workloads keep using local PE; public-FQDN callers swung by Microsoft's internal router (TTL 5 s). Overall RTO depends on app + DB failover (target < 1 hr). | No DNS edits at the HSM layer during failover. |
| Identity | **Microsoft Entra ID** + **Managed Identities + HSM Local RBAC** | No secrets in code; per-key least privilege |
| Admin access | **Remote-access VPN → Azure Bastion → Jump VMs** | No public RDP/SSH |
| Security Domain | **Offline-generated, M-of-N (e.g. 3-of-5)** | Generated once on Primary, auto-replicated to DR — no second ceremony |
| Replication setup | **Multi-Region Replication blade → + Add Region** (portal equivalent of `az keyvault region add`) | Native path links the pools as one logical HSM and provisions the internal traffic routing automatically |
| Workloads in scope | VM OS/Data Disks (via DES), SQL Managed Instance (TDE with CMK), application-tier wrap/unwrap | All services requiring customer-managed encryption |
| **Cost** | **Multi-region replication ≈ doubles Managed HSM spend** (a full secondary HSM cluster is provisioned in DR, billed separately). Plus two per-region Private DNS zones and existing global peering. **No customer Traffic Manager profile cost.** | Budget for **2 × HSM cluster cost** from day one. |

### 1.6 Risk register & operational risks

The architecture is resilient by design. The two risks below are operational, and both have mitigations baked into the relevant phases.

| # | Risk | What goes wrong | Mitigation (where enforced) |
|---|---|---|---|
| **R1** | **App code/config hard-codes a non-canonical HSM hostname** (regional sub-FQDN or PE IP) | Microsoft's internal routing layer only flips traffic Primary→DR for clients that resolve the **canonical** FQDN. Hard-coded regional sub-FQDN or PE IP bypasses the router and keeps targeting the dead region. | **Architecture:** [§9.3](#93-application-tier-envelope-encryption) and [§9.4](#94-repeat-in-dr) mandate the canonical `https://<hsm>.managedhsm.azure.net/keys/<key>` form everywhere. **Audit:** [§10.5](#105-periodic-configuration-audits-quarterly) check B scans app config for `*.<region>.managedhsm.azure.net` or PE IPs. **Drill:** [§11.2](#112-quarterly-drill-no-production-impact) step 6 validates the platform router converges within one TTL (5 s). |
| **R2** | **Private DNS misconfiguration via wrong VNet linking** | Linking a VNet to the **wrong** regional zone (e.g. a DR app spoke accidentally linked to the Primary zone) silently mis-routes HSM traffic and breaks Active-Active — surfaces only at failover. | **Architecture:** [§1.5](#15-private-dns-zone--primary-regional-zone-in-the-central-connectivity-subscription-split-horizon) / [§2.3](#23-dr-regional-private-dns-zone-in-the-central-connectivity-subscription) mandate each zone is linked only to its own region's VNets, tagged `region=primary` / `region=dr`. **Audit:** [§10.5](#105-periodic-configuration-audits-quarterly) check A: every link's VNet location must match its zone's region tag. **Drill:** [§11.2](#112-quarterly-drill-no-production-impact) step 3 validates DR FQDN resolves to `10.22.1.4`. |

---

## 2. Prerequisites

Confirm each item before starting. Track owners and dates.

### 2.1 Azure tenant & subscription
- [ ] Microsoft Entra tenant identified (single tenant).
- [ ] One Azure subscription per environment (recommended: `org-prod`, `org-nonprod`). This guide targets `org-prod`.
- [ ] Subscription registered for required resource providers (portal: **Subscriptions → \<your sub\> → Settings → Resource providers**): `Microsoft.KeyVault`, `Microsoft.Network`, `Microsoft.Compute`, `Microsoft.Sql`, `Microsoft.Storage`, `Microsoft.Insights`, `Microsoft.OperationalInsights`.
- [ ] **Quotas approved** in both Primary and DR regions for Managed HSM (1 each). Check via **Subscriptions → \<your sub\> → Settings → Usage + quotas → Filter Provider = Key Vault**.
- [ ] **Platform/networking team sign-off** that the existing hub-and-spoke in each region can host one additional spoke VNet peered to the hub, and that the central connectivity subscription can host one additional Private DNS zone per region.

### 2.2 Roles required to perform the deployment

| Activity | Role required |
|---|---|
| Create resource groups, networks, HSM resource (portal click-paths in Phases 1, 2, 4) | **Contributor** on subscription (or **Owner** for role assignments) |
| Assign Azure RBAC (control-plane) | **User Access Administrator** or **Owner** |
| Assign HSM data-plane local RBAC (portal: HSM → **Local RBAC** blade) | **Managed HSM Administrator** (post-activation) |
| Security Domain ceremony ([§5](#phase-5--activate-hsm--generate-security-domain-m-of-n-ceremony)) | At least **N quorum officers** with physical token/key material |

### 2.3 People & artefacts
- [ ] **N Security Domain custodians** identified (e.g. 5 distinct officers from Security/Infra/Compliance/App/Audit).
- [ ] N × **RSA 3072/4096 key pairs** generated **offline** on hardened workstations. Public keys collected as `.cer`/`.pem`. Private keys held by each custodian (hardware token / smart card / encrypted USB).
- [ ] Quorum **M** decided (recommended: M=3, N=5).
- [ ] Naming/tagging standard agreed with CCoE.
- [ ] Change ticket approved.
- [ ] ExpressRoute circuit(s) provisioned (Primary + DR).

### 2.4 Tooling
- [ ] A modern browser with access to <https://portal.azure.com>.
- [ ] **For Phase 5 only**, an admin workstation with Azure CLI ≥ `2.61` and `az extension add --name keyvault`. (Hardened jump VM in the existing Primary app spoke is the recommended location for that one CLI step.)
- [ ] OpenSSL on the same workstation, for verifying officer public-key material.

---

## 3. Naming, IP Plan, and Variables

Use these values throughout the guide. Adjust to your organisation's standard if different.

### 3.1 Resource naming

**EXISTING** rows live in the org's production hub-and-spoke — you don't create them, you just capture their names/IDs from the platform team in Phase 1.0. **NEW** rows are the only resources this guide actually creates.

| Resource | Status | Primary (example: South India) | DR (example: Central India) |
|---|---|---|---|
| Hub VNet | EXISTING | *(captured)* | *(captured)* |
| ExpressRoute Gateway | EXISTING | *(captured)* | *(captured)* |
| Azure Firewall (or NVA) | EXISTING | *(captured — private IP for any UDRs)* | *(captured)* |
| Azure Bastion (in hub) | EXISTING | *(captured)* | *(captured)* |
| On-prem-facing DNS forwarder VMs | EXISTING | *(capture private IPs)* | *(capture private IPs)* |
| App / workload spoke(s) | EXISTING | *(captured — VMs and SQL MI live here)* | *(captured)* |
| Resource group — HSM | NEW | `rg-org-hsm-pri` | `rg-org-hsm-dr` |
| HSM PE spoke VNet | NEW | `vnet-org-hsm-pri` | `vnet-org-hsm-dr` |
| Private DNS zone (per region, split-horizon, in **central connectivity subscription**) | NEW | `privatelink.managedhsm.azure.net` (linked only to Primary hub + Primary app spokes + Primary HSM PE spoke) | `privatelink.managedhsm.azure.net` (linked only to DR hub + DR app spokes + DR HSM PE spoke) |
| Global FQDN routing | PLATFORM-MANAGED | Provisioned automatically by Multi-Region Replication → Add Region under `<hsm>.managedhsm.azure.net` (Microsoft-internal, DNS TTL 5 s) | Same — no customer Traffic Manager profile, no resource group, no per-profile cost |
| Managed HSM | NEW | `org-hsm-prod` | **same** `org-hsm-prod` (replica, added via Multi-Region Replication blade) |
| Private endpoint to HSM | NEW | `pe-hsm-pri` | `pe-hsm-dr` |

> **First-time readers:** the Managed HSM pool **name is identical** in both regions because the DR HSM is a *replica*, not a separate vault. Apps use a single key URI: `https://org-hsm-prod.managedhsm.azure.net/keys/<key-name>`.

### 3.2 IP plan

| Scope | Status | CIDR |
|---|---|---|
| Hub Primary (existing GatewaySubnet, AzureBastionSubnet, Firewall subnet, DNS forwarder subnet) | EXISTING | *(captured)* |
| App/workload spokes Primary | EXISTING | *(captured)* |
| **HSM PE Spoke Primary** VNet | NEW | `10.12.1.0/24` *(example — request a /24 from IPAM)* |
|  → `snet-pe-hsm` (Primary) | NEW | `10.12.1.0/28` *(PE receives `10.12.1.4`; Azure reserves `.0`–`.3`)* |
| Hub DR | EXISTING | *(captured)* |
| App/workload spokes DR | EXISTING | *(captured)* |
| **HSM PE Spoke DR** VNet | NEW | `10.22.1.0/24` *(example — request a /24)* |
|  → `snet-pe-hsm` (DR) | NEW | `10.22.1.0/28` *(PE receives `10.22.1.4`)* |

### 3.3 Values to capture before clicking anything

Open Notepad (or your change ticket) and fill in these values — every portal page below assumes you have them ready.

| Variable | Where to find it | Your value |
|---|---|---|
| `TENANT_ID` | Entra ID → Overview → Tenant ID | `<your-entra-tenant-id>` |
| `SUB_ID` (workload subscription) | Subscriptions blade | `<your-workload-subscription-id>` |
| `CONNECTIVITY_SUB_ID` (central DNS) | Subscriptions blade | `<connectivity-subscription-id>` |
| `LOC_PRI` / `LOC_DR` | Pick from Azure regions list | `southindia` / `centralindia` |
| `HUB_VNET_PRI` / `_RG` / `_ID` | Virtual networks blade in connectivity sub | *(from platform team)* |
| `APP_SPOKE_VNET_PRI` / `_RG` / `_ID` | Virtual networks blade | *(from platform team)* |
| `WORKLOAD_RG_PRI` | Resource groups blade | *(existing prod RG holding VMs/DES/SQL MI)* |
| `DNS_FWD_IPS_PRI` | Hub → Network interfaces of forwarder VMs | `<ip1>,<ip2>` |
| `FW_PRIVATE_IP_PRI` | Existing Azure Firewall → Overview | *(only if hub forces east-west through firewall)* |
| `HUB_VNET_DR` / `APP_SPOKE_VNET_DR` / `WORKLOAD_RG_DR` / `DNS_FWD_IPS_DR` / `FW_PRIVATE_IP_DR` | DR equivalents — capture the same way | *(from platform team)* |
| `CONN_DNS_RG` / `CONN_DNS_RG_DR` | Resource groups blade in connectivity sub (one RG per regional zone — strongly recommended; if your central pattern shares a single RG, both zones live there with `region=primary`/`region=dr` tags) | *(from connectivity team)* |
| `RG_HSM_PRI` / `RG_HSM_DR` | **NEW** — you'll create these in 1.1 / 2.2 | `rg-org-hsm-pri` / `rg-org-hsm-dr` |
| `HSM_NAME` | **NEW** — you choose | `org-hsm-prod` |
| `ADMIN_OID_1..3` | Entra ID → Users → \<officer\> → Object ID | `<entra-objectid-officer-N>` |

---

## Phase 0 — Subscription, Identity, and RBAC Preparation

**Goal:** subscription ready, providers registered, break-glass account and admin groups created.

### 0.1 Register resource providers (portal)
1. Sign in to <https://portal.azure.com>.
2. Search bar → **Subscriptions** → click your workload subscription.
3. Left menu → **Settings → Resource providers**.
4. In the filter box, type each of the following and click **Register** if Status is *NotRegistered*:
   - `Microsoft.KeyVault`
   - `Microsoft.Network`
   - `Microsoft.Compute`
   - `Microsoft.Sql`
   - `Microsoft.Storage`
   - `Microsoft.Insights`
   - `Microsoft.OperationalInsights`
5. Refresh until each shows **Registered**.

### 0.2 Create Entra groups
1. Search bar → **Microsoft Entra ID** → left menu → **Groups** → **+ New group**.
2. Create the following four groups one at a time:

| Group name | Group type | Membership type | Purpose |
|---|---|---|---|
| `grp-org-hsm-admins` | Security | Assigned | Full HSM admin (quorum officers) |
| `grp-org-hsm-crypto-officers` | Security | Assigned | Create/rotate/delete keys |
| `grp-org-hsm-crypto-users` | Security | Assigned | Wrap/unwrap/sign/verify only (workload identities) |
| `grp-org-hsm-auditors` | Security | Assigned | Read-only + logs |

3. For each group after creation, click into it and copy the **Object ID** from Overview — you'll use these in Phase 4 and Phase 8.

### 0.3 Break-glass identity
1. Entra ID → **Users → + New user → Create new user**.
2. Username: `bg-hsm-admin@<your-tenant>.onmicrosoft.com`. Set a strong random password.
3. After creation: **Authentication methods → + Add → FIDO2 security key** (or Temporary Access Pass if you'll register the key offline).
4. Entra ID → **Protection → Conditional Access → Policies** → ensure this user is **excluded** from any policy that could lock it out (e.g. block legacy auth is fine; require-compliant-device may not be).
5. Add the user to `grp-org-hsm-admins`.
6. Print the credentials, seal in an envelope, log to physical safe with the Security team. **Test login once a year, otherwise never use.**

### 0.4 Tagging baseline
For every new resource you create from this point on, set these tags in the **Tags** tab of the create blade: `env=prod`, `owner=<team>`, `costcenter=<cc>`, `dataclass=restricted`, `dr=active-active`.

---

## Phase 1 — Network Foundation (Primary Region)

**Goal:** **Reuse the org's existing hub** (ExpressRoute Gateway, Firewall, Bastion, DNS forwarder VMs are already there). Add one new spoke VNet for the HSM Private Endpoint, peer it into the existing hub, and link the central Private DNS zone to it.

### 1.0 Discover & capture existing hub assets (read-only — portal)

**Do not change anything in the existing hub** — this is a strictly read-only inventory step.

1. Switch subscription: portal top-right account → **Switch directory/subscription** → pick the **connectivity subscription** (where the hub lives).
2. Search bar → **Virtual networks** → open `HUB_VNET_PRI`.
3. From **Overview** capture: **Resource group**, **Location**, **Address space**, **Resource ID** (click *JSON View* → copy the `id`).
4. Left menu → **Subnets** → confirm presence of `GatewaySubnet`, `AzureBastionSubnet`, the Firewall subnet, and the DNS-forwarder subnet. Note their CIDRs.
5. Left menu → **Peerings** → record all existing peerings (you should see peerings to existing app/workload spokes; you'll add the HSM PE spoke alongside these in [§1.4](#14-peer-new-hsm-pe-spoke--existing-hub)).
6. Back to search bar → **Virtual network gateways** → confirm the Primary hub has an **ExpressRoute** gateway; capture name + SKU.
7. Search bar → **Bastions** → confirm the hub Bastion exists; capture name.
8. For each existing app/workload spoke that will need to call the HSM: **Virtual networks** → open it → JSON View → copy the `id` into `APP_SPOKE_VNET_PRI_ID`. Note its address space (you'll need it for the NSG rule in [§1.3](#13-nsg-on-the-pe-subnet-allow-443-from-approved-spokes-only)).
9. For each DNS forwarder VM: **Virtual machines** → open it → **Networking** → record its **NIC private IP**.

> **Heads-up on Firewall + UDRs:** if the existing hub uses Azure Firewall (or an NVA) as the **east-west** chokepoint and the app spokes have a UDR `0.0.0.0/0 → <FW_PRIVATE_IP>`, you must add a **more-specific UDR** in those spokes for the new HSM PE subnet (`10.12.1.0/28`) with **next-hop = Virtual network** — otherwise PE traffic will be sent to the firewall, which usually breaks Private Link. Confirm with the platform team before [§1.4](#14-peer-new-hsm-pe-spoke--existing-hub). To add the UDR (if needed): **Route tables** → existing route table attached to the app spoke → **Routes → + Add** → Name `udr-hsmpe-bypass-fw-pri`, Destination IP addresses `10.12.1.0/28`, Next hop type `Virtual network`.

### 1.1 Resource group for the new HSM + PE spoke (portal)
1. Switch back to your **workload subscription** (top-right account).
2. Search bar → **Resource groups → + Create**.
3. Subscription: workload sub. Resource group: **`rg-org-hsm-pri`**. Region: **South India** (your Primary).
4. **Tags** tab → add the baseline tags from [§0.4](#04-tagging-baseline).
5. Review + create → Create.

### 1.2 New HSM Private-Endpoint spoke VNet (portal)
1. Search bar → **Virtual networks → + Create**.
2. **Basics** tab:
   - Subscription: workload sub.
   - Resource group: `rg-org-hsm-pri`.
   - Name: **`vnet-org-hsm-pri`**.
   - Region: South India.
3. **IP addresses** tab:
   - Address space: **`10.12.1.0/24`**.
   - Remove any auto-created default subnet.
   - **+ Add subnet** → Name **`snet-pe-hsm`**, Address range **`10.12.1.0/28`**. Click **Add**.
4. Review + create → Create.
5. After creation: open `vnet-org-hsm-pri` → **Subnets** → click **`snet-pe-hsm`** → set **Private endpoint network policies: Disabled** → Save. *(This is required for Private Endpoint creation in Phase 7.)*

### 1.3 NSG on the PE subnet (allow 443 from approved spokes only)
1. Search bar → **Network security groups → + Create**.
2. Resource group `rg-org-hsm-pri`, Region South India, Name **`nsg-pe-hsm-pri`**. Create.
3. Open the NSG → **Inbound security rules → + Add**:
   - Name: `allow-app-spoke-443`. Priority: `100`. Direction: Inbound. Action: Allow.
   - Source: `IP Addresses`. Source IP addresses/CIDR: **`<app-spoke-cidr-pri>`** (the address space of the existing Primary app spoke from [§1.0](#10-discover--capture-existing-hub-assets-readonly--portal)).
   - Destination: `IP Addresses`. Destination IP addresses/CIDR: `10.12.1.0/28`.
   - Service: `HTTPS`. Protocol: TCP. Destination port ranges: `443`.
   - Add.
4. **+ Add** another rule:
   - Name: `deny-all-in`. Priority: `4096`. Direction: Inbound. Action: Deny. Source: `Any`. Destination: `Any`. Protocol: Any. Port: `*`. Add.
5. Open `vnet-org-hsm-pri` → **Subnets** → **`snet-pe-hsm`** → **Network security group**: pick `nsg-pe-hsm-pri` → Save.

### 1.4 Peer new HSM PE spoke ↔ existing hub

Run as two separate peerings — one from each side.

**Side A — hub → new HSM PE spoke** *(usually run by the platform team after change-ticket review; requires Network Contributor on the hub VNet)*:
1. Switch to the connectivity subscription.
2. **Virtual networks → `HUB_VNET_PRI`** → **Peerings → + Add**.
3. **This virtual network**:
   - Peering link name: `peer-hub-to-hsmpe-pri`.
   - Traffic to remote virtual network: **Allow**.
   - Traffic forwarded from remote virtual network: **Allow**.
   - Virtual network gateway: **Use this virtual network's gateway** (so the new HSM PE spoke can reach on-prem via the existing ExR GW). *Most hubs already have this enabled — leave it as-is.*
4. **Remote virtual network**:
   - Peering link name: `peer-hsmpe-to-hub-pri`.
   - Virtual network deployment model: Resource Manager.
   - Subscription: workload sub. Virtual network: `vnet-org-hsm-pri`.
   - Traffic to remote virtual network: **Allow**.
   - Traffic forwarded from remote virtual network: **Allow**.
   - Virtual network gateway: **Use the remote virtual network's gateway** *(this is the half that lets the HSM PE spoke use the existing hub's gateway).*
5. **Add**.

The portal's "Add peering" blade now creates **both** peerings in one shot (the older two-side workflow is no longer required). If your platform team uses the older two-side flow, they will create the hub-side first and you create the matching spoke-side from the spoke. Verify under **Peerings** on both VNets — both should show **Peering status: Connected**.

> **Spoke-to-spoke is intentionally NOT peered.** Traffic from the existing app spoke → HSM PE spoke flows via Private Link on the Azure backbone (host-routed by the PE's NIC); no explicit VNet peering needed.

### 1.5 Private DNS zone — **Primary regional zone in the central connectivity subscription** (split-horizon)

> **Why per-region zones?** Each region gets its own `privatelink.managedhsm.azure.net` zone, linked only to that region's VNets. The same HSM FQDN resolves to the **local** PE IP in each region — Primary workloads always hit the Primary PE, DR workloads always hit the DR PE. No cross-region hops, no manual DNS swing at failover. Global FQDN routing is handled separately by Microsoft's internal, platform-managed routing layer (provisioned by the Multi-Region Replication blade in Phase 6). **No customer Traffic Manager profile.**
>
> **Where the zone lives:** matching the org's existing central-DNS pattern, both regional zones live in the **connectivity subscription** (`CONN_DNS_RG` for Primary, `CONN_DNS_RG_DR` for DR — different RGs strongly recommended).

1. Switch to the **connectivity subscription**.
2. Search bar → **Private DNS zones → + Create**.
3. **Basics**:
   - Subscription: connectivity sub. Resource group: **`CONN_DNS_RG`** (the existing central DNS RG for Primary-region zones).
   - Name: **`privatelink.managedhsm.azure.net`** *(must be exactly this — Azure derives the public-to-private CNAME mapping from this name).*
4. **Tags** tab → add `region=primary` (this is what the quarterly audit in [§10.5](#105-periodic-configuration-audits-quarterly) uses to detect cross-region link mistakes). Also add the baseline tags from [§0.4](#04-tagging-baseline).
5. Review + create → Create.
6. Open the new zone → **Virtual network links → + Add** *(repeat per VNet to link)*:

| Link name | Virtual network | Registration |
|---|---|---|
| `link-hub-pri` | `HUB_VNET_PRI` (existing Primary hub) | **Disable** auto-registration |
| `link-app-spoke-pri` | `APP_SPOKE_VNET_PRI` (existing Primary app spoke) — repeat per app spoke that will call the HSM | **Disable** |
| `link-hsmpe-pri` | `vnet-org-hsm-pri` (the new HSM PE spoke from [§1.2](#12-new-hsm-private-endpoint-spoke-vnet-portal)) | **Disable** |

If a VNet sits in a different subscription, click **+ Add** → tick **I know the resource ID** → paste the VNet's full resource ID.

> **Do NOT link this Primary zone to any DR VNet.** DR VNets get their own zone (Phase 2). Linking a VNet to the wrong zone is risk **R2** in [§1.6](#16-risk-register--operational-risks).

---

## Phase 2 — Network Foundation (DR Region)

Same shape as Phase 1, in DR. **The DR hub also already exists** — reuse it.

### 2.1 Discover & capture existing DR hub assets (read-only)
Repeat [§1.0](#10-discover--capture-existing-hub-assets-readonly--portal) against the DR hub. Capture `HUB_VNET_DR_ID`, `APP_SPOKE_VNET_DR_ID`, `DNS_FWD_IPS_DR`, `FW_PRIVATE_IP_DR`.

### 2.2 Resource group, new HSM PE spoke, NSG, hub peering (portal)

Repeat in the workload subscription with DR-region values:

| Step | Mirror of | Use these values |
|---|---|---|
| Create RG | [§1.1](#11-resource-group-for-the-new-hsm--pe-spoke-portal) | Name `rg-org-hsm-dr`, Region **Central India** |
| Create VNet | [§1.2](#12-new-hsm-private-endpoint-spoke-vnet-portal) | Name `vnet-org-hsm-dr`, Address space `10.22.1.0/24`, Subnet `snet-pe-hsm` `10.22.1.0/28`, **Private endpoint network policies: Disabled** |
| Create NSG | [§1.3](#13-nsg-on-the-pe-subnet-allow-443-from-approved-spokes-only) | Name `nsg-pe-hsm-dr`, allow `<app-spoke-cidr-dr>` → `10.22.1.0/28` TCP/443, then deny-all-in. Attach to `snet-pe-hsm` in `vnet-org-hsm-dr`. |
| Peer to hub | [§1.4](#14-peer-new-hsm-pe-spoke--existing-hub) | Link names `peer-hub-to-hsmpe-dr` and `peer-hsmpe-to-hub-dr`. Same gateway settings. |

### 2.3 DR regional Private DNS zone (in the central connectivity subscription)

> **Why a second zone with the same name?** A VNet can be linked to **only one Private DNS zone per domain name** ([docs](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview#restrictions)) — a single shared zone for both regions is not feasible. Two zones with the same name *are* supported provided they live in **different resource groups** (this guide uses `CONN_DNS_RG` for Primary and `CONN_DNS_RG_DR` for DR, both in the central connectivity subscription).

1. Switch to the **connectivity subscription**.
2. **Private DNS zones → + Create**.
3. **Basics**:
   - Subscription: connectivity sub. Resource group: **`CONN_DNS_RG_DR`**.
   - Name: **`privatelink.managedhsm.azure.net`**.
4. **Tags** tab → add `region=dr` + baseline tags.
5. Review + create → Create.
6. Open the new DR-region zone → **Virtual network links → + Add** *(repeat per VNet)*:

| Link name | Virtual network |
|---|---|
| `link-hub-dr` | `HUB_VNET_DR` (existing DR hub) |
| `link-app-spoke-dr` | `APP_SPOKE_VNET_DR` (existing DR app spoke) — repeat per spoke |
| `link-hsmpe-dr` | `vnet-org-hsm-dr` (new DR HSM PE spoke from [§2.2](#22-resource-group-new-hsm-pe-spoke-nsg-hub-peering-portal)) |

All links: **Registration: Disabled**.

> **Recap:** the FQDN `org-hsm-prod.privatelink.managedhsm.azure.net` is the same in both regions, but each region's zone holds **only the local PE's A record**. A Primary-region VM always resolves to `10.12.1.4`; a DR-region VM always resolves to `10.22.1.4`. No record drift, no round-robin surprises, and no manual DNS swing at failover.

---

## Phase 3 — Global wiring (DNS forwarding)

> **What is already done by the platform team — NOT in this phase:**
> - **Hub-to-hub peering** between existing Primary and DR hubs.
> - **DR ExpressRoute circuit + Gateway.**
>
> **What is done automatically by Azure — NOT in this phase:**
> - **Global FQDN routing** under `<hsm>.managedhsm.azure.net`. Microsoft provisions this internal, platform-managed routing layer automatically when you add the second region in Phase 6 ([§6.1](#61-preferred--native-multi-region-replication-multi-region-replication-blade)). Health-checks both regional HSM pools, DNS TTL 5 s. **No customer Traffic Manager profile to create or operate** — see [§3.2](#32-global-fqdn-routing-platform-managed).

### 3.1 On-prem DNS forwarding (use the org's existing forwarder VMs)

The org already runs **self-hosted DNS forwarder VMs** inside each regional hub. You do **not** deploy Microsoft DNS Private Resolver — you piggy-back on what is already there.

**Inside Azure**, any VM in any of the VNets you linked in [§1.5](#15-private-dns-zone--primary-regional-zone-in-the-central-connectivity-subscription-split-horizon) / [§2.3](#23-dr-regional-private-dns-zone-in-the-central-connectivity-subscription) resolves `org-hsm-prod.privatelink.managedhsm.azure.net` to its **local** PE IP automatically — nothing to configure.

**On-prem clients** must be able to resolve `*.privatelink.managedhsm.azure.net` to the regional PE via the existing forwarder VMs. Two valid patterns; pick whichever matches what your platform team already uses for other Private Link services:

- **Pattern A — forwarder VMs query Azure-provided DNS (`168.63.129.16`):** On each regional forwarder VM, add a conditional-forward rule: zone `privatelink.managedhsm.azure.net` → target `168.63.129.16`.
- **Pattern B — forwarder VMs query a DNS Private Resolver in the hub:** if the platform team has deployed one, capture its inbound endpoint IP and point the conditional forward at that IP.

Then on the **on-prem DNS server(s)**, add the same conditional-forward rule pointing at the regional forwarder VM IPs:
- On-prem sites homed to Primary → forward to **`DNS_FWD_IPS_PRI`** first, **`DNS_FWD_IPS_DR`** as fallback.
- On-prem sites homed to DR → forward to **`DNS_FWD_IPS_DR`** first, **`DNS_FWD_IPS_PRI`** as fallback.

This work happens on Windows/Linux DNS servers, not in the Azure portal — coordinate with the on-prem DNS team. The address `168.63.129.16` is Azure's static wire-server DNS endpoint, reachable from any VM in any VNet; it returns answers from whatever Private DNS zones are linked to that VNet.

### 3.2 Global FQDN routing (platform-managed)

The FQDN `org-hsm-prod.managedhsm.azure.net` (without the `privatelink.` infix) is the **public/global** name. When you extend the HSM into a second region in Phase 6 ([§6.1](#61-preferred--native-multi-region-replication-multi-region-replication-blade)), **Azure automatically provisions an internal, Microsoft-managed traffic-routing layer for this FQDN**. There is **no customer-deployed Azure Traffic Manager profile** for the HSM. No profile name, no `*.trafficmanager.net` hostname, no endpoint enable/disable, no probe configuration that you own.

| Behaviour | Value |
|---|---|
| Resolves to | An IP in the region currently selected by the platform router |
| Default DNS TTL | **5 seconds** |
| Health probing | Performed by Microsoft on the internal HSM control plane (private to the platform; not visible to customers) — does **not** require public access on the HSM |
| Behaviour when one region is unhealthy | The router stops answering with that region's endpoint and returns the surviving region's endpoint. Fully automatic. |
| Behaviour with Private Endpoints in **both** regions ([Phase 7](#phase-7--lock-down-network-private-endpoints-nsg-public-access-off)) | A request that lands on the surviving region's PE is served by that region's HSM. A request that lands on the failed region's PE is redirected by the platform router to the surviving region's HSM. No client-side or operator action required. |

**Customer-side rule:** all workloads must call the HSM at the canonical URL `https://org-hsm-prod.managedhsm.azure.net/...`. Do **not** hard-code regional sub-FQDNs (`org-hsm-prod.<region>.managedhsm.azure.net`), PE IPs, or any non-canonical hostname. This is risk **R1** in [§1.6](#16-risk-register--operational-risks), audited quarterly in [§10.5](#105-periodic-configuration-audits-quarterly) check B.

> **No customer-managed Traffic Manager profile in this design.** Microsoft's native multi-region replication already provisions the internal routing layer under the canonical FQDN with TTL 5 s. A customer-deployed TM profile in front of that would be redundant and would risk bypassing private DNS. **Do not create a Traffic Manager profile for this HSM.**

### 3.3 Validate connectivity (before continuing)

From a small jump VM in your **existing Primary app spoke** (reach it via the existing hub Bastion — portal: **Bastions → \<hub-bastion\> → Connect**):

- `Test-NetConnection org-hsm-prod.privatelink.managedhsm.azure.net -Port 443` → succeeds (after Phase 7 creates the PE).
- `Resolve-DnsName org-hsm-prod.privatelink.managedhsm.azure.net` from a **Primary** VNet → `10.12.1.4`.
- From a DR VNet → `10.22.1.4`.
- `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` from on-prem → CNAME chain through the platform-managed routing layer; resolves to a healthy regional endpoint.

---

## Phase 4 — Provision Managed HSM (Primary)

### 4.1 Choose initial administrators
Collect Entra **Object IDs** of the officers who will participate in the Security Domain ceremony (minimum N=3, recommended 5). These become **initial administrators** with full data-plane rights on the HSM, written into the HSM's local-RBAC root scope *before* the SD exists. They are the only identities that can run the activation in [§5.2](#52-generate-encrypt-and-activate-the-hsm-via-cli---only-cli-required-step).

To get an Object ID via portal: **Entra ID → Users → \<officer\> → Object ID** (top of Overview).

### 4.2 Create the HSM (portal)
1. Switch to the workload subscription.
2. Search bar → **Key Vaults → + Create → Managed HSM** *(use the dropdown on the + Create button; the default Create button still creates a standard Key Vault).*
3. **Basics** tab:
   - Subscription: workload sub.
   - Resource group: **`rg-org-hsm-pri`**.
   - HSM name: **`org-hsm-prod`** *(this becomes the canonical FQDN — must be globally unique in `managedhsm.azure.net`).*
   - Region: **South India**.
   - Pricing tier: **Standard B1** *(only SKU currently available for Managed HSM)*.
   - HSM administrator: click **Add HSM administrators** → pick the N officers (or the `grp-org-hsm-admins` group) → Select.
4. **Networking** tab:
   - Connectivity method: **Disable public access** *(leave it off from day one; you'll activate over the private path in [§4.3](#43-network-access-during-activation)).*
5. **Tags** tab → baseline tags from [§0.4](#04-tagging-baseline).
6. **Review + create** → wait for validation → **Create**.

   Provisioning takes ~20–30 minutes for the 3-node cluster. State will go through **Provisioning → Provisioned (pending security domain)**.

7. After creation: open the HSM → **Overview** → confirm **Status: Provisioned**, **Activated: No**. Left menu → **Properties** → confirm **Soft delete retention: 90 days**, **Purge protection: Enabled** *(both required; cannot be reduced once set).*

> **If you don't see the toggle for "Purge protection" on creation:** Managed HSM enables it by default and it cannot be disabled — you'll see it in **Properties** as **Enabled** after creation.

### 4.3 Network access during activation

> **Recommended (the only path used in this guide):** activate the HSM over the **private path** — from a jump VM in your existing Primary app spoke, reached via the existing hub Bastion. Public network access stays **Disabled** throughout, including during Phase 5.

**Order of operations to enable the private path before activation:**

1. Skip ahead to [§7.1](#71-create-private-endpoint-to-primary-hsm-portal) and [§7.2](#72-register-primary-pe-in-the-primary-region-private-dns-zone-portal) now — stand up the Primary Private Endpoint and register it in the Primary regional Private DNS zone.
2. Create a small Linux/Windows **jump VM** in your **existing Primary app spoke** — **Virtual machines → + Create → Azure virtual machine**. Choose the smallest SKU (`Standard_B2s`). Pick the existing app spoke as the VNet/subnet. No public IP. System-assigned managed identity = On.
3. Open **Bastions → \<existing hub Bastion\> → Connect** → pick the jump VM → enter credentials.
4. Inside the jump VM: `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` should return `10.12.1.4`; `Test-NetConnection org-hsm-prod.managedhsm.azure.net -Port 443` should succeed.
5. Return to [Phase 5](#phase-5--activate-hsm--generate-security-domain-m-of-n-ceremony) and run the Security Domain ceremony from this jump VM.

> **Discouraged fallback** (only if the private path cannot be ready in time): temporarily enable public access with a tight IP allow-list, complete activation, then immediately disable public access in [§7.4](#74-disable-public-access-on-both-hsms-portal). Document the exception in the change ticket.
>
> Portal click-path for the fallback: open the HSM → **Networking** → **Public access** → **Enabled from selected virtual networks and IP addresses** → under *Firewall*, **+ Add your client IP address** (or paste `<admin-egress-public-ip>/32`) → Save. **Revoke immediately after [§5.3](#53-verify-activation-portal).**

---

## Phase 5 — Activate HSM & Generate Security Domain (M-of-N Ceremony)

> **This is the most security-critical step in the whole project.** Loss of the Security Domain = permanent loss of all keys in the HSM (no Microsoft recovery). Treat the ceremony as a controlled event.

> **Plain-English summary:** N officers walk into a room, each holding a public key they generated offline. You run **one** Azure CLI command. The HSM hardware generates its own root-of-trust material *inside the box*, encrypts that material so that any M of the N public keys can later decrypt it, and writes out a single encrypted file (the **Security Domain file**). At the moment that command succeeds, the HSM transitions from `Pending` to `Activated`.

### 5.1 Pre-ceremony
- [ ] Conference room booked. Video/audio recording per company policy.
- [ ] Custodians present (N people, e.g. 5).
- [ ] Each custodian brings their **public-key file** (`.cer`/PEM) generated offline.
- [ ] Verify each `.cer` belongs to its custodian (fingerprint read aloud, compared against pre-shared record).
- [ ] Jump VM from [§4.3](#43-network-access-during-activation) is reachable via Bastion and has Azure CLI ≥ 2.61 + `az extension add --name keyvault` already installed.
- [ ] All N `.cer` files have been copied to `./sd-certs/` on the jump VM (e.g. via Bastion file transfer or a controlled storage SAS).
- [ ] Storage locations for the encrypted **Security Domain file** decided (see [§5.4](#54-custody-and-storage-of-the-security-domain-file-and-officer-private-keys)).

### 5.2 Generate, encrypt, and activate the HSM via CLI — **only CLI-required step**

> **Why CLI for this one step?** The Azure portal does **not** expose the *Security Domain download/activation* operation. The Managed HSM blade only shows the post-activation state. This single step must be executed from `az keyvault security-domain download`. Everything else in the guide stays in the portal.

On the Bastion-attached jump VM, signed in as a quorum officer (`az login` → pick the officer's Entra account):

```bash
# Files: ./sd-certs/officer1.cer officer2.cer ... officer5.cer
az keyvault security-domain download \
  --hsm-name "org-hsm-prod" \
  --sd-wrapping-keys ./sd-certs/officer1.cer ./sd-certs/officer2.cer \
                     ./sd-certs/officer3.cer ./sd-certs/officer4.cer \
                     ./sd-certs/officer5.cer \
  --sd-quorum 3 \
  --security-domain-file org-hsm-prod-SD.json
```

What this single command does on the HSM side:
1. **Generates** the Security Domain inside the HSM hardware.
2. **Encrypts** it using the N supplied public keys with an M-of-N quorum policy (here, 3-of-5).
3. **Transitions the HSM from `Pending` to `Activated`**.

The encrypted SD file (`org-hsm-prod-SD.json`) that lands on the jump VM is the *artefact* of activation — there is no separate "activate" command, and there is no portal button for this. Treat the file as the crown-jewels artefact described in [§5.4](#54-custody-and-storage-of-the-security-domain-file-and-officer-private-keys).

### 5.3 Verify activation (portal)
1. Portal → **Key Vaults** filter set to **Managed HSMs** → open **`org-hsm-prod`**.
2. **Overview** should now show **Activated: Yes** and **Status: Activated**.
3. Left menu → **Properties** → **Status: The Managed HSM is operational**.

You can also confirm from the jump VM CLI: `az keyvault show --hsm-name org-hsm-prod --query "properties.statusMessage"` → `"The Managed HSM is operational"`.

### 5.4 Custody and storage of the Security Domain file and officer private keys
- **Security Domain file (`org-hsm-prod-SD.json`)** — store in **three** geographically separated, access-controlled locations:
  1. Your physical vault/safe, encrypted USB + paper-printed base64.
  2. Azure Storage account in a different subscription, **immutable** blob (legal hold or time-based), encrypted with a customer-managed key from a *different* HSM/Key Vault.
  3. Offline cold-storage drive, sealed envelope, in DR datacentre.
- **Officer private keys** — each custodian holds their own. Spread across at least 3 physical sites. Each in a hardware token (FIDO2 / smart card) or encrypted USB with passphrase.
- **Never** store M or more private keys in the same physical location.

### 5.5 Post-ceremony record
Write and sign:
- Names of N custodians and which public-key fingerprint each provided.
- Quorum M value (e.g. 3).
- SHA-256 of `org-hsm-prod-SD.json`.
- Storage locations of SD file copies.
- Date, time, witness signatures.

Store the record in your compliance repository.

---

## Phase 6 — Provision DR HSM & Enable Multi-Region Replication

> The DR HSM is **not a separate vault**. It is a replica of the Primary HSM — same Security Domain, same keys, same key URIs.

### 6.1 **Preferred** — Native multi-region replication (Multi-Region Replication blade)

1. Portal → open the Primary HSM **`org-hsm-prod`**.
2. Left menu → **Settings → Multi-Region Replication**.
3. The blade shows a map with the Primary region (South India) marked. Click **+ Add Region** at the top.
4. In the **Add Region** pane:
   - Region: **Central India**.
   - Confirm.
5. Click **Add / OK**. The blade adds the new row with **Provisioning State = Creating**.
6. **Wait until Provisioning State = Succeeded** for the DR row (~30 min). **Do not perform any operations on the Primary HSM** in the meantime — no key creates, no role changes, no SD operations.

After completion, this single action:
- Provisioned the DR HSM cluster in Central India.
- Imported the Security Domain from Primary automatically (**no second M-of-N ceremony**).
- **Replicates both keys *and* HSM-local role assignments** to the DR pool — anything you create or grant on Primary appears on DR within the ~6-min eventual-consistency window ([§6.4](#64-replication-lag--eventual-consistency-window)).
- Wired the DR pool into the **internal, Microsoft-managed routing layer** that fronts the global `<hsm>.managedhsm.azure.net` FQDN (DNS TTL 5 s) — see [§3.2](#32-global-fqdn-routing-platform-managed). **No customer Traffic Manager profile is created.**
- Both regions now appear under the single `org-hsm-prod` resource in the Multi-Region Replication blade.

Proceed to **[§6.3](#63-verify-multi-region-replication-functional-test-portal--cli)** to verify replication functionally, then to **Phase 7** to create the DR Private Endpoint.

### 6.2 Fallback — Manual replica with Security Domain import

Use this path only if the **Multi-Region Replication** blade is not available in your region/tenant, or if a pre-existing DR HSM must be brought into the replication pair. It produces the same end state but requires the Phase 5 SD file and **M physical custodians** to re-cooperate.

#### 6.2.1 Create the DR HSM shell (same name, in DR region)

Repeat the portal click-path from [§4.2](#42-create-the-hsm-portal) with:
- Resource group: **`rg-org-hsm-dr`**.
- HSM name: **`org-hsm-prod`** *(same name — Managed HSM multi-region replication requires it).*
- Region: **Central India**.
- Administrators: same N officers.
- Public network access: **Disabled**.

The DR HSM is born in **pending Security Domain** state.

#### 6.2.2 Upload Security Domain to DR HSM (CLI — portal does not expose this either)

On the same jump VM used in [§5.2](#52-generate-encrypt-and-activate-the-hsm-via-cli---only-cli-required-step), with **M custodians' private keys** present (M custodians must physically participate, or pre-stage the M keys on one trusted workstation under four-eyes control):

```bash
az keyvault security-domain upload \
  --hsm-name "org-hsm-prod" --resource-group "rg-org-hsm-dr" \
  --sd-file org-hsm-prod-SD.json \
  --sd-wrapping-keys ./officer-private-keys/officer1.key \
                     ./officer-private-keys/officer2.key \
                     ./officer-private-keys/officer3.key
```

After upload, DR HSM is **Activated** and is a true mirror. Microsoft's internal, platform-managed routing layer under `<hsm>.managedhsm.azure.net` ([§3.2](#32-global-fqdn-routing-platform-managed)) is also surfaced for the manual path once both pools share the same Security Domain.

### 6.3 Verify multi-region replication (functional test, portal + CLI)

1. Portal → **`org-hsm-prod`** (any region) → **Multi-Region Replication** → both rows show **Succeeded**.
2. Left menu → **Settings → Keys → + Generate/Import**:
   - Options: **Generate**.
   - Name: **`canary-replication-test`**.
   - Key type: **RSA-HSM**. RSA key size: **3072**.
   - Set permitted operations: **Sign**, **Verify**.
   - Create.
3. From a Primary jump VM, copy the **Key Identifier** (versioned `kid`) from the key's Overview blade.
4. From a **DR jump VM** in your existing DR app spoke (DNS resolves to `10.22.1.4`): open the same Managed HSM in portal, **Keys → `canary-replication-test`** — should show the same key with the same `kid`. (Allow up to ~6 min for the DR pool to converge — see [§6.4](#64-replication-lag--eventual-consistency-window).)
5. Optional cross-region cryptographic test (CLI on each side):
   ```bash
   # On Primary jump VM
   echo -n "canary-test" | az keyvault key sign --id "<kid>" --algorithm RS256 --value @- > sig.b64
   # On DR jump VM
   az keyvault key verify --id "<kid>" --algorithm RS256 --digest <sha256-of-canary-test> --signature @sig.b64
   ```
6. Clean up: portal → **Keys → `canary-replication-test` → Delete**.

### 6.4 Replication lag — eventual consistency window

Managed HSM multi-region replication is **eventually consistent**: write operations (key create/update/delete, role assignments, role-definition changes) can take **up to ~6 minutes** to propagate. The HSM service serialises and replicates over the Azure backbone, but there is no synchronous-commit guarantee.

**Operational rules that fall out of this:**

1. **Pre-failover quiet period.** Before any planned regional failover or maintenance, allow at least 6 minutes after the last control-plane write. Where possible, validate `HsmReplicationLatencyMs` is at baseline (see [§10.3](#103-alerts-recommended-baseline)).
2. **No back-to-back writer flipping.** Don't write in Primary and immediately route the next workload call to DR — the change may not be visible yet. Pin to the writer region until convergence, or rely on app-side retry ([§9.5](#95-application-side-resilience-mandatory-for-production-workloads)).
3. **Bulk onboarding.** Batch the work and *wait* before declaring DR equivalence — don't drive the canary test ([§6.3](#63-verify-multi-region-replication-functional-test-portal--cli)) immediately after the last write.
4. **Operational documentation.** All key-rotation runbooks and role-assignment tickets must reference the 6-minute window so on-call doesn't mistake transient `404 KeyNotFound` / `403 Forbidden` for a real fault.
5. **Alerting.** Keep the `HsmReplicationLatencyMs` alert from [§10.3](#103-alerts-recommended-baseline) (P1 if > 1000 ms sustained 5 min).

---

## Phase 7 — Lock Down Network (Private Endpoints, NSG, Public Access OFF)

### 7.1 Create Private Endpoint to Primary HSM (portal)
1. Switch to workload subscription. Portal → **`org-hsm-prod`**.
2. Left menu → **Networking → Private endpoint connections → + Private endpoint**.
3. **Basics**:
   - Resource group: **`rg-org-hsm-pri`**.
   - Name: **`pe-hsm-pri`**.
   - Region: **South India**.
4. **Resource**:
   - Connection method: *Connect to an Azure resource in my directory*.
   - Resource type: **`Microsoft.KeyVault/managedHSMs`**.
   - Resource: **`org-hsm-prod`**.
   - Target sub-resource: **`managedhsm`**.
5. **Virtual Network**:
   - Virtual network: **`vnet-org-hsm-pri`**.
   - Subnet: **`snet-pe-hsm`**.
6. **DNS** *(do NOT integrate with Private DNS zone in this step — we'll attach the regional zone explicitly in [§7.2](#72-register-primary-pe-in-the-primary-region-private-dns-zone-portal))*:
   - Integrate with private DNS zone: **No**.
7. Review + create → Create. Wait until **Provisioning state: Succeeded**.

### 7.2 Register Primary PE in the **Primary-region** Private DNS zone (portal)

The PE's **DNS zone group** writes its A record into the Private DNS zone you point it at. Because the Primary regional zone lives in the **connectivity subscription** (`CONN_DNS_RG`), and the PE lives in the workload subscription, you need rights on both.

1. Portal → open the PE **`pe-hsm-pri`** (Search bar → Private endpoints).
2. Left menu → **DNS configuration → + Add configuration**.
3. **Add private DNS zone group** pane:
   - Configuration name: `zg-hsm-pri`.
   - Subscription: **connectivity subscription**.
   - Resource group: **`CONN_DNS_RG`**.
   - Private DNS zone: **`privatelink.managedhsm.azure.net`** (the Primary-region zone you created in [§1.5](#15-private-dns-zone--primary-regional-zone-in-the-central-connectivity-subscription-split-horizon)).
4. Add.

Verify: from a Bastion-attached jump VM in any **Primary** linked spoke, `nslookup org-hsm-prod.privatelink.managedhsm.azure.net` returns **`10.12.1.4`**. From a DR VNet it must NOT return this IP — it should return `10.22.1.4` from the DR-region zone ([§7.3](#73-repeat-for-dr--register-dr-pe-in-the-dr-region-private-dns-zone-portal)).

### 7.3 Repeat for DR — register DR PE in the **DR-region** Private DNS zone (portal)

1. Portal → **`org-hsm-prod`** → **Networking → Private endpoint connections → + Private endpoint**.
2. **Basics**: RG `rg-org-hsm-dr`, Name **`pe-hsm-dr`**, Region **Central India**.
3. **Resource**: same `org-hsm-prod`, sub-resource `managedhsm`.
4. **Virtual Network**: `vnet-org-hsm-dr`, subnet `snet-pe-hsm`.
5. **DNS**: Integrate with private DNS zone: **No**.
6. Review + create → Create.
7. Open the new PE **`pe-hsm-dr`** → **DNS configuration → + Add configuration**:
   - Configuration name: `zg-hsm-dr`.
   - Subscription: connectivity subscription.
   - Resource group: **`CONN_DNS_RG_DR`** *(the DR-region zone — different RG from Primary).*
   - Private DNS zone: **`privatelink.managedhsm.azure.net`** (the DR-region zone created in [§2.3](#23-dr-regional-private-dns-zone-in-the-central-connectivity-subscription)).
8. Add.

> **If your central pattern keeps BOTH zones in the same RG** (uncommon, but supported), the dropdown will list both zones with identical names; disambiguate by the **resource ID** shown in the picker tooltip, or use **JSON View** on each zone to confirm its `tags.region` value.

> **Split-horizon is now complete:**
>
> | Source VNet | Zone consulted | A record returned |
> |---|---|---|
> | Any Primary-linked VNet | Primary regional zone in `CONN_DNS_RG` | `10.12.1.4` |
> | Any DR-linked VNet | DR regional zone in `CONN_DNS_RG_DR` | `10.22.1.4` |

### 7.4 Disable public access on BOTH HSMs (portal)
1. Portal → **`org-hsm-prod`** → **Networking → Firewalls and virtual networks** tab.
2. **Public access**: **Disabled**.
3. **Default action**: **Deny** (auto-implied when public access is Disabled).
4. Save.
5. The **same `org-hsm-prod` resource** governs both regional pools via the Multi-Region Replication blade — you only need to set the public-access toggle once. If you used the manual fallback path ([§6.2](#62-fallback--manual-replica-with-security-domain-import)), repeat the click-path on the DR HSM resource as well.

From this point onward, **all HSM data-plane traffic must traverse Private Endpoints** — i.e. from inside the VNets or via ExpressRoute + DNS forwarder from on-prem.

### 7.5 Smoke test from inside the VNet
From a Bastion-attached jump VM in your existing Primary app spoke:
- PowerShell: `Resolve-DnsName org-hsm-prod.privatelink.managedhsm.azure.net` → `10.12.1.4`.
- `Test-NetConnection ... -Port 443` → succeeds.
- If the VM has Az CLI: `az login --identity` then `az keyvault key list --hsm-name org-hsm-prod` returns 200 (or 403 — meaning auth is needed; connectivity is healthy).

---

## Phase 8 — HSM Data-Plane RBAC and Key Creation

> **First-timer note:** Managed HSM uses its **own local RBAC** at three scopes (HSM-wide `/`, all keys `/keys`, individual key `/keys/{name}`). This is **separate from Azure RBAC** — a user who is *Owner* of the Azure subscription has **no** ability to use keys until you also grant them an HSM-local role here. Likewise, removing someone from Azure RBAC does **not** revoke their HSM-local grants.

> **DR-side RBAC: nothing to reconfigure after failover.** All HSM-local role assignments, custom role definitions, and keys are protected by the Security Domain and are **replicated to the DR pool automatically**. Granting a workload identity `Managed HSM Crypto User` on `/keys/cmk-app-prod` here grants it that role on **both** pools. Allow ~6 min for convergence ([§6.4](#64-replication-lag--eventual-consistency-window)).

### 8.1 Assign administrator role (portal)
The officers from Phase 4 already have `Managed HSM Administrator`. Now add the admin group.

1. Portal → **`org-hsm-prod`** → left menu → **Local RBAC**.
2. **+ Add → Add role assignment**.
3. **Role**: search for and pick **Managed HSM Administrator**.
4. **Scope**: `/` *(the HSM root scope).*
5. **Members**: User, group, or service principal → **+ Select members** → pick **`grp-org-hsm-admins`** → Select.
6. **Review + assign**.

### 8.2 Assign crypto officer role (key lifecycle)
1. **Local RBAC → + Add → Add role assignment**.
2. **Role**: **Managed HSM Crypto Officer**.
3. **Scope**: `/keys`.
4. **Members**: pick **`grp-org-hsm-crypto-officers`**.
5. Review + assign.

### 8.3 Create CMK keys per workload (portal)
1. **Keys → + Generate/Import** *(repeat for each key below).*

| Key name | Key type | Size | Permitted operations |
|---|---|---|---|
| `cmk-vmdisk-prod` | **RSA-HSM** | **3072** | Wrap Key, Unwrap Key |
| `cmk-sqlmi-prod` | **RSA-HSM** | **3072** | Wrap Key, Unwrap Key, Encrypt, Decrypt *(includes encrypt/decrypt so the same key can be reused for ad-hoc cryptographic operations during operational tasks without minting a second key)* |
| `cmk-app-prod` | **RSA-HSM** | **3072** | Wrap Key, Unwrap Key, Encrypt, Decrypt |

For each: set **Activation date / Expiration date** if your policy mandates a lifetime (otherwise leave blank). **Enabled = Yes**. **Tags**: baseline. Create.

Each key is **immediately replicated** to the DR HSM (allow ~6 min — see [§6.4](#64-replication-lag--eventual-consistency-window)).

Capture and document the **versioned key URI** for each key — on each key's Overview blade, copy the **Key Identifier**:
- `https://org-hsm-prod.managedhsm.azure.net/keys/cmk-vmdisk-prod/<version>`

### 8.4 Assign least-privilege roles to workload identities (portal)

For each workload identity (DES managed identity, SQL MI managed identity, VM/App managed identity), assign **scoped** rights on the specific key it needs.

Example for the Disk Encryption Set you'll create in [§9.1](#91-virtual-machines--cmk-via-disk-encryption-set-portal):

1. Portal → **`org-hsm-prod`** → **Local RBAC → + Add → Add role assignment**.
2. **Role**: **Managed HSM Crypto Service Encryption User**.
3. **Scope**: `/keys/cmk-vmdisk-prod`.
4. **Members**: **+ Select members** → search for the DES name `des-vmdisk-prod` (or paste its object ID from **Disk Encryption Sets → \<des\> → Identity**) → Select.
5. Review + assign.

Repeat per key for SQL MI managed identity and VM/App managed identities, picking the smallest applicable role from [Appendix A](#appendix-a--rbac-role-reference).

---

## Phase 9 — Workload Integration (VM Disks, SQL MI TDE, App Tier)

> Do the **Primary region first**, validate end-to-end, then mirror in DR.

### 9.1 Virtual Machines — CMK via Disk Encryption Set (portal)
1. Search bar → **Disk Encryption Sets → + Create**.
2. **Basics**:
   - Subscription: workload sub. Resource group: **`WORKLOAD_RG_PRI`**.
   - Disk encryption set name: **`des-vmdisk-prod`**. Region: **South India**.
   - Encryption type: **Encryption at-rest with a customer-managed key**.
   - Key Vault and key: **Select a key vault and key** → switch the picker to **Managed HSM** → pick HSM `org-hsm-prod` → Key `cmk-vmdisk-prod` → leave **Version** empty (recommended — gives auto-rotation).
   - Auto key rotation: **Enabled** (since you used a versionless URI).
3. **Identity**: System-assigned managed identity → **On**.
4. Review + create → Create.
5. **Grant the DES MI access on the key:** repeat [§8.4](#84-assign-least-privilege-roles-to-workload-identities-portal) using role **Managed HSM Crypto Service Encryption User**, scope `/keys/cmk-vmdisk-prod`, member = `des-vmdisk-prod`.

6. **Create a NEW managed disk that uses the DES from day one:** **Disks → + Create** → in the **Encryption** tab → **Customer-managed key** → DES `des-vmdisk-prod`. Attach to your application VMs in the existing app subnet.

#### 9.1.1 Converting **existing** managed disks to CMK (portal)

Switching a disk from platform-managed key (PMK) to CMK only rewraps the disk's **DEK** with your HSM-held CMK — the encrypted blocks on the disk are **not** rewritten and no data is copied. It is a metadata operation. The only constraint is operational: the disk must be in a state where ARM can update its encryption settings.

**Pre-flight (run once before bulk-updating any disks):**
- The DES managed identity must already hold **Managed HSM Crypto Service Encryption User** on `/keys/cmk-vmdisk-prod` (assigned in [§8.4](#84-assign-least-privilege-roles-to-workload-identities-portal)). Without this, the encryption change fails with **403** against the HSM.
- The DES must be in the **same region** as the disk being updated. A Primary-region DES cannot encrypt a DR-region disk — that is why [§9.4](#94-repeat-in-dr) recreates a separate DES in DR.
- Confirm the disk SKU supports CMK (Premium_LRS / Standard_LRS / StandardSSD_LRS / Premium_ZRS / StandardSSD_ZRS are supported; Ultra and Premium v2 have extra restrictions — verify before bulk runs).

**Case A — disk is currently unattached.** Update in place; no VM impact.

1. Portal → **Disks → \<diskName\>** → **Encryption** blade.
2. **Key management** → **Customer-managed key**.
3. **Disk encryption set** → pick `des-vmdisk-prod`.
4. **Save**. The Azure activity log shows a single `Microsoft.Compute/disks/write` operation; no data is rewritten.

**Case B — disk is attached to a VM (OS or data disk).** There is **no online conversion** — the portal will not let you change encryption while the VM is running. Convert all disks in one maintenance window:

1. **VM → Overview → Stop** (this deallocates the VM — the OS disk and data disks all become eligible for an encryption change).
2. For the **OS disk** and **each data disk** in **VM → Disks**:
   - Click the disk name → **Encryption** blade.
   - **Key management** → **Customer-managed key** → DES `des-vmdisk-prod` → **Save**.
3. **VM → Overview → Start**.

> The portal shortcut **VM → Disks → \<disk\> → Encryption → Customer-managed key** prompts you to stop the VM if it is running — accept the prompt and complete the conversion in the same window. Don't leave the VM stopped with mixed-state disks (some CMK, some PMK) across multiple maintenance windows.

**Case C — bulk migration of an existing estate.** Don't drive this from the portal disk-by-disk. Use [Azure Resource Graph Explorer](https://portal.azure.com/#blade/HubsExtension/ArgQueryBlade) to enumerate every PMK disk in scope, export to CSV, group by attached VM, then schedule maintenance windows VM-by-VM through Case B. Sample query:

```kusto
Resources
| where type =~ 'microsoft.compute/disks'
  and resourceGroup =~ 'WORKLOAD_RG_PRI'
  and properties.encryption.type == 'EncryptionAtRestWithPlatformKey'
| project name, vmId = tostring(managedBy), sku = tostring(sku.name), sizeGB = properties.diskSizeGB
```

For estates beyond a handful of VMs, prefer the CLI path in the companion guide (`HSM_Implementation_Steps.md` §9.1.1) over portal clicks — the portal does not offer a multi-disk bulk-edit blade.

**Things to know up front:**
1. **PMK → CMK is supported; CMK → PMK on the same disk is not a normal operation.** Treat the move as one-way and plan accordingly.
2. **Versionless key URI on the DES** (recommended in step 2 above) means future CMK rotations auto-apply to every disk pointing at this DES — disks do not need to be re-converted on rotation.
3. **Snapshots and images** taken **before** the conversion are still PMK-encrypted. If you restore from them later, the restored disk lands as PMK and must be re-converted via Case A.
4. **Existing Azure Backup recovery points** for the VM remain valid across the conversion, but the next backup after conversion will be a full (not incremental) — size the backup window accordingly.
5. Allow ~1 min per disk for the metadata update; the VM stop/start dominates the actual outage window, not the encryption change.

### 9.2 SQL Managed Instance — TDE with CMK (portal)
1. SQL MI must already exist in a delegated subnet inside your existing app spoke.
2. **SQL MI → Settings → Identity** → confirm **System-assigned managed identity** is On. Copy the **Object (principal) ID**.
3. Grant the SQL MI MI access on the key (same as [§8.4](#84-assign-least-privilege-roles-to-workload-identities-portal)):
   - Role: **Managed HSM Crypto Service Encryption User**.
   - Scope: `/keys/cmk-sqlmi-prod`.
   - Member: SQL MI managed identity.
4. **SQL MI → Security → Transparent data encryption**:
   - Transparent data encryption: **Customer-managed key**.
   - Identity: **System-assigned**.
   - Key: **Select a key** → switch to **Enter a key identifier** → paste the versioned URI of `cmk-sqlmi-prod` (`https://org-hsm-prod.managedhsm.azure.net/keys/cmk-sqlmi-prod/<version>`).
   - **Save**.
5. Wait for the rotation to apply; the blade refreshes with **TDE Protector: customer-managed key** and shows the `kid`.

> **Terminology note:** the portal label reads "Customer-managed key in Azure Key Vault" even when the underlying key lives in a **Managed HSM** at `*.managedhsm.azure.net`. SQL MI treats both as the same key-store family — confirm via the `kid` value, not the label.

### 9.3 Application tier (envelope encryption)
For any organisation app that wraps its own DEKs:
1. Enable **system-assigned managed identity** on the VM / App Service / AKS workload. Portal: **\<resource\> → Identity → System assigned → On**.
2. Grant it **Managed HSM Crypto User** on `/keys/cmk-app-prod` ([§8.4](#84-assign-least-privilege-roles-to-workload-identities-portal)).
3. App uses **Azure Key Vault SDK** (`Azure.Security.KeyVault.Keys.Cryptography`) pointed at **`https://org-hsm-prod.managedhsm.azure.net/keys/cmk-app-prod`** for `WrapKey`/`UnwrapKey`. This URL only resolves correctly when the workload runs inside a VNet that is linked to the regional `privatelink.managedhsm.azure.net` zone ([§1.5](#15-private-dns-zone--primary-regional-zone-in-the-central-connectivity-subscription-split-horizon) / [§2.3](#23-dr-regional-private-dns-zone-in-the-central-connectivity-subscription)). Do **not** hard-code the regional `*.<region>.managedhsm.azure.net` FQDN or a PE IP — both bypass split-horizon DNS and break R1 in [§1.6](#16-risk-register--operational-risks). Do **not** introduce a customer-deployed Traffic Manager hostname; the global routing layer is platform-managed ([§3.2](#32-global-fqdn-routing-platform-managed)).
4. Cache the **wrapped DEK** alongside ciphertext; only the wrapped DEK leaves the HSM — the raw DEK never does.

### 9.4 Repeat in DR
Recreate **DES (DR)** in `WORKLOAD_RG_DR`, **SQL MI (DR)** TDE config, and DR app workloads pointing to the **same key URI** (`https://org-hsm-prod.managedhsm.azure.net/keys/<name>`). Each region's DES/SQL MI/app uses its own managed identity but the same HSM-local role assignment (replicated automatically).

> **DNS in the active-active design:** each region has its own Private DNS zone linked only to its own VNets, so a DR workload resolving `org-hsm-prod.privatelink.managedhsm.azure.net` always lands on the **DR PE** (`10.22.1.4`) and a Primary workload always lands on the **Primary PE** (`10.12.1.4`). No cross-region hops in steady state and no per-workload region-pinning logic needed.

### 9.5 Application-side resilience (mandatory for production workloads)

Even with Active-Active DNS and platform-managed global routing, applications must defend against the **eventual-consistency window** of replication (up to ~6 minutes, [§6.4](#64-replication-lag--eventual-consistency-window)) and against transient network blips. Every application calling the HSM **must** implement:

1. **Retry with exponential back-off** on transient failures (HTTP 408, 429, 5xx, socket timeouts). The Azure Key Vault SDKs ship a built-in retry policy — keep it enabled and tune `MaxRetries = 5`, base delay 800 ms. This also covers transient `404 KeyNotFound` / `403 Forbidden` that can appear briefly when a key or role assignment was just written in the *other* region.
2. **Region-aware retry / fallback to the writer region.** Keep at least one application instance per region; let the load balancer/queue route retries to a healthy region. Do **not** introduce DNS overrides or a customer-deployed Traffic Manager hostname — always use the canonical `<hsm>.managedhsm.azure.net` FQDN so split-horizon DNS continues to work.
3. **Short DNS TTL awareness.** Azure Private DNS A records default to 10 s TTL — adequate. The platform-managed global router uses TTL 5 s. Do not cache resolved IPs in the application beyond the TTL.
4. **Wrapped-DEK caching.** Cache the *wrapped* DEK for the lifetime of the request/session so a transient HSM blip doesn't fail user-facing operations.
5. **Circuit breaker + fail-fast.** After N consecutive HSM failures within W seconds, open a circuit and surface a clear health signal rather than letting the app silently degrade.
6. **Honour the 6-minute pre-failover quiet period** ([§6.4](#64-replication-lag--eventual-consistency-window)). Operational tooling that performs key rotation, role grants, or new key creation must record the timestamp of the last write so any subsequent *planned* failover honours the convergence window.

---

## Phase 10 — Backup, Monitoring & Logging

### 10.1 HSM full backup (control-plane, periodic + event-triggered) — **CLI/REST only**

> **The Azure portal does not currently expose `keyvault backup start` / `restore start` for Managed HSM.** The Multi-Region Replication blade also does not replace the full-backup process — replication mirrors live state, but **only a full backup gives you a recoverable artefact** against ransomware/accidental deletion. Run backups from CLI (or Azure Automation / Logic App scheduling the same CLI).

```bash
STG="<backup-storage-account>"
CONTAINER="hsm-backups"
SAS="<container-sas-with-create-write>"

az keyvault backup start --hsm-name "org-hsm-prod" \
  --storage-account-name "$STG" \
  --blob-container-name "$CONTAINER" \
  --storage-container-SAS-token "$SAS"
```

**Backup cadence:**

| Trigger | Frequency / condition | Rationale |
|---|---|---|
| Scheduled full backup | **Daily** off-peak (Azure Automation / Logic App) | Caps key-management RPO at 24 h even in the rare event both regional pools are lost |
| Event-triggered full backup | After any **bulk key creation**, **key rotation campaign**, **mass role-assignment change**, or **before any planned HSM maintenance** | Narrows the window where a fresh key/role exists only in the live HSM and not in backup storage |
| Retention | 12 months, **immutable** (legal-hold or time-based) blob container, in a **different region** (e.g. South East Asia) | Protects against ransomware/accidental deletion and regional storage outages |
| Encryption of backup blob | Customer-managed key from a **different** HSM / Key Vault than `org-hsm-prod` | Avoids circular dependency — you can't use the HSM you're restoring to encrypt its own backup |

**Restore drill (annual):** at least once per year, take the most recent backup blob and run `az keyvault restore start` into a **throwaway test HSM** in a non-production subscription. Verify the restore completes inside the **30-minute time limit** Azure enforces; verify keys and role assignments come back; delete the test HSM at the end.

### 10.2 Diagnostic settings → Log Analytics + Storage (portal)
1. Portal → **`org-hsm-prod`** → **Monitoring → Diagnostic settings → + Add diagnostic setting**.
2. **Diagnostic setting name**: `diag-hsm`.
3. **Logs**: tick **AuditEvent** and **AzurePolicyEvaluationDetails**.
4. **Metrics**: tick **AllMetrics**.
5. **Destination details**: tick **Send to Log Analytics workspace** → pick `law-org-pri` (or your central LAW).
6. Save.
7. If you used the manual fallback path ([§6.2](#62-fallback--manual-replica-with-security-domain-import)), repeat the click-path on the DR HSM resource. *(With the native multi-region replication path from [§6.1](#61-preferred--native-multi-region-replication-multi-region-replication-blade), the single `org-hsm-prod` resource already emits logs from both regional pools.)*

### 10.3 Alerts (recommended baseline, portal)

The list below is a **production-grade baseline**, not a rigid floor — tune thresholds and severities to your environment. The first three are non-negotiable for any HSM holding CMKs; the last two are strongly recommended once multi-region replication is live.

Create via **Monitor → Alerts → + Create → Alert rule**, scope = the HSM resource:

| Signal | Condition | Severity / action |
|---|---|---|
| Custom log search: `AzureDiagnostics \| where ResourceProvider == "MICROSOFT.KEYVAULT" and OperationName in ("VaultDelete", "Purge", "PurgeDeletedKey")` | Count > 0 over 5 min | **P1** — page Security on-call |
| Custom log search: failed-auth rate | > 5/min sustained | **P2** |
| Custom log search: key version not used in 30 days | scheduled query | **P3** — review for cleanup |
| Metric: HSM region health degraded | Status != Healthy | **P1** — initiate DR runbook |
| Metric: `HsmReplicationLatencyMs` | > 1000 ms sustained 5 min | **P1** |

For each, **Actions → + Action group** → assign to your Security on-call team (Email + SMS + PagerDuty webhook).

### 10.4 Activity log alerts (portal)
1. **Monitor → Alerts → + Create → Alert rule** → Scope: the HSM.
2. **Condition → Add condition**: Signal `Administrative` → Operation `Microsoft.KeyVault/managedHSMs/write` or `…/delete` → Status `Started/Succeeded/Failed`.
3. Filter by **caller** to exclude pipeline service principals (e.g. `not contains 'sp-cicd-'`).
4. Action group: Security on-call.

### 10.5 Periodic configuration audits (quarterly)

These audits catch the two operational risks called out in [§1.6 Risk Register](#16-risk-register--operational-risks) **before** they bite during a real failover. Schedule quarterly, align with the DR drill in [§11.2](#112-quarterly-drill-no-production-impact).

**Check A — Private DNS link hygiene (R2)** *(portal)*:
1. Switch to the connectivity subscription.
2. For each of `CONN_DNS_RG` and `CONN_DNS_RG_DR`: open the Private DNS zone `privatelink.managedhsm.azure.net` → **Virtual network links** → review each row's **Virtual network** column.
3. For each linked VNet, open it (click through) → confirm its **Location** matches the zone's `region` tag.
4. **Any mismatch is a P1** — remove the offending link immediately, then validate from a workload in the affected region that DNS still resolves correctly.

**Check B — Workload FQDN drift (R1)** *(portal)*:
1. App Configuration → **Configuration explorer** → filter by value for `.southindia.managedhsm` or `.centralindia.managedhsm` or `10.12.1.4` or `10.22.1.4`. Also scan for any `trafficmanager.net` entries that mention this HSM (leftover from older designs).
2. App Service → \<each app\> → **Configuration → Application settings**: same value scan.
3. Function App → \<each\> → **Configuration → Application settings**: same.
4. Key Vault references in the same App Service → check the referenced URLs.
5. **Any finding is a P2** — migrate the app config to the canonical `https://org-hsm-prod.managedhsm.azure.net/...` URL and redeploy.

Record the run as a change-ticket entry. Findings from A are **P1** (potential failover misroute); findings from B are **P2** (canonical-FQDN discipline).

---

## Phase 11 — DR Failover Drill & Runbook

### 11.1 Failover model (Active-Active)
- The HSM key store is **Active-Active**: both regional pools serve their own local PE in steady state, and the global `<hsm>.managedhsm.azure.net` FQDN is health-routed by Microsoft's internal, platform-managed routing layer ([§3.2](#32-global-fqdn-routing-platform-managed)) with DNS TTL 5 s.
- **There is no manual DNS swing and no manual endpoint toggle in the normal failover path.** If the Primary region or its HSM endpoint becomes unhealthy, the platform router stops returning the Primary endpoint and returns the DR endpoint — fully automatic.
- Workloads inside each region continue to resolve the **private** FQDN via their region-local Private DNS zone and continue using their local PE. If a region itself is lost, the workloads in that region are gone too — the surviving region's workloads keep using their local HSM with no change required.
- "Failover" therefore means swinging the **application + DB tier** from Primary to DR (Front Door / customer-side traffic director for the app layer, SQL MI auto-failover group). The HSM piece is automatic.

### 11.2 Quarterly drill (no production impact)
1. Pick a non-prod workload deployed in DR.
2. From DR Bastion → DR jump VM, sign in to the HSM via Az CLI on the jump VM (`az login --identity`) and run a wrap/unwrap against `https://org-hsm-prod.managedhsm.azure.net/keys/cmk-app-prod`.
3. Confirm DNS resolves to **`10.22.1.4`** (DR PE, via the DR-region Private DNS zone) and call latency p50 < 30 ms.
4. Force-fail an app instance from Primary to DR (Front Door / customer-side traffic director priority swap — Front Door portal: **Origin groups → \<group\> → Origins**, set priorities).
5. Verify SQL MI TDE unseal succeeds in DR — open SQL MI (DR) → **Transparent data encryption**: still showing the HSM key URI, no error.
6. **Validate platform-managed global routing.** From a public/on-prem client outside both Azure regions (or via the Cloud Shell in a third region), run `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` repeatedly while the Primary HSM PE is intentionally made unreachable in the drill subscription (e.g. temporarily detach the Primary PE from its DNS zone group via the PE's **DNS configuration** blade, or use a non-prod replicated HSM). Confirm the resolved endpoint converges on the DR-region endpoint within one TTL (~5 s) and that `Test-NetConnection ... -Port 443` succeeds against the new target.
7. Restore the drill HSM's Primary PE / DNS zone group; record metrics; close the drill ticket.

### 11.3 Real failover runbook (region loss)
1. Declare incident; activate DR ICs.
2. **Verify the platform-managed global router has already converged** on the DR endpoint. From a healthy out-of-region location: `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` should now point at the DR-region endpoint; `Test-NetConnection ... -Port 443` should succeed within one TTL (~5 s) of the Primary becoming unhealthy. **No customer action is required to trigger this** — if DNS has not converged after ~30 s, raise a Severity-A support ticket against Managed HSM rather than attempting any client-side workaround.
3. Swing Front Door / your traffic director for the application tier: DR = priority 1, Primary = priority 2 (drain). Portal: **Front Door → Origin groups → \<group\> → Origins**.
4. Promote DR SQL MI (auto-failover group) — TDE protector already on the same HSM URI → no key re-config. Portal: **SQL Managed Instance → Failover groups → \<group\> → Forced failover**.
5. Re-point app config (if any region-specific overrides exist).
6. Validate end-to-end transaction. DR workloads continue resolving the **private** HSM FQDN to `10.22.1.4` via the DR Private DNS zone — no change required.
7. Communicate.

> **The manual private-DNS swing in [§11.5](#115-last-resort-private-dns-fallback-do-not-use-in-normal-failover) is a last-resort fallback only.**

**HSM key-store RPO: ≈ 0** (replication near real-time on the Azure backbone; eventually consistent within ~6 min).
**HSM data-plane failover: automatic** via the platform-managed router under the canonical FQDN, ~5 s TTL.
**Overall solution RTO:** depends on app + DB failover. Target < 1 hour.

### 11.4 Security Domain restore drill (annual)
Once a year, in a **test subscription**:
1. Deploy a throwaway Managed HSM (repeat [§4.2](#42-create-the-hsm-portal)).
2. Bring M custodians together.
3. Upload `org-hsm-prod-SD.json` to the test HSM via `az keyvault security-domain upload` ([§6.2.2](#622-upload-security-domain-to-dr-hsm-cli--portal-does-not-expose-this-either)).
4. Confirm activation succeeds in the portal **Overview** blade.
5. Delete the test HSM.

Document outcome with custodian signatures. Combine with a partial **custodian access check** every 6 months — each custodian briefly demonstrates they can still unlock their private key (no actual SD operation performed) so loss of access is detected long before it would matter.

### 11.5 **Last-resort** private-DNS fallback (do NOT use in normal failover)

The Active-Active design ([§1.5](#15-private-dns-zone--primary-regional-zone-in-the-central-connectivity-subscription-split-horizon), [§2.3](#23-dr-regional-private-dns-zone-in-the-central-connectivity-subscription), [§7.3](#73-repeat-for-dr--register-dr-pe-in-the-dr-region-private-dns-zone-portal)) makes this unnecessary in steady-state failover. **Use the steps below ONLY** in the rare scenario where the DR-region Private DNS zone itself has been damaged or is mis-pointing, *and* the platform router has been unable to converge.

Run as an operator who holds **Private DNS Zone Contributor** on the **DR-region** zone (`privatelink.managedhsm.azure.net` in `CONN_DNS_RG_DR`).

**Portal steps:**
1. Switch to the connectivity subscription.
2. Search bar → **Private DNS zones** → open the DR-region `privatelink.managedhsm.azure.net` (in `CONN_DNS_RG_DR`).
3. **Recordsets → + Record set**:
   - Name: `org-hsm-prod`.
   - Type: **A**.
   - TTL: **10 seconds**.
   - IP address: **`10.22.1.4`** *(the DR PE IP).*
4. OK.
5. Verify from a DR jump VM: `Resolve-DnsName org-hsm-prod.privatelink.managedhsm.azure.net` → `10.22.1.4`.

Fail-back is unnecessary — once the DR-region zone is healthy and the Primary region returns, each region's workloads resume using their own local PE without operator action. Pre-stage this runbook as an Azure Automation runbook gated on incident-commander approval; **never trigger it automatically**, and never edit the *Primary*-region zone from a DR-side runbook.

---

## Phase 12 — Go-Live Checklist & Hand-Over

### 12.1 Pre go-live
- [ ] All Phase 1–10 steps complete in **both** regions.
- [ ] Public access **Disabled** on the HSM (Networking → Public access shows **Disabled**).
- [ ] Soft-delete **90 days**, Purge protection **Enabled** (Properties blade) — verified.
- [ ] **Active-Active DNS confirmed:** Primary VNets resolve `org-hsm-prod.privatelink.managedhsm.azure.net` to `10.12.1.4`; DR VNets resolve the same name to `10.22.1.4`. No shared/single zone.
- [ ] **Platform-managed global routing confirmed:** from a public/out-of-region client, `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` returns a healthy regional endpoint; `Test-NetConnection ... -Port 443` succeeds. **No customer-deployed Traffic Manager profile is present for this HSM** (search `Traffic Manager profiles` in the portal — no result for this HSM). A `*.trafficmanager.net` hostname in any app config is a **Go-Live blocker**.
- [ ] **Multi-region replication healthy** — HSM → Multi-Region Replication blade shows both `southindia` and `centralindia` with **Provisioning state: Succeeded**; canary test ([§6.3](#63-verify-multi-region-replication-functional-test-portal--cli)) passes.
- [ ] **6-minute eventual-consistency window** documented in all key-rotation and role-assignment runbooks ([§6.4](#64-replication-lag--eventual-consistency-window)).
- [ ] Security Domain file SHA-256 recorded; 3 copies stored; custodian list signed.
- [ ] **Backup**: daily schedule running; last successful HSM backup blob URL + checksum on file; event-triggered backup configured on key-rotation pipeline ([§10.1](#101-hsm-full-backup-control-plane-periodic--event-triggered--clirest-only)).
- [ ] **Backup restore drill** executed at least once into a throwaway test HSM, completed under the 30-min limit.
- [ ] DR drill ([§11.2](#112-quarterly-drill-no-production-impact)) executed and signed off, **including the platform-routing convergence validation in step 6**.
- [ ] All workload identities have **scoped** roles (no HSM-wide `Crypto User`).
- [ ] Break-glass user tested (login + role usage) and credentials sealed.
- [ ] Log Analytics receiving `AuditEvent` from the HSM (Monitor → Logs → run `AzureDiagnostics | where ResourceProvider == "MICROSOFT.KEYVAULT" | take 5`).
- [ ] Alerts ([§10.3](#103-alerts-recommended-baseline-portal)) firing on test triggers, including `HsmReplicationLatencyMs`.
- [ ] On-prem DNS forwarders are region-aware per [§3.1](#31-on-prem-dns-forwarding-use-the-orgs-existing-forwarder-vms) (Primary-homed sites → Primary forwarder first; DR-homed sites → DR forwarder first).

### 12.2 Documentation hand-over to operations
Deliverables:
1. This document plus diagrams.
2. RBAC matrix ([Appendix A](#appendix-a--rbac-role-reference)).
3. Security Domain custody record.
4. Backup schedule (daily + event-triggered) and storage account location.
5. Runbooks: failover ([§11.3](#113-real-failover-runbook-region-loss)), key rotation (with 6-min quiet period), SD restore, **last-resort private-DNS fallback ([§11.5](#115-last-resort-private-dns-fallback-do-not-use-in-normal-failover))**.
6. Contact list: HSM admins, Security on-call, Azure TAM.

---

## Appendix A — RBAC Role Reference

> Use this when assigning HSM-local roles in [§8](#phase-8--hsm-data-plane-rbac-and-key-creation). Always pick the **smallest** role that lets a workload do its job, and scope it as tightly as possible (prefer `/keys/<name>` over `/keys` over `/`).

| Role | Scope (recommended) | Used by | Operations |
|---|---|---|---|
| Managed HSM Administrator | `/` | Quorum officers / `grp-org-hsm-admins` | Full control: role assignments, key management, SD operations |
| Managed HSM Crypto Officer | `/keys` | `grp-org-hsm-crypto-officers` | Create / rotate / delete keys |
| Managed HSM Crypto User | `/keys/<key>` | Application managed identities | Wrap, Unwrap, Sign, Verify, Encrypt, Decrypt — **no** key lifecycle |
| Managed HSM Crypto Service Encryption User | `/keys/<key>` | DES, SQL MI, Storage account MI | Wrap / Unwrap only (what Azure platform services need to call the HSM on the workload's behalf) |
| Managed HSM Crypto Auditor | `/` | `grp-org-hsm-auditors` | Read keys and audit logs, no cryptographic ops |
| Managed HSM Backup | `/` | Backup automation service principal | Trigger full backup, read SAS |

---

## Appendix B — Security Domain Ceremony (Portal-Aware Detailed)

The portal cannot perform the SD ceremony — see [§5.2](#52-generate-encrypt-and-activate-the-hsm-via-cli---only-cli-required-step). This appendix expands the ceremony for organisations that need a more detailed runbook.

### B.1 Custodian onboarding (offline, weeks before ceremony)
1. Each of the N custodians generates an **RSA-3072 or RSA-4096** key pair on a **hardened, air-gapped workstation** (no network access). Recommended: `openssl genrsa -aes256 -out officerN.key 3072` then `openssl req -new -x509 -key officerN.key -out officerN.cer -days 3650 -subj "/CN=officerN-org-hsm-prod"`.
2. Private key (`officerN.key`) is stored on a hardware token / smart card / encrypted USB and physically kept by the custodian. **Never leaves the custodian.**
3. Public key (`officerN.cer`) is exported and handed in to the ceremony coordinator. Capture the SHA-256 fingerprint of each `.cer`.
4. Coordinator publishes the list of fingerprints to all custodians for cross-verification.

### B.2 Day-of-ceremony checklist
- [ ] Room booked; access list checked; recording running.
- [ ] All N custodians physically present with their `.cer` files (USB).
- [ ] Coordinator reads each `.cer` fingerprint aloud; custodian confirms.
- [ ] All N `.cer` files copied to `./sd-certs/` on the jump VM.
- [ ] Coordinator signs in via `az login` as a quorum officer.
- [ ] Run [§5.2](#52-generate-encrypt-and-activate-the-hsm-via-cli---only-cli-required-step) command. **Do not exit the shell until the SD file is verified.**
- [ ] Compute and read aloud SHA-256 of `org-hsm-prod-SD.json`; all custodians sign the witness sheet recording that hash.
- [ ] Copy the SD file to all three secure locations from [§5.4](#54-custody-and-storage-of-the-security-domain-file-and-officer-private-keys), each time re-verifying the SHA-256 at the destination.
- [ ] Securely wipe `org-hsm-prod-SD.json` from the jump VM disk (do not leave it in `/tmp`, `~`, or Cloud Shell home).
- [ ] All custodians depart with their own private-key media intact.

### B.3 Post-ceremony custody record (template)
```
Ceremony date / time:
Location:
HSM name / region:
Quorum: M of N
Custodians (name, role, public-key SHA-256):
  1.
  2.
  3.
  4.
  5.
Security Domain file SHA-256:
Storage copy 1 location / SHA-256 verified by:
Storage copy 2 location / SHA-256 verified by:
Storage copy 3 location / SHA-256 verified by:
Witnesses (signatures):
```

---

## Appendix C — Troubleshooting

### C.1 HSM stuck at `Provisioning` for > 1 hour
- Check Activity Log on the HSM resource for deployment errors.
- Confirm quota for Managed HSM in the region (Subscriptions → Usage + quotas).
- Open a Severity-B support ticket: *Resource and region* = Key Vault → Managed HSM.

### C.2 `403 Forbidden` from a workload identity that should have access
- Reproduce with the same identity via Cloud Shell: `az keyvault key show --hsm-name org-hsm-prod -n <key>`.
- HSM → **Local RBAC**: confirm the role assignment exists at the right scope (`/keys/<key>` or `/keys`).
- Remember: control-plane Azure RBAC (e.g. *Owner*) does **NOT** grant data-plane HSM rights.
- If the role was assigned in the **last ~6 minutes**, wait — eventual consistency ([§6.4](#64-replication-lag--eventual-consistency-window)).

### C.3 `404 KeyNotFound` from a workload immediately after a key was created
- Same cause: replication lag ([§6.4](#64-replication-lag--eventual-consistency-window)). Wait up to 6 minutes and retry.
- If persistent, confirm the key actually exists in **HSM → Keys** blade.

### C.4 DNS resolves to public IP / Public DNS instead of PE
- Confirm the workload's VNet is linked to the correct **regional** Private DNS zone ([§1.5](#15-private-dns-zone--primary-regional-zone-in-the-central-connectivity-subscription-split-horizon) / [§2.3](#23-dr-regional-private-dns-zone-in-the-central-connectivity-subscription)).
- Confirm the PE has a **DNS zone group** attached ([§7.2](#72-register-primary-pe-in-the-primary-region-private-dns-zone-portal) / [§7.3](#73-repeat-for-dr--register-dr-pe-in-the-dr-region-private-dns-zone-portal)).
- From the workload VM: `Resolve-DnsName org-hsm-prod.privatelink.managedhsm.azure.net` — should return the regional PE IP. If it returns no record, the zone link or A record is missing.

### C.5 DR workload resolving to Primary PE IP (`10.12.1.4` instead of `10.22.1.4`)
- **Risk R2 has tripped.** Stop traffic to the misrouted workload.
- Connectivity sub → Private DNS zones → review **Virtual network links** on both regional zones.
- A DR-region VNet must NOT appear in the Primary zone's link list (and vice-versa). Remove the offending link, then re-add it to the correct regional zone.
- Validate with `Resolve-DnsName` from the workload before reopening traffic.

### C.6 "Add Region" greyed-out or "region not available"
- Confirm both regions are listed as supported for Managed HSM multi-region replication ([Microsoft Learn](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/multi-region-replication#azure-region-support)). US East, Canada East, West Europe, Qatar Central, Poland Central, and West India cannot currently be extended regions.
- Confirm Managed HSM quota is approved in the target extended region.
- Confirm the HSM is **Activated** (Phase 5 done) — the blade rejects Add Region on a *Pending* HSM.

### C.7 "Multi-Region Replication" blade missing
- Confirm Azure Portal version is current (hard refresh).
- Confirm your account has at least **Reader** on the HSM and **Managed HSM Administrator** local RBAC.
- If still missing, fall back to the manual path in [§6.2](#62-fallback--manual-replica-with-security-domain-import).

### C.8 Backup blob fails with `AuthorizationFailure`
- Confirm the SAS token has `cw` (create + write) on the container, not just `rl` (read + list).
- Confirm the HSM has **Outbound** access to the storage account (it uses the Azure backbone; no PE on the storage side is required from the HSM, but if the storage account has its own firewall, add the HSM's outbound IPs or use a service endpoint).

### C.9 SQL MI TDE blade rejects the HSM key URI
- Confirm the SQL MI managed identity has **Managed HSM Crypto Service Encryption User** on `/keys/cmk-sqlmi-prod` (not just *Crypto User*).
- Confirm the key URI uses the **canonical FQDN** (`org-hsm-prod.managedhsm.azure.net`), not a regional sub-FQDN or PE IP.
- Wait ~6 min if the role assignment was recent (replication lag).

---

> **End of Azure portal edition.** For the CLI commands behind any of these portal click-paths, see the matching section in [`HSM_Implementation_Steps.md`](./HSM_Implementation_Steps.md) — section numbers and phase names are identical between the two documents.
