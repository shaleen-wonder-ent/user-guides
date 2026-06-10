# Customer-Managed Keys on Azure Managed HSM — **Public-Network Edition**
## Dual-Region Deployment Guide (no VNets, no Private Endpoints — Azure portal)

> **What this document is.** A simplified, **public-network** variant of [`HSM_Implementation_Azure_Steps.md`](./HSM_Implementation_Azure_Steps.md). Everything related to **hub-and-spoke networking, Private Endpoints, Private DNS zones, on-prem DNS forwarders, ExpressRoute, VNet peerings, NSGs, and Bastion is removed.** All clients reach the HSM over the **public internet** via its canonical FQDN `https://<hsm>.managedhsm.azure.net/...`, gated by an **IP allow-list** on the HSM's built-in firewall.
>
> Everything **non-networking** is identical to the private-network edition: same Security Domain ceremony, same multi-region replication, same local RBAC, same key creation, same workload integration on the data-plane, same backup, monitoring, alerts, and DR drill. Phase numbering is preserved (with Phases 1, 2, 3, 7 marked **N/A — public network**) so the two documents can be cross-referenced section-by-section.
>
> **When to use this edition.**
> - You don't yet have a hub-and-spoke topology, ExpressRoute, or Private DNS infrastructure.
> - Workloads outside Azure (developer laptops, SaaS callers, on-prem batch jobs) need to reach the HSM and you don't want to terminate them onto a VNet.
> - You're prototyping / proof-of-concept and don't want to wait on the network team.
> - You explicitly accept that the HSM's data-plane endpoint is publicly resolvable (TLS-protected; AAD-authenticated; firewalled to specific IPs — but DNS-discoverable).
>
> **When NOT to use this edition.**
> - Regulatory requirement that the HSM data path must traverse a private network only (most banking / PCI / restricted-data regimes).
> - Internal corporate policy forbidding any public-network exposure for crown-jewel cryptographic services.
> - You need on-prem callers to reach the HSM **without** their egress IP being added to the allow-list (use the private-network edition + ExpressRoute instead).
>
> **Two unavoidable portal limitations to know up-front** (same as the private edition):
> 1. **The Security Domain ceremony in Phase 5** must be run from `az keyvault security-domain download` on an admin workstation — the portal has no UI for it.
> 2. **HSM full backups/restores in §10.1** are CLI / REST-only.
>
> Everything else — the HSM resource, **Multi-Region Replication** blade, **Networking → Firewall** IP allow-list, **Local RBAC**, **Keys**, **Disk Encryption Sets**, SQL MI **Transparent data encryption**, **Diagnostic settings**, **Alerts** — is **100 % portal**.
>
> **Example primary region:** South India · **Example DR region:** Central India *(swap for any two Azure regions where Managed HSM is available).*
> **HSM pool name (both regions — same name, one logical HSM):** `org-hsm-prod` *(replace `org` with your org code).*

---

## What this guide builds, in plain English

Same end state as the private edition for the **cryptographic** layer:

- **One logical Managed HSM** named `org-hsm-prod`, with **two physically separate clusters** (Primary + DR) under the same canonical FQDN.
- Microsoft's internal, platform-managed routing layer under `<hsm>.managedhsm.azure.net` (TTL 5 s) automatically swings between the two regional pools.
- Same keys in both regions (replication ≤ ~6 min).
- M-of-N (e.g. 3-of-5) quorum protects the Security Domain.

**Different** from the private edition:

- **No hub VNet, no spoke VNet, no Private Endpoint, no Private DNS zone, no DNS forwarder, no Bastion** — clients call `https://org-hsm-prod.managedhsm.azure.net/...` directly over the public internet.
- **No split-horizon DNS** — public DNS for `<hsm>.managedhsm.azure.net` resolves to whichever regional pool the platform router currently selects. There is no per-region resolution.
- **Access control = TLS + AAD + IP allow-list + HSM-local RBAC.** Public network access stays **Enabled**, but the **firewall is set to Deny-by-default** and only your approved egress IPs / IP ranges are allowed in.

---

## The journey, phase by phase

