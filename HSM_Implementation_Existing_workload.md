# Azure Key Vault Managed HSM — Existing Workloads (Central India, Single Region)

> This guide explains the Contoso Managed HSM rollout for **existing, already-running** SQL Managed Instances and VMs in **Central India only**. It is deliberately simplified for **process understanding**: multi-region replication, disaster recovery, and Storage are out of scope here. The goal is for the team to clearly understand **what changes**, **what stays the same**, **when downtime happens and when it doesn't**, and **which scenarios apply to which VM / MI**.

---

## Table of Contents

**Scope & Mental Model**

- [What's Out of Scope in This Guide](#whats-out-of-scope-in-this-guide)
- [The Big Picture — What Are We Doing?](#the-big-picture--what-are-we-doing)
- [How It All Fits Together (Single Region, Existing Workloads)](#how-it-all-fits-together-single-region-existing-workloads)

**Concepts You Need Before the Cutover**

- [The Three Separate Things You Create](#the-three-separate-things-you-create)
- [What Lives Where](#what-lives-where)
- [How DEK Actually Works — At Rest vs When Accessed](#how-dek-actually-works--at-rest-vs-when-accessed)
- [Important: DEK vs DES — They Sound Similar but Are Completely Different](#important-dek-vs-des--they-sound-similar-but-are-completely-different)
- [Why This Design Is Secure](#why-this-design-is-secure)
- [The Full Lifecycle (Single Region, VMs + SQL MI Only)](#the-full-lifecycle-single-region-vms--sql-mi-only)

**Section by Section**

- [Terminology & Assumptions](#terminology--assumptions)
- [Section 1: High-Level Architecture](#section-1-high-level-architecture)
- [Section 2: What Microsoft Does Behind the Scenes](#section-2-what-microsoft-does-behind-the-scenes)
- [Section 3: The Deployment Steps (Phase 0)](#section-3-the-deployment-steps-phase-0)
- [Section 4: Migrating EXISTING SQL Managed Instances (Phase 1a)](#section-4-migrating-existing-sql-managed-instances-phase-1a)
  - [Scenarios for the existing SQL MI estate](#scenarios-for-the-existing-sql-mi-estate-central-india)
  - [Pre-flight checklist](#pre-flight-checklist-for-each-in-scope-sql-mi)
  - [The migration steps (per SQL MI)](#the-migration-steps-per-sql-mi)
  - [What happens if it goes wrong mid-switch?](#what-happens-if-it-goes-wrong-mid-switch)
  - [⚠️ When SQL MI *can* see downtime](#-when-sql-mi-can-see-downtime--hsm-access-lost-after-the-protector-switch)
  - [Other SQL MI nuances worth knowing](#other-sql-mi-nuances-worth-knowing)
- [Section 5: Migrating EXISTING VM Disks (Phase 1b)](#section-5-migrating-existing-vm-disks-phase-1b)
  - [Pick the scenario that matches each VM](#pick-the-scenario-that-matches-each-vm--this-drives-the-downtime-the-steps-and-the-risk)
  - [Scenario A — Managed disks, never had ADE](#scenario-a--managed-disks-never-had-ade--most-common-simplest)
  - [Scenario B — Managed disks that have ever had ADE (rebuild)](#scenario-b--managed-disks-that-have-ever-had-ade--rebuild-required)
  - [Scenario C — Unmanaged disks (convert first)](#scenario-c--unmanaged-disks-page-blob-vhds--convert-first-then-go-to-a-or-b)
  - [Scenario D — AD domain controllers](#scenario-d--ad-domain-controllers--cuts-across-all-scenarios-above)
  - [Eligibility gate](#eligibility-gate--must-be-run-for-every-disk-before-any-window-is-scheduled)
  - [⚠️ Tier-0 key-state changes (whole-fleet blast radius)](#-ops-note--tier-0-key-state-changes-whole-fleet-blast-radius)
- [Section 6: Network (Private Endpoint)](#section-6-network-private-endpoint)
- [Section 7: RBAC](#section-7-rbac)
- [Section 8: Security Domain](#section-8-security-domain)
- [Section 9: Disaster Recovery](#section-9-disaster-recovery)
- [Section 10: Key Rotation](#section-10-key-rotation)
- [Section 11: Resource Inventory (Single Region)](#section-11-resource-inventory-single-region)
- [Section 12: Pricing (Single Region)](#section-12-pricing-single-region)

**Quick Reference**

- [Side-by-Side: SQL MI vs VM Disks](#side-by-side-sql-mi-vs-vm-disks-single-region-in-scope)

---

## What's Out of Scope in This Guide

To keep this document focused on **process understanding**, the following are deliberately **not covered**:

- ❌ **Multi-region replication / Disaster Recovery / second region** 
- ❌ **Storage account migration** 
- ❌ **Failover groups / geo-replicas** 
- ❌ **Azure Site Recovery (ASR)** 

If any of those apply to a specific workload, switch to the multi-region guide for that workload.

---

## The Big Picture — What Are We Doing?

**Today:** Contoso's existing SQL Managed Instances and VMs in Central India are **already encrypted at rest**, but Microsoft owns the encryption keys (Microsoft-managed / platform-managed keys).

**Goal:** Move the **ownership of those wrapping keys** from Microsoft to Contoso by storing them in a Managed HSM — so Contoso can revoke access at any time and meet PCI-DSS requirements.

**What actually changes:** the **wrapping key** that protects each workload's Data Encryption Key (DEK). The data on disk, and the DEKs themselves, are **not** re-encrypted.

> 💡 **Mindset:** This is a **protector swap**, not a re-encryption. The DEK that was protecting your data yesterday will protect it tomorrow — only the key that *wraps* that DEK changes from a Microsoft key to a Contoso key in the HSM.

**Why this matters for understanding downtime:**

- Because no data is being re-encrypted, the operation itself is fast (seconds to minutes per resource).
- For **SQL MI**: the protector swap is **online** — the database keeps serving queries.
- For **VM disks**: the swap is also fast at the storage layer, but Azure requires the VM to be **stopped and deallocated** to change which Disk Encryption Set (DES) a disk is associated with. **The downtime is from the stop/start, not from any re-encryption.**

---

## How It All Fits Together (Single Region, Existing Workloads)

```
1. Create HSM in Central India  (no impact on running workloads)
2. Activate Security Domain (3 RSA keys + offline backup file)
3. Create CMK keys inside the HSM
4. MIGRATE each existing workload to use the CMK as its wrapping key:

   EXISTING SQL MI:
   ┌──────────┐      ┌─────┐      ┌──────────────┐      ┌─────┐
   │ Database │─DEK──│ OLD │  ──► │ NEW: CMK     │──in──│ HSM │
   │ data     │      │ MS  │swap  │(TDE protector│      │     │
   │          │      │ key │      │  flipped)    │      │     │
   └──────────┘      └─────┘      └──────────────┘      └─────┘
   ✅ ONLINE — NO database downtime for the protector switch.
      (See Section 4 for the rare cases where it can become
       "Inaccessible" AFTER the switch.)

   EXISTING VM Disks:
   ┌──────────┐      ┌─────┐      ┌─────┐      ┌─────┐      ┌─────┐
   │ Disk     │─DEK──│ OLD │ ──►  │ DES │──────│ CMK │──in──│ HSM │
   │ data     │      │ PMK │swap  │*new │      │     │      │     │
   └──────────┘      └─────┘      └─────┘      └─────┘      └─────┘
   ⚠️ VM MUST be STOPPED & DEALLOCATED to change which DES
      its disks are associated with. Downtime is the stop/start
      window (~15-30 min per VM), NOT re-encryption time.
      (See Section 5 for the scenarios where this gets longer.)

5. The DEK is NOT regenerated. Only the wrapping key changes.
6. The data on disk is NOT re-encrypted. The bits don't move.
7. After all workloads are migrated, you rotate keys on a schedule.
```

---

## The Three Separate Things You Create

| Thing | Who Creates It | When | Where It Lives | Used For |
|---|---|---|---|---|
| **3 RSA keys + Security Domain** | Security Officers (one-time) | HSM activation | Offline (safe deposit box) | Only for HSM disaster recovery (rebuilding the vault itself) |
| **CMK keys** (`Contoso-SQLMI-CMK`, `Contoso-DISK-CMK`) | Key Management Team (Crypto User role) | After HSM is active, **before any migration cutover** | Inside the HSM hardware (never leaves) | Wrapping / unwrapping DEKs |
| **Service connection** (TDE protector switch on SQL MI; new DES for VMs) | Key Management Team or Infra Team | **Migration window** for each existing workload | Azure resource configuration | Telling existing Azure services “stop using the MS key — use this HSM key now” |

---

## What Lives Where

| Thing | Where It Lives | What It Is |
|---|---|---|
| **CMK** (`Contoso-SQLMI-CMK`, `Contoso-DISK-CMK`) | **Inside HSM hardware** — never leaves | The wrapping key that protects DEKs |
| **DEK** (Data Encryption Key) | **Next to the data** (database / disk) — in **encrypted** form | The key that actually encrypts your data |
| **DEK (plaintext)** | **In memory only** — temporarily, when data is accessed | Unwrapped DEK, used for decryption, never written to disk |
| **DES** (Disk Encryption Set) | **Azure Resource Manager** — it's an Azure resource like a VM | A bridge / config object that connects VM disks to the HSM key |
| **Security Domain** | **Offline** — safe deposit box | Master key to rebuild the entire HSM if it is ever lost |

> **For existing workloads, the DEK already exists.** It was created by Azure on day one when the database / disk was first provisioned. The migration does **not** create a new DEK — it only changes the CMK that wraps that existing DEK. This is why the operation is near-instant for SQL MI and very fast (only stop/start time) for VMs.

---

## How DEK Actually Works — At Rest vs When Accessed

```
AT REST (on disk):
┌──────────────┐     ┌───────────────────────┐
│ Encrypted    │     │ Encrypted DEK         │
│ Data         │     │ (wrapped by CMK)      │
│              │     │                       │
│ (can't read  │     │ (can't use without    │
│  without DEK)│     │  CMK in HSM)          │
└──────────────┘     └───────────────────────┘
Both sit together on disk. Both useless without the HSM.


WHEN DATA IS ACCESSED (in memory):
┌──────────────┐     ┌───────────────────────┐     ┌─────────┐
│ Encrypted    │     │ Encrypted DEK ────────┼────►│   HSM   │
│ Data         │     │                       │     │         │
│              │     │ Plaintext DEK ◄───────┼─────│ Unwraps │
│              │◄────│ (in memory only)      │     │ using   │
│ Decrypted!   │     │                       │     │   CMK   │
└──────────────┘     └───────────────────────┘     └─────────┘
Plaintext DEK exists ONLY in memory, temporarily. Never written to disk.
```

---

## Important: DEK vs DES — They Sound Similar but Are Completely Different

| | DEK | DES |
|---|---|---|
| **Full name** | Data Encryption Key | Disk Encryption Set |
| **What it is** | An actual encryption **key** | An Azure **resource** (a configuration object) |
| **Where it lives** | **Next to the data** (encrypted) + **in memory** (plaintext, temporarily) | **Azure Resource Manager** (like a VM) |
| **Purpose** | Encrypts / decrypts the actual data | **Connects** a VM disk to the CMK in the HSM |
| **Who creates it** | Azure created it when the resource was first provisioned (already exists for existing workloads) | **You** create it manually as part of the VM migration |
| **Lives inside HSM?** | ❌ No — lives with the data (encrypted) | ❌ No — lives in Azure Resource Manager |
| **Needed for which services here?** | ✅ Both SQL MI and VM disks already have DEKs today | ✅ VM disks only — SQL MI does NOT use DES |
| **Analogy** | The actual padlock on the box | The cable that connects the padlock to the safe |

---

## Why This Design Is Secure

Even if an attacker gets access to:

| What They Steal | Can They Read Data? | Why? |
|---|---|---|
| The encrypted data | ❌ No | They don't have the DEK |
| The encrypted DEK | ❌ No | They can't unwrap it without the CMK |
| The encrypted data + encrypted DEK | ❌ No | Still need CMK (in HSM) to unwrap DEK |
| The HSM endpoint URL | ❌ No | They need Azure AD credentials + RBAC permission |
| Azure AD credentials without RBAC | ❌ No | RBAC denies access to the key |

**The only way to read the data:** valid Azure AD credentials **+** correct RBAC role **+** network access to the HSM **+** the CMK is available and enabled. All four conditions must be true simultaneously.

---

## The Full Lifecycle (Single Region, VMs + SQL MI Only)

```
┌────────────────────────────────────────────────────────────────┐
│  PHASE 0  —  Stand up the HSM  (no impact on running workloads)│
│  • Create the HSM in Central India                             │
│  • Activate Security Domain (3 officers, 2-of-3 quorum)        │
│  • Create the two CMKs:                                        │
│      - Contoso-SQLMI-CMK                                       │
│      - Contoso-DISK-CMK                                        │
│  • Configure RBAC + Private Endpoint                           │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  PRE-MIGRATION ASSESSMENT (per existing workload)              │
│  • Inventory the in-scope SQL MIs and VMs                      │
│  • Capture current encryption settings (rollback baseline)     │
│  • Confirm backups are healthy                                 │
│  • For VMs: identify scenario (managed/unmanaged, ADE history) │
│  • Schedule maintenance window (only needed for VMs)           │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  PHASE 1-PRE — ELIGIBILITY GATE (mandatory for VMs)            │
│  • For each VM disk: az disk show ... --query                  │
│     "encryptionSettingsCollection"                             │
│     → non-null = ADE-EVER → needs REBUILD path                 │
│  • For each VM: managed vs unmanaged disk audit                │
│     → unmanaged disks were retired 31-Mar-2026; must be        │
│       converted to managed FIRST in a separate window          │
│  • Confirm CMK is RSA 2048 or 3072 bits (NOT 4096) for SQL MI  │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  PHASE 1a  —  Migrate SQL MIs to CMK   (ONLINE — NO DOWNTIME)  │
│  • Grant SQL MI managed identity access to Contoso-SQLMI-CMK   │
│  • Switch TDE protector from "Service-managed" → CMK           │
│  • Validate                                                    │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  PHASE 1b  —  Migrate VM disks to CMK   (REQUIRES VM RESTART)  │
│  • Create the DES (one-time) → grant DES access to disk CMK    │
│  • For each VM: stop/deallocate → update disks to use DES →    │
│    start VM → validate. ~15-30 min downtime per VM.            │
│  • ADE-ever VMs follow the rebuild path instead (see Sect. 5)  │
└────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌────────────────────────────────────────────────────────────────┐
│  STEADY STATE                                                  │
│  • All in-scope SQL MIs and VMs use Contoso CMK                │
│  • Rotate keys on schedule (no DR / no ASR concerns here)      │
└────────────────────────────────────────────────────────────────┘
```

---

## Section by Section — Plain English (Existing Workloads)

---

### Terminology & Assumptions

**One Azure region in scope:**

- **Central India (Pune)** — the only region this guide covers. The existing SQL MIs and VMs being migrated are assumed to live here.

This guide assumes a **single-region** deployment so the team can build clear mental models around the protector swap, the DEK behavior, and the downtime characteristics. **Multi-region replication, geo-replicas, failover groups, and DR are explicitly out of scope** — those concerns are addressed in the multi-region version of this document.

---

### Section 1: High-Level Architecture

One Managed HSM resource in **Central India** (`Contoso-PROD-HSM`). All keys, RBAC, and policies live in this single HSM. There is no replicated mirror, no DR pair, and no second region in this guide.

**What's different for existing workloads:** the HSM is being introduced **alongside** running services, not greenfield. Until you actually flip each workload's protector, those workloads continue to use Microsoft-managed keys exactly as before. There is no "all-or-nothing" cutover — it's per-workload, and you can pause between workloads.

**Cost note:** Single region, single HSM — ~$2,304/month for the HSM hardware. Costs start the moment the HSM is provisioned, **not** when you start migrating workloads. So there's a real incentive to keep the gap between "HSM ready" and "workloads migrated" as short as possible.

---

### Section 2: What Microsoft Does Behind the Scenes

Identical to the fresh-deployment guide. Marvell LiquidSecurity HSM, 3 partitions on different racks inside the Central India datacenter, Confidential Compute envelope, etc. The hardware story doesn't change just because the workloads pre-exist.

---

### Section 3: The Deployment Steps (Phase 0)

Phase 0 has **zero impact** on running workloads. You can run all of Phase 0 while production traffic continues flowing through Microsoft-managed keys. Existing SQL MIs and VMs don't even know the HSM has been created.

Quick recap (see the fresh-deployment guide for full detail):

1. **Create the HSM** in Central India with soft-delete + purge protection
2. **Activate Security Domain** with 3 officers (2-of-3 quorum)
3. **Create the two CMK keys**:
   - `Contoso-SQLMI-CMK` — RSA 2,048 or 3,072 bits (SQL MI does **not** support 4,096-bit keys as a TDE protector)
   - `Contoso-DISK-CMK` — RSA (any supported size)
4. **Set up RBAC** (remember the role-name trap: **Crypto User** = key management, **Crypto Officer** does NOT manage keys)
5. **Set up the Private Endpoint** in the workload VNet so VMs / management calls can reach the HSM privately

> ⚠️ **Do NOT begin Phase 1 migrations until Phase 0 is fully complete and the HSM is reachable from the workload subnets.** A failed reachability test at the moment of the SQL MI protector switch is the most common cause of a stuck migration.

---

### Section 4: Migrating EXISTING SQL Managed Instances (Phase 1a)

> **Will this cause downtime?  → NO** (in the normal case).
>
> The TDE protector switch on a SQL MI is an **online operation**. The database stays available throughout. The DEK that already encrypts the database pages is **not replaced** — SQL MI just re-wraps that same DEK with the new CMK on the next checkpoint. **No re-encryption of database pages happens.**
>
> The only situation where users might see an interruption is the *failure* mode (HSM unreachable / RBAC misconfigured) covered in the timing table further down — and that is preventable with the pre-flight checklist.

```
BEFORE migration:
Database data ← DEK ← Microsoft-managed key (in MS infrastructure)

AFTER migration:
Database data ← (same DEK) ← Contoso CMK (in Contoso HSM)

The DEK is unchanged. Only the wrapper key changes. Online operation.
```

---

#### Scenarios for the existing SQL MI estate (Central India)

You will likely encounter a mix of the following. Identify which scenario each MI falls into **before** scheduling the cutover; the steps are nearly identical, but the validation depth differs.

**Scenario A — A standalone SQL MI with a single database**

The simplest case. One MI, one DB. No replicas. No failover group.

- Cutover is online. **Zero downtime.**
- Steps: enable the MI's managed identity → grant it `Crypto Service Encryption User` on `Contoso-SQLMI-CMK` → flip the TDE protector to the HSM key → validate.
- Total elapsed wall-clock time per MI: typically 5-15 minutes including validation.

**Scenario B — A standalone SQL MI hosting multiple databases**

This is the common case for Contoso. **Important to understand:**

- The TDE protector is set **at the MI level, not the DB level.** When you flip the protector, **every database on that MI flips at the same time, in one operation.**
- This is a feature, not a footgun — you don't need to run the cutover N times for N databases.
- **Zero downtime** for any of the databases on that MI.
- Validation is heavier: smoke-test a write/read against **every** database on the MI, not just one.

**Scenario C — A SQL MI that is part of a failover group (geo-replication)**

> **This scenario is OUT OF SCOPE for this single-region guide.**
>
> Failover groups by definition span two regions. Migrating a failover-group SQL MI requires Phase-0 work in both regions and a coordinated cutover across both ends. Use the **multi-region** version of this guide for that scenario.

If you discover a failover group during the inventory step, **flag it and skip the MI for now** — do not attempt a single-region cutover on one half of a failover pair.

---

#### Pre-flight checklist for each in-scope SQL MI

Run these checks before the cutover window. None of them require taking the MI offline.

| Check | Why it matters |
|---|---|
| HSM has `purge protection` enabled | Azure SQL refuses to use any vault without it |
| HSM has soft-delete enabled (it is by default) | Mandatory for TDE protector use |
| **CMK is RSA 2,048 or 3,072 bits (NOT 4,096)** | SQL MI silently rejects 4,096-bit keys — the TDE setup will not complete and the failure is not obvious from the portal |
| CMK status is **Enabled**, activation date in the past, expiration in the future | SQL MI validates key state before accepting the protector |
| SQL MI has a **System-Assigned or User-Assigned Managed Identity** | Required for the MI to authenticate to the HSM. **Field/CSA recommendation: prefer User-Assigned (UAMI)**. UAMI's lifecycle is decoupled from the SQL MI (survives recreation), and switching from SAMI to UAMI later requires reconfiguring the TDE protector — operationally heavier than getting it right the first time. |
| Network path: SQL MI subnet → HSM endpoint resolves | The MI must reach the HSM at the moment of the switch and on every restart afterwards. SQL MI is a managed service — its TDE-protector traffic to HSM does **not** route through Contoso's Private Endpoint. It relies on either (a) the HSM firewall being open to the SQL MI's outbound IPs, or (b) the *Allow trusted Microsoft services to bypass the firewall* setting being enabled. See Section 6 for details. |
| Database is not in a transitional state (restoring, copying, scaling) | The protector switch can fail mid-operation |
| **Within per-HSM caps**: ≤ 500 General Purpose DBs / ≤ 200 Business Critical DBs / ≤ 500 Hyperscale page servers per HSM | Exceeding caps causes throttling, latency spikes and connection timeouts. With one HSM in this guide, all in-scope MIs share the same caps. |
| **LTR + PITR retention windows are documented** | Drives how long every old CMK version must be retained post-rotation — historical backups are pinned to the protector version that was active at backup creation time |
| Recent successful backup confirmed | Rollback insurance |

---

#### The migration steps (per SQL MI)

```
Step 1: Enable Managed Identity on the SQL MI (UAMI preferred)
        — no downtime

Step 2: Grant that identity the
        "Managed HSM Crypto Service Encryption User" role,
        scoped to /keys/Contoso-SQLMI-CMK

Step 3: Switch the TDE protector:
          Portal: Security → Transparent Data Encryption →
                  Customer-Managed Key → select HSM key
          CLI:    az sql mi tde-key set ...

Step 4: Validate:
          • Run a write + read against EACH database on the MI
          • Check sys.dm_db_encryption_keys → encryption_state = 3
          • Check the TDE protector shows the HSM key URL

Step 5: Document the cutover; close the change ticket
```

---

#### What happens if it goes wrong mid-switch?

- The protector switch is **validated at the API contract** — Azure checks HSM reachability and key state before accepting the change. If those checks fail, the operation is rejected and the MI keeps using the old MS-managed key. **No data loss, no downtime.**
- ⚠️ **Do not interpret "validated up front" as "guaranteed safe afterwards."** If HSM access is intermittently flaky, or if RBAC propagation is still in-flight, the switch can succeed at T=0 and the database can become **Inaccessible** later. See the timing table below.
- If validation fails after the fact, you can switch the protector **back** to "Service-managed" the same way. The DEK was never replaced, so reverting just re-wraps it with the MS key again.

#### ⚠️ When SQL MI *can* see downtime — HSM access lost AFTER the protector switch

This is the only realistic downtime scenario for SQL MI in this migration. It is **caused by ongoing HSM connectivity loss**, not by the cutover itself.

| Failure type | Time to `Inaccessible` | Recovery if access restored < 30 min | Recovery if access restored ≥ 30 min |
|---|---|---|---|
| **5XX** (network / transient) | 24-hour buffer | Auto-heal within ~1 hour | **Manual portal recovery required.** Server-level settings can be lost: tags, elastic-pool membership, read scale, auto-pause, PITR history, LTR policy. |
| **4XX** (auth / RBAC / disabled key) | 30 minutes | Auto-heal within ~1 hour | **Manual portal recovery required.** Same server-level settings loss as above. |
| Generic unreachable | Up to 10 minutes before connections start being denied | — | — |

**Required:** set Azure Resource Health alerts and Activity Log alerts on the SQL MI so protector-access loss is detected within 10 minutes.

**⚠️ Audit-defensibility note:** Microsoft documentation explicitly advises validating **all** server- and database-level settings on the recovered database after the 30-minute window has expired. Build a runbook that captures a complete settings snapshot **before** the protector switch, and validate against that snapshot during recovery.

---

#### Other SQL MI nuances worth knowing

- **Auto-rotation pickup:** When automated key rotation is enabled, the TDE protector picks up new key versions **within 24 hours** of detection. Plan rotation cycles around this window.
- **No version pinning:** SQL MI's TDE protector configuration **always resolves to the latest enabled version** of the key. Operators **cannot** pin SQL MI to an older key version. If a rollback is required, the only option is to introduce a **new** key (or a new key version) and switch the protector to it.
- **Long-term retention (LTR) backups:** Existing pre-migration LTR backups remain encrypted with whatever protector was active when they were taken (Microsoft retains the old MS key automatically). **Post-migration backups, however, are pinned to the CMK version that was active when they were taken.** Every CMK version must be retained for the **longest retention horizon** (PITR + LTR — potentially years) or those backups become unrestorable.
- **Point-in-time restore (PITR):** The backup chain spans the protector switch transparently.

---

### Section 5: Migrating EXISTING VM Disks (Phase 1b)

> **Will this cause downtime? → YES — and you need to know exactly when, and exactly why.**
>
> **Why downtime is required (and what it is *not*):** Switching the encryption set on a VM disk is **not** because the data has to be re-encrypted. The DEK that already encrypts the disk is **not replaced** — Azure just re-wraps that same DEK with the new CMK. **No data is rewritten.**
>
> The downtime exists because the **Azure platform** requires the VM to be **stopped and deallocated** before the disk's encryption-set association can be changed. This is a platform constraint on the disk-management API, not a property of encryption itself.
>
> Translation for the team: **"stop the VM, change the disk's pointer, start the VM"** — typically **15-30 minutes per VM** of actual outage, plus whatever validation time you need.

```
BEFORE migration:
Disk data ← DEK ← Platform-Managed Key (Microsoft-owned)

AFTER migration:
Disk data ← (same DEK) ← DES (new) ← Contoso CMK (in Contoso HSM)

Same as SQL MI: the underlying DEK doesn't change.
What changes is what wraps it (and a brand-new DES is inserted as the bridge).
```

---

#### Pick the scenario that matches each VM — this drives the downtime, the steps, and the risk

You will likely have a mix of all four scenarios in the existing fleet. **Run the eligibility check (below) against every disk in scope first**, then sort each VM into one of these scenarios.

| Scenario | What it looks like | Downtime | Effort | Risk |
|---|---|---|---|---|
| **A** — Managed disks, never had ADE | The standard modern VM | ~15-30 min per VM (stop / update DES / start) | Low | 🟢 |
| **B** — Managed disks, ADE-ever (current OR historical) | UDE flag persists on the disk | **Hours to days per VM** (full rebuild required) | High | 🔴 |
| **C** — Unmanaged disks (page-blob VHDs) | Older VMs not yet converted | **Two windows**: convert window + Scenario A window | Medium | 🟡 |
| **D** — AD domain controllers (cuts across A/B/C) | Any VM running the DC role | Same as A/B/C **but** must be done one DC at a time | — | 🔴 (because of fleet impact) |

A given VM can be in **C → A** (convert first, then standard cutover) or **C → B** (convert first, then rebuild). It cannot bypass the eligibility gate.

---

#### Scenario A — Managed disks, never had ADE  (most common, simplest)

**Downtime per VM:** ~15-30 minutes — stop / deallocate, swap the DES association on each disk, start, validate.

**Why this is the easy path:** the disk already has a managed DEK created by Azure on day one; SSE+CMK with a DES is a directly supported in-place operation; no data is moved or rewritten.

**One-time prep (do once for the whole estate):**

```
Step A: Create the DES in Central India
          - Type: EncryptionAtRestWithCustomerKey
          - Pointed at Contoso-DISK-CMK
          - System-Assigned Managed Identity enabled

Step B: Grant the DES identity the
        "Managed HSM Crypto Service Encryption User" role,
        scoped to /keys/Contoso-DISK-CMK
```

**Per-VM cutover:**

```
Step 1: Stop and deallocate the VM
          az vm deallocate -g <rg> -n <vm>
        ── Downtime starts here ──

Step 2: Update each disk attached to the VM (OS + every data disk)
        to reference the new DES:
          az disk update --encryption-type EncryptionAtRestWithCustomerKey
                         --disk-encryption-set <DES-id>
        (Azure re-wraps the existing DEK using the CMK. Fast.)

Step 3: Start the VM
          az vm start -g <rg> -n <vm>
        ── Downtime ends when the VM is fully booted and the
           workload is healthy ──

Step 4: Validate:
          • VM boots
          • All workload services start
          • For DC VMs: dcdiag /v + repadmin /replsummary clean
          • For SQL on VM: services up, DBs online
```

**Rollback:** Update the disk(s) back to `EncryptionAtRestWithPlatformKey`. The original DEK is re-wrapped with the platform key. **One critical caveat:** rolling back to PMK is supported only for disks **without incremental snapshots taken since the CMK was enabled**. If incremental snapshots exist, the disk and its snapshots are pinned to CMK and the only way back is to copy the data to a brand-new managed disk that does not use CMK — a multi-hour operation per disk.

---

#### Scenario B — Managed disks that have ever had ADE  (rebuild required)

**Downtime: substantial — hours to days per VM, depending on the OS and the data size.** This is not an in-place cutover.

**Why a rebuild is required:** Azure Disk Encryption (ADE) sets a Unified Data Encryption (UDE) flag on the disk. **The UDE flag persists even after ADE is disabled, and snapshots / copies inherit it.** ADE-ever disks **cannot** be encrypted with SSE+CMK in place.

**ADE itself is being retired on 15-Sep-2028;** encryption-at-host with CMK is the strategic replacement, and **encryption-at-host cannot be enabled on any VM that ever had ADE**. So the rebuild path is unavoidable for these VMs regardless of timing.

**OS-specific extra constraint — Linux:** in-place ADE decryption on a **Linux OS disk** is **not supported**. The OS disk must be replaced as part of the rebuild. (Windows OS disks can be ADE-decrypted in place.)

**The rebuild path:**

```
Step 1: Stop the VM.
        On Windows: disable ADE on the OS + data disks.
        On Linux:   skip ADE-disable on the OS disk (not supported);
                    plan to replace the OS disk entirely.
        ── Downtime starts ──

Step 2: Export the (decrypted) disk's VHD blob.

Step 3: Create a new managed disk via the "Upload" method
        — this is the only operation that does NOT propagate
        the UDE flag.

Step 4: Attach the new disk(s) to a new VM with the target DES
        already associated.

Step 5: Cut over the workload to the new VM
        (DNS / load balancer / connection strings / AD service principal).
        ── Downtime ends ──

Step 6: Decommission the old VM only after the new one is validated.
```

**Treat each Scenario B VM as its own project**, not as part of a batch cutover.

---

#### Scenario C — Unmanaged disks (page-blob VHDs)  (convert first, then go to A or B)

**Downtime: two separate windows** — first to convert to managed disks, then the relevant Scenario A or B window.

**Why a separate conversion window:** Unmanaged disks were **fully retired on 31-Mar-2026.** Any VM still on unmanaged disks today is in stop-ship territory regardless of CMK plans. The conversion is unrelated to CMK — it's a prerequisite that must happen first.

```
Step 1: Stop and deallocate the VM
        ── Downtime window 1 starts ──

Step 2: Convert to managed disks
          az vm convert -g <rg> -n <vm>

Step 3: Start the VM, validate the workload
        ── Downtime window 1 ends ──

(Now the VM is in Scenario A — or, if it ever had ADE, Scenario B.)

Step 4: Schedule a separate window for the actual CMK cutover.
```

Conversion windows and CMK windows are deliberately kept separate so a failure in one doesn't bleed into the other.

---

#### Scenario D — AD domain controllers  (cuts across all scenarios above)

A DC is just a VM, so it falls into A, B, or C above. **The extra rule for DCs is operational, not technical:**

- **Migrate one DC at a time.** Validate it's fully back (`dcdiag /v`, `repadmin /replsummary` clean, test authentication from a client) **before** starting the next DC.
- Confirm at least one **other** DC in the same site is healthy and authenticating before each cutover begins.
- Confirm AD replication is healthy before you start the very first DC (`repadmin /replsummary`). Don't migrate on top of an existing replication issue.
- Never schedule all DCs in parallel. The blast radius of getting it wrong is the entire identity plane.

---

#### Eligibility gate — must be run for every disk BEFORE any window is scheduled

This is the **single most-missed step**. Without it, the team will discover an ADE-ever or unmanaged disk **mid-cutover**, with the VM already stopped, and have to abort.

**Check 1 — ADE history (Scenario B detector):**

```bash
az disk show -g <rg> -n <disk> \
  --query "{name:name, udePresent: encryptionSettingsCollection != null}" -o table
```

Any disk where `udePresent == true` is **Scenario B** (rebuild path), not Scenario A.

**Check 2 — managed vs unmanaged (Scenario C detector):**

```bash
az vm show -g <rg> -n <vm> --query "storageProfile" -o json
```

Any disk that does not have a `managedDisk` block is unmanaged → **Scenario C** (convert first).

**Check 3 — RSA key size (CMK side):** `Contoso-DISK-CMK` can be RSA 2,048 / 3,072 / 4,096 — disks support all three. (This contrasts with SQL MI, which rejects 4,096.)

**Check 4 — no ADE + SSE+CMK "double encryption" plans:** Azure's "double encryption at rest" feature stacks **PMK + CMK at the SSE layer** — it does **not** combine ADE with SSE+CMK. Don't propose ADE + SSE+CMK as a defense-in-depth design; it isn't supported.

**Check 5 — subscription-move constraint:** Disks, snapshots, and images encrypted with CMK **cannot be moved between subscriptions**. They can be moved between resource groups within the same subscription only when the VM is deallocated. If landing-zone consolidation is planned, **do it before** CMK enablement, not after.

---

#### ⚠️ Ops note — Tier-0 key-state changes (whole-fleet blast radius)

When the disk CMK is **disabled, deleted, or expired**, **any** VM with an OS or data disk using that key will automatically shut down, and disk I/O begins failing approximately **one hour** after the key state change.

In this single-region design, **one accidental "Disable" toggle on `Contoso-DISK-CMK` will shut down every VM that has been migrated to CMK — including every domain controller — within ~1 hour.** There is no in-region failover that survives this; every CMK-using VM in scope shares this single key.

**Operational rules:**

- Treat any state change to `Contoso-DISK-CMK` as a **Tier-0**, peer-reviewed, change-ticketed operation
- Restrict `Crypto Officer` and `Managed HSM Administrator` role assignments on this key to a small named group
- Set Activity Log alerts on disable / delete / expiration events for this specific key
- **Never** test key-state changes against the production disk CMK — use a separate test-only key in the same HSM if validation is needed

---

### Section 6: Network (Private Endpoint)

One Private Endpoint in Central India, in the workload VNet, exposing the HSM. Existing-workload nuances:

- **Existing SQL MIs and VMs are already in subnets.** Confirm those subnets have a route to the HSM Private Endpoint (private DNS zone link, NSG rules, UDR / firewall hops). Many existing environments have legacy NSGs that need updating.
- **Test the path from the actual workload subnet** before the migration window — not from a jump box in a different subnet.
- **Trusted-services bypass — what it actually does:** Managed HSM's *“Allow trusted Microsoft services to bypass the firewall”* setting controls whether **Azure platform services** (SQL MI, DES) can reach the HSM when the HSM firewall is in *Selected networks* mode. This is **independent** of whether Contoso's VNet reaches HSM via Private Endpoint — Contoso's PE governs only **Contoso-VNet → HSM** traffic, not **platform-service → HSM** traffic.
  - **For SQL MI specifically:** SQL MI is a managed service; its TDE-protector traffic to HSM **does not** route through Contoso's VNet PE. SQL MI relies on either (a) the HSM firewall being open to the SQL MI's outbound IPs, or (b) trusted Microsoft services bypass enabled on the HSM firewall.
  - **For DES:** Same architecture — DES's wrap/unwrap calls go via the Microsoft backbone, not Contoso's VNet.
  - ⚠️ Disabling public access on Managed HSM **without** one of these two paths in place will break SQL MI TDE configuration and break post-migration TDE-protector access — **even if Contoso's VNet has PE.** This is the most common post-migration outage cause.

---

### Section 7: RBAC

- For each in-scope SQL MI, assign the SQL MI's managed identity (UAMI preferred) the `Managed HSM Crypto Service Encryption User` role, scoped to `/keys/Contoso-SQLMI-CMK`.
- For the DES, assign its system-assigned managed identity the same role, scoped to `/keys/Contoso-DISK-CMK`.
- Test the RBAC by attempting a wrap/unwrap from the workload subnet **before** the protector switch.
- Remember the role-name trap: **Crypto User** = key management; **Crypto Officer** does NOT manage keys.

---

### Section 8: Security Domain

Identical to the fresh-deployment guide. The Security Domain is about the HSM itself — it doesn't care whether the workloads are new or existing. Three RSA keys held by three named officers, 2-of-3 quorum to recover. Lives offline.

---

### Section 9: Disaster Recovery

> **Out of scope for this single-region guide.**
>
> This document is intentionally focused on Central India only so the team can build a clean mental model of the protector swap, the DEK behavior, and the downtime characteristics. Multi-region replication, geo-replicas, failover groups, ASR, and HSM-side DR are addressed in the **multi-region** version of this guide.
>
> If the team encounters any of the following during inventory — **flag and pause**, do not proceed with this guide:
> - SQL MI in a failover group
> - VM with Azure Site Recovery (ASR) replication enabled
> - Any cross-region dependency on the workload being migrated

---

### Section 10: Key Rotation

Managed HSM supports **automatic key rotation** on a schedule, plus on-demand rotation.

**Existing-workload nuance:**

- For SQL MIs that were already running before migration, the first rotation after migration is the first time the new key version is exercised end-to-end. **Treat the first post-migration rotation as a planned exercise, not a routine activity** — schedule it, validate it, and only after that move to set-and-forget.

**⚠️ Ops note — rotation pickup latency:**

| Service | Pickup latency for new key version | VM / DB reboot? |
|---|---|---|
| SQL MI (auto-rotation) | within **24 hours** of detection | None |
| Managed disks via DES (auto-rotation) | within **1 hour** — DES updates all referenced disks, snapshots, and images | None — VMs are **not** rebooted during automatic key rotation |

**⚠️ Mandatory retention rule:** Keep **all previous CMK versions** for at least the longest backup retention horizon (PITR + LTR). Newer rotations always use the latest version, but historical backups are pinned to the version that was active at backup time. For a SQL MI with a 7-year LTR policy, this means every CMK version generated during those 7 years must be retained — a multi-year retention policy is a multi-year key-management commitment.

---

### Section 11: Resource Inventory (Single Region)

**Created during Phase 0 (one-time):**

- 1 Managed HSM resource (`Contoso-PROD-HSM` in Central India)
- 2 CMK keys (`Contoso-SQLMI-CMK`, `Contoso-DISK-CMK`)
- 1 Private Endpoint + 1 Private DNS Zone in the workload VNet
- RBAC role assignments

**Created during Phase 1b (one-time, before any VM cutover):**

- 1 Disk Encryption Set (DES) in Central India, pointed at `Contoso-DISK-CMK`. **Track its managed-identity `principalId`** in the inventory — it's the only reliable way to attribute wrap/unwrap activity in HSM audit logs during incident response.

**Already exists (from the existing workloads) — to be re-pointed, not re-created:**

- N existing SQL MIs (each gets its TDE protector switched in Phase 1a)
- N existing OS + data disks on VMs (each gets its encryption set switched in Phase 1b)

**Already exists (created by Azure automatically):**

- DEKs — one per existing resource, already encrypting data today

---

### Section 12: Pricing (Single Region)

| What | Monthly Cost |
|---|---|
| HSM pool (Central India) | ~$2,304 |
| Keys + operations | ~$15-20 |
| Private Endpoint | ~$10 |
| **Total** | **~$2,330 / month** |

Cost starts the moment the HSM is provisioned in Phase 0, **not** when workloads are migrated. There is a real incentive to keep the gap between “HSM ready” and “workloads migrated” as short as possible.

**One-time costs to budget for:**

- Engineering hours for per-VM cutover windows (DCs migrated serially)
- Rebuild-path effort for any Scenario B VMs (ADE-ever)
- Convert-path effort for any Scenario C VMs (unmanaged)

---


## Side-by-Side: SQL MI vs VM Disks (Single Region, In Scope)

| Aspect | Existing SQL MI | Existing VM Disks |
|---|---|---|
| **Downtime required** | ❌ No — online protector switch | ⚠️ **Yes** — VM stop/deallocate per VM (Azure platform rule, OS-agnostic) |
| **Re-encryption of data** | ❌ No (DEK reused, just re-wrapped) | ❌ No (DEK reused, just re-wrapped) |
| **Rebuild required for some?** | N/A | ⚠️ **Yes** for any disk that has ever had ADE (UDE flag persists), and for any unmanaged disk (retired 31-Mar-2026) |
| **New identity needed** | Managed Identity on the SQL MI (UAMI recommended) | New DES (created during migration) with its own system-assigned managed identity |
| **Cutover scope per call** | All databases on the MI flip together (protector is at MI level) | One disk at a time per `az disk update`; in practice all disks on a VM are updated in one window |
| **Rollback** | Switch protector back to "Service-managed" — instant | Switch disk back to `EncryptionAtRestWithPlatformKey` **only if disk has no incremental snapshots**; otherwise data must be copied to a new non-CMK disk |
| **Key-disable blast radius** | DB → `Inaccessible` in 30 min (4XX) / 24 h (5XX); manual recovery may lose server-level settings | **Every VM using the key shuts down within ~1 hour** — disabling `Contoso-DISK-CMK` takes down every CMK-using VM |
| **CMK key used** | `Contoso-SQLMI-CMK` | `Contoso-DISK-CMK` |
| **CMK key size constraint** | RSA 2,048 or 3,072 only (NOT 4,096) | RSA 2,048 / 3,072 / 4,096 all supported |
| **RBAC role needed** | `Managed HSM Crypto Service Encryption User` | `Managed HSM Crypto Service Encryption User` |
| **Risk level** | 🟡 Medium (throttling caps + post-cutover Inaccessible timers) | 🔴 **High** (downtime per VM + ADE-ever eligibility + key-disable shuts down all CMK-using VMs in ~1 hour + no-snapshot rollback constraint) |
| **Recommended order** | Phase 1a — do all in-scope MIs first | Phase 1b — eligibility gate first, then one VM at a time (one DC at a time for any DC VMs) |



---

> *"We're not turning encryption on for the first time — Contoso's data in Central India is already encrypted today. We're moving the **ownership of the keys** from Microsoft to Contoso. The actual data on disk doesn't change. The data-encryption keys don't change. What changes is which key wraps them, and who controls that wrapping key.*
>
> *For SQL MI, the cutover is online — no downtime. For VMs, each one needs a brief stop/start window — typically 15-30 minutes — done one VM at a time (one DC at a time for any domain controllers) so the workload stays up. ADE-ever VMs and unmanaged-disk VMs are the two stop-ship cases that need a different path entirely.*
>
> *The whole estate moves to one Contoso-owned HSM in Central India that costs ~$2,330/month and is the foundation for PCI-DSS 4.0 customer-managed-key compliance."*
