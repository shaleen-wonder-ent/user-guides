# Customer-Managed Keys on Azure Managed HSM
## Dual-Region Hub-and-Spoke Deployment Guide

> **Audience:** infrastructure, security, and platform engineers deploying Azure Managed HSM for the **first time** to back customer-managed keys (CMK) for VM disks, SQL Managed Instance TDE, and application-tier envelope encryption.
>
> **Architecture reference:** `managed-hsm-dual-region-hub-spoke-azure-simplified.png` and `managed-hsm-dual-region-hub-spoke-azure-tech.png`

<img width="1522" height="846" alt="image" src="https://github.com/user-attachments/assets/2b4a2b79-e4a3-4cdc-98d1-8d4db3cd0393" />
<img width="2580" height="1065" alt="image" src="https://github.com/user-attachments/assets/9b930e1f-61aa-4090-ae64-6fdb44e5872f" />


>
> **Example primary region:** South India · **Example DR region:** Central India *(swap for any two Azure paired regions)*
> **HSM pool name (both regions):** `org-hsm-prod` *(placeholder — replace `org` everywhere with your organisation's short code, e.g. `acme`, `contoso`)*
> **Posture:** Always Private (public access disabled), Always Available (multi-region replication, **Active-Active** with region-aware DNS and Azure Traffic Manager-based automatic failover).

---

## What this guide builds, in plain English

You are going to deploy a **highly secure, highly available cryptographic key store on Azure**, used by your applications, virtual machines, and databases to encrypt data with keys that **you** control — and that **no one, not even Microsoft, can read or extract**.

The end state looks like this:

- **Two physically separate HSM clusters** in two Azure regions. Both share the same keys via built-in **multi-region replication**, so you can lose an entire region and your encryption keys are still available.
- **No public internet access** to the key store. Applications reach it only through **Azure Private Endpoints** inside your own virtual networks, or from on-premises over **ExpressRoute**.
- **A hub-and-spoke network in each region** so administration traffic, application traffic, and HSM traffic are cleanly separated.
- **Active-Active operation** — applications in each region talk to their *local* HSM by default (low latency, no cross-region hop), and Azure Traffic Manager **automatically** fails over if a region goes down. No human DNS edit needed during an outage.
- **An M-of-N quorum** (e.g. any 3 of 5 officers) controls the HSM's root key material — no single person can compromise or destroy the keys.

## The journey, phase by phase (plain English)

Read this first so you understand *why* each phase exists before you run any commands.

| Phase | What you do | Why it matters |
|---|---|---|
| **0. Subscription & identity prep** | Register Azure resource providers; create Microsoft Entra ID groups for admins, key managers, key users, auditors; create a "break-glass" emergency admin account. | These groups become the basis of every permission grant later. Doing this once up-front avoids ad-hoc permission drift. |
| **1. Primary network** | Build the **hub-and-spoke** in the primary region: hub VNet with ExpressRoute gateway, Azure Bastion (jump-host gateway), DNS Private Resolver; two spokes — one for the HSM's Private Endpoint, one for client workloads. | The HSM cannot be exposed to the public internet — so you need a private network in which it can live and be reached. |
| **2. DR network** | Repeat Phase 1 in the DR region. **Important:** each region gets its **own** Private DNS zone (split-horizon), so a region's workloads always resolve the HSM name to that region's local Private Endpoint. | Region-aware DNS is what makes Active-Active work — no manual DNS edits at failover. |
| **3. Global connectivity & Traffic Manager** | Peer the two hub VNets, stand up the DR ExpressRoute, configure on-prem DNS forwarders, and create the **Traffic Manager** profile for the global HSM FQDN. | Traffic Manager is what automatically fails the global endpoint over when a region goes down. |
| **4. Provision Primary HSM** | Run one CLI command and wait ~20–30 minutes for the 3-node HSM cluster to provision. It comes up in *Pending* state — usable for activation, not yet for keys. | This is the HSM itself. It is **not yet activated** — keys cannot be created until Phase 5. |
| **5. Security Domain ceremony** | Bring N officers (e.g. 5) into a room, each holding their offline-generated public key. Run one command that **generates** the HSM's root key material, **encrypts** it under those N public keys with an M-of-N quorum, and **activates** the HSM. The output is one encrypted Security Domain file. | This is the most critical step of the whole project. Lose the Security Domain file *and* enough officer private keys, and **every key in the HSM is permanently unrecoverable** — Microsoft cannot help. |
| **6. DR HSM & replication** | Add the DR region to the existing HSM with one CLI command (`az keyvault region add`). Azure provisions the DR cluster, copies the Security Domain, and links the two pools as one logical HSM. Then run a canary test to prove a key created on Primary is usable on DR. | This is what gives you region-redundant keys — the same key URI works in both regions. |
| **7. Private Endpoints & lock-down** | Create one Private Endpoint per region pointing to its local HSM, register each into its **own** regional Private DNS zone, then **disable public network access** on both HSMs. | After this, the only way to reach the HSM is through your private network. The internet path is closed for good. |
| **8. Data-plane RBAC and keys** | Grant your Entra groups the right HSM-local roles (Administrator, Crypto Officer, Crypto User). Create one key per workload (one for VM disks, one for SQL TDE, one for app envelope encryption). | The HSM uses **its own local RBAC**, distinct from Azure RBAC. Every workload identity gets least-privilege access to *only the key it needs*. |
| **9. Workload integration** | Wire each workload to its key: Disk Encryption Set for VM disks, TDE protector for SQL Managed Instance, SDK calls for application envelope encryption. Add SDK retry + circuit-breaker logic to handle the ~6-minute replication lag. | This is where encryption actually starts using your HSM keys. |
| **10. Backup, logging, alerts** | Set up **daily** HSM backups to an immutable storage container in a third region. Stream HSM audit logs to Log Analytics. Configure alerts for deletion attempts, auth failures, replication lag. | Backups protect against accidental key deletion or ransomware on the storage tier. Logs are mandatory for audit. |
| **11. DR drill & runbook** | Run a quarterly drill that simulates Traffic Manager failover. Once a year, restore the Security Domain into a throwaway test HSM to prove the ceremony artefacts still work. | A DR plan you don't rehearse is not a DR plan. The annual restore is what catches "we lost a custodian's key two years ago and didn't notice". |
| **12. Go-live checklist** | Tick the gates: public access disabled, soft-delete + purge protection on, backups verified, DR drill signed off, alerts firing on test triggers, runbooks handed over. | Final gate before production traffic touches the HSM. |

## How to read this guide

- **Phases are sequential.** Do not skip ahead — Phase 5 (activation) depends on Phase 4 (provisioning), Phase 7 (lock-down) depends on Phase 6 (replication), and so on.
- **Code blocks are real commands**, written for `bash` / Azure CLI. Set the variables in [§3.3](#33-cli-variables-paste-into-your-shell) first, then paste blocks in order. Anything wrapped in `<placeholder>` needs a real value you supply.
- **Boxes that start with ⚠️ or "Important"** mark places where a mistake is hard or impossible to reverse (e.g. Security Domain custody, public-access toggle).
- **Replace `org` everywhere** with your organisation's short code. Swap the example region names if you are not deploying in India.
- **First time with Azure Managed HSM?** Read the Concept primer below before starting Phase 1.

## Concept primer (read once, refer back as needed)

| Term | Plain-English meaning |
|---|---|
| **Managed HSM** | A single-tenant, FIPS 140-2 Level 3 cryptographic appliance that Azure runs *for you*. Unlike standard Azure Key Vault, the hardware is dedicated to your tenant. Microsoft cannot see your keys. |
| **Security Domain (SD)** | The HSM's "root of trust". It is generated *inside* the HSM hardware, then encrypted to the N officers' public keys before it ever leaves. Required to activate the HSM and to bring a replica online. If lost beyond recovery, every key in the HSM dies with it. |
| **M-of-N quorum** | The "any M of N officers must cooperate" rule that protects the Security Domain. E.g. 3-of-5 means any 3 of the 5 designated officers' private keys, together, can unseal the SD. No single officer (or two of them together) can. |
| **Hub-and-spoke** | An Azure networking pattern. The "hub" VNet centralises shared services (gateway, firewall, Bastion, DNS). "Spoke" VNets host workloads and connect only to the hub. Spokes don't talk to each other directly — keeps blast-radius small. |
| **Private Endpoint (PE)** | A NIC inside your VNet that gives a Private Link service (here, the HSM) a private IP from your address space. Traffic to that IP stays on the Microsoft backbone, never traverses the public internet. |
| **Private DNS zone** | An Azure-private DNS namespace. Here we create one per region for `privatelink.managedhsm.azure.net` so that the HSM's FQDN resolves to the **local** Private Endpoint IP in each region. |
| **Traffic Manager** | A DNS-based global traffic director. Health-probes endpoints and answers DNS based on which one is up. We use it to make the **global** HSM FQDN automatically fail over to the DR region. |
| **Multi-region replication** | A Managed HSM feature where two regional HSM clusters share the same Security Domain and continuously replicate keys and role assignments. Eventually consistent within ~6 minutes — plan for this in your apps. |
| **Disk Encryption Set (DES)** | An Azure object that binds a managed disk's encryption to a specific key in a Key Vault or Managed HSM. Each VM disk pointed at the DES is then encrypted using that key. |
| **TDE (Transparent Data Encryption)** | SQL's at-rest encryption. With CMK + Managed HSM, the TDE "protector" key lives in the HSM; SQL wraps/unwraps the database encryption keys against it. |
| **Envelope encryption** | App pattern where you encrypt data with a fast symmetric key (the DEK), then encrypt that DEK with an HSM-held key. Only the (small) DEK touches the HSM — bulk data never does. |
| **Break-glass account** | An emergency-only Entra ID account with strong MFA, excluded from conditional access policies that could lock it out, used only when normal admin paths fail. |

---

## Table of Contents

1. [Overview & Design Decisions](#1-overview--design-decisions)
2. [Prerequisites](#2-prerequisites)
3. [Naming, IP Plan, and Variables](#3-naming-ip-plan-and-variables)
4. [Phase 0 — Subscription, Identity, and RBAC Preparation](#phase-0--subscription-identity-and-rbac-preparation)
5. [Phase 1 — Network Foundation (Primary Region)](#phase-1--network-foundation-primary-region)
6. [Phase 2 — Network Foundation (DR Region)](#phase-2--network-foundation-dr-region)
7. [Phase 3 — Global Connectivity (Peering, ExpressRoute, DNS Link)](#phase-3--global-connectivity-peering-expressroute-dns-link)
8. [Phase 4 — Provision Managed HSM (Primary)](#phase-4--provision-managed-hsm-primary)
9. [Phase 5 — Activate HSM & Generate Security Domain (M-of-N Ceremony)](#phase-5--activate-hsm--generate-security-domain-m-of-n-ceremony)
10. [Phase 6 — Provision DR HSM & Verify Multi-Region Replication](#phase-6--provision-dr-hsm--enable-multi-region-replication)
11. [Phase 7 — Lock Down Network (Private Endpoints, NSG, Public Access OFF)](#phase-7--lock-down-network-private-endpoints-nsg-public-access-off)
12. [Phase 8 — HSM Data-Plane RBAC and Key Creation](#phase-8--hsm-data-plane-rbac-and-key-creation)
13. [Phase 9 — Workload Integration (VM Disks, SQL MI TDE, App Tier, Resilience)](#phase-9--workload-integration-vm-disks-sql-mi-tde-app-tier)
14. [Phase 10 — Backup, Monitoring & Logging](#phase-10--backup-monitoring--logging)
15. [Phase 11 — DR Failover Drill and Runbook (Active-Active)](#phase-11--dr-failover-drill--runbook)
16. [Phase 12 — Go-Live Checklist & Hand-Over](#phase-12--go-live-checklist--hand-over)
17. [Appendix A — RBAC Role Reference](#appendix-a--rbac-role-reference)
18. [Appendix B — Security Domain Ceremony (Detailed)](#appendix-b--security-domain-ceremony-detailed)
19. [Appendix C — Troubleshooting](#appendix-c--troubleshooting)

---

## 1. Overview & Design Decisions

> **First-time reader?** Treat this table as the executive summary — each row maps directly to one or more phases below. You do not need to memorise it; come back to it whenever you need to remember *why* a decision was made.

| Area | Decision | Rationale |
|---|---|---|
| Key store | **Azure Managed HSM** (single-tenant, FIPS 140-2 Level 3) | Regulatory and audit requirement for sensitive workloads; tenant-isolated HSM, not shared with other Azure customers |
| Topology | **Hub-and-Spoke per region**, dual region | Centralised egress/admin, isolated workload spokes, small blast radius if any one spoke is compromised |
| Regions | **Primary + DR**, an Azure paired-region set (example: South India + Central India) | Data residency compliance; paired regions get coordinated platform updates and prioritised recovery |
| Network | **Private only** (Public access DISABLED on HSM) | Zero exposure to public internet; data-plane reached only via Private Endpoint |
| Multi-region mode | **Active-Active** via Managed HSM multi-region replication; both pools serve local data-plane traffic | Each region's workloads hit their **local** HSM PE in steady state — no cross-region latency, capacity is not stranded, and failover is exercised continuously |
| Cross-region | **Global VNet Peering (Hub-to-Hub)** + **HSM multi-region replication** | Same key URI in both regions; replication is near real-time but **eventually consistent (up to ~6 minutes for key/role propagation)** — operations and applications must plan for this window |
| DNS | **Region-specific Private DNS zones** (one `privatelink.managedhsm.azure.net` per region) + **Azure Traffic Manager** for the global `*.managedhsm.azure.net` FQDN | Split-horizon resolution: each region's VNets resolve the HSM FQDN to the local PE IP; Traffic Manager handles automatic failover at the global endpoint without manual DNS edits |
| RTO/RPO | **Key store RPO ≈ 0** (replication on the Azure backbone), **automatic** swing via Traffic Manager. Overall solution RTO depends on app + DB failover (target < 1 hour) | No human DNS edit required inside a failed region |
| Identity | **Microsoft Entra ID** for control-plane; **Managed Identities + Local RBAC** for data-plane | No secrets in code; per-key scoped least privilege |
| Admin access | **Remote-access VPN → Azure Bastion → Jump/Client VMs** with MFA | No public RDP/SSH; every admin session is auditable |
| Security Domain | **Offline-generated, M-of-N (e.g. 3-of-5)** | Same Security Domain imported into both HSM pools so DR is a true mirror |
| Replication setup | **`az keyvault region add` (native multi-region replication)** preferred; manual SD-upload retained as fallback | Native path links the pools as one logical HSM, configures Traffic Manager automatically, and shows both regions under one resource in the portal |
| Workloads in scope | VM OS/Data Disks (via DES), SQL Managed Instance (TDE with CMK), application-tier wrap/unwrap | All services requiring customer-managed encryption |

---

## 2. Prerequisites

Confirm each item before starting. Track owners and dates.

### 2.1 Azure tenant & subscription
- [ ] Microsoft Entra tenant identified (single tenant).
- [ ] One Azure subscription per environment (recommended: `org-prod`, `org-nonprod`). This guide targets `org-prod`.
- [ ] Subscription registered for required resource providers:
  - `Microsoft.KeyVault`, `Microsoft.Network`, `Microsoft.Compute`, `Microsoft.Sql`, `Microsoft.Storage`, `Microsoft.Insights`.
- [ ] Quotas approved in both **primary** and **DR** regions for: Managed HSM (1 each), ExpressRoute Gateway, Bastion, VMs, SQL MI, Public IPs (Bastion only).

### 2.2 Roles required to perform the deployment
| Activity | Role required |
|---|---|
| Create resource groups, networks, HSM resource | **Contributor** on subscription (or **Owner** for role assignments) |
| Assign RBAC (control-plane) | **User Access Administrator** or **Owner** |
| Assign HSM data-plane RBAC | **Managed HSM Administrator** (post-activation) |
| Security Domain ceremony | At least **N quorum officers** with physical token/key material |

### 2.3 People & artefacts
- [ ] **N Security Domain custodians** identified (e.g. 5 officers). Distinct individuals, different functions (Security, Infra, Compliance, App, Audit).
- [ ] N x **RSA 3072/4096 key pairs** generated **offline** (one per officer, on hardened workstations). Public keys collected as `.cer`/`.pem`. Private keys held by each custodian (hardware token / encrypted USB / smart card) — never on shared storage.
- [ ] Quorum **M** decided (recommended: `M=3, N=5`).
- [ ] Naming/tagging standard agreed with your Cloud Centre of Excellence (CCoE).
- [ ] Change ticket approved.
- [ ] ExpressRoute circuit(s) provisioned by carrier (Primary + DR). LOA / cross-connect complete.

### 2.4 Tooling on admin workstation
- [ ] Azure CLI ≥ `2.61` and `az extension add --name keyvault` (latest).
- [ ] PowerShell `Az.KeyVault` module (optional).
- [ ] OpenSSL for verifying public-key material.
- [ ] A workstation that can reach the HSM **only during activation** (public endpoint must be temporarily allowed; see Phase 5). For all subsequent operations, use Bastion → Jump VM inside the network.

---

## 3. Naming, IP Plan, and Variables

Use these values throughout the guide. Adjust to your organisation's standard if different. The prefix `org` is a stand-in for your organisation's short code — replace it everywhere (e.g. `acme`, `contoso`).

### 3.1 Resource naming

| Resource | Primary (example: South India) | DR (example: Central India) |
|---|---|---|
| Resource group — hub | `rg-org-hub-pri` | `rg-org-hub-dr` |
| Resource group — HSM | `rg-org-hsm-pri` | `rg-org-hsm-dr` |
| Resource group — workloads | `rg-org-app-pri` | `rg-org-app-dr` |
| Resource group — global (TM, shared DNS infra) | `rg-org-global` (location: any — Traffic Manager is a global resource) | — |
| Hub VNet | `vnet-org-hub-pri` | `vnet-org-hub-dr` |
| HSM client spoke VNet | `vnet-org-app-pri` | `vnet-org-app-dr` |
| HSM private-endpoint spoke VNet | `vnet-org-hsm-pri` | `vnet-org-hsm-dr` |
| Bastion | `bas-org-pri` | `bas-org-dr` |
| ExpressRoute gateway | `ergw-org-pri` | `ergw-org-dr` |
| DNS Private Resolver | `dnspr-org-pri` | `dnspr-org-dr` |
| Private DNS zone (regional, split-horizon) | `privatelink.managedhsm.azure.net` (in `rg-org-hub-pri`, linked to Primary VNets only) | `privatelink.managedhsm.azure.net` (in `rg-org-hub-dr`, linked to DR VNets only) |
| Traffic Manager profile (global endpoint) | `tm-org-hsm-prod` (in `rg-org-global`) | — |
| Managed HSM | `org-hsm-prod` | `org-hsm-prod` (same name — replica) |
| Private endpoint to HSM | `pe-hsm-pri` | `pe-hsm-dr` |

> **Note for first-time readers:** the Managed HSM pool **name is identical** in both regions. The DR HSM is a *replica* of the Primary, not a separate vault. Applications use a single key URI: `https://org-hsm-prod.managedhsm.azure.net/keys/<key-name>`. Resolution of that URI is split-horizon by design — Primary VNets resolve to the Primary PE IP, DR VNets resolve to the DR PE IP (see Phase 1 §1.10, Phase 2, and Phase 7).

### 3.2 IP plan

Pick non-overlapping CIDRs that fit your existing on-prem and Azure address spaces. Sizes shown are comfortable defaults — do not shrink them without checking subnet requirements (Bastion needs `/26`, ExpressRoute Gateway needs `/27` minimum).

| Scope | CIDR |
|---|---|
| Hub Primary | `10.10.0.0/16` |
|  → `GatewaySubnet` | `10.10.0.0/27` |
|  → `AzureBastionSubnet` | `10.10.1.0/26` |
|  → `snet-dnspr-inbound` | `10.10.2.0/28` |
|  → `snet-dnspr-outbound` | `10.10.2.16/28` |
| HSM Client Spoke Primary | `10.11.0.0/16` |
|  → `snet-app` | `10.11.1.0/24` |
|  → `snet-sqlmi` | `10.11.2.0/27` (delegated) |
| HSM PE Spoke Primary | `10.12.0.0/16` |
|  → `snet-pe-hsm` | `10.12.1.0/24` |
| Hub DR | `10.20.0.0/16` (same subnet split as Primary) |
| HSM Client Spoke DR | `10.21.0.0/16` |
| HSM PE Spoke DR | `10.22.0.0/16` |

### 3.3 CLI variables (paste into your shell)

Set these once at the start of your session — every command block below assumes they are defined.

```bash
# Tenant / subscription
TENANT_ID="<your-entra-tenant-id>"
SUB_ID="<your-subscription-id>"
az account set --subscription "$SUB_ID"

# Regions (replace with your chosen Azure paired-region set)
LOC_PRI="southindia"
LOC_DR="centralindia"

# Resource groups
RG_HUB_PRI="rg-org-hub-pri"
RG_HUB_DR="rg-org-hub-dr"
RG_HSM_PRI="rg-org-hsm-pri"
RG_HSM_DR="rg-org-hsm-dr"
RG_APP_PRI="rg-org-app-pri"
RG_APP_DR="rg-org-app-dr"
RG_GLOBAL="rg-org-global"

# HSM
HSM_NAME="org-hsm-prod"

# Officer object IDs (Entra users / groups that will administer the HSM)
ADMIN_OID_1="<entra-objectid-officer-1>"
ADMIN_OID_2="<entra-objectid-officer-2>"
ADMIN_OID_3="<entra-objectid-officer-3>"
```

---

## Phase 0 — Subscription, Identity, and RBAC Preparation

**Goal:** subscription ready, providers registered, break-glass account and admin groups created.

**Why this matters (first-timer note):** if you skip the up-front identity work and assign permissions ad-hoc to individual users later, you will end up with orphaned permissions that are painful to clean up and risky during personnel changes. Doing it group-first means every later grant attaches to a stable, auditable group.

### 0.1 Register resource providers
```bash
for ns in Microsoft.KeyVault Microsoft.Network Microsoft.Compute Microsoft.Sql Microsoft.Storage Microsoft.Insights Microsoft.OperationalInsights; do
  az provider register --namespace "$ns"
done
```
Re-run `az provider show -n Microsoft.KeyVault --query registrationState` until it returns `Registered`.

### 0.2 Create Entra groups (admins, operators, auditors)
Create four groups in Entra ID (Portal → Entra ID → Groups → New group):
- `grp-org-hsm-admins` — full HSM admin (quorum officers).
- `grp-org-hsm-crypto-officers` — create/rotate/delete keys.
- `grp-org-hsm-crypto-users` — wrap/unwrap/sign/verify only (workload identities).
- `grp-org-hsm-auditors` — read-only + logs.

Record their **Object IDs**; you will use them in Phase 8.

### 0.3 Break-glass identity
Create one **break-glass user** (cloud-only Entra account, FIDO2 key, excluded from any Conditional Access policy that could lock it out). Add to `grp-org-hsm-admins`. Store the credentials in a sealed envelope with the Security team.

> **Why a break-glass account?** If every admin loses their access (Conditional Access misconfiguration, lost FIDO2 keys, identity-provider outage), this account is the only way back in. It must be tested annually but otherwise never used.

### 0.4 Tagging baseline
Decide minimum tags applied to every resource: `env=prod`, `owner=<team>`, `costcenter=<cc>`, `dataclass=restricted`, `dr=active-active`.

---

## Phase 1 — Network Foundation (Primary Region)

**Goal:** Hub VNet (with ExpressRoute Gateway, Bastion, DNS Resolver) + two spokes (HSM client + HSM PE) in South India, peered hub-to-spoke.

### 1.1 Resource groups
```bash
az group create -n "$RG_HUB_PRI" -l "$LOC_PRI"
az group create -n "$RG_HSM_PRI" -l "$LOC_PRI"
az group create -n "$RG_APP_PRI" -l "$LOC_PRI"
```

### 1.2 Hub VNet + subnets
```bash
az network vnet create -g "$RG_HUB_PRI" -n vnet-org-hub-pri -l "$LOC_PRI" \
  --address-prefixes 10.10.0.0/16

az network vnet subnet create -g "$RG_HUB_PRI" --vnet-name vnet-org-hub-pri \
  -n GatewaySubnet --address-prefixes 10.10.0.0/27

az network vnet subnet create -g "$RG_HUB_PRI" --vnet-name vnet-org-hub-pri \
  -n AzureBastionSubnet --address-prefixes 10.10.1.0/26

az network vnet subnet create -g "$RG_HUB_PRI" --vnet-name vnet-org-hub-pri \
  -n snet-dnspr-inbound --address-prefixes 10.10.2.0/28 \
  --delegations Microsoft.Network/dnsResolvers

az network vnet subnet create -g "$RG_HUB_PRI" --vnet-name vnet-org-hub-pri \
  -n snet-dnspr-outbound --address-prefixes 10.10.2.16/28 \
  --delegations Microsoft.Network/dnsResolvers
```

### 1.3 ExpressRoute Gateway (Primary)
> Requires existing ExpressRoute **circuit** authorisation from your carrier.
```bash
az network public-ip create -g "$RG_HUB_PRI" -n pip-ergw-org-pri \
  --sku Standard --allocation-method Static --zone 1 2 3

az network vnet-gateway create -g "$RG_HUB_PRI" -n ergw-org-pri \
  --public-ip-address pip-ergw-org-pri \
  --vnet vnet-org-hub-pri --gateway-type ExpressRoute --sku ErGw1AZ
```
Then create a **connection** linking the gateway to the ExpressRoute circuit resource ID (carrier-supplied). Validate BGP peering with your on-prem network team.

### 1.4 Azure Bastion (Standard SKU minimum)
```bash
az network public-ip create -g "$RG_HUB_PRI" -n pip-bas-org-pri \
  --sku Standard --allocation-method Static

az network bastion create -g "$RG_HUB_PRI" -n bas-org-pri -l "$LOC_PRI" \
  --vnet-name vnet-org-hub-pri --public-ip-address pip-bas-org-pri \
  --sku Standard --enable-tunneling true
```

### 1.5 DNS Private Resolver
```bash
az dns-resolver create -g "$RG_HUB_PRI" -n dnspr-org-pri -l "$LOC_PRI" \
  --id "/subscriptions/$SUB_ID/resourceGroups/$RG_HUB_PRI/providers/Microsoft.Network/virtualNetworks/vnet-org-hub-pri"

az dns-resolver inbound-endpoint create -g "$RG_HUB_PRI" \
  --dns-resolver-name dnspr-org-pri -n in-ep --location "$LOC_PRI" \
  --ip-configurations '[{"private-ip-allocation-method":"Dynamic","id":"/subscriptions/'$SUB_ID'/resourceGroups/'$RG_HUB_PRI'/providers/Microsoft.Network/virtualNetworks/vnet-org-hub-pri/subnets/snet-dnspr-inbound"}]'
```
Record the inbound endpoint IP — on-prem DNS will forward `privatelink.managedhsm.azure.net` to it.

### 1.6 HSM Client Spoke
```bash
az network vnet create -g "$RG_APP_PRI" -n vnet-org-app-pri -l "$LOC_PRI" \
  --address-prefixes 10.11.0.0/16

az network vnet subnet create -g "$RG_APP_PRI" --vnet-name vnet-org-app-pri \
  -n snet-app --address-prefixes 10.11.1.0/24

az network vnet subnet create -g "$RG_APP_PRI" --vnet-name vnet-org-app-pri \
  -n snet-sqlmi --address-prefixes 10.11.2.0/27 \
  --delegations Microsoft.Sql/managedInstances
```

### 1.7 HSM Private-Endpoint Spoke
```bash
az network vnet create -g "$RG_HSM_PRI" -n vnet-org-hsm-pri -l "$LOC_PRI" \
  --address-prefixes 10.12.0.0/16

az network vnet subnet create -g "$RG_HSM_PRI" --vnet-name vnet-org-hsm-pri \
  -n snet-pe-hsm --address-prefixes 10.12.1.0/24 \
  --private-endpoint-network-policies Disabled
```

### 1.8 NSG on PE subnet (allow 443 from approved spokes only)
```bash
az network nsg create -g "$RG_HSM_PRI" -n nsg-pe-hsm-pri -l "$LOC_PRI"

az network nsg rule create -g "$RG_HSM_PRI" --nsg-name nsg-pe-hsm-pri \
  -n allow-app-spoke-443 --priority 100 --direction Inbound --access Allow \
  --protocol Tcp --source-address-prefixes 10.11.0.0/16 \
  --destination-address-prefixes 10.12.1.0/24 --destination-port-ranges 443

az network nsg rule create -g "$RG_HSM_PRI" --nsg-name nsg-pe-hsm-pri \
  -n deny-all-in --priority 4096 --direction Inbound --access Deny \
  --protocol '*' --source-address-prefixes '*' \
  --destination-address-prefixes '*' --destination-port-ranges '*'

az network vnet subnet update -g "$RG_HSM_PRI" --vnet-name vnet-org-hsm-pri \
  -n snet-pe-hsm --network-security-group nsg-pe-hsm-pri
```

### 1.9 Peer hub ↔ spokes (Primary)
```bash
# hub -> app spoke
az network vnet peering create -g "$RG_HUB_PRI" -n peer-hub-to-app \
  --vnet-name vnet-org-hub-pri \
  --remote-vnet "/subscriptions/$SUB_ID/resourceGroups/$RG_APP_PRI/providers/Microsoft.Network/virtualNetworks/vnet-org-app-pri" \
  --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit

# app spoke -> hub
az network vnet peering create -g "$RG_APP_PRI" -n peer-app-to-hub \
  --vnet-name vnet-org-app-pri \
  --remote-vnet "/subscriptions/$SUB_ID/resourceGroups/$RG_HUB_PRI/providers/Microsoft.Network/virtualNetworks/vnet-org-hub-pri" \
  --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways

# hub <-> hsm spoke (same pattern)
az network vnet peering create -g "$RG_HUB_PRI" -n peer-hub-to-hsm \
  --vnet-name vnet-org-hub-pri \
  --remote-vnet "/subscriptions/$SUB_ID/resourceGroups/$RG_HSM_PRI/providers/Microsoft.Network/virtualNetworks/vnet-org-hsm-pri" \
  --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit

az network vnet peering create -g "$RG_HSM_PRI" -n peer-hsm-to-hub \
  --vnet-name vnet-org-hsm-pri \
  --remote-vnet "/subscriptions/$SUB_ID/resourceGroups/$RG_HUB_PRI/providers/Microsoft.Network/virtualNetworks/vnet-org-hub-pri" \
  --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways
```

> **Spoke-to-spoke is intentionally NOT peered.** Traffic from app spoke → HSM PE spoke flows via Private Link on the Azure backbone, not via VNet peering.

### 1.10 Private DNS zone — **regional zone for Primary** (split-horizon design)

> **Why per-region zones?** Each region gets its own `privatelink.managedhsm.azure.net` zone, linked only to that region's VNets. That way the same HSM FQDN resolves to the **local** Private Endpoint IP in each region — Primary workloads always hit the Primary PE, DR workloads always hit the DR PE. No cross-region hops in steady state, and no manual DNS edits during failover. Failover at the global `*.managedhsm.azure.net` FQDN is handled separately by Azure Traffic Manager (Phase 3 §3.4). A single shared zone with manual DNS-swing was deliberately rejected: it required a human edit inside a potentially-failed region and stranded the DR HSM's capacity in steady state.

```bash
az network private-dns zone create -g "$RG_HUB_PRI" -n privatelink.managedhsm.azure.net

# Link to Primary hub
az network private-dns link vnet create -g "$RG_HUB_PRI" \
  -n link-hub-pri -z privatelink.managedhsm.azure.net \
  --virtual-network vnet-org-hub-pri --registration-enabled false

# Link to Primary spokes (so VMs and SQL MI resolve PE FQDN to the Primary PE)
az network private-dns link vnet create -g "$RG_HUB_PRI" \
  -n link-app-pri -z privatelink.managedhsm.azure.net \
  --virtual-network "/subscriptions/$SUB_ID/resourceGroups/$RG_APP_PRI/providers/Microsoft.Network/virtualNetworks/vnet-org-app-pri" \
  --registration-enabled false

az network private-dns link vnet create -g "$RG_HUB_PRI" \
  -n link-hsm-pri -z privatelink.managedhsm.azure.net \
  --virtual-network "/subscriptions/$SUB_ID/resourceGroups/$RG_HSM_PRI/providers/Microsoft.Network/virtualNetworks/vnet-org-hsm-pri" \
  --registration-enabled false
```

> **Do NOT link this Primary zone to any DR VNet.** DR VNets get their own zone (Phase 2) so resolution stays local to each region.

---

## Phase 2 — Network Foundation (DR Region)

Repeat **Phase 1** in the DR region (example: Central India) with the DR names and CIDRs (10.20.x / 10.21.x / 10.22.x). Salient differences:

- Resource groups: `rg-org-hub-dr`, `rg-org-hsm-dr`, `rg-org-app-dr`.
- VNets: `vnet-org-hub-dr` (10.20.0.0/16), `vnet-org-app-dr` (10.21.0.0/16), `vnet-org-hsm-dr` (10.22.0.0/16).
- HSM PE subnet `snet-pe-hsm` = `10.22.1.0/24`.
- **Create a separate `privatelink.managedhsm.azure.net` zone in the DR hub RG** and link it **only to DR VNets** (split-horizon resolution):

```bash
az network private-dns zone create -g "$RG_HUB_DR" -n privatelink.managedhsm.azure.net

for VNET_RG_PAIR in "vnet-org-hub-dr:$RG_HUB_DR" "vnet-org-app-dr:$RG_APP_DR" "vnet-org-hsm-dr:$RG_HSM_DR"; do
  VN="${VNET_RG_PAIR%%:*}"; VRG="${VNET_RG_PAIR##*:}"
  az network private-dns link vnet create -g "$RG_HUB_DR" \
    -n "link-${VN}" -z privatelink.managedhsm.azure.net \
    --virtual-network "/subscriptions/$SUB_ID/resourceGroups/$VRG/providers/Microsoft.Network/virtualNetworks/$VN" \
    --registration-enabled false
done
```

> **Why per-region zones:** the FQDN `org-hsm-prod.privatelink.managedhsm.azure.net` is the same in both regions, but each region's zone holds **only the local PE's A record**. A Primary-region VM will always resolve to the Primary PE (10.12.1.4); a DR-region VM will always resolve to the DR PE (10.22.1.4). No record drift, no coexisting A records, no round-robin surprises, and no manual DNS swing required during failover. Failover at the global `*.managedhsm.azure.net` FQDN is handled by Azure Traffic Manager (Phase 3 §3.4).

---

## Phase 3 — Global Connectivity (Peering, ExpressRoute, DNS Link)

### 3.1 Global VNet Peering — Hub-to-Hub
```bash
az network vnet peering create -g "$RG_HUB_PRI" -n peer-hub-pri-to-dr \
  --vnet-name vnet-org-hub-pri \
  --remote-vnet "/subscriptions/$SUB_ID/resourceGroups/$RG_HUB_DR/providers/Microsoft.Network/virtualNetworks/vnet-org-hub-dr" \
  --allow-vnet-access --allow-forwarded-traffic

az network vnet peering create -g "$RG_HUB_DR" -n peer-hub-dr-to-pri \
  --vnet-name vnet-org-hub-dr \
  --remote-vnet "/subscriptions/$SUB_ID/resourceGroups/$RG_HUB_PRI/providers/Microsoft.Network/virtualNetworks/vnet-org-hub-pri" \
  --allow-vnet-access --allow-forwarded-traffic
```
> Do **not** enable gateway transit on the hub-to-hub link unless you have a deliberate transit design — each region keeps its own ExpressRoute Gateway.

### 3.2 ExpressRoute (DR)
Create `pip-ergw-org-dr`, `ergw-org-dr` and a circuit connection in `rg-org-hub-dr` exactly as in §1.3. Coordinate with the carrier for the DR circuit.

### 3.3 On-prem DNS forwarding (region-aware)

On the on-prem DNS, configure **conditional forwarders** for `privatelink.managedhsm.azure.net` so that on-prem clients resolve to the nearest healthy regional PE:

- **On-prem sites homed to Primary** (geographically closer to the Primary region): forwarder order = **Primary Resolver inbound IP first, DR second**.
- **On-prem sites homed to DR** (geographically closer to the DR region): forwarder order = **DR Resolver inbound IP first, Primary second**.

This gives steady-state local resolution and an automatic on-prem fall-through if the local Resolver becomes unreachable. Because each region's Private DNS zone is separate and holds only its **own** PE A record (§1.10 and Phase 2), an on-prem query that lands on the Primary Resolver gets the Primary PE IP, and a query that lands on the DR Resolver gets the DR PE IP — no record collisions, no round-robin.

### 3.4 Azure Traffic Manager — global automatic failover (recommended)

The FQDN `org-hsm-prod.managedhsm.azure.net` (without the `privatelink.` infix) is the **public/global** name. With native multi-region replication (Phase 6 §6.1) Azure provisions a Traffic Manager profile that health-probes both regional endpoints and answers DNS based on which is healthy. If you used the native `az keyvault region add` path in Phase 6, this is already in place — verify and stop. If you took the manual SD-import fallback, create the profile explicitly:

```bash
# Create a Traffic Manager profile (Priority routing = active-passive at the global DNS layer,
# while data-plane stays active-active per region via the regional Private DNS zones above)
az network traffic-manager profile create \
  -g rg-org-global -n tm-org-hsm-prod \
  --routing-method Priority \
  --unique-dns-name org-hsm-prod-tm \
  --ttl 30 \
  --protocol HTTPS --port 443 --path "/keys?api-version=7.4"

# Add Primary HSM endpoint (priority 1)
az network traffic-manager endpoint create \
  -g rg-org-global --profile-name tm-org-hsm-prod \
  -n hsm-pri --type externalEndpoints \
  --target org-hsm-prod.southindia.managedhsm.azure.net \
  --priority 1 --endpoint-status Enabled

# Add DR HSM endpoint (priority 2)
az network traffic-manager endpoint create \
  -g rg-org-global --profile-name tm-org-hsm-prod \
  -n hsm-dr --type externalEndpoints \
  --target org-hsm-prod.centralindia.managedhsm.azure.net \
  --priority 2 --endpoint-status Enabled
```

> **Why Priority routing and not Performance:** in-region data-plane traffic already resolves to the **local** PE via split-horizon Private DNS — it never reaches Traffic Manager. TM only matters for (a) hybrid/SaaS callers that hit the public FQDN, and (b) automatic failover to the DR endpoint if the Primary becomes unhealthy. Priority routing makes the failover deterministic and easy to reason about.

> **Manual DNS swing is no longer the primary failover mechanism.** It is retained in §11.5 only as a last-resort runbook in the (rare) scenario where Traffic Manager and Azure-managed DNS for the HSM endpoint are unavailable at the same time as the Primary region.

### 3.5 Validate connectivity (before continuing)
From a jump VM in `snet-app` (deploy a small temporary VM via Bastion):
- `Test-NetConnection <primary-pe-fqdn> -Port 443` → succeeds (after Phase 7).
- `Resolve-DnsName org-hsm-prod.privatelink.managedhsm.azure.net` from a **Primary** VNet → `10.12.1.4` (Primary PE).
- `Resolve-DnsName org-hsm-prod.privatelink.managedhsm.azure.net` from a **DR** VNet → `10.22.1.4` (DR PE).
- `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` from on-prem → CNAME chain through Traffic Manager, finally resolves (via the on-prem forwarder → Private Resolver) to the **local-region** PE IP.
- From on-prem: same checks via ExpressRoute, confirming the region-aware forwarder order from §3.3.

---

## Phase 4 — Provision Managed HSM (Primary)

### 4.1 Choose initial administrators
Collect Entra **Object IDs** of the officers who will participate in the Security Domain ceremony (minimum N=3, recommended 5). These become **initial administrators** with full data-plane rights on the HSM.

> **What `--administrators` actually grants:** these object IDs are written into the HSM's local-RBAC root scope with the `Managed HSM Administrator` role *before* the Security Domain exists. They are the only identities that can run the activation command in §5.2. Choose carefully — you cannot remove them until after the HSM is activated and additional admins are assigned in Phase 8.

### 4.2 Create the HSM (provisioning state)
```bash
ADMINS="$ADMIN_OID_1 $ADMIN_OID_2 $ADMIN_OID_3"

az keyvault create --hsm-name "$HSM_NAME" \
  --resource-group "$RG_HSM_PRI" --location "$LOC_PRI" \
  --administrators $ADMINS \
  --retention-days 90 \
  --enable-purge-protection true
```
- `--retention-days 90` → soft-delete window (cannot be reduced once set).
- `--enable-purge-protection true` → required to block premature destruction.

The HSM provisions a **3-node cluster** (T1 SKU) in ~20–30 min and lands in state **`Provisioning` → `Active (pending security domain)`**.

### 4.3 Network access during activation

> **Clarification:** Managed HSM activation (the Security Domain ceremony in §5) requires **data-plane access** from the administrator workstation to the HSM — not specifically the public internet. If you have already provisioned a Private Endpoint (Phase 7) and your admin workstation can reach it (via Bastion + jump VM, or via ExpressRoute from on-prem), you can keep public access **Disabled** throughout. The IP-allow-listed public path below is the **practical fallback** when the private path isn't ready yet.

**Option A — Private path already in place (preferred):** skip §4.3, jump straight to §7.1–§7.3 to stand up Private Endpoints, then run §5 from a jump VM inside `snet-app`.

**Option B — Temporary public access with IP allow-list (fallback for green-field):**
```bash
# Allow only the admin team's egress IPs during activation; revoke in §7.4
az keyvault update-hsm --hsm-name "$HSM_NAME" -g "$RG_HSM_PRI" \
  --public-network-access Enabled \
  --default-action Deny \
  --bypass AzureServices

az keyvault network-rule add --hsm-name "$HSM_NAME" \
  --ip-address <admin-egress-public-ip>/32
```

---

## Phase 5 — Activate HSM & Generate Security Domain (M-of-N Ceremony)

> **This is the most security-critical step in the whole project.** Loss of the Security Domain = permanent loss of all keys in the HSM (no Microsoft recovery). Treat the ceremony as a controlled event.

> **Plain-English summary of what happens here:** you walk into a room with N officers, each holding a public key they generated offline. You run **one** Azure CLI command. The HSM hardware generates its own root-of-trust material *inside the box*, encrypts that material so that any M of the N public keys can later decrypt it, and writes out a single encrypted file (the **Security Domain file**). At the moment this command succeeds, the HSM transitions from `Pending` to `Activated` and is ready to hold keys.

### 5.1 Pre-ceremony
- [ ] Conference room booked. Video/audio recording per company policy.
- [ ] Custodians present (N people, e.g. 5).
- [ ] Each custodian brings their **public key file** (`.cer`/PEM) generated offline.
- [ ] Verify each `.cer` belongs to its custodian (fingerprint read aloud, compared against pre-shared record).
- [ ] An administrator workstation has Azure CLI + access to HSM activation endpoint.
- [ ] Storage location for the encrypted **Security Domain file** decided (see §5.4).

### 5.2 Generate, encrypt, and activate the HSM ("download Security Domain")

> **Naming note:** the CLI verb is `download`, but this single command performs **three** actions on the HSM side: (1) **generates** the Security Domain inside the HSM hardware, (2) **encrypts** it using the N supplied public keys with an M-of-N quorum policy, and (3) **transitions the HSM from `Pending` to `Activated`**. The encrypted SD file that lands on your workstation is the *artefact* of that activation — there is no separate "activate" command.

```bash
# Concatenate the N public-key certificates into a directory, e.g. ./sd-certs/
# Files: officer1.cer officer2.cer ... officer5.cer

az keyvault security-domain download \
  --hsm-name "$HSM_NAME" \
  --sd-wrapping-keys ./sd-certs/officer1.cer ./sd-certs/officer2.cer \
                     ./sd-certs/officer3.cer ./sd-certs/officer4.cer \
                     ./sd-certs/officer5.cer \
  --sd-quorum 3 \
  --security-domain-file org-hsm-prod-SD.json
```
- A **single encrypted Security Domain file** is produced. It is encrypted to the N public keys; ANY 3-of-5 private keys can later reconstruct it.
- The HSM is now **Activated**. Keys can be created from this point on.

### 5.3 Verify activation
```bash
az keyvault show --hsm-name "$HSM_NAME" --query "properties.statusMessage"
# expect: "The Managed HSM is operational"
```

### 5.4 Custody and storage of the Security Domain file and officer private keys
- **Security Domain file (`org-hsm-prod-SD.json`)** — store in **three** geographically separated, access-controlled locations:
  1. Your physical vault/safe, encrypted USB + paper-printed base64.
  2. Azure Storage account in a different subscription, immutable blob, customer-managed key from a *different* HSM/Key Vault.
  3. Offline cold-storage drive, sealed envelope, in DR datacentre.
- **Officer private keys** — each custodian holds their own. Spread across at least 3 physical sites. Each private key in a hardware token (FIDO2 / smart card) or encrypted USB with passphrase.
- **Never** store M or more private keys in the same physical location.

### 5.5 Post-ceremony record
Write and sign:
- Names of N custodians and which public key fingerprint each provided.
- Quorum M value (e.g. 3).
- SHA-256 of `org-hsm-prod-SD.json`.
- Storage locations of SD file copies.
- Date, time, witness signatures.

Store the record in your compliance repository.

---

## Phase 6 — Provision DR HSM & Enable Multi-Region Replication

> The DR HSM is **not a separate vault**. It is a replica of the Primary HSM — same Security Domain, same keys, same key URIs.
>
> **Two paths are supported.** The Azure-native path (§6.1) is **preferred** for any new deployment because it links the two pools as one logical HSM, provisions Traffic Manager for the global endpoint automatically, and shows both regions under a single resource in the portal. The manual SD-import path (§6.2) is retained as a fallback for environments where the native command isn't yet available or where a pre-existing DR HSM must be brought into the replication pair.

### 6.1 **Preferred** — Native multi-region replication (`az keyvault region add`)

From an admin workstation that can reach the Primary HSM's data plane (via Bastion + jump VM after Phase 7, or via the temporary public allow-list from §4.3 during initial bring-up):

```bash
# Add Central India as an extended region of the existing Primary HSM
az keyvault region add \
  --hsm-name "$HSM_NAME" \
  --region-name "$LOC_DR"

# Verify
az keyvault region list --hsm-name "$HSM_NAME" -o table
# Expect two rows: southindia (Primary, Provisioning state Succeeded) and centralindia (Provisioning state Succeeded)
```

This single command:
- Provisions the DR HSM cluster in Central India.
- Imports the Security Domain from the Primary automatically (no second M-of-N ceremony required).
- Registers the DR pool into the same Traffic Manager profile that fronts the global `*.managedhsm.azure.net` FQDN.
- Surfaces both regions under one resource in the portal (`Multi-region replication` blade).

Proceed to **§6.3** to verify replication functionally, then to **Phase 7** to create the DR Private Endpoint and register it in the DR-local Private DNS zone.

### 6.2 Fallback — Manual replica with Security Domain import

Use this path only if §6.1 is not available in your CLI version or region. It produces the same end state but requires the Phase 5 SD file and **M physical custodians** to re-cooperate.

#### 6.2.1 Create the DR HSM shell (same name, in DR region)
```bash
az keyvault create --hsm-name "$HSM_NAME" \
  --resource-group "$RG_HSM_DR" --location "$LOC_DR" \
  --administrators $ADMINS \
  --retention-days 90 \
  --enable-purge-protection true
```

> Same HSM **pool name** is by design — Managed HSM multi-region replication requires it. Azure distinguishes the two by region.

#### 6.2.2 Upload Security Domain to DR HSM
The DR HSM is born in **pending Security Domain** state. Upload the SD file generated in Phase 5.

```bash
az keyvault security-domain upload \
  --hsm-name "$HSM_NAME" --resource-group "$RG_HSM_DR" \
  --sd-file org-hsm-prod-SD.json \
  --sd-wrapping-keys ./officer-private-keys/officer1.key \
                     ./officer-private-keys/officer2.key \
                     ./officer-private-keys/officer3.key
```
- This requires **M (quorum) private keys** — so **M custodians must physically participate** (or pre-stage the M keys on one trusted workstation under four-eyes control).
- After upload, DR HSM is **Activated** and is a true mirror.
- If you took this fallback path, **also create the Traffic Manager profile from §3.4 manually** — it is not auto-provisioned.

### 6.3 Verify multi-region replication (functional test)

With both HSMs sharing the same Security Domain, replication is established and maintained by the Managed HSM service. There is **no documented CLI guarantee** that exposes the region list reliably for the manual path (the `properties.regions` field is not a stable contract; only the native path in §6.1 surfaces a clean `az keyvault region list`). Verify replication **functionally**:

```bash
# 1. Create a canary key on the Primary HSM (from a jump VM that can reach the Primary PE)
az keyvault key create --hsm-name "$HSM_NAME" -n canary-replication-test \
  --kty RSA-HSM --size 3072 --ops sign verify

# Capture the kid
KID=$(az keyvault key show --hsm-name "$HSM_NAME" -n canary-replication-test --query key.kid -o tsv)

# 2. From a jump VM in the DR app spoke (snet-app in vnet-org-app-dr) — DNS there resolves
#    org-hsm-prod.privatelink.managedhsm.azure.net to the DR PE (10.22.1.4)
az keyvault key show --hsm-name "$HSM_NAME" -n canary-replication-test --query key.kid -o tsv
# Expect: same kid value — but allow up to ~6 minutes for the DR pool to converge (see 6.4)

# 3. Sign on Primary, verify on DR (proves the private key material is replicated, not just metadata)
echo -n "canary-test" | az keyvault key sign --id "$KID" --algorithm RS256 --value @- > sig.b64    # run from Primary
az keyvault key verify --id "$KID" --algorithm RS256 --digest <sha256-of-canary-test> --signature @sig.b64   # run from DR

# 4. Clean up
az keyvault key delete --hsm-name "$HSM_NAME" -n canary-replication-test
```

### 6.4 Replication lag — eventual consistency window

Managed HSM multi-region replication is **eventually consistent**: per Microsoft's published behaviour, write operations (key create/update/delete, role assignments, role definition changes) can take **up to ~6 minutes** to propagate from the writer pool to the extended pool. The HSM service serialises and replicates over the Azure backbone, but there is no synchronous-commit guarantee.

**Operational rules that fall out of this:**

1. **Pre-failover quiet period.** Before any *planned* regional failover or maintenance, allow at least 6 minutes after the last control-plane write (new key, new role assignment, key rotation, etc.). Where possible, validate that `HsmReplicationLatencyMs` is at baseline before proceeding.
2. **No back-to-back writer flipping.** Do not perform a write in Primary and immediately route the next workload call to DR — the change may not be visible yet. Either pin the workload to the writer region until convergence is confirmed, or rely on application-side retry (§9.5).
3. **Bulk onboarding.** When onboarding many keys/roles at once, batch the work and then *wait* before declaring DR equivalence — don't drive the canary test (§6.3) immediately after the last write.
4. **Operational documentation.** All key-rotation runbooks and role-assignment change tickets must reference the 6-minute window so on-call doesn't mistake transient `404 KeyNotFound` / `403 Forbidden` for a real fault.
5. **Alerting.** Keep the `HsmReplicationLatencyMs` alert from §10.3 (P1 if > 1000 ms sustained 5 min) — it is the operational signal that the replication path itself is unhealthy versus just propagating normally.

---

## Phase 7 — Lock Down Network (Private Endpoints, NSG, Public Access OFF)

### 7.1 Create Private Endpoint to Primary HSM
```bash
HSM_ID_PRI=$(az keyvault show --hsm-name "$HSM_NAME" -g "$RG_HSM_PRI" --query id -o tsv)

az network private-endpoint create -g "$RG_HSM_PRI" -n pe-hsm-pri -l "$LOC_PRI" \
  --vnet-name vnet-org-hsm-pri --subnet snet-pe-hsm \
  --private-connection-resource-id "$HSM_ID_PRI" \
  --group-id managedhsm \
  --connection-name conn-hsm-pri
```

### 7.2 Register Primary PE in the **Primary-region** Private DNS zone
```bash
DNS_ZONE_ID_PRI=$(az network private-dns zone show -g "$RG_HUB_PRI" \
  -n privatelink.managedhsm.azure.net --query id -o tsv)

az network private-endpoint dns-zone-group create -g "$RG_HSM_PRI" \
  --endpoint-name pe-hsm-pri -n zg-hsm-pri \
  --private-dns-zone "$DNS_ZONE_ID_PRI" \
  --zone-name privatelink-managedhsm-azure-net
```
Verify: `nslookup org-hsm-prod.privatelink.managedhsm.azure.net` from a VM in any **Primary** linked spoke returns **10.12.1.4**. The same query from a DR VNet must NOT resolve to this IP — it should resolve to the DR PE IP from the DR-region zone (§7.3).
### 7.3 Repeat for DR — register DR PE in the **DR-region** Private DNS zone
```bash
HSM_ID_DR=$(az keyvault show --hsm-name "$HSM_NAME" -g "$RG_HSM_DR" --query id -o tsv)

az network private-endpoint create -g "$RG_HSM_DR" -n pe-hsm-dr -l "$LOC_DR" \
  --vnet-name vnet-org-hsm-dr --subnet snet-pe-hsm \
  --private-connection-resource-id "$HSM_ID_DR" \
  --group-id managedhsm \
  --connection-name conn-hsm-dr

# Register DR PE into the DR-region Private DNS zone (created in Phase 2)
DNS_ZONE_ID_DR=$(az network private-dns zone show -g "$RG_HUB_DR" \
  -n privatelink.managedhsm.azure.net --query id -o tsv)

az network private-endpoint dns-zone-group create -g "$RG_HSM_DR" \
  --endpoint-name pe-hsm-dr -n zg-hsm-dr \
  --private-dns-zone "$DNS_ZONE_ID_DR" \
  --zone-name privatelink-managedhsm-azure-net
```

> **Split-horizon is now complete.** Each region's zone holds **only its own PE A record**:
>
> | Source VNet | Zone consulted | A record returned |
> |---|---|---|
> | Any Primary VNet (`vnet-org-*-pri`) | Primary zone in `rg-org-hub-pri` | `10.12.1.4` |
> | Any DR VNet (`vnet-org-*-dr`) | DR zone in `rg-org-hub-dr` | `10.22.1.4` |
>
> There is **no shared zone, no coexisting A records, no resolver round-robin, and no manual DNS swing required for failover**. Workloads always hit the **local** HSM in steady state. The global `*.managedhsm.azure.net` FQDN is health-routed by Traffic Manager (§3.4) for the rare callers that go through the public name.

### 7.4 Disable public access on BOTH HSMs
```bash
for RG in "$RG_HSM_PRI" "$RG_HSM_DR"; do
  az keyvault update-hsm --hsm-name "$HSM_NAME" -g "$RG" \
    --public-network-access Disabled \
    --default-action Deny
done
```
From this point onward, **all HSM data-plane traffic must traverse Private Endpoints** — i.e. from inside the VNets or via ExpressRoute + DNS forwarder from on-prem.

### 7.5 Smoke test from inside the VNet
From a Bastion-attached jump VM in `snet-app` (Primary):
```bash
az login --identity   # if VM has system-assigned MI; or use az login --service-principal
az keyvault key list --hsm-name "$HSM_NAME"
```
If DNS resolves to 10.12.1.4 and a 200/403 is returned (403 means auth needed — that's fine, connectivity works), private path is healthy.

---

## Phase 8 — HSM Data-Plane RBAC and Key Creation

> **First-timer note:** Managed HSM uses its **own local RBAC** at three scopes (HSM-wide `/`, all keys `/keys`, individual key `/keys/{name}`). This is **separate from Azure RBAC** — a user who is `Owner` of the Azure subscription has **no** ability to use keys until you also grant them an HSM-local role here. Likewise, removing someone from Azure RBAC does **not** revoke their HSM-local grants; that has to be done explicitly with `az keyvault role assignment delete`.

### 8.1 Assign administrator role
The officers from Phase 4 already have `Managed HSM Administrator`. Add the admin group:
```bash
ADMIN_GROUP_OID=$(az ad group show -g grp-org-hsm-admins --query id -o tsv)

az keyvault role assignment create --hsm-name "$HSM_NAME" \
  --role "Managed HSM Administrator" \
  --assignee-object-id "$ADMIN_GROUP_OID" \
  --assignee-principal-type Group \
  --scope "/"
```

### 8.2 Assign crypto officer role (key lifecycle)
```bash
CO_OID=$(az ad group show -g grp-org-hsm-crypto-officers --query id -o tsv)
az keyvault role assignment create --hsm-name "$HSM_NAME" \
  --role "Managed HSM Crypto Officer" \
  --assignee-object-id "$CO_OID" --assignee-principal-type Group \
  --scope "/keys"
```

### 8.3 Create CMK keys (per workload)
```bash
# Key for VM Disk Encryption Set
az keyvault key create --hsm-name "$HSM_NAME" -n cmk-vmdisk-prod \
  --kty RSA-HSM --size 3072 --ops wrapKey unwrapKey

# Key for SQL Managed Instance TDE — include encrypt/decrypt so the same key can be
# reused for ad-hoc cryptographic operations during operational tasks (e.g. validation,
# manual unseal tests) without minting a second key.
az keyvault key create --hsm-name "$HSM_NAME" -n cmk-sqlmi-prod \
  --kty RSA-HSM --size 3072 --ops wrapKey unwrapKey encrypt decrypt

# Key for application-tier envelope encryption
az keyvault key create --hsm-name "$HSM_NAME" -n cmk-app-prod \
  --kty RSA-HSM --size 3072 --ops wrapKey unwrapKey encrypt decrypt
```
Each key is **immediately replicated** to the DR HSM (gold thick arrow in the diagram — *HSM Multi-Region Replication, same Security Domain*).

Capture and document the **versioned key URI**:
```bash
az keyvault key show --hsm-name "$HSM_NAME" -n cmk-vmdisk-prod --query key.kid -o tsv
# https://org-hsm-prod.managedhsm.azure.net/keys/cmk-vmdisk-prod/<version>
```

### 8.4 Assign least-privilege roles to workload identities
For each workload identity (DES, SQL MI, VM MI), assign **scoped** rights:
```bash
# DES principal id
DES_OID=$(az disk-encryption-set show -n des-vmdisk-prod -g "$RG_APP_PRI" --query identity.principalId -o tsv)

az keyvault role assignment create --hsm-name "$HSM_NAME" \
  --role "Managed HSM Crypto Service Encryption User" \
  --assignee-object-id "$DES_OID" --assignee-principal-type ServicePrincipal \
  --scope "/keys/cmk-vmdisk-prod"
```
Repeat per key for SQL MI managed identity and VM/App managed identities, picking the smallest applicable built-in role from Appendix A.

---

## Phase 9 — Workload Integration (VM Disks, SQL MI TDE, App Tier)

> Do the **Primary region first**, validate end-to-end, then mirror in DR.

### 9.1 Virtual Machines — CMK via Disk Encryption Set
```bash
# Create DES bound to the HSM key (versionless URI is recommended for auto-rotation)
KID_VMDISK="https://${HSM_NAME}.managedhsm.azure.net/keys/cmk-vmdisk-prod"

az disk-encryption-set create -g "$RG_APP_PRI" -n des-vmdisk-prod -l "$LOC_PRI" \
  --source-vault "$HSM_NAME" \
  --key-url "$KID_VMDISK" \
  --encryption-type EncryptionAtRestWithCustomerKey \
  --enable-auto-key-rotation true \
  --mi-system-assigned

# Grant the DES MI access on the key (see §8.4)

# Create or update a managed disk to use the DES
az disk create -g "$RG_APP_PRI" -n disk-app01-os -l "$LOC_PRI" \
  --size-gb 128 --sku Premium_LRS \
  --encryption-type EncryptionAtRestWithCustomerKey \
  --disk-encryption-set des-vmdisk-prod
```
Attach to your application VMs in `snet-app`. Existing VMs can be re-encrypted by changing the DES on the disk (involves a stop/start).

### 9.2 SQL Managed Instance — TDE with CMK
```bash
# SQL MI must already exist in snet-sqlmi (delegated subnet from §1.6)

# Assign SQL MI managed identity Crypto User on the key
SQLMI_OID=$(az sql mi show -g "$RG_APP_PRI" -n sqlmi-org-pri --query identity.principalId -o tsv)
az keyvault role assignment create --hsm-name "$HSM_NAME" \
  --role "Managed HSM Crypto Service Encryption User" \
  --assignee-object-id "$SQLMI_OID" --assignee-principal-type ServicePrincipal \
  --scope "/keys/cmk-sqlmi-prod"

# Set the TDE protector to the HSM key
KID_SQLMI=$(az keyvault key show --hsm-name "$HSM_NAME" -n cmk-sqlmi-prod --query key.kid -o tsv)

az sql mi key create -g "$RG_APP_PRI" --mi sqlmi-org-pri --kid "$KID_SQLMI"
az sql mi tde-key set -g "$RG_APP_PRI" --mi sqlmi-org-pri \
  --server-key-type AzureKeyVault --kid "$KID_SQLMI"
```
Verify: `az sql mi tde-key show -g "$RG_APP_PRI" --mi sqlmi-org-pri`.

> **Terminology note:** the SQL CLI uses `--server-key-type AzureKeyVault` and the portal labels it "Azure Key Vault" even when the underlying key lives in a **Managed HSM** at `*.managedhsm.azure.net`. This is expected — SQL MI treats both as the same key-store family. Confirm via the `kid` value, not the label.

### 9.3 Application tier (envelope encryption)
For any organisation app that wraps its own DEKs:
1. Enable **system-assigned managed identity** on the VM / App Service / AKS workload.
2. Grant it `Managed HSM Crypto User` on `/keys/cmk-app-prod`.
3. App uses **Azure Key Vault SDK** (`Azure.Security.KeyVault.Keys.Cryptography`) pointed at `https://org-hsm-prod.managedhsm.azure.net/keys/cmk-app-prod` for `WrapKey`/`UnwrapKey`.
4. Cache the **wrapped DEK** alongside ciphertext; only the wrapped DEK leaves the HSM — the raw key never does.

### 9.4 Repeat in DR
Recreate **DES (DR)** in `rg-org-app-dr`, **SQL MI (DR)**, and DR app workloads pointing to the **same key URI** (`https://org-hsm-prod.managedhsm.azure.net/keys/<name>`).

> **DNS in the active-active design:** because each region has its own Private DNS zone (§1.10, Phase 2) linked only to its own VNets, a DR workload resolving `org-hsm-prod.privatelink.managedhsm.azure.net` always lands on the **DR PE** (10.22.1.4) and a Primary workload always lands on the **Primary PE** (10.12.1.4). No cross-region hops occur in steady state and no per-workload region-pinning logic is needed. The global `*.managedhsm.azure.net` FQDN is health-routed by Traffic Manager (§3.4) for any caller that goes through the public name.

### 9.5 Application-side resilience (mandatory for production workloads)

Even with Active-Active DNS and Traffic Manager, applications must defend against the **eventual-consistency window** of replication (up to ~6 minutes, see §6.4) and against transient network blips. Every application calling the HSM **must** implement:

1. **Retry with exponential back-off** on transient failures (HTTP 408, 429, 5xx, socket timeouts). The Azure Key Vault SDKs ship a built-in retry policy — keep it enabled and tune `MaxRetries = 5`, base delay 800 ms. This also covers the `404 KeyNotFound` / `403 Forbidden` that can appear briefly when a key or role assignment was just written in the *other* region and has not yet replicated.
2. **Region-aware retry / fallback to the writer region.** If repeated `404 KeyNotFound` or `403 Forbidden` errors are seen against the local PE shortly after a known control-plane change, retry against the **global** Traffic Manager FQDN (`org-hsm-prod.managedhsm.azure.net`) — this will route to the region that performed the write while replication completes.
3. **Short DNS TTL awareness.** Azure Private DNS A records default to 10 s TTL — adequate. The Traffic Manager profile in §3.4 uses TTL 30. Do not cache resolved IPs in the application beyond the TTL.
4. **Wrapped-DEK caching.** Cache the *wrapped* DEK in the application tier for the lifetime of the request/session so a transient HSM blip doesn't fail user-facing operations.
5. **Circuit breaker + fail-fast.** After N consecutive HSM failures within W seconds, open a circuit and surface a clear health signal to the platform so an operator can confirm Traffic Manager has flipped to DR (and, in the very rare worst case, invoke the manual DNS fallback in §11.5) rather than letting the app silently degrade.
6. **Honour the 6-minute pre-failover quiet period (§6.4).** Operational tooling that performs key rotation, role grants, or new key creation must record the timestamp of the last write so that any subsequent *planned* failover honours the convergence window. Unplanned failovers are handled by the SDK retry + region-aware retry above.

---

## Phase 10 — Backup, Monitoring & Logging

### 10.1 HSM full backup (control-plane, periodic + event-triggered)
```bash
STG="<backup-storage-account>"
CONTAINER="hsm-backups"
SAS="<container-sas-with-create-write>"

az keyvault backup start --hsm-name "$HSM_NAME" \
  --storage-account-name "$STG" \
  --blob-container-name "$CONTAINER" \
  --storage-container-SAS-token "$SAS"
```

**Backup cadence (revised — weekly alone is insufficient for an RPO-sensitive key store):**

| Trigger | Frequency / condition | Rationale |
|---|---|---|
| Scheduled full backup | **Daily** off-peak (Azure Automation / Logic App) | Caps key-management RPO at 24 h even in the unlikely event both regional pools are lost |
| Event-triggered full backup | After any **bulk key creation**, **key rotation campaign**, **mass role-assignment change**, or **before any planned HSM maintenance** | Narrows the window where a fresh key/role exists only in the live HSM and not in backup storage |
| Retention | 12 months, **immutable** (legal-hold or time-based immutability) blob container, in a **different region** (e.g. South East Asia) | Protects against ransomware/accidental deletion and regional storage outages |
| Encryption of backup blob | Customer-managed key from a **different** HSM / Key Vault than `org-hsm-prod` | Avoids circular dependency — you cannot use the HSM you are restoring to encrypt its own backup |

**Restore drill (annual, distinct from the SD restore drill in §11.4):** at least once per year, take the most recent backup blob and run `az keyvault restore start` into a **throwaway test HSM** in a non-production subscription. Verify:
- the restore completes inside the **30-minute time limit** Azure enforces on HSM restore jobs;
- the restored HSM exposes the expected keys and role assignments;
- the team has practiced the restore commands and SAS-token handling under timed conditions.
Delete the test HSM at the end of the drill. Record the result alongside the SD ceremony log.

### 10.2 Diagnostic settings → Log Analytics + Storage
```bash
LAW_ID=$(az monitor log-analytics workspace show -g rg-org-mon -n law-org-pri --query id -o tsv)

for RG in "$RG_HSM_PRI" "$RG_HSM_DR"; do
  HSM_ID=$(az keyvault show --hsm-name "$HSM_NAME" -g "$RG" --query id -o tsv)
  az monitor diagnostic-settings create --name diag-hsm \
    --resource "$HSM_ID" \
    --workspace "$LAW_ID" \
    --logs '[{"category":"AuditEvent","enabled":true},
              {"category":"AzurePolicyEvaluationDetails","enabled":true}]' \
    --metrics '[{"category":"AllMetrics","enabled":true}]'
done
```

### 10.3 Alerts (minimum set)
Create Azure Monitor alerts for:
| Signal | Alert |
|---|---|
| `AuditEvent` where `operationName == "VaultDelete"` or `Purge*` | P1 — page Security on-call |
| Failed authentication rate > 5/min | P2 |
| Key version not used in 30 days (custom KQL) | P3 — review for cleanup |
| HSM region health degraded | P1 — initiate DR runbook |
| Replication lag (`HsmReplicationLatencyMs`) > 1000 ms sustained 5 min | P1 |

### 10.4 Activity log alerts
Alert on any `Microsoft.KeyVault/managedHSMs/write` or `delete` from non-pipeline principals.

---

## Phase 11 — DR Failover Drill & Runbook

### 11.1 Failover model (Active-Active)
- The HSM key store is **Active-Active**: both regional pools serve their own local PE in steady state, and the global `*.managedhsm.azure.net` FQDN is health-routed by Traffic Manager (§3.4).
- **There is no manual DNS swing in the normal failover path.** If the Primary region or its HSM endpoint becomes unhealthy, Traffic Manager fails the global FQDN over to the DR endpoint automatically (TTL 30 s).
- Workloads inside each region continue to resolve the **private** FQDN via their region-local Private DNS zone and continue using their local PE. If a region itself is lost, the workloads in that region are gone too — the surviving region's workloads keep using their local DR HSM with no change required.
- "Failover" therefore means swinging the **application + DB tier** from Primary to DR (Front Door / Traffic Manager priority swap, SQL MI auto-failover group). The HSM piece is automatic.

### 11.2 Quarterly drill (no production impact)
1. Pick a non-prod workload deployed in DR.
2. From DR Bastion → DR jump VM, run wrap/unwrap against `https://org-hsm-prod.managedhsm.azure.net/keys/cmk-app-prod`.
3. Confirm DNS resolves to **10.22.1.4** (DR PE, via the DR-region Private DNS zone) and call latency p50 < 30 ms.
4. Force-fail an app instance from Primary to DR (Front Door / Traffic Manager priority swap).
5. Verify SQL MI TDE unseal succeeds in DR.
6. **Simulate a Traffic Manager failover** for the global FQDN: disable the Primary endpoint in the TM profile, confirm `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` from a public/on-prem client now CNAMEs to the DR target within one TTL.
7. Re-enable the Primary TM endpoint; record metrics; restore.

### 11.3 Real failover runbook (region loss)
1. Declare incident; activate DR ICs.
2. **Verify Traffic Manager has already flipped** the global HSM FQDN to the DR endpoint (TM probes the regional `*.<region>.managedhsm.azure.net` health URL on a 30-s cadence). If TM has not flipped because the probe is ambiguous, manually set the Primary TM endpoint to `Disabled`.
3. Front Door / Traffic Manager priority for the application tier: DR = 1, Primary = 2 (drain).
4. Promote DR SQL MI (auto-failover group) — TDE protector already on the same HSM URI → no key re-config.
5. Re-point app config (if any region-specific overrides exist).
6. Validate end-to-end transaction. Expect to see DR workloads continue resolving the **private** HSM FQDN to `10.22.1.4` via the DR Private DNS zone — no change required.
7. Communicate.

> **The manual private-DNS swing in §11.5 is a last-resort fallback only** — invoke it only if both (a) Traffic Manager is not failing over for the global FQDN, *and* (b) some workload tier still relies on the private FQDN and is somehow misrouted. In the Active-Active design this should never happen, because each region's private zone already points at its own PE.

**HSM key-store RPO: ≈ 0 (replication is near real-time on the Azure backbone; eventually consistent within ~6 minutes per §6.4).**
**HSM data-plane failover: automatic via Traffic Manager, ~30 s detection + TTL.**
**Overall solution RTO: depends on application + DB failover. Target < 1 hour.**

### 11.4 Security Domain restore drill (annual)
Once a year, in a **test subscription**:
1. Deploy a throwaway Managed HSM.
2. Bring M custodians together.
3. Upload `org-hsm-prod-SD.json` to the test HSM (§6.2).
4. Confirm activation succeeds.
5. Delete the test HSM.
Document outcome with custodian signatures. Combine with a partial **custodian access check** every 6 months — each custodian briefly demonstrates they can still unlock their private key (no actual SD operation performed) so loss of access is detected long before it would matter in an incident.

### 11.5 **Last-resort** private-DNS fallback (do NOT use in normal failover)

The Active-Active design (§1.10, Phase 2, §7.3) makes this runbook unnecessary in steady-state failover — each region's Private DNS zone is independent and points at its own local PE, and the global FQDN is health-routed by Traffic Manager. Use the steps below **only** in the rare scenario where the DR-region Private DNS zone itself has been damaged or is mis-pointing, and you must force DR resolution to the DR PE IP manually.

Run from a hardened operator workstation that holds **Private DNS Zone Contributor** on the **DR-region** zone (`privatelink.managedhsm.azure.net` in `rg-org-hub-dr`).

```bash
# Variables
ZONE="privatelink.managedhsm.azure.net"
ZONE_RG="$RG_HUB_DR"        # DR-region zone
RECORD="org-hsm-prod"
PE_IP_DR="10.22.1.4"        # DR PE IP

# 1. Ensure only the DR PE IP is present in the DR-region zone
az network private-dns record-set a list -g "$ZONE_RG" -z "$ZONE" \
  --query "[?name=='$RECORD'].aRecords" -o json

az network private-dns record-set a add-record \
  -g "$ZONE_RG" -z "$ZONE" -n "$RECORD" --ipv4-address "$PE_IP_DR"

az network private-dns record-set a update \
  -g "$ZONE_RG" -z "$ZONE" -n "$RECORD" --set ttl=10

# 2. Verify from a DR jump VM
# Resolve-DnsName org-hsm-prod.privatelink.managedhsm.azure.net  -> expect 10.22.1.4
```

Fail-back is unnecessary — once the DR-region zone is healthy and the Primary region returns, each region's workloads resume using their own local PE without operator action. Pre-stage this script as an Azure Automation runbook gated on incident commander approval; **never trigger it automatically**, and never edit the *Primary*-region zone from a DR-side runbook (it cannot help DR workloads anyway, since DR VNets are not linked to that zone).

---

## Phase 12 — Go-Live Checklist & Hand-Over

### 12.1 Pre go-live
- [ ] All Phase 1–10 steps complete in **both** regions.
- [ ] Public access **Disabled** on both HSMs (`az keyvault show-hsm ... --query properties.publicNetworkAccess` returns `Disabled`).
- [ ] Soft-delete **90 days**, Purge protection **ON** — verified on both.
- [ ] **Active-Active DNS confirmed:** Primary VNets resolve `org-hsm-prod.privatelink.managedhsm.azure.net` to `10.12.1.4`; DR VNets resolve the same name to `10.22.1.4`. No shared/single zone.
- [ ] **Traffic Manager profile** (`tm-org-hsm-prod`) shows both regional endpoints `Online` and Priority routing configured (§3.4).
- [ ] **Multi-region replication healthy** — `az keyvault region list --hsm-name $HSM_NAME` shows both `southindia` and `centralindia` with provisioning state `Succeeded`; canary test (§6.3) passes.
- [ ] **6-minute eventual-consistency window** documented in all key-rotation and role-assignment runbooks (§6.4).
- [ ] Security Domain file SHA-256 recorded; 3 copies stored; custodian list signed.
- [ ] **Backup**: daily schedule running; last successful HSM backup blob URL + checksum on file; event-triggered backup configured on key rotation pipeline (§10.1).
- [ ] **Backup restore drill** executed at least once into a throwaway test HSM, completed under the 30-min limit (§10.1).
- [ ] DR drill (§11.2) executed and signed off, **including the Traffic Manager failover simulation in step 6**.
- [ ] All workload identities have **scoped** roles (no HSM-wide `Crypto User`).
- [ ] Break-glass user tested (login + role usage) and credentials sealed.
- [ ] Log Analytics receiving `AuditEvent` from both HSMs.
- [ ] Alerts (§10.3) firing on test triggers, including `HsmReplicationLatencyMs`.
- [ ] On-prem DNS forwarders are region-aware per §3.3 (Primary-homed sites → Primary Resolver first; DR-homed sites → DR Resolver first).

### 12.2 Documentation hand-over to operations
Deliverables:
1. This `steps.md` plus diagrams.
2. RBAC matrix (Appendix A).
3. Security Domain custody record.
4. Backup schedule (daily + event-triggered) and storage account location.
5. Runbooks: failover (§11.3), Traffic Manager endpoint enable/disable, key rotation (with 6-min quiet period), SD restore, **last-resort private DNS fallback (§11.5)**.
6. Contact list: HSM admins, Security on-call, Azure TAM.

---

## Appendix A — RBAC Role Reference

| Role | Scope | Typical assignee |
|---|---|---|
| `Managed HSM Administrator` | `/` | Quorum officers (Phase 5 admins), `grp-org-hsm-admins` |
| `Managed HSM Crypto Officer` | `/keys` | Key lifecycle team (create/rotate/delete) |
| `Managed HSM Crypto User` | `/keys/{name}` | Application MIs that need wrap/unwrap/sign/verify |
| `Managed HSM Crypto Service Encryption User` | `/keys/{name}` | Azure platform services: DES, SQL MI TDE, Storage |
| `Managed HSM Crypto Auditor` | `/` | `grp-org-hsm-auditors` (read-only metadata) |
| `Managed HSM Backup` | `/` | Backup automation principal (full HSM backup only) |
| `Managed HSM Policy Administrator` | `/` | Compliance team (manages role assignments) |

**Rule of thumb:** workload identities are **never** at scope `/` and **never** Administrator. Scope each MI to the specific key it needs.

---

## Appendix B — Security Domain Ceremony (Detailed)

### B.1 Roles
- **Ceremony Master** — drives the script, no key material.
- **N Custodians** — each holds 1 private key; M=3 must cooperate to recover.
- **Witness** — independent (Audit/Compliance) — signs the log.
- **Scribe** — records timestamps, fingerprints, commands executed.

### B.2 Offline key generation (each custodian, before the ceremony)
```bash
# On a hardened, network-isolated workstation:
openssl genrsa -out officerX.key 3072
openssl req -new -x509 -key officerX.key -out officerX.cer -days 3650 \
  -subj "/CN=HSM Officer X"
openssl x509 -in officerX.cer -noout -fingerprint -sha256
# Custodian records and prints the fingerprint.
```
- Private key → custodian's hardware token / encrypted USB.
- Public certificate → handed to Ceremony Master on a clean USB.

### B.3 Ceremony day script
1. Roll call; identity verification.
2. Each custodian's public cert collected and fingerprint **read aloud and compared** to the printed record.
3. Ceremony Master runs `az keyvault security-domain download` (Phase 5.2).
4. SD file SHA-256 computed and read aloud; all custodians sign.
5. SD file copied to its 3 storage locations (witnessed).
6. Custodians take their private keys to their separate custody locations.
7. Ceremony log signed by all parties; lodged with the Compliance team.

### B.4 Recovery prerequisites (for §6.2 / §11.4)
- SD file (`org-hsm-prod-SD.json`).
- At least **M** custodians physically together (or M private keys consolidated under four-eyes control on a clean workstation).
- Azure CLI access to the target HSM in pending-SD state.

---

## Appendix C — Troubleshooting

| Symptom | Likely cause | Resolution |
|---|---|---|
| `az keyvault key list` from VM fails with `Forbidden` | Workload MI not assigned a role on the HSM | Assign `Managed HSM Crypto User` (or service-specific role) on `/keys/<name>` |
| `Could not resolve org-hsm-prod...` from on-prem | On-prem DNS not forwarding to DNS Private Resolver | Add conditional forwarder for `privatelink.managedhsm.azure.net` to Resolver inbound IPs |
| DNS resolves to public IP (e.g. `40.x.x.x`) | Private DNS zone not linked to the VNet, or PE DNS zone group missing | Re-run §7.2 / §1.10 link create |
| DNS returns wrong region PE (e.g. Primary workload getting DR IP) | Primary VNet was accidentally linked to the DR-region zone, or vice versa | Per the Active-Active design (§1.10, Phase 2), each regional zone must be linked **only** to its own region's VNets. Audit `az network private-dns link vnet list` on both zones and remove cross-region links. |
| HSM create fails on quota | Region quota = 0 | Open support ticket: *Service & subscription limits → Managed HSM* |
| SD upload to DR fails — `quorum not met` | Fewer than M private keys provided | Bring more custodians; verify keys correspond to certs used in download |
| SQL MI TDE rotation hangs | SQL MI MI doesn't have role on new key version | Assign `Managed HSM Crypto Service Encryption User` on `/keys/cmk-sqlmi-prod` before rotation |
| Replication lag alert | Cross-region link degraded | Check Azure status; failover read traffic to healthy region |
| Locked out (all officers lost keys) | Catastrophic — only break-glass user remains | Use break-glass to assign Administrator to new officers; re-run §5.2 to rotate SD with new key set (advanced; engage Microsoft) |

---

**End of guide.** Treat this document as a living artefact — update it with every change in IPs, quorum, custodians, or workload onboarding.