| Phase | What you do (portal) | Why it matters |
|---|---|---|
| **0. Subscription & identity prep** | Entra ID → Groups; Subscription → Resource providers. | Permissions baseline before any HSM resource exists. |
| **1. Network foundation (Primary)** | **N/A — public network edition.** | — |
| **2. Network foundation (DR)** | **N/A — public network edition.** | — |
| **3. Global wiring (DNS forwarding)** | **N/A — public network edition.** Clients use public DNS for the canonical FQDN. | — |
| **4. Provision Primary HSM** | Key Vaults blade → **+ Create** → **Managed HSM**. ~20–30 min. Public access **Enabled**; firewall **Deny by default**; allow-list a small initial admin IP. | The HSM itself. |
| **5. Security Domain ceremony** | **CLI-only step.** | Most security-critical step. Lose SD + M keys = permanent key loss. |
| **6. DR HSM & replication** | HSM → **Multi-Region Replication** → **+ Add Region**. ~30 min. | One logical HSM, two regions, RPO ≈ 0. |
| **7. Lock-down network** | **N/A — public network edition.** Replaced by [§7-public](#phase-7--public-firewall-allow-list-tightening) — tighten the HSM firewall allow-list. | Public access stays on, but the allow-list is the gate. |
| **8. Data-plane RBAC and keys** | HSM → **Local RBAC**; **Keys** → **+ Generate/Import**. | HSM-local RBAC, separate from Azure RBAC. |
| **9. Workload integration** | Disk Encryption Sets; SQL MI TDE; App with managed identity + SDK against the canonical public FQDN. | This is where encryption actually starts using your HSM keys. |
| **10. Backup, logging, alerts** | Diagnostic settings; Alerts (portal). Backup itself runs via CLI/REST. | Backups protect against accidental deletion; logs are mandatory for audit. |
| **11. DR drill & runbook** | Quarterly portal-driven validation; annual SD restore drill (CLI). | Rehearse the failover. |
| **12. Go-live checklist** | Final gates: firewall allow-list reviewed, soft-delete + purge protection on, backups verified, drills signed off. | Final gate before production traffic. |

---

## Concept primer (delta vs. the private edition)

| Term | Plain-English meaning (public-network context) |
|---|---|
| **Managed HSM firewall** | A built-in IP allow-list on the HSM resource (Networking → Firewall blade). With **public access Enabled** + **Default action: Deny**, only listed IPs / CIDRs can reach the data plane. AAD auth is still mandatory on top. |
| **Canonical FQDN** | `https://<hsm>.managedhsm.azure.net` — the only hostname any client should call. Microsoft's platform router resolves it to a healthy regional pool. |
| **No split-horizon DNS** | Without Private DNS, there's only one public answer per moment. The router (TTL 5 s) decides which regional pool answers. |
| **Trusted services bypass** | A toggle on Networking → Firewall that lets **Azure Disk Encryption / SQL MI TDE / Storage CMK** call the HSM even when their egress IP isn't on the allow-list, provided the requesting Azure service identity is granted HSM-local RBAC. **Recommended ON** so you don't have to chase per-region Azure service IPs. |

Everything else (Security Domain, M-of-N, multi-region replication, envelope encryption, DES, TDE Protector, eventual consistency) is identical to the private edition.

---

## Table of Contents

1. [Overview & Design Decisions](#1-overview--design-decisions)
2. [Prerequisites](#2-prerequisites)
3. [Naming and Variables](#3-naming-and-variables)
4. [Phase 0 — Subscription, Identity, and RBAC Preparation](#phase-0--subscription-identity-and-rbac-preparation)
5. [Phase 1 — Network Foundation (Primary)](#phase-1--network-foundation-primary)
6. [Phase 2 — Network Foundation (DR)](#phase-2--network-foundation-dr)
7. [Phase 3 — Global wiring (DNS forwarding)](#phase-3--global-wiring-dns-forwarding)
8. [Phase 4 — Provision Managed HSM (Primary)](#phase-4--provision-managed-hsm-primary)
9. [Phase 5 — Activate HSM & Generate Security Domain (M-of-N Ceremony)](#phase-5--activate-hsm--generate-security-domain-m-of-n-ceremony)
10. [Phase 6 — Provision DR HSM & Enable Multi-Region Replication](#phase-6--provision-dr-hsm--enable-multi-region-replication)
11. [Phase 7 — Public Firewall Allow-List Tightening](#phase-7--public-firewall-allow-list-tightening)
12. [Phase 8 — HSM Data-Plane RBAC and Key Creation](#phase-8--hsm-data-plane-rbac-and-key-creation)
13. [Phase 9 — Workload Integration (VM Disks, SQL MI TDE, App Tier)](#phase-9--workload-integration-vm-disks-sql-mi-tde-app-tier)
14. [Phase 10 — Backup, Monitoring & Logging](#phase-10--backup-monitoring--logging)
15. [Phase 11 — DR Failover Drill and Runbook (Active-Active)](#phase-11--dr-failover-drill--runbook)
16. [Phase 12 — Go-Live Checklist & Hand-Over](#phase-12--go-live-checklist--hand-over)
17. [Appendix A — RBAC Role Reference](#appendix-a--rbac-role-reference)
18. [Appendix B — Security Domain Ceremony (Portal-Aware Detailed)](#appendix-b--security-domain-ceremony-portal-aware-detailed)
19. [Appendix C — Troubleshooting (Public-Network Specific)](#appendix-c--troubleshooting-publicnetwork-specific)

---

## 1. Overview & Design Decisions

| Area | Decision | Rationale |
|---|---|---|
| Key store | **Azure Managed HSM** (single-tenant, FIPS 140-3 Level 3) | Regulatory/audit requirement; tenant-isolated, not shared |
| Topology | **No VNets / no Private Endpoints — public-network access** | Simplifies onboarding; suits non-VNet clients (laptops, SaaS, on-prem without ExR) |
| Regions | **Primary + DR** Azure region (example: South India + Central India) | Data residency; HA via multi-region replication |
| Network | **Public access ENABLED**, **Firewall: Deny by default**, **IP allow-list** + **Trusted Azure services bypass** | Only approved egress IPs and approved Azure services can reach the HSM data plane |
| Multi-region mode | **Active-Active** via Managed HSM multi-region replication | Both regional pools serve traffic; platform router picks a healthy one per request |
| Cross-region | **HSM multi-region replication** (Microsoft-managed backbone — no customer peering required) | Same key URI in both regions; replication eventually consistent within ~6 min |
| DNS | **Public DNS only**, single canonical FQDN `org-hsm-prod.managedhsm.azure.net`. No customer Private DNS zone. No customer Traffic Manager profile. | Platform router fronts the FQDN automatically; TTL 5 s. |
| RTO/RPO | Key store RPO ≈ 0 (replication eventually consistent ≤ ~6 min). HSM failover automatic at the platform router (TTL 5 s). Overall RTO depends on app + DB failover (target < 1 hr). | No DNS edits at the HSM layer during failover. |
| Identity | **Microsoft Entra ID** + **Managed Identities / users + HSM Local RBAC** | No secrets in code; per-key least privilege |
| Admin access | **Bastion-less** — admins call the HSM directly over public FQDN from their workstations. Workstation egress IPs allow-listed. | No need for jump VMs or Bastion. |
| Security Domain | **Offline-generated, M-of-N (e.g. 3-of-5)** | Same as private edition. |
| Replication setup | **Multi-Region Replication blade → + Add Region** | Same as private edition. |
| Workloads in scope | VM OS/Data Disks (via DES), SQL MI (TDE with CMK), application-tier wrap/unwrap | Same as private edition; clients hit the public FQDN. |
| **Cost** | **HSM cluster cost × 2** (Primary + DR replica). **No Private Endpoint / Private DNS / Bastion costs.** | Significantly cheaper than the private edition once networking is excluded. |

### 1.6 Risk register & operational risks (public-network specific)

| # | Risk | What goes wrong | Mitigation |
|---|---|---|---|
| **R1** | **App code/config hard-codes a non-canonical HSM hostname** (regional sub-FQDN) | Bypasses the platform router → won't fail over. | [§9.3](#93-application-tier-envelope-encryption) mandates the canonical FQDN. [§10.5](#105-periodic-configuration-audits-quarterly) audits for drift. |
| **R3** | **HSM firewall allow-list drift** (someone adds a wide CIDR like `0.0.0.0/0` "temporarily" and forgets) | Public exposure beyond intended audience. AAD + RBAC still protect data, but it widens the attack surface. | Activity-log alert on `Microsoft.KeyVault/managedHSMs/write` ([§10.4](#104-activity-log-alerts-portal)) flags any firewall change. [§10.5](#105-periodic-configuration-audits-quarterly) check D quarterly-reviews the allow-list. |
| **R4** | **Compromised allow-listed IP** (e.g. a workstation belonging to an allow-listed analyst is owned) | Attacker can attempt HSM data-plane calls. AAD auth + Conditional Access + HSM-local RBAC still required. | Conditional Access on all admin accounts (MFA + compliant device). HSM-local RBAC scoped to specific keys. Alert on unusual `Microsoft.KeyVault` data-plane patterns. |
| **R5** | **Trusted Azure services bypass abused** | If you enable bypass and grant HSM-local RBAC too broadly (e.g. `Crypto User` to a wide service principal), Azure services anywhere could call the HSM. | Always scope HSM-local roles to `/keys/<name>`, never `/keys` or `/`. Audit role assignments quarterly. |

The **R2 (Private DNS misconfiguration)** risk from the private edition does not apply here — there is no private DNS.

---

## 2. Prerequisites

### 2.1 Azure tenant & subscription
- [ ] Microsoft Entra tenant identified.
- [ ] One Azure subscription per environment.
- [ ] Subscription registered for `Microsoft.KeyVault`, `Microsoft.Compute`, `Microsoft.Sql`, `Microsoft.Storage`, `Microsoft.Insights`, `Microsoft.OperationalInsights` *(no `Microsoft.Network` required for this edition unless workloads already use VNets for their own reasons).*
- [ ] **Quotas approved** in both regions for Managed HSM.

### 2.2 Roles required to perform the deployment
| Activity | Role required |
|---|---|
| Create the HSM resource | **Contributor** on subscription (or **Owner** for role assignments) |
| Assign Azure RBAC | **User Access Administrator** or **Owner** |
| Assign HSM data-plane local RBAC | **Managed HSM Administrator** (post-activation) |
| Security Domain ceremony | At least **N quorum officers** |

### 2.3 People & artefacts
- [ ] **N Security Domain custodians** identified (recommended N=5).
- [ ] N × **RSA 3072/4096 key pairs** generated **offline**. Public keys as `.cer`. Private keys held by each custodian.
- [ ] Quorum **M** decided (recommended M=3).
- [ ] **List of allow-listed egress IPs / CIDRs** agreed and signed off:
  - Admin workstations / VPN egress NAT IPs.
  - CI/CD runners that will deploy DES / SQL TDE config.
  - On-prem batch egress (if any callers run on-prem).
  - **Do NOT include `0.0.0.0/0`.**
- [ ] Change ticket approved.

### 2.4 Tooling
- [ ] Modern browser with access to <https://portal.azure.com>.
- [ ] **For Phase 5 only**, an admin workstation with Azure CLI ≥ `2.61` and `az extension add --name keyvault`. The workstation's egress IP must be on the allow-list ([§4.2](#42-create-the-hsm-portal)).
- [ ] OpenSSL on the same workstation, for verifying officer public-key material.

---

## 3. Naming and Variables

### 3.1 Resource naming

| Resource | Status | Primary (example: South India) | DR (example: Central India) |
|---|---|---|---|
| Resource group — HSM | NEW | `rg-org-hsm-pri` | `rg-org-hsm-dr` |
| Managed HSM | NEW | `org-hsm-prod` | **same** `org-hsm-prod` (replica via Multi-Region Replication blade) |
| Global FQDN routing | PLATFORM-MANAGED | Provisioned automatically by Multi-Region Replication; Microsoft-internal, DNS TTL 5 s | Same — no customer-managed component |

> **First-time readers:** the Managed HSM pool name is identical in both regions because the DR HSM is a *replica*, not a separate vault. Apps use a single key URI: `https://org-hsm-prod.managedhsm.azure.net/keys/<key-name>`.

### 3.2 Values to capture before clicking anything

| Variable | Where to find it | Your value |
|---|---|---|
| `TENANT_ID` | Entra ID → Overview → Tenant ID | `<your-entra-tenant-id>` |
| `SUB_ID` | Subscriptions blade | `<your-subscription-id>` |
| `LOC_PRI` / `LOC_DR` | Azure regions | `southindia` / `centralindia` |
| `RG_HSM_PRI` / `RG_HSM_DR` | **NEW** — you'll create in 4.1 / 6.2 | `rg-org-hsm-pri` / `rg-org-hsm-dr` |
| `HSM_NAME` | **NEW** — you choose | `org-hsm-prod` |
| `ADMIN_OID_1..3` | Entra ID → Users → \<officer\> → Object ID | `<entra-objectid-officer-N>` |
| `ALLOWED_IPS` | From Security / Network team | e.g. `203.0.113.10/32, 198.51.100.0/24` |

---

## Phase 0 — Subscription, Identity, and RBAC Preparation

Identical to the private edition. Reproduced here for self-containment.

### 0.1 Register resource providers (portal)
1. Sign in to <https://portal.azure.com>.
2. Search bar → **Subscriptions** → click your subscription.
3. Left menu → **Settings → Resource providers**.
4. Filter for each of the following and click **Register** if *NotRegistered*:
   - `Microsoft.KeyVault`
   - `Microsoft.Compute`
   - `Microsoft.Sql`
   - `Microsoft.Storage`
   - `Microsoft.Insights`
   - `Microsoft.OperationalInsights`

### 0.2 Create Entra groups
1. Search bar → **Microsoft Entra ID** → **Groups** → **+ New group**.

| Group name | Group type | Membership type | Purpose |
|---|---|---|---|
| `grp-org-hsm-admins` | Security | Assigned | Full HSM admin (quorum officers) |
| `grp-org-hsm-crypto-officers` | Security | Assigned | Create/rotate/delete keys |
| `grp-org-hsm-crypto-users` | Security | Assigned | Wrap/unwrap/sign/verify only (workload identities) |
| `grp-org-hsm-auditors` | Security | Assigned | Read-only + logs |

Copy each group's **Object ID** from Overview.

### 0.3 Break-glass identity
1. Entra ID → **Users → + New user → Create new user**.
2. Username: `bg-hsm-admin@<your-tenant>.onmicrosoft.com`. Strong random password.
3. **Authentication methods → + Add → FIDO2 security key**.
4. **Conditional Access** → exclude from policies that could lock it out.
5. Add to `grp-org-hsm-admins`.
6. Seal credentials in a physical safe; test login once a year.

### 0.4 Tagging baseline
For every new resource: `env=prod`, `owner=<team>`, `costcenter=<cc>`, `dataclass=restricted`, `dr=active-active`, **`network=public`** (the last one helps audit tools distinguish this edition from the private edition).

### 0.5 Conditional Access (strongly recommended in this edition)
Because the HSM is reachable from the public internet, layered identity controls become more important:

1. Entra ID → **Protection → Conditional Access → Policies → + New policy**.
2. Name: `cap-hsm-admins-mfa-compliant`.
3. **Assignments → Users**: `grp-org-hsm-admins` + `grp-org-hsm-crypto-officers`.
4. **Assignments → Target resources → Cloud apps**: select **Azure Key Vault** (Managed HSM data plane uses the same AAD app id).
5. **Conditions → Locations**: optionally restrict to your corporate egress IP ranges (defence in depth on top of HSM firewall).
6. **Access controls → Grant**: require **Multi-factor authentication** AND **Require device to be marked as compliant** (Intune-managed).
7. **Enable policy: On**.

---

## Phase 1 — Network Foundation (Primary)

**N/A — public network edition.** No VNets, no spokes, no NSGs created.

## Phase 2 — Network Foundation (DR)

**N/A — public network edition.**

## Phase 3 — Global wiring (DNS forwarding)

**N/A — public network edition.** All clients use **public DNS** to resolve `org-hsm-prod.managedhsm.azure.net`. Microsoft's platform-managed router fronts this FQDN automatically once the second region is added in [Phase 6](#phase-6--provision-dr-hsm--enable-multi-region-replication) — TTL 5 s. No customer-deployed Traffic Manager profile.

---

## Phase 4 — Provision Managed HSM (Primary)

### 4.1 Resource group
1. Search bar → **Resource groups → + Create**.
2. Subscription: workload sub. Resource group: **`rg-org-hsm-pri`**. Region: **South India**.
3. **Tags** tab → baseline from [§0.4](#04-tagging-baseline).
4. Review + create → Create.

### 4.2 Create the HSM (portal)
1. Search bar → **Key Vaults → + Create → Managed HSM** *(use the dropdown on the + Create button).*
2. **Basics** tab:
   - Subscription: workload sub.
   - Resource group: **`rg-org-hsm-pri`**.
   - HSM name: **`org-hsm-prod`** *(globally unique in `managedhsm.azure.net`).*
   - Region: **South India**.
   - Pricing tier: **Standard B1**.
   - HSM administrator: **Add HSM administrators** → pick the N officers (or `grp-org-hsm-admins`).
3. **Networking** tab:
   - Connectivity method: **Enable public access**.
   - **Default action**: **Deny**.
   - **Firewall → Allow Azure services on the trusted services list to bypass this firewall**: **Yes** *(recommended — lets DES, SQL MI TDE, Storage CMK call the HSM without per-region IP wrangling; still requires HSM-local RBAC on the calling identity).*
   - **Firewall → IP address or CIDR**: paste each entry from `ALLOWED_IPS` (initial admin workstation IP at minimum). Click **+ Add** between entries.
4. **Tags** tab → baseline.
5. **Review + create** → wait for validation → **Create**. ~20–30 min. State: **Provisioning → Provisioned (pending security domain)**.
6. After creation: open the HSM → **Overview** → confirm **Status: Provisioned**, **Activated: No**. **Properties** → confirm **Soft delete retention: 90 days**, **Purge protection: Enabled**.

> **Day-0 allow-list is intentionally small.** Add only the IPs needed for the SD ceremony admin workstation and the initial admin. Widen in [Phase 7](#phase-7--public-firewall-allow-list-tightening) after activation, then prune again in [§10.5](#105-periodic-configuration-audits-quarterly) reviews.

---

## Phase 5 — Activate HSM & Generate Security Domain (M-of-N Ceremony)

> **Identical to the private edition.** Reproduced verbatim because the SD ceremony does not depend on networking choices.

### 5.1 Pre-ceremony
- [ ] Conference room booked. Video/audio recording per policy.
- [ ] N custodians present, each with their public-key file (`.cer`/PEM).
- [ ] Verify each `.cer` belongs to its custodian (fingerprint read aloud).
- [ ] Admin workstation has Azure CLI ≥ 2.61 + `az extension add --name keyvault` installed.
- [ ] **Workstation's egress public IP is on the HSM firewall allow-list** ([§4.2](#42-create-the-hsm-portal)). Verify with `Test-NetConnection org-hsm-prod.managedhsm.azure.net -Port 443` → success.
- [ ] All N `.cer` files copied to `./sd-certs/` on the workstation.

### 5.2 Generate, encrypt, and activate the HSM via CLI — **only CLI-required step**

```bash
az login   # sign in as a quorum officer

az keyvault security-domain download \
  --hsm-name "org-hsm-prod" \
  --sd-wrapping-keys ./sd-certs/officer1.cer ./sd-certs/officer2.cer \
                     ./sd-certs/officer3.cer ./sd-certs/officer4.cer \
                     ./sd-certs/officer5.cer \
  --sd-quorum 3 \
  --security-domain-file org-hsm-prod-SD.json
```

This single command generates the SD inside the HSM hardware, encrypts it to the N public keys with M-of-N quorum, and transitions the HSM from `Pending` → `Activated`.

### 5.3 Verify activation (portal)
1. Portal → **Key Vaults** filter set to **Managed HSMs** → open **`org-hsm-prod`**.
2. **Overview** → **Activated: Yes**, **Status: Activated**.
3. **Properties → Status: The Managed HSM is operational**.

### 5.4 Custody and storage of the Security Domain file
- Store `org-hsm-prod-SD.json` in **three** geographically separated, access-controlled locations (physical safe + immutable blob in different sub + offline cold-storage drive).
- Officer private keys: each custodian holds their own, spread across ≥ 3 sites.
- Never co-locate M or more private keys.

### 5.5 Post-ceremony record
Sign and file:
- Custodian names + public-key fingerprints.
- Quorum M value.
- SHA-256 of `org-hsm-prod-SD.json`.
- Storage locations.
- Date, time, witness signatures.

---

## Phase 6 — Provision DR HSM & Enable Multi-Region Replication

> The DR HSM is not a separate vault. It is a replica of Primary — same SD, same keys, same key URIs.

### 6.1 **Preferred** — Native multi-region replication (Multi-Region Replication blade)

1. Portal → open Primary HSM `org-hsm-prod`.
2. Left menu → **Settings → Multi-Region Replication**.
3. Click **+ Add Region** at the top.
4. **Add Region** pane → Region: **Central India** → Confirm.
5. Click **Add / OK**. Provisioning State = **Creating**.
6. **Wait until Provisioning State = Succeeded** for the DR row (~30 min). Don't perform Primary writes meanwhile.

This single action:
- Provisions the DR HSM cluster in Central India.
- Imports the SD from Primary automatically (**no second M-of-N ceremony**).
- **Replicates both keys and HSM-local role assignments** to the DR pool (~6-min eventual consistency).
- Wires the DR pool into the internal Microsoft-managed routing layer under `<hsm>.managedhsm.azure.net` (TTL 5 s).
- The DR pool inherits the **same firewall + RBAC** configuration as Primary. There is no separate firewall to configure in DR — the `org-hsm-prod` resource governs both pools.

### 6.2 Verify multi-region replication (functional test)

1. Portal → `org-hsm-prod` (any region) → **Multi-Region Replication** → both rows show **Succeeded**.
2. **Settings → Keys → + Generate/Import**:
   - Name: `canary-replication-test`.
   - Key type: **RSA-HSM**. Size: **3072**.
   - Permitted operations: **Sign**, **Verify**.
   - Create.
3. From the admin workstation, capture the **Key Identifier** (versioned `kid`) from the key's Overview blade.
4. After ~6 min, reload the same Managed HSM page → the key is present (it is the same logical key in both pools).
5. Optional cryptographic test (CLI):
   ```bash
   echo -n "canary-test" | az keyvault key sign --id "<kid>" --algorithm RS256 --value @- > sig.b64
   az keyvault key verify --id "<kid>" --algorithm RS256 --digest <sha256-of-canary-test> --signature @sig.b64
   ```
6. Clean up: **Keys → `canary-replication-test` → Delete**.

### 6.3 Replication lag — eventual consistency window

Same rules as the private edition:
- Up to **~6 minutes** for key creates/updates/deletes and role-assignment changes to propagate.
- Allow a 6-minute quiet period before any planned regional cutover.
- Application code must retry on transient `404 KeyNotFound` / `403 Forbidden` (see [§9.5](#95-application-side-resilience-mandatory-for-production-workloads)).
- Alert on `HsmReplicationLatencyMs > 1000 ms` sustained 5 min ([§10.3](#103-alerts-recommended-baseline-portal)).

---

## Phase 7 — Public Firewall Allow-List Tightening

> Replaces the private edition's "lock down network" phase. Public access stays **Enabled** — the firewall allow-list is the gate.

### 7.1 Review and finalise the production allow-list
1. Portal → `org-hsm-prod` → **Networking → Firewalls and virtual networks** tab.
2. Confirm **Public access: Enabled** and **Default action: Deny**.
3. **Allowed IPs and CIDRs** — review the existing list. For production, the entries should be:
   - **Admin / quorum officer workstation egress IPs** (corporate NAT or VPN gateways, /32 where possible).
   - **CI/CD runner egress IPs** that will deploy DES / SQL TDE configuration (note: most workload integration uses Azure-internal trusted services — see 7.2 — so CI/CD usually only needs Azure control-plane access via Azure Resource Manager, not direct HSM data-plane access).
   - **On-prem application egress NAT IPs**, if any on-prem app calls the HSM directly.
   - **Monitoring / backup tooling egress IPs**, if backup is driven from a non-Azure host.
4. **Remove** any temporary or wildcard entries from Phase 4.
5. **Trusted Azure services bypass**: confirm **Yes** (so DES / SQL MI TDE / Storage CMK can reach the HSM without per-Azure-region IP allow-listing).
6. **Save**.

> **Do NOT add `0.0.0.0/0`.** If you genuinely need a globally-reachable callable HSM, that's not this edition — re-architect around Azure Front Door + Web Application Firewall + per-caller AAD app-only auth, then revisit.

### 7.2 Trusted Azure services bypass — what it does and doesn't do
- **Does:** allows traffic from Microsoft-managed services that hold a system-assigned managed identity with HSM-local RBAC — including Azure Disk Encryption (DES), SQL MI TDE, Storage CMK, App Service / Function App CMK — to reach the HSM data plane regardless of source IP. **Authentication and HSM-local RBAC are still enforced.**
- **Does NOT:** allow arbitrary Azure resources or arbitrary tenants. Each calling identity must still be granted an HSM-local role at `/keys/<name>` (Phase 8).
- **Audit:** review HSM Local RBAC assignments quarterly ([§10.5](#105-periodic-configuration-audits-quarterly) check E) — every grant to a managed identity should map to a documented workload.

### 7.3 Smoke test from outside the allow-list
From any IP not on the allow-list (a phone hotspot, a personal cloud VM):
- `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` → returns a public IP (this is expected — DNS resolution is not firewalled).
- `Test-NetConnection org-hsm-prod.managedhsm.azure.net -Port 443` → **TCP connect succeeds** (Microsoft front-end is reachable) **but** any HTTP call should be rejected with HTTP 403 / connection-reset by the HSM firewall before AAD even sees it.
- Confirm the rejection appears in the HSM diagnostic logs as a denied request ([§10.2](#102-diagnostic-settings--log-analytics--storage-portal)).

---

## Phase 8 — HSM Data-Plane RBAC and Key Creation

Identical to the private edition. The HSM-local RBAC model is independent of networking.

### 8.1 Assign administrator role (portal)
1. Portal → `org-hsm-prod` → **Local RBAC → + Add → Add role assignment**.
2. **Role**: **Managed HSM Administrator**. **Scope**: `/`. **Members**: `grp-org-hsm-admins`.
3. Review + assign.

### 8.2 Assign crypto officer role
1. **Local RBAC → + Add → Add role assignment**.
2. **Role**: **Managed HSM Crypto Officer**. **Scope**: `/keys`. **Members**: `grp-org-hsm-crypto-officers`.
3. Review + assign.

### 8.3 Create CMK keys per workload (portal)
1. **Keys → + Generate/Import** for each:

| Key name | Key type | Size | Permitted operations |
|---|---|---|---|
| `cmk-vmdisk-prod` | **RSA-HSM** | **3072** | Wrap Key, Unwrap Key |
| `cmk-sqlmi-prod` | **RSA-HSM** | **3072** | Wrap Key, Unwrap Key, Encrypt, Decrypt |
| `cmk-app-prod` | **RSA-HSM** | **3072** | Wrap Key, Unwrap Key, Encrypt, Decrypt |

Each key replicates to the DR pool within ~6 min ([§6.3](#63-replication-lag--eventual-consistency-window)).

Capture each key's versioned **Key Identifier** URI from the Overview blade: `https://org-hsm-prod.managedhsm.azure.net/keys/<name>/<version>`.

### 8.4 Assign least-privilege roles to workload identities (portal)

For each workload identity (DES MI, SQL MI MI, App MI):

1. Portal → `org-hsm-prod` → **Local RBAC → + Add → Add role assignment**.
2. **Role**: **Managed HSM Crypto Service Encryption User** (for DES / SQL MI / Storage) or **Managed HSM Crypto User** (for app-tier envelope encryption — see [§9.3](#93-application-tier-envelope-encryption)).
3. **Scope**: `/keys/<specific-key>` *(never `/keys` or `/` for workload identities).*
4. **Members**: pick the workload's managed identity.
5. Review + assign.

---

## Phase 9 — Workload Integration (VM Disks, SQL MI TDE, App Tier)

Do **Primary region first**, validate end-to-end, then mirror in DR.

> **Networking note:** because DES / SQL MI TDE / Storage CMK rely on **trusted Azure services bypass** ([§7.2](#72-trusted-azure-services-bypass--what-it-does-and-doesnt-do)), there is no IP allow-listing to do per Azure service. Just assign HSM-local RBAC ([§8.4](#84-assign-leastprivilege-roles-to-workload-identities-portal)) and configure the workload to point at the canonical HSM FQDN.

### 9.1 Virtual Machines — CMK via Disk Encryption Set (portal)
1. Search bar → **Disk Encryption Sets → + Create**.
2. **Basics**:
   - Subscription: workload sub. Resource group: any prod RG (e.g. `rg-org-app-pri`).
   - Disk encryption set name: **`des-vmdisk-prod`**. Region: **South India**.
   - Encryption type: **Encryption at-rest with a customer-managed key**.
   - Key Vault and key: **Select a key vault and key** → switch picker to **Managed HSM** → pick HSM `org-hsm-prod` → Key `cmk-vmdisk-prod` → leave **Version** empty (gives auto-rotation).
   - Auto key rotation: **Enabled**.
3. **Identity**: System-assigned managed identity → **On**.
4. Review + create → Create.
5. Grant the DES MI access on the key — repeat [§8.4](#84-assign-leastprivilege-roles-to-workload-identities-portal) with role **Managed HSM Crypto Service Encryption User**, scope `/keys/cmk-vmdisk-prod`, member `des-vmdisk-prod`.
6. **Disks → + Create** → **Encryption** tab → **Customer-managed key** → DES `des-vmdisk-prod`. Attach to VMs.
7. Re-encrypt existing VM disks: **VM → Disks → \<disk\> → Encryption → Customer-managed key**.

### 9.2 SQL Managed Instance — TDE with CMK (portal)
1. **SQL MI → Settings → Identity** → confirm **System-assigned managed identity: On**. Copy **Object (principal) ID**.
2. Grant the SQL MI MI access on the key ([§8.4](#84-assign-leastprivilege-roles-to-workload-identities-portal)) — role **Managed HSM Crypto Service Encryption User**, scope `/keys/cmk-sqlmi-prod`.
3. **SQL MI → Security → Transparent data encryption**:
   - TDE: **Customer-managed key**.
   - Identity: **System-assigned**.
   - Key: **Enter a key identifier** → paste versioned URI `https://org-hsm-prod.managedhsm.azure.net/keys/cmk-sqlmi-prod/<version>`.
   - **Save**.
4. The blade refreshes with **TDE Protector: customer-managed key**.

### 9.3 Application tier (envelope encryption)
For any app that wraps its own DEKs:
1. Enable **system-assigned managed identity** on the VM / App Service / AKS workload: **\<resource\> → Identity → System assigned → On**.
2. Grant it **Managed HSM Crypto User** on `/keys/cmk-app-prod` ([§8.4](#84-assign-leastprivilege-roles-to-workload-identities-portal)).
3. App uses the Azure Key Vault SDK (`Azure.Security.KeyVault.Keys.Cryptography`) pointed at **`https://org-hsm-prod.managedhsm.azure.net/keys/cmk-app-prod`** for `WrapKey`/`UnwrapKey`. Always the canonical FQDN — never a regional sub-FQDN.
4. **Egress consideration:** if the app runs in Azure with trusted services bypass enabled, no IP allow-listing is needed. If it runs outside Azure, its egress IP must be on the HSM firewall allow-list.
5. Cache the **wrapped DEK** alongside ciphertext; only the wrapped DEK leaves the HSM.

### 9.4 Repeat in DR
Recreate **DES (DR)** in a DR resource group, **SQL MI (DR)** TDE config, and DR app workloads pointing to the **same key URI** (`https://org-hsm-prod.managedhsm.azure.net/keys/<name>`). Each region's DES/SQL MI/app uses its own managed identity, but the same HSM-local role assignment (replicated automatically — give it ~6 min).

### 9.5 Application-side resilience (mandatory for production workloads)
Even with platform-managed global routing, apps must defend against the **eventual-consistency window** and transient blips:
1. **Retry with exponential back-off** on transient failures (HTTP 408, 429, 5xx, timeouts, transient 404/403 right after a write). Keep the SDK's built-in retry on; `MaxRetries = 5`, base delay 800 ms.
2. **Region-aware app deployment** — keep at least one app instance per Azure region; let your load balancer drive request distribution. Always use the canonical HSM FQDN.
3. **Short DNS TTL awareness** — public DNS TTL is 5 s for the platform router; don't cache resolved IPs beyond that.
4. **Wrapped-DEK caching** for the request/session lifetime.
5. **Circuit breaker + fail-fast** after N consecutive HSM failures within W seconds.
6. **Honour the 6-minute pre-failover quiet period** ([§6.3](#63-replication-lag--eventual-consistency-window)) in any planned cutover.

---

## Phase 10 — Backup, Monitoring & Logging

Same as the private edition (networking-agnostic).

### 10.1 HSM full backup (control-plane, periodic + event-triggered) — **CLI/REST only**

```bash
STG="<backup-storage-account>"
CONTAINER="hsm-backups"
SAS="<container-sas-with-create-write>"

az keyvault backup start --hsm-name "org-hsm-prod" \
  --storage-account-name "$STG" \
  --blob-container-name "$CONTAINER" \
  --storage-container-SAS-token "$SAS"
```

**Cadence:**

| Trigger | Frequency / condition | Rationale |
|---|---|---|
| Scheduled full backup | **Daily** off-peak (Automation / Logic App) | Caps RPO at 24 h even if both regions lost |
| Event-triggered full backup | After bulk key create / rotation / role-assignment changes / before planned maintenance | Narrows the unprotected-write window |
| Retention | 12 months, **immutable** blob, in a **different region** | Ransomware + regional outage protection |
| Encryption of backup blob | CMK from a **different** HSM / Key Vault | Avoid circular dependency |

**Restore drill (annual):** restore the latest backup into a throwaway test HSM in a non-prod subscription. Verify completion under the 30-min Azure limit.

### 10.2 Diagnostic settings → Log Analytics + Storage (portal)
1. Portal → `org-hsm-prod` → **Monitoring → Diagnostic settings → + Add diagnostic setting**.
2. Name: `diag-hsm`.
3. **Logs**: tick **AuditEvent** and **AzurePolicyEvaluationDetails**.
4. **Metrics**: tick **AllMetrics**.
5. **Destination**: Log Analytics workspace → pick `law-org` (or your central LAW).
6. Save.

The single `org-hsm-prod` resource emits logs from both regional pools.

### 10.3 Alerts (recommended baseline, portal)
Create via **Monitor → Alerts → + Create → Alert rule**, scope = the HSM resource:

| Signal | Condition | Severity / action |
|---|---|---|
| Custom log search: `AzureDiagnostics \| where ResourceProvider == "MICROSOFT.KEYVAULT" and OperationName in ("VaultDelete", "Purge", "PurgeDeletedKey")` | Count > 0 over 5 min | **P1** — page Security on-call |
| Custom log search: failed-auth rate | > 5/min sustained | **P2** |
| Custom log search: **denied-by-firewall** rate | > 20/min sustained | **P2** — investigate scan / leak |
| Custom log search: key version not used in 30 days | scheduled query | **P3** |
| Metric: HSM region health degraded | Status != Healthy | **P1** — initiate DR runbook |
| Metric: `HsmReplicationLatencyMs` | > 1000 ms sustained 5 min | **P1** |

### 10.4 Activity log alerts (portal)
1. **Monitor → Alerts → + Create → Alert rule** → Scope: the HSM.
2. Condition → Signal `Administrative` → Operation `Microsoft.KeyVault/managedHSMs/write` or `…/delete`.
3. **Action group → + Add** → Security on-call.
4. **In this edition, the "write" alert is especially important** — any firewall allow-list change is captured here ([§1.6 R3](#16-risk-register--operational-risks-publicnetwork-specific)).

### 10.5 Periodic configuration audits (quarterly)
Schedule quarterly, align with the DR drill in [§11.2](#112-quarterly-drill-no-production-impact).

**Check B — Workload FQDN drift (R1)** *(portal)*:
1. App Configuration / App Service / Function App: scan **Configuration → Application settings** for `.southindia.managedhsm` or `.centralindia.managedhsm` patterns. Any finding is **P2** — migrate to the canonical URL.

**Check D — Firewall allow-list drift (R3)** *(portal)*:
1. Portal → `org-hsm-prod` → **Networking → Firewalls and virtual networks**.
2. Compare the **IP address or CIDR** list against the signed-off baseline from [§7.1](#71-review-and-finalise-the-production-allowlist).
3. Any unknown entry, any wide CIDR (e.g. `/16` or larger that isn't approved), or any `0.0.0.0/0` is a **P1** — remove immediately, cross-check Activity Log for who added it.
4. Confirm **Trusted services bypass** is still **Yes**.
5. Confirm **Default action: Deny**.

**Check E — HSM Local RBAC drift (R5)** *(portal)*:
1. Portal → `org-hsm-prod` → **Local RBAC**.
2. For every assignment: scope **must** be `/keys/<name>` for workload identities, never `/keys` or `/`.
3. Cross-reference each Members entry against the documented workload inventory; any orphan grant is **P2** — revoke after change-ticket review.

Record each run as a change-ticket entry.

---

## Phase 11 — DR Failover Drill & Runbook

### 11.1 Failover model (Active-Active)
- Both regional pools are reachable via the same canonical FQDN. Microsoft's platform router (TTL 5 s) picks a healthy regional pool per request.
- **No manual DNS swing**, **no manual endpoint toggle** at the HSM layer for normal failover.
- "Failover" means swinging the **application + DB tier**; the HSM piece is automatic.

### 11.2 Quarterly drill (no production impact)
1. Pick a non-prod workload deployed in DR.
2. From an allow-listed admin workstation, run a wrap/unwrap against `https://org-hsm-prod.managedhsm.azure.net/keys/cmk-app-prod`. Verify success.
3. Force-fail an app instance from Primary to DR (Front Door priority swap).
4. Verify SQL MI TDE unseal succeeds in DR — open SQL MI (DR) → **Transparent data encryption**: still showing the HSM key URI, no error.
5. **Validate platform-managed global routing.** From a public client, run `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` repeatedly while a non-prod replicated HSM has its Primary pool intentionally taken out of service via a Severity-A drill ticket. Confirm convergence on the DR-region endpoint within one TTL (~5 s) and that `Test-NetConnection ... -Port 443` succeeds against the new target.
6. Restore the drill HSM; record metrics; close the drill ticket.

### 11.3 Real failover runbook (region loss)
1. Declare incident; activate DR ICs.
2. **Verify the platform-managed global router has already converged** on the DR endpoint. From a healthy out-of-region location: `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` should now point at the DR-region endpoint; `Test-NetConnection ... -Port 443` should succeed within ~5 s of Primary becoming unhealthy. **No customer action is required to trigger this** — if DNS has not converged after ~30 s, raise a Severity-A support ticket against Managed HSM.
3. Swing Front Door for the application tier: DR = priority 1, Primary = priority 2 (drain).
4. Promote DR SQL MI (auto-failover group) — TDE protector already on the same HSM URI → no key re-config.
5. Validate end-to-end transaction.
6. Communicate.

**HSM key-store RPO: ≈ 0** (replication near real-time; eventually consistent within ~6 min).
**HSM data-plane failover: automatic** via the platform-managed router, ~5 s TTL.
**Overall solution RTO:** depends on app + DB failover. Target < 1 hour.

### 11.4 Security Domain restore drill (annual)
Once a year, in a **test subscription**:
1. Deploy a throwaway Managed HSM (repeat [§4.2](#42-create-the-hsm-portal)).
2. Bring M custodians together.
3. Upload `org-hsm-prod-SD.json` to the test HSM via `az keyvault security-domain upload`.
4. Confirm activation succeeds in the portal **Overview** blade.
5. Delete the test HSM.

---

## Phase 12 — Go-Live Checklist & Hand-Over

### 12.1 Pre go-live
- [ ] All non-N/A phases complete in both regions.
- [ ] **Public access Enabled** + **Default action: Deny** + **finalised IP allow-list** confirmed against the signed-off baseline.
- [ ] **No `0.0.0.0/0`** anywhere in the allow-list.
- [ ] **Trusted Azure services bypass: Yes** confirmed.
- [ ] **Conditional Access policy** ([§0.5](#05-conditional-access-strongly-recommended-in-this-edition)) covers all admin / crypto-officer groups: MFA + compliant device.
- [ ] Soft-delete **90 days**, Purge protection **Enabled** — verified on the Properties blade.
- [ ] Platform-managed global routing confirmed: `Resolve-DnsName org-hsm-prod.managedhsm.azure.net` returns a healthy regional endpoint; `Test-NetConnection ... -Port 443` succeeds from an allow-listed location.
- [ ] **No customer-deployed Traffic Manager profile** is present for this HSM.
- [ ] **Multi-region replication healthy** — Multi-Region Replication blade shows both regions **Succeeded**; canary test ([§6.2](#62-verify-multi-region-replication-functional-test)) passes.
- [ ] **6-minute eventual-consistency window** documented in all key-rotation and role-assignment runbooks.
- [ ] Security Domain file SHA-256 recorded; 3 copies stored; custodian list signed.
- [ ] **Backup**: daily schedule running; last successful backup blob URL + checksum on file; event-triggered backup configured.
- [ ] **Backup restore drill** executed at least once into a throwaway test HSM, completed under the 30-min limit.
- [ ] DR drill ([§11.2](#112-quarterly-drill-no-production-impact)) executed and signed off.
- [ ] All workload identities have **scoped** roles (no HSM-wide `Crypto User`).
- [ ] Break-glass user tested and credentials sealed.
- [ ] Log Analytics receiving `AuditEvent`.
- [ ] Alerts ([§10.3](#103-alerts-recommended-baseline-portal)) firing on test triggers, including the **denied-by-firewall** alert.

### 12.2 Documentation hand-over to operations
1. This document plus the [private edition](./HSM_Implementation_Azure_Steps.md) and original [CLI guide](./HSM_Implementation_Steps.md) for cross-reference.
2. RBAC matrix ([Appendix A](#appendix-a--rbac-role-reference)).
3. **Signed-off firewall allow-list baseline** + change-ticket procedure for adding/removing entries.
4. Security Domain custody record.
5. Backup schedule and storage account location.
6. Runbooks: failover ([§11.3](#113-real-failover-runbook-region-loss)), key rotation (with 6-min quiet period), SD restore.
7. Contact list: HSM admins, Security on-call, Azure TAM.

---

## Appendix A — RBAC Role Reference

Identical to the private edition. Reproduced for self-containment.

| Role | Scope (recommended) | Used by | Operations |
|---|---|---|---|
| Managed HSM Administrator | `/` | Quorum officers / `grp-org-hsm-admins` | Full control: role assignments, key management, SD operations |
| Managed HSM Crypto Officer | `/keys` | `grp-org-hsm-crypto-officers` | Create / rotate / delete keys |
| Managed HSM Crypto User | `/keys/<key>` | Application managed identities | Wrap, Unwrap, Sign, Verify, Encrypt, Decrypt — **no** key lifecycle |
| Managed HSM Crypto Service Encryption User | `/keys/<key>` | DES, SQL MI, Storage account MI | Wrap / Unwrap only |
| Managed HSM Crypto Auditor | `/` | `grp-org-hsm-auditors` | Read keys and audit logs, no crypto ops |
| Managed HSM Backup | `/` | Backup automation SP | Trigger full backup, read SAS |

---

## Appendix B — Security Domain Ceremony (Portal-Aware Detailed)

Identical to the private edition. Cross-reference [`HSM_Implementation_Azure_Steps.md` Appendix B](./HSM_Implementation_Azure_Steps.md#appendix-b--security-domain-ceremony-portal-aware-detailed).

**Public-network nuance:** the admin workstation that runs `az keyvault security-domain download` must be on the HSM firewall allow-list ([§4.2](#42-create-the-hsm-portal)) before the ceremony. Test connectivity with `Test-NetConnection org-hsm-prod.managedhsm.azure.net -Port 443` immediately before starting.

---

## Appendix C — Troubleshooting (Public-Network Specific)

### C.1 HSM stuck at `Provisioning` for > 1 hour
Same as private edition: check Activity Log, confirm quota, open Sev-B support ticket.

### C.2 `403 Forbidden` from a workload identity that should have access
- Reproduce: `az keyvault key show --hsm-name org-hsm-prod -n <key>`.
- HSM → **Local RBAC**: confirm the role assignment exists at the right scope.
- Control-plane Azure RBAC does NOT grant data-plane HSM rights.
- If role assigned in last ~6 min, wait — eventual consistency.

### C.3 `404 KeyNotFound` immediately after key creation
Replication lag. Wait up to 6 min. If persistent, confirm the key exists in **HSM → Keys**.

### C.4 `Forbidden` / `connection reset` from an allow-listed IP that worked yesterday
- HSM → **Networking** → confirm the IP is still in the allow-list (check Activity Log for unexpected `Microsoft.KeyVault/managedHSMs/write` events).
- Confirm your egress IP hasn't changed (corporate NAT pool rotation, VPN gateway swap).
- Run `curl ifconfig.me` from the source to confirm current egress IP.

### C.5 `Forbidden` from an Azure service that should benefit from trusted-services bypass
- HSM → **Networking → Firewalls and virtual networks** → confirm **Allow Azure services on the trusted services list to bypass this firewall: Yes**.
- Confirm the service's managed identity has the correct HSM-local role at the right scope ([§8.4](#84-assign-leastprivilege-roles-to-workload-identities-portal)).
- The trusted services list does not include arbitrary Azure services — confirm yours is listed (DES, SQL MI TDE, Storage CMK, App Service / Function App CMK are all included).

### C.6 "Add Region" greyed-out or "region not available"
- Confirm both regions are supported for Managed HSM multi-region replication.
- Confirm quota is approved in the target region.
- Confirm the HSM is **Activated**.

### C.7 "Multi-Region Replication" blade missing
- Hard-refresh portal.
- Confirm Reader on HSM + Managed HSM Administrator local RBAC.

### C.8 Backup blob fails with `AuthorizationFailure`
- SAS token needs `cw` (create + write).
- Storage account firewall — if locked down, add the HSM's outbound IPs or use a service endpoint (Storage side).

### C.9 SQL MI TDE blade rejects the HSM key URI
- SQL MI MI needs **Managed HSM Crypto Service Encryption User** on `/keys/cmk-sqlmi-prod`.
- Key URI must use the canonical FQDN.
- Wait ~6 min if role just assigned (replication lag).

### C.10 Unexpected spike in denied-by-firewall log entries
- Likely an internet scanner hitting `*.managedhsm.azure.net`. The HSM firewall is doing its job — TCP connect succeeds, but data-plane requests are dropped before AAD.
- Cross-check the source IPs in the diagnostic log against threat intel.
- If a known legitimate caller is being denied, add the source IP to the allow-list **only after** confirming with the workload owner and raising a change ticket.

---

> **End of public-network edition.** For the full networked / private-endpoint version of the same deployment, see [`HSM_Implementation_Azure_Steps.md`](./HSM_Implementation_Azure_Steps.md). For the CLI commands behind any portal click-path, see [`HSM_Implementation_Steps.md`](./HSM_Implementation_Steps.md).
