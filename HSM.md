# Azure Managed HSM for CMK Encryption — Deployment Model, Hardware Setup & Implementation Plan for Contoso (South India + Central India)

---

## Table of Contents

1. [Hardware Setup — Who Does What?](#1-hardware-setup--who-does-what)
2. [Deployment Model for South India + Central India](#2-deployment-model-for-south-india--central-india)
3. [How the Deployment Actually Happens — Step by Step](#3-how-the-deployment-actually-happens--step-by-step)
4. [Phase 1: SQL Managed Instance + AD Servers](#4-phase-1-sql-managed-instance--ad-servers)
5. [Phase 2: Remaining Services](#5-phase-2-remaining-services)
6. [Recommended Deployment Timeline](#6-recommended-deployment-timeline)
7. [Pricing Summary](#7-pricing-summary)
8. [Questions We Need Answered to Finalize the Architecture](#8-questions-we-need-answered-to-finalize-the-architecture)
9. [Summary for Leadership](#9-summary-for-leadership)

---

## 1. Hardware Setup — Who Does What?

A common and important question: **does Contoso need to procure or manage any hardware?**

**No.** Microsoft handles 100% of the physical HSM infrastructure. When Contoso creates a Managed HSM resource, Azure automatically provisions dedicated, single-tenant Marvell LiquidSecurity HSM partitions in the requested region. Contoso never sees, touches, or manages any physical device.

### Complete Responsibility Breakdown

| Layer | Responsibility | Details |
|---|---|---|
| **Physical HSM hardware** | **Microsoft** | Purchases, racks, networks, and secures Marvell LiquidSecurity HSM adapters in Azure datacenters |
| **Hardware maintenance** | **Microsoft** | Firmware updates, hardware replacements, power/cooling — all invisible to Contoso |
| **HSM cluster provisioning** | **Microsoft** | Automatically provisions a 3-partition HSM pool with HA across Availability Zones when Contoso creates the resource |
| **Confidential Compute isolation** | **Microsoft** | All HSM operations execute inside Trusted Execution Environments (TEE) — even Microsoft operators cannot access key material at any point |
| **Network connectivity** | **Microsoft + Contoso** | Microsoft provisions the HSM endpoint; Contoso configures Private Endpoint + Private DNS in their VNet |
| **Resource creation** | **Contoso** | Creates the Managed HSM pool via Azure Portal, CLI, or ARM/Bicep template — the same experience as creating any other Azure resource |
| **Security Domain activation** | **Contoso** | Downloads the Security Domain (root of trust) using a minimum of 3 RSA key pairs — this is the step that gives Contoso exclusive cryptographic control |
| **Key creation & lifecycle** | **Contoso** | Creates, rotates, and manages encryption keys via the Key Vault API |
| **Access control (RBAC)** | **Contoso** | Assigns roles (Crypto Officer, Crypto User, etc.) using Microsoft Entra ID |
| **Integration with Azure services** | **Contoso** | Points SQL MI, Storage, VMs, etc. to use CMK from the Managed HSM |

### Key Message for Leadership

Contoso does not procure, install, or manage any hardware. Microsoft provisions dedicated, FIPS 140-3 Level 3 validated HSM hardware automatically. The Security Domain activation is the critical moment — that's when Contoso takes exclusive ownership of the root of trust. After that point, Microsoft has zero ability to access or recover Contoso's keys.

> ⚠️ **Important responsibility:** If Contoso loses the Security Domain backup files and private keys, the keys stored in the HSM are **permanently and irrecoverably lost**. These must be stored securely offline with appropriate custodianship procedures.

---

## 2. Deployment Model for South India + Central India

Contoso currently operates across **two Azure regions — South India (SI) and Central India (CI).** We've confirmed that **Azure Key Vault Managed HSM is available in both regions.**

The deployment model depends on how these regions relate to each other:

| Scenario | Recommended Deployment | Monthly Cost |
|---|---|---|
| **SI = Primary Production, CI = DR** | 1 Managed HSM pool in SI (primary) + 1 Managed HSM pool in CI (DR standby) | ~$4,608/month |
| **SI and CI = Active-Active** (both serve production traffic) | 1 Managed HSM pool in each region with multi-region replication enabled | ~$4,608/month |
| **SI and CI = Independent environments** | 1 Managed HSM pool per region (no replication) | ~$4,608/month |
| **Single region only** (if DR is not required for keys) | 1 Managed HSM pool in primary region | ~$2,304/month |

> *Reminder: High availability within each region is built-in at no extra cost — each pool automatically runs as a 3-partition cluster spread across Availability Zones. The second pool is only for cross-region DR or active-active requirements.*

### Two-Region Deployment Architecture

```
South India (Primary)                    Central India (DR/Secondary)
┌──────────────────────────┐            ┌──────────────────────────┐
│ Managed HSM Pool: SI     │            │ Managed HSM Pool: CI     │
│ • Create resource        │            │ • Create resource        │
│ • Activate Security      │            │ • Activate Security      │
│   Domain (separate SD)   │            │   Domain (separate SD)   │
│ • Create keys            │            │ • Create keys (or        │
│ • Integrate SQL MI,      │            │   replicate from SI)     │
│   AD VMs, etc.           │            │ • Integrate DR workloads │
│                          │            │                          │
│ ~$2,304/month            │            │ ~$2,304/month            │
└──────────────────────────┘            └──────────────────────────┘
```

Each region gets its own independent HSM pool with its own Security Domain. If multi-region replication is enabled, keys can sync automatically between the two pools.

---

## 3. How the Deployment Actually Happens — Step by Step

From Contoso's perspective, the entire deployment follows four steps:

### Step 1: Create the Managed HSM Resource (~20–30 minutes)

Contoso creates the resource via Azure Portal, CLI, or ARM template. Behind the scenes, Azure automatically:

- Allocates dedicated Marvell LiquidSecurity HSM partitions in the requested region
- Provisions a 3-partition cluster across Availability Zones for automatic HA
- Configures Confidential Compute Infrastructure (TEE) for key isolation
- Exposes a secure HTTPS endpoint (e.g., `https://Contoso-prod-hsm-si.managedhsm.azure.net`)

After this step, the HSM is **provisioned but not yet usable** — it requires activation.

### Step 2: Activate — Download Security Domain (~15 minutes)

This is the most critical step. Contoso generates a minimum of 3 RSA key pairs and downloads the Security Domain — this is the root of trust that gives Contoso exclusive cryptographic control:

- Generate 3 RSA key pairs (for disaster recovery quorum)
- Download the Security Domain file from the HSM
- **Store the Security Domain file and private keys securely offline**

After activation, the HSM is **live and ready for use**. Microsoft has zero ability to access keys from this point forward.

### Step 3: Configure Access & Create Keys (~30 minutes–1 hour)

- Assign RBAC roles using Microsoft Entra ID (Crypto Officer, Crypto User, Crypto Service Encryption User)
- Create CMK keys for each workload (e.g., `Contoso-SQLMI-CMK`, `Contoso-DISK-CMK`)
- Configure Private Endpoint for secure VNet access

### Step 4: Integrate Azure Services (per service, ~30 minutes–1 hour each)

- Point SQL MI TDE protector to the Managed HSM key
- Create Disk Encryption Sets for AD VM disks referencing the Managed HSM key
- (Phase 2) Connect Storage, Cosmos DB, and other services

---

## 4. Phase 1: SQL Managed Instance + AD Servers

### ✅ SQL Managed Instance — Fully Supported with Managed HSM

SQL MI supports **Transparent Data Encryption (TDE) with customer-managed keys** stored in Managed HSM. This is a native, well-documented integration.

**How it works:**

```
SQL Managed Instance → TDE Protector → Key in Managed HSM Pool
         ↑                                        ↑
   Managed Identity                     Wrap/Unwrap/Get permissions
   (auto-assigned)                      (granted via Azure RBAC)
```

**Setup steps:**

1. Enable **System-Assigned Managed Identity** on the SQL MI instance
2. In the Managed HSM pool, assign the **Managed HSM Crypto Service Encryption User** role to the SQL MI's managed identity
3. Create (or import) an RSA-HSM key in the Managed HSM pool
4. Configure the SQL MI TDE protector to point to the Managed HSM key
5. Verify TDE is active with CMK

If Contoso has SQL MI in both SI and CI, each region's SQL MI should reference the Managed HSM pool **in the same region** for low latency and availability.

---

### ⚠️ AD Servers — Clarification Needed from Contoso

The CMK approach for AD servers depends on **how they are deployed**. This is a critical distinction:

| AD Deployment Model | CMK with Managed HSM? | Details |
|---|---|---|
| **Self-hosted AD Domain Controllers on Azure VMs** | ✅ **Supported** | VM OS and data disks can be encrypted using **Azure Disk Encryption** or **Server-Side Encryption** with CMK keys from Managed HSM |
| **Azure AD Domain Services (managed service)** | ❌ **Not supported** | Azure AD DS is fully managed by Microsoft — CMK encryption is not available; only platform-managed keys are used |

**If self-hosted AD on Azure VMs** (which we believe is more likely for Contoso's enterprise environment), the deployment would be:

```
AD Domain Controller VM (SI/CI)
    └── OS Disk ──→ Disk Encryption Set ──→ Key in Managed HSM Pool (same region)
    └── Data Disk ──→ Disk Encryption Set ──→ Key in Managed HSM Pool (same region)
```

**Setup steps:**

1. Create a **Disk Encryption Set (DES)** in each region, linked to the Managed HSM key
2. Assign the DES's managed identity the **Managed HSM Crypto Service Encryption User** role
3. Associate the DES with the AD VM's OS and data disks
4. Verify encryption status and validate domain services are unaffected

> **We need Contoso to confirm which AD deployment model is in use** — this determines whether CMK is feasible for this workload in Phase 1.

---

## 5. Phase 2: Remaining Services

Once Phase 1 is validated, extending CMK to additional Azure services follows the same pattern — each service's managed identity gets RBAC access to the Managed HSM key.

### Common Services and CMK Readiness

| Azure Service | CMK with Managed HSM | Integration Method |
|---|---|---|
| Azure Storage (Blob, File, Queue, Table) | ✅ | Storage account encryption settings → Managed HSM key |
| Azure Cosmos DB | ✅ | Account-level encryption with CMK |
| Azure Disk Encryption (all VMs) | ✅ | Disk Encryption Set → Managed HSM key |
| Azure Data Lake Storage | ✅ | Storage encryption settings |
| Azure Kubernetes Service (AKS) | ✅ | Disk + etcd encryption with CMK |
| Microsoft 365 / Purview Customer Key | ✅ | Managed HSM key reference |
| Azure SQL Database (if applicable) | ✅ | TDE with CMK (same as SQL MI) |
| Azure Information Protection | ✅ | Managed HSM key reference |

---

## 6. Recommended Deployment Timeline

| Phase | Scope | Key Actions | Estimated Duration |
|---|---|---|---|
| **Phase 0** | Foundation | Provision Managed HSM pool(s) in SI and CI; activate Security Domain; establish RBAC policies; define key naming and rotation standards; configure Private Endpoints | 1–2 weeks |
| **Phase 1a** | SQL MI (CMK) | Configure TDE with CMK on SQL MI in primary region; validate; extend to secondary region | 2–3 weeks |
| **Phase 1b** | AD Servers (CMK) | Create Disk Encryption Sets; encrypt AD VM disks with CMK; validate domain services unaffected | 2–3 weeks |
| **Phase 1 — Validation** | Soak period | Monitor for 2–4 weeks in production; validate key operations, backup/restore, and failover procedures | 2–4 weeks |
| **Phase 2** | Remaining services | Extend CMK to Storage, Cosmos DB, additional VMs, AKS, etc. — one service at a time | 4–8 weeks (rolling) |

---

## 7. Pricing Summary

### Per-Component Pricing

| Component | Cost | Notes |
|---|---|---|
| **Managed HSM Pool (Standard B1)** | ~$3.20/hour (~$2,304/month per pool) | Fixed cost — billed continuously once created; cannot be paused |
| **Per HSM-protected key** | $1.00/key/month (RSA 2048) or $5.00/key/month (RSA 3072/4096, ECC) | Based on key type |
| **Per operation** | $0.03 per 10,000 operations | Encrypt, decrypt, sign, wrap/unwrap, etc. |

### Estimated Monthly Cost for Contoso

| Deployment | Monthly Cost |
|---|---|
| 1 Managed HSM pool (single region) | ~$2,304 + key/operation fees |
| 2 Managed HSM pools (SI + CI) | ~$4,608 + key/operation fees |

### Cost Comparison with Legacy Dedicated HSM

The now-retired Azure Dedicated HSM would have cost ~$6,984/month for a production HA pair in a single region.

- **Managed HSM (two regional pools):** ~$4,608/month — approximately **34% cost reduction** with significantly better HA, SLA, and zero infrastructure management burden
- **Managed HSM (single region):** ~$2,304/month — approximately **67% cost reduction**

> *We would recommend confirming per-key pricing details with your Microsoft account team, as some advanced key type charges may also apply within the Managed HSM pool.*

---

## 8. Questions We Need Answered to Finalize the Architecture

To provide Contoso with a detailed architecture diagram and step-by-step implementation plan, we need clarity on the following:

| # | Question | Why It Matters |
|---|---|---|
| 1 | **SI/CI relationship** — Is it Primary/DR, Active-Active, or independent environments? | Determines whether we need 1 or 2 Managed HSM pools, and whether multi-region replication should be enabled |
| 2 | **AD server deployment model** — Are your AD servers self-hosted Domain Controllers on Azure VMs, or Azure AD Domain Services (managed)? | Self-hosted VMs → ✅ CMK supported via Disk Encryption. Azure AD DS → ❌ CMK not supported. This is a critical distinction for Phase 1 scoping |
| 3 | **SQL MI count and regions** — How many SQL MI instances does Contoso have, and in which region(s)? | Determines the number of keys needed and RBAC assignments per region |
| 4 | **Phase 2 service inventory** — Which additional Azure services are in scope for CMK in Phase 2? | Allows us to validate CMK support for each service and plan integration sequencing |
| 5 | **Cross-region DR requirement** — Does Contoso's compliance framework mandate that encryption keys must be redundant across regions? | Determines if the second Managed HSM pool is required or optional |
| 6 | **Key rotation policy** — Does Contoso have an existing key rotation cadence requirement (e.g., annual, quarterly)? | Influences key management automation and operational procedures |
| 7 | **Network architecture** — Are SI and CI connected via VNet peering, and do workloads in both regions need to access the HSM? | Determines Private Endpoint configuration and DNS setup |
| 8 | **Non-production environment** — Does Contoso have a dev/test environment where we can pilot the Managed HSM deployment before production? | We strongly recommend validating in non-production first |

---

## 9. Summary for Leadership

| Concern | Answer |
|---|---|
| **Does Contoso need to procure hardware?** | **No** — Microsoft provisions dedicated HSM hardware automatically |
| **Does Microsoft access Contoso's keys?** | **No — never.** HSM operations run inside Confidential Compute Infrastructure; even Microsoft operators cannot access key material |
| **Is this dedicated hardware?** | **Yes** — single-tenant, FIPS 140-3 Level 3 validated, cryptographically isolated |
| **Is this a security downgrade from Dedicated HSM?** | **No — it's an upgrade.** FIPS 140-3 L3 (newer standard), PCI DSS 4.0 certified, built-in HA |
| **What does it cost?** | ~$2,304/month per region (HA included) — approximately 67% less than the legacy Dedicated HSM model per region |
| **Do we need special approval or minimum spend?** | **No** — self-service, pay-as-you-go, no $5M commitment gate |
| **Who manages infrastructure?** | **Microsoft** — Contoso focuses solely on key management, access policies, and service integration |
| **How long to deploy?** | Phase 0 (foundation): 1–2 weeks. Phase 1 (SQL MI + AD): 4–6 weeks including validation. Phase 2 (remaining): 4–8 weeks rolling |

**This represents a modernization, not a downgrade.** The newer service is purpose-built for cloud-native operations while maintaining the same (or higher) cryptographic assurance standards, with significantly reduced operational burden and cost.

---

