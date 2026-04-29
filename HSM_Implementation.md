# Azure Key Vault Managed HSM — Production Deployment Architecture & Flow for Contoso

---

## Document Purpose

This document provides Contoso with:
1. End-to-end production deployment architecture
2. Visual diagrams for the two-region (South India + Central India) setup
3. Detailed flows for Phase 1 (SQL MI + AD Servers) and Phase 2 (Remaining Services)
4. Security and key management flow
5. Operational procedures (backup, rotation, DR)

---

## Terminology

| Abbreviation | Meaning |
|---|---|
| **South India** | Azure region — Contoso's primary production region (Chennai) |
| **Central India** | Azure region — Contoso's disaster recovery region (Pune) |
| **Extended HSM** | Multi-region replication feature — a single Managed HSM resource with presence in two regions (not two separate HSMs) |

> **Note on regions:** This document uses the full Azure region names "South India" and "Central India" throughout. If Contoso uses "SI" and "CI" internally to refer to organizational domains (e.g., business units), these are distinct from the Azure region references.

---

## Assumptions (Pending Contoso Confirmation)

| # | Assumption | Status |
|---|---|---|
| 1 | South India = Primary Production, Central India = Disaster Recovery | ⚠️ Needs confirmation |
| 2 | AD Servers = Self-hosted Domain Controllers on Azure VMs | ⚠️ Needs confirmation |
| 3 | Private Endpoint access (no public endpoint) | Assumed for production |
| 4 | Key rotation cadence = Annual | ⚠️ Needs confirmation |
| 5 | Non-production environment available for pilot | ⚠️ Needs confirmation |

---

## 1. High-Level Architecture — Two-Region Deployment

With **multi-region replication enabled**, the Managed HSM is a **single logical resource** with presence in both regions. Keys, roles, and permissions created in the primary region **automatically replicate** to the extended region. Contoso does not create separate keys or RBAC assignments per region.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                           CONTOSO AZURE ENVIRONMENT                              │
│                                                                                  │
│  ┌──────────────────────────────────┐    ┌──────────────────────────────────┐    │
│  │  SOUTH INDIA — Chennai (Primary) │    │  CENTRAL INDIA — Pune (DR)       │    │
│  │                                  │    │                                  │    │
│  │  ┌────────────────────────────┐  │    │  ┌────────────────────────────┐  │    │
│  │  │ Azure Key Vault Managed HSM│  │    │  │ Extended Region (replica)  │  │    │
│  │  │ Contoso-PROD-HSM           │  │    │  │ Contoso-PROD-HSM           │  │    │
│  │  │                            │  │    │  │                            │  │    │
│  │  │ ┌─────┐ ┌─────┐ ┌─────┐    │  │    │  │ ┌─────┐ ┌─────┐ ┌─────┐    │  │    │
│  │  │ │HSM 1│ │HSM 2│ │HSM 3│    │  │    │  │ │HSM 1│ │HSM 2│ │HSM 3│    │  │    │
│  │  │ │Part.│ │Part.│ │Part.│    │  │    │  │ │Part.│ │Part.│ │Part.│    │  │    │
│  │  │ └─────┘ └─────┘ └─────┘    │  │    │  │ └─────┘ └─────┘ └─────┘    │  │    │
│  │  │ (3-partition HA cluster)*  │  │    │  │ (3-partition HA cluster)   │  │    │
│  │  │                            │  │    │  │                            │  │    │
│  │  │ Keys (source of truth):    │  │    │  │ Keys (auto-replicated):    │  │    │
│  │  │ • Contoso-SQLMI-CMK        │──┼────┼─►│ • Contoso-SQLMI-CMK        │  │    │
│  │  │ • Contoso-DISK-CMK         │──┼────┼─►│ • Contoso-DISK-CMK         │  │    │
│  │  │ • Contoso-STORAGE-CMK      │──┼────┼─►│ • Contoso-STORAGE-CMK      │  │    │
│  │  │                            │  │    │  │                            │  │    │
│  │  │ RBAC (auto-replicated):    │──┼────┼─►│ RBAC (auto-replicated)     │  │    │
│  │  └─────────────┬──────────────┘  │    │  └─────────────┬──────────────┘  │    │
│  │                │ Private Endp    │    │                │ Private Endp    │    │
│  │                │                 │    │                │                 │    │
│  │  ┌─────────────▼──────────────┐  │    │  ┌─────────────▼──────────────┐  │    │
│  │  │     Contoso VNet           │  │    │  │     Contoso VNet           │  │    │
│  │  │     (South India)          │  │    │  │     (Central India)        │  │    │
│  │  │                            │  │    │  │                            │  │    │
│  │  │ ┌─────────┐ ┌───────────┐  │  │    │  │ ┌─────────┐ ┌───────────┐  │  │    │
│  │  │ │ SQL MI  │ │ AD Server │  │  │    │  │ │ SQL MI  │ │ AD Server │  │  │    │
│  │  │ │(Primary)│ │  VM (DC)  │  │  │    │  │ │  (DR)   │ │  VM (DC)  │  │  │    │
│  │  │ └─────────┘ └───────────┘  │  │    │  │ └─────────┘ └───────────┘  │  │    │
│  │  │                            │  │    │  │                            │  │    │
│  │  │ ┌─────────┐ ┌───────────┐  │  │    │  │ ┌─────────┐ ┌───────────┐  │  │    │
│  │  │ │ Storage │ │ Other VMs │  │  │    │  │ │ Storage │ │ Other VMs │  │  │    │
│  │  │ │Accounts │ │ (Phase 2) │  │  │    │  │ │Accounts │ │ (Phase 2) │  │  │    │
│  │  │ └─────────┘ └───────────┘  │  │    │  │ └─────────┘ └───────────┘  │  │    │
│  │  └────────────────────────────┘  │    │  └────────────────────────────┘  │    │
│  └──────────────────────────────────┘    └──────────────────────────────────┘    │
│                                                                                  │
│        ◄──── Multi-region replication (keys, RBAC, policies sync auto) ────►     │
│                                                                                  │
│  ONE HSM resource — Contoso-PROD-HSM — with two regional presences               │
│                                                                                  │
│  * South India does not currently offer Availability Zones. Partitions are       │
│    distributed across separate fault domains and racks for high availability.    │
│    Central India (Pune) supports Availability Zones.                             │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Key architectural points:**
- **One Managed HSM resource** (`Contoso-PROD-HSM`) with multi-region replication to Central India
- **One set of keys** — keys are created once in the primary region and automatically replicated
- **One set of RBAC assignments** — roles and permissions replicate automatically
- Contoso does **not** create or manage separate keys or RBAC per region

---

## 2. Managed HSM Internal Architecture (What Microsoft Provisions)

```
┌─────────────────────────────────────────────────────────────────────┐
│  AZURE DATACENTER — South India / Chennai (Primary Region)          │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  Confidential Compute Infrastructure (TEE)                     │ │
│  │                                                                │ │
│  │  ┌───────────────────────────────────────────────────────────┐ │ │
│  │  │  Managed HSM Pool: Contoso-PROD-HSM                       │ │ │
│  │  │                                                           │ │ │
│  │  │   Fault Domain 1    Fault Domain 2    Fault Domain 3      │ │ │
│  │  │   (Rack 1)          (Rack 2)          (Rack 3)            │ │ │
│  │  │   ┌─────────┐        ┌─────────┐        ┌─────────┐       │ │ │
│  │  │   │Marvell  │        │Marvell  │        │Marvell  │       │ │ │
│  │  │   │Liquid   │◄──────►│Liquid   │◄──────►│Liquid   │       │ │ │
│  │  │   │Security │        │Security │        │Security │       │ │ │
│  │  │   │HSM      │        │HSM      │        │HSM      │       │ │ │
│  │  │   │Partition│        │Partition│        │Partition│       │ │ │
│  │  │   └─────────┘        └─────────┘        └─────────┘       │ │ │
│  │  │       ▲                  ▲                  ▲             │ │ │
│  │  │       │     Automatic Sync & Failover       │             │ │ │
│  │  │       └──────────────────┼──────────────────┘             │ │ │
│  │  │                          │                                │ │ │
│  │  │              FIPS 140-3 Level 3                           │ │ │
│  │  │              Single-Tenant Isolation                      │ │ │
│  │  │              Keys NEVER leave HSM boundary                │ │ │
│  │  └──────────────────────────┼────────────────────────────────┘ │ │
│  │                             │                                  │ │
│  │  Microsoft CANNOT access ───┤                                  │ │
│  │  key material at this layer │                                  │ │
│  └─────────────────────────────┼──────────────────────────────────┘ │
│                                │                                    │
│                     ┌──────────▼──────────┐                         │
│                     │  HTTPS Endpoint     │                         │
│                     │  *.managedhsm.      │                         │
│                     │   azure.net         │                         │
│                     └──────────┬──────────┘                         │
│                                │                                    │
└────────────────────────────────┼────────────────────────────────────┘
                                 │
                      Private Endpoint
                                 │
                     ┌───────────▼────────────┐
                     │  Contoso VNet           │
                     │  (South India)          │
                     └────────────────────────┘
```

> **Note on partition distribution:** Managed HSM distributes its three partitions across servers deployed in different racks and load-balanced for high availability. In regions that support Availability Zones (e.g., Central India / Pune), partitions are spread across AZs. In regions without AZ support (e.g., South India / Chennai), partitions are distributed across **separate fault domains and racks**, which still provides intra-region resilience through a different mechanism. The HA guarantee is the same — automatic failover if any single partition becomes unavailable.

**What Microsoft provisions (invisible to Contoso):**
- Physical Marvell LiquidSecurity HSM adapters
- 3-partition cluster spread across fault domains / racks (or Availability Zones where available)
- Confidential Compute envelope (Trusted Execution Environment)
- Automatic partition synchronization and failover
- HTTPS endpoint for API access

**What Contoso sees:**
- A single logical HSM endpoint
- Keys and RBAC policies
- Private Endpoint in their VNet

---

## 3. Deployment Flow — End-to-End Process

```
┌──────────────────────────────────────────────────────────────────────┐
│                     DEPLOYMENT FLOW                                  │
└──────────────────────────────────────────────────────────────────────┘

    ┌─────────┐         ┌──────────────┐         ┌──────────────┐
    │  START  │────────►│  Phase 0:    │────────►│  Phase 0:    │
    │         │         │  Provision   │         │  Activate    │
    └─────────┘         │  HSM Pool    │         │  Security    │
                        │              │         │  Domain      │
                        │  • CLI/Portal│         │              │
                        │  • ~20-30 min│         │  • 3 RSA keys│
                        │              │         │  • Download  │
                        │  Microsoft   │         │    SD file   │
                        │  allocates   │         │  • Store     │
                        │  hardware    │         │    OFFLINE   │
                        │  (auto)      │         │  • ~15 min   │
                        │              │         │              │
                        │    MUST      │         │              │
                        │  enable:     │         │              │
                        │  • soft-     │         │              │
                        │    delete    │         │              │
                        │  • purge     │         │              │
                        │    protection│         │              │
                        └──────────────┘         └──────┬───────┘
                                                        │
                                                        ▼
    ┌──────────────┐                             ┌──────────────┐
    │  Phase 0:    │◄────────────────────────────│  Phase 0:    │
    │  Enable      │                             │  Configure   │
    │  Multi-Region│                             │  RBAC        │
    │  Replication │                             │              │
    │              │                             │  • HSM Admin │
    │  • Add       │                             │  • Crypto    │
    │    Central   │                             │    User      │
    │    India as  │                             │  • Crypto    │
    │    extended  │                             │    Service   │
    │    region    │                             │    Encryption│
    │              │                             │    User      │
    └──────┬───────┘                             └──────────────┘
           │
           ▼
    ┌──────────────┐       ┌──────────────┐
    │  Phase 0:    │◄──────│  Phase 0:    │
    │  Create Keys │       │  Private     │
    │              │       │  Endpoints   │
    │  • SQLMI-CMK │       │              │
    │  • DISK-CMK  │       │  • South     │
    │  • STORAGE-  │       │    India PE  │
    │    CMK       │       │  • Central   │
    │              │       │    India PE  │
    │  (one set —  │       │  • DNS config│
    │   auto-      │       │    both      │
    │   replicated)│       │    regions   │
    └──────┬───────┘       └──────────────┘
           │
           ▼
    ┌──────────────┐
    │  Phase 1a:   │
    │  SQL MI CMK  │
    │              │
    │  • Enable MI │
    │    identity  │
    │  • Grant RBAC│
    │  • Set TDE   │
    │    protector │
    │  • Validate  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Phase 1b:   │
    │  AD Server   │
    │  VM Disk CMK │
    │              │
    │  • Create DES│
    │    (BOTH     │
    │    regions)  │
    │  • Grant RBAC│
    │  • Encrypt   │
    │    OS + Data │
    │    disks     │
    │  • Validate  │
    │    AD health │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Validation  │
    │  Soak Period │
    │  (2-4 weeks) │
    │              │
    │  • Monitor   │
    │  • Test DR   │
    │  • Test key  │
    │    rotation  │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │  Phase 2:    │
    │  Remaining   │
    │  Services    │
    │              │
    │  • Storage   │
    │  • Cosmos DB │
    │  • Other VMs │
    │  • AKS       │
    │  (rolling)   │
    └──────┬───────┘
           │
           ▼
    ┌─────────┐
    │  DONE   │
    └─────────┘
```

### Phase 0: HSM Provisioning — CLI Command

```bash
# Provision Managed HSM with soft-delete and purge protection ENABLED
az keyvault create \
  --hsm-name Contoso-PROD-HSM \
  --resource-group Contoso-Prod-RG \
  --location southindia \
  --administrators <admin-object-id-1> <admin-object-id-2> \
  --sku Standard_B1 \
  --retention-days 90 \
  --enable-purge-protection true
```

> ⚠️ **Purge protection is mandatory and irreversible:**
> - **SQL MI TDE will reject** a key from a Managed HSM that does not have both soft-delete and purge protection enabled. Phase 1a will fail at the `az sql mi tde-key set` step without this.
> - **PCI-DSS compliance** requires demonstrable key lifecycle controls — purge protection ensures deleted keys/HSMs cannot be permanently destroyed before the retention period expires.
> - **Billing impact:** If the HSM resource is deleted, it remains in a soft-deleted state for the entire retention period (7–90 days, set at creation via `--retention-days`). Purge protection blocks early purging, meaning **the HSM continues to be billed for the full retention period** even after deletion. This is by design and cannot be changed after enablement.
> - Soft-delete is enabled by default on Managed HSM. Purge protection **must be explicitly enabled** via `--enable-purge-protection true`.

---

## 4. Phase 1a: SQL Managed Instance — CMK Integration Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                 SQL MI ←→ Managed HSM Integration                   │
└─────────────────────────────────────────────────────────────────────┘

┌───────────────────┐                     ┌───────────────────────────┐
│                   │                     │                           │
│  SQL Managed      │                     │  Managed HSM Pool         │
│  Instance         │                     │  (Contoso-PROD-HSM)       │
│                   │                     │                           │
│  ┌─────────────┐  │     ② Grant RBAC    │  ┌───────────────────┐    │
│  │ System-     │──┼────────────────────►│  │ Role Assignment:  │    │
│  │ Assigned    │  │     (Crypto Service │  │ "Managed HSM      │    │
│  │ Managed     │  │      Encryption     │  │  Crypto Service   │    │
│  │ Identity    │  │      User)          │  │  Encryption User" │    │
│  └─────────────┘  │     scoped to       │  │  on /keys/        │    │
│                   │    /keys/Contoso-   │  │  Contoso-SQLMI-CMK│    │
│                   │    SQLMI-CMK        │  └───────────────────┘    │
│                   │                     │                           │
│  ┌─────────────┐  │    ④ Wrap/Unwrap    │  ┌───────────────────┐    │
│  │ TDE         │──┼────────────────────►│  │ Key:              │    │
│  │ Protector   │  │◄────────────────────┼──│ Contoso-SQLMI-CMK │    │
│  │ (points to  │  │    (encrypted DEK)  │  │ Type: RSA-HSM     │    │
│  │  HSM key)   │  │                     │  │ Size: 2048/3072   │    │
│  └─────────────┘  │                     │  └───────────────────┘    │
│                   │                     │                           │
│  ┌─────────────┐  │                     │  Prerequisites:           │
│  │ Database    │  │                     │   Soft-delete: enabled    │
│  │ Encryption  │  │                     │   Purge protection:       │
│  │ Key (DEK)   │  │                     │     enabled               │
│  │ [encrypted  │  │                     │  (SQL MI will reject TDE  │
│  │  by CMK]    │  │                     │   protector without both) │
│  └─────────────┘  │                     │                           │
│                   │                     │                           │
└───────────────────┘                     └───────────────────────────┘

FLOW:
① Enable System-Assigned Managed Identity on SQL MI
② Assign "Managed HSM Crypto Service Encryption User" role to MI identity
   → Scoped to /keys/Contoso-SQLMI-CMK (least-privilege)
③ Create key Contoso-SQLMI-CMK in Managed HSM (RSA-HSM, 2048-bit)
   → Key auto-replicates to Central India extended region
④ Set SQL MI TDE protector → points to HSM key URI
   → ⚠️ Will fail if purge protection is not enabled on the HSM
⑤ SQL MI generates a DEK (Data Encryption Key) → wraps it using HSM key
⑥ All database data encrypted with DEK; DEK protected by HSM key
⑦ On read: SQL MI calls HSM to unwrap DEK → decrypts data
```

### SQL MI — Step-by-Step Commands

```bash
# ① Enable Managed Identity on SQL MI
az sql mi update \
  --name contoso-sqlmi \
  --resource-group Contoso-Prod-RG \
  --assign-identity

# ② Get the MI's identity object ID
MI_IDENTITY=$(az sql mi show \
  --name contoso-sqlmi \
  --resource-group Contoso-Prod-RG \
  --query identity.principalId -o tsv)

# ③ Create the CMK key (created once — auto-replicates to extended region)
az keyvault key create \
  --hsm-name Contoso-PROD-HSM \
  --name Contoso-SQLMI-CMK \
  --kty RSA-HSM \
  --size 2048

# ④ Assign RBAC role in Managed HSM — scoped to specific key
az keyvault role assignment create \
  --hsm-name Contoso-PROD-HSM \
  --role "Managed HSM Crypto Service Encryption User" \
  --assignee $MI_IDENTITY \
  --scope /keys/Contoso-SQLMI-CMK

# ⑤ Get the key URI
KEY_URI=$(az keyvault key show \
  --hsm-name Contoso-PROD-HSM \
  --name Contoso-SQLMI-CMK \
  --query key.kid -o tsv)

# ⑥ Set TDE protector to CMK
#    ⚠️ This will FAIL if purge protection is not enabled on the HSM
az sql mi tde-key set \
  --server-key-type AzureKeyVault \
  --kid $KEY_URI \
  --managed-instance contoso-sqlmi \
  --resource-group Contoso-Prod-RG
```

---

## 5. Phase 1b: AD Server VMs — Disk Encryption with CMK Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│           AD Server VM ←→ Managed HSM Disk Encryption               │
└─────────────────────────────────────────────────────────────────────┘

┌───────────────────────┐       ┌────────────────┐      ┌──────────────────┐
│                       │       │                │      │                  │
│  AD Domain Controller │       │ Disk Encryption│      │  Managed HSM     │
│  VM                   │       │ Set (DES)      │      │  Pool            │
│                       │       │                │      │                  │
│  ┌─────────────────┐  │       │  ┌──────────┐  │ RBAC │  ┌────────────┐  │
│  │ OS Disk         │──┼──────►│  │ DES      │──┼─────►│  │ Key:       │  │
│  │ (encrypted)     │  │       │  │ Managed  │  │      │  │ Contoso-   │  │
│  └─────────────────┘  │       │  │ Identity │  │      │  │ DISK-CMK   │  │
│                       │       │  └──────────┘  │      │  │            │  │
│  ┌─────────────────┐  │       │                │      │  │ RSA-HSM    │  │
│  │ Data Disk       │──┼──────►│  Points to     │      │  │ 2048-bit   │  │
│  │ (AD DB/SYSVOL)  │  │       │  HSM Key ──────┼─────►│  │            │  │
│  │ (encrypted)     │  │       │                │      │  └────────────┘  │
│  └─────────────────┘  │       │                │      │                  │
│                       │       │                │      │                  │
└───────────────────────┘       └────────────────┘      └──────────────────┘

FLOW:
① Create key Contoso-DISK-CMK in Managed HSM (auto-replicates to Central India)
② Create Disk Encryption Set (DES) in South India, linked to HSM key
③ Create a SECOND DES in Central India, linked to same HSM key
   → Required for ASR-replicated VMs in DR region
④ DES gets a System-Assigned Managed Identity
⑤ Assign "Managed HSM Crypto Service Encryption User" role to DES identity
   → Scoped to /keys/Contoso-DISK-CMK (least-privilege)
⑥ Associate DES with AD VM's OS disk and Data disk(s)
⑦ Azure encrypts disk data using DEK → DEK wrapped by HSM key
⑧ On VM boot: Azure calls HSM to unwrap DEK → disk accessible
```

> ⚠️ **DES in both regions:** Unlike keys and RBAC (which auto-replicate), a **Disk Encryption Set is a regional Azure Compute resource**. If Contoso uses Azure Site Recovery (ASR) to replicate AD VMs to Central India, a DES must be **pre-created in Central India** referencing the same Managed HSM key. ASR requires a target-region DES to encrypt replica disks.

### AD Server Disk Encryption — Step-by-Step Commands

```bash
# ① Create the CMK key for disk encryption (created once — auto-replicates)
az keyvault key create \
  --hsm-name Contoso-PROD-HSM \
  --name Contoso-DISK-CMK \
  --kty RSA-HSM \
  --size 2048

# ② Get key URI
DISK_KEY_URI=$(az keyvault key show \
  --hsm-name Contoso-PROD-HSM \
  --name Contoso-DISK-CMK \
  --query key.kid -o tsv)

# ③ Create Disk Encryption Set — SOUTH INDIA (primary)
az disk-encryption-set create \
  --name Contoso-DES-SouthIndia \
  --resource-group Contoso-Prod-RG \
  --location southindia \
  --key-url $DISK_KEY_URI \
  --source-vault /subscriptions/<sub-id>/resourceGroups/Contoso-Prod-RG/providers/Microsoft.KeyVault/managedHSMs/Contoso-PROD-HSM \
  --encryption-type EncryptionAtRestWithCustomerKey

# ④ Create Disk Encryption Set — CENTRAL INDIA (for ASR/DR replicas)
az disk-encryption-set create \
  --name Contoso-DES-CentralIndia \
  --resource-group Contoso-DR-RG \
  --location centralindia \
  --key-url $DISK_KEY_URI \
  --source-vault /subscriptions/<sub-id>/resourceGroups/Contoso-Prod-RG/providers/Microsoft.KeyVault/managedHSMs/Contoso-PROD-HSM \
  --encryption-type EncryptionAtRestWithCustomerKey

# ⑤ Get DES identity (South India)
DES_IDENTITY=$(az disk-encryption-set show \
  --name Contoso-DES-SouthIndia \
  --resource-group Contoso-Prod-RG \
  --query identity.principalId -o tsv)

# ⑥ Grant DES access to Managed HSM key — scoped to specific key
az keyvault role assignment create \
  --hsm-name Contoso-PROD-HSM \
  --role "Managed HSM Crypto Service Encryption User" \
  --assignee $DES_IDENTITY \
  --scope /keys/Contoso-DISK-CMK

# ⑦ Repeat for Central India DES identity
DES_IDENTITY_CI=$(az disk-encryption-set show \
  --name Contoso-DES-CentralIndia \
  --resource-group Contoso-DR-RG \
  --query identity.principalId -o tsv)

az keyvault role assignment create \
  --hsm-name Contoso-PROD-HSM \
  --role "Managed HSM Crypto Service Encryption User" \
  --assignee $DES_IDENTITY_CI \
  --scope /keys/Contoso-DISK-CMK

# ⑧ Update AD VM OS disk to use DES (South India)
az disk update \
  --name <ad-vm-os-disk-name> \
  --resource-group Contoso-Prod-RG \
  --encryption-type EncryptionAtRestWithCustomerKey \
  --disk-encryption-set /subscriptions/<sub-id>/resourceGroups/Contoso-Prod-RG/providers/Microsoft.Compute/diskEncryptionSets/Contoso-DES-SouthIndia

# ⑨ Update AD VM Data disk(s) to use DES (South India)
az disk update \
  --name <ad-vm-data-disk-name> \
  --resource-group Contoso-Prod-RG \
  --encryption-type EncryptionAtRestWithCustomerKey \
  --disk-encryption-set /subscriptions/<sub-id>/resourceGroups/Contoso-Prod-RG/providers/Microsoft.Compute/diskEncryptionSets/Contoso-DES-SouthIndia
```

> ⚠️ **Important:** Changing disk encryption on an existing VM may require deallocating the VM. Plan a maintenance window for AD servers and ensure a secondary DC is available during the operation.

---

## 6. Network Architecture — Private Endpoint Flow

Private Endpoints are required in **both regions** to allow workloads to securely access the Managed HSM.

> ⚠️ **Operational constraint:** When public access is disabled on the Managed HSM, **all administrative operations** (key creation, RBAC changes, rotation) must also be performed from within the VNet — via a **jump box, Azure Bastion, or VPN-connected workstation**. Contoso's administrators will not be able to manage the HSM from the public internet.

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         NETWORK ARCHITECTURE (Both Regions)                      │
│                                                                                  │
│  SOUTH INDIA / Chennai (Primary)          CENTRAL INDIA / Pune (DR/Extended)     │
│  ┌────────────────────────────────┐      ┌────────────────────────────────┐      │
│  │ Contoso VNet (South India)     │      │ Contoso VNet (Central India)   │      │
│  │                                │      │                                │      │
│  │ ┌──────────────────────────┐   │      │ ┌──────────────────────────┐   │      │
│  │ │ Workload Subnet          │   │      │ │ Workload Subnet          │   │      │
│  │ │                          │   │      │ │                          │   │      │
│  │ │ ┌────────┐ ┌──────────┐  │   │      │ │ ┌────────┐ ┌──────────┐  │   │      │
│  │ │ │ SQL MI │ │ AD VMs   │  │   │      │ │ │ SQL MI │ │ AD VMs   │  │   │      │
│  │ │ └───┬────┘ └────┬─────┘  │   │      │ │ └───┬────┘ └────┬─────┘  │   │      │
│  │ │     │            │       │   │      │ │     │            │       │   │      │
│  │ └─────┼────────────┼───────┘   │      │ └─────┼────────────┼───────┘   │      │
│  │       │            │           │      │       │            │           │      │
│  │ ┌─────▼────────────▼───────┐   │      │ ┌─────▼────────────▼───────┐   │      │
│  │ │ Private Endpoint Subnet  │   │      │ │ Private Endpoint Subnet  │   │      │
│  │ │                          │   │      │ │                          │   │      │
│  │ │ PE: pe-contoso-hsm-si    │   │      │ │ PE: pe-contoso-hsm-ci    │   │      │
│  │ │ NIC IP: 10.1.x.x         │   │      │ │ NIC IP: 10.2.x.x         │   │      │
│  │ └──────────┬───────────────┘   │      │ └──────────┬───────────────┘   │      │
│  │            │                   │      │            │                   │      │
│  │ ┌──────────▼───────────────┐   │      │ ┌──────────▼───────────────┐   │      │
│  │ │ Private DNS Zone         │   │      │ │ Private DNS Zone         │   │      │
│  │ │ privatelink.managedhsm.  │   │      │ │ privatelink.managedhsm.  │   │      │
│  │ │ azure.net                │   │      │ │ azure.net                │   │      │
│  │ │ → 10.1.x.x               │   │      │ │ → 10.2.x.x               │   │      │
│  │ └──────────────────────────┘   │      │ └──────────────────────────┘   │      │
│  │                                │      │                                │      │
│  │ ┌──────────────────────────┐   │      │ ┌──────────────────────────┐   │      │
│  │ │ Management Subnet        │   │      │ │ Management Subnet        │   │      │
│  │ │                          │   │      │ │                          │   │      │
│  │ │ ┌────────────────────┐   │   │      │ │ ┌────────────────────┐   │   │      │
│  │ │ │ Jump Box / Bastion │   │   │      │ │ │ Jump Box / Bastion │   │   │      │
│  │ │ │ (HSM admin access) │   │   │      │ │ │ (HSM admin access) │   │   │      │
│  │ │ └────────────────────┘   │   │      │ │ └────────────────────┘   │   │      │
│  │ └──────────────────────────┘   │      │ └──────────────────────────┘   │      │
│  └────────────────────────────────┘      └────────────────────────────────┘      │
│                                                                                  │
│                     Both PEs → Same HSM: Contoso-PROD-HSM                        │
│                     Public access: DISABLED                                      │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘

SECURITY:
• All traffic to HSM stays within Azure backbone (no internet traversal)
• Public endpoint DISABLED — only Private Endpoint access
• NSG rules on PE subnets for additional control
• DNS resolution ensures workloads resolve HSM to private IP in their region
• Admin operations require VNet access (jump box / Azure Bastion / VPN)
```

---

## 7. Key Management & RBAC Architecture

> ⚠️ **Critical: Crypto Officer ≠ Key Management.** The Managed HSM role names are counterintuitive. Please review the role descriptions below carefully — they reflect Microsoft's official built-in role definitions.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     RBAC & KEY MANAGEMENT MODEL                     │
└─────────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────────┐
                    │    Microsoft Entra ID       │
                    │    (Azure AD)               │
                    │                             │
                    │  ┌───────────────────────┐  │
                    │  │ Contoso Security Team │  │
                    │  │ (Human Admins)        │  │
                    │  └───────────┬───────────┘  │
                    │              │              │
                    └──────────────┼──────────────┘
                                   │
                    ┌──────────────▼──────────────────────────────────┐
                    │          MANAGED HSM RBAC ROLES                 │
                    │                                                 │
                    │  ┌───────────────────────────────────────────┐  │
                    │  │  Managed HSM Administrator                │  │
                    │  │  → Contoso Security Team (2-3 people)     │  │
                    │  │  → Can: Manage role assignments and       │  │
                    │  │         role definitions, initial HSM     │  │
                    │  │         setup and Security Domain mgmt    │  │
                    │  │  → Cannot: Perform any key operations     │  │
                    │  └───────────────────────────────────────────┘  │
                    │                                                 │
                    │  ┌───────────────────────────────────────────┐  │
                    │  │  Managed HSM Crypto Officer               │  │
                    │  │  → Contoso Governance / Compliance Team   │  │
                    │  │  → Can: Manage role assignments,          │  │
                    │  │         purge or recover deleted keys,    │  │
                    │  │         export keys                       │  │
                    │  │  → Cannot: Create, write, rotate, delete, │  │
                    │  │         import keys or perform crypto     │  │
                    │  │         operations (encrypt, sign, wrap)  │  │
                    │  │      This is a GOVERNANCE role, not a     │  │
                    │  │     day-to-day key management role        │  │
                    │  └───────────────────────────────────────────┘  │
                    │                                                 │
                    │  ┌───────────────────────────────────────────┐  │
                    │  │  Managed HSM Crypto User                  │  │
                    │  │  → Contoso Key Management Team            │  │
                    │  │  → Can: Create, write, rotate, delete,    │  │
                    │  │         read, backup, restore, import,    │  │
                    │  │         release keys AND perform all      │  │
                    │  │      crypto operations (encrypt, decrypt, │  │
                    │  │         sign, verify, wrap, unwrap)       │  │
                    │  │  → Cannot: Purge/recover deleted keys     │  │
                    │  │      export keys, manage role definitions │  │
                    │  │     This is the KEY MANAGEMENT role —     │  │
                    │  │     despite the name "Crypto User"        │  │
                    │  └───────────────────────────────────────────┘  │
                    │                                                 │
                    │  ┌────────────────────────────────────────────┐ │
                    │  │  Managed HSM Crypto Service Encryption User│ │
                    │  │  → SQL MI Managed Identity                 │ │
                    │  │  → Disk Encryption Set Managed Identity    │ │
                    │  │  → Storage Account Managed Identity        │ │
                    │  │  → Can: Wrap/Unwrap/Get keys (CMK only)    │ │
                    │  │  → Scoped per-key for least-privilege      │ │
                    │  └────────────────────────────────────────────┘ │
                    │                                                 │
                    └─────────────────────────────────────────────────┘
```

### Role Mapping — Official vs. Common Misunderstanding

| Built-in Role | What People Assume | What It Actually Does |
|---|---|---|
| **Managed HSM Crypto Officer** | "Key administrator — creates and manages keys" | **Governance role** — manages RBAC, purges/recovers deleted keys, exports keys. **Cannot** create, rotate, or delete keys. |
| **Managed HSM Crypto User** | "Application user — only uses keys" | **Key lifecycle role** — creates, rotates, deletes, imports keys AND performs all crypto operations. **Cannot** purge deleted keys or manage roles. |

> **Source:** [Managed HSM built-in roles — Microsoft Learn](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/built-in-roles)

### Key Naming Convention (Unified — No Per-Region Suffixes)

| Key Name | Purpose | Used By | Replication |
|---|---|---|---|
| `Contoso-SQLMI-CMK` | SQL MI TDE encryption | SQL MI Managed Identity | Auto-replicated to Central India |
| `Contoso-DISK-CMK` | VM Disk encryption | Disk Encryption Set | Auto-replicated to Central India |
| `Contoso-STORAGE-CMK` | Storage account encryption (Phase 2) | Storage Account Identity | Auto-replicated to Central India |

> **Note:** Keys are created once in the primary region. Multi-region replication automatically synchronizes keys, roles, and permissions to the extended region. Contoso does not create separate keys per region.

---

## 8. Security Domain — Backup & Recovery Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│              SECURITY DOMAIN — CUSTODY & RECOVERY MODEL             │
└─────────────────────────────────────────────────────────────────────┘

                ACTIVATION (One-time setup)
                ─────────────────────────────

    Security Officer 1         Security Officer 2         Security Officer 3
    ┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
    │ Generate RSA     │       │ Generate RSA     │       │ Generate RSA     │
    │ Key Pair #1      │       │ Key Pair #2      │       │ Key Pair #3      │
    │                  │       │                  │       │                  │
    │ cert_0.cer ──────┼───┐   │ cert_1.cer ──────┼───┐   │ cert_2.cer ──────┼───┐
    │ cert_0.key       │   │   │ cert_1.key       │   │   │ cert_2.key       │   │
    └──────────────────┘   │   └──────────────────┘   │   └──────────────────┘   │
                           │                          │                          │
                           ▼                          ▼                          ▼
                           ┌──────────────────────────────────────────────────┐
                           │  az keyvault security-domain download            │
                           │    --hsm-name Contoso-PROD-HSM                   │
                           │    --sd-wrapping-keys cert_0.cer cert_1.cer      │
                           │                       cert_2.cer                 │
                           │    --sd-quorum 2                                 │
                           │    --security-domain-file Contoso-SD.json        │
                           └──────────────────────────┬───────────────────────┘
                                                      │
                                                      ▼
                    ┌──────────────────────────────────────────────────┐
                    │  Security Domain File: Contoso-SD.json           │
                    │  (Encrypted — requires ANY 2 of 3 private keys   │
                    │   to decrypt = quorum of 2)                      │
                    └──────────────────────────────────────────────────┘

                STORAGE (Offline — CRITICAL)
                ────────────────────────────

    ┌─────────────────────────────────────────────────────────────────┐
    │  MUST store separately and securely:                            │
    │                                                                 │
    │  • Contoso-SD.json    → Secure vault / safe deposit box         │
    │  • cert_0.key         → Officer 1's secure storage              │
    │  • cert_1.key         → Officer 2's secure storage              │
    │  • cert_2.key         → Officer 3's secure storage              │
    │                                                                 │
    │       If quorum (2 of 3) private keys + SD file are ALL lost,   │
    │     ALL KEYS IN THE HSM ARE PERMANENTLY UNRECOVERABLE.          │
    └─────────────────────────────────────────────────────────────────┘

                RECOVERY SCENARIO
                ─────────────────

    Regional disaster → Need to restore HSM in new region

    Officer 1 (cert_0.key) + Officer 2 (cert_1.key) + Contoso-SD.json
                           │
                           ▼
    ┌──────────────────────────────────────────────────────────┐
    │  az keyvault security-domain upload                      │
    │    --hsm-name Contoso-RECOVERY-HSM                       │
    │    --sd-file Contoso-SD.json                             │
    │    --sd-wrapping-keys cert_0.key cert_1.key              │
    │    --sd-quorum 2                                         │
    └──────────────────────────────────────────────────────────┘
                           │
                           ▼
    HSM restored with all keys intact in new region ✅
```

---

## 9. Disaster Recovery — Cross-Region Flow

With multi-region replication, the Managed HSM is a **single resource with presence in both regions**. This is not two separate HSMs — it is one HSM with an extended region.

```
┌─────────────────────────────────────────────────────────────────────┐
│  DISASTER RECOVERY — South India (Primary) → Central India (DR)     │
└─────────────────────────────────────────────────────────────────────┘

    NORMAL OPERATIONS
    ─────────────────

    South India / Chennai (Active)          Central India / Pune (Extended)
    ┌──────────────────────┐               ┌──────────────────────┐
    │ Contoso-PROD-HSM     │  Auto-Sync    │ Contoso-PROD-HSM     │
    │ (Primary region)     │──────────────►│ (Extended region)    │
    │                      │               │                      │
    │ Keys: Active         │  Keys, RBAC,  │ Keys: Replicated     │
    │ SQL MI: Active       │  policies     │ SQL MI: Standby      │
    │ AD VMs: Active       │  sync auto    │ AD VMs: Standby      │
    └──────────────────────┘               └──────────────────────┘

    ONE resource — two regional presences
    SLA: 99.99% with multi-region replication enabled


    FAILOVER SCENARIO (South India region outage)
    ──────────────────────────────────────────────

    South India (DOWN ❌)                   Central India / Pune (ACTIVE ✅)
    ┌──────────────────────┐               ┌──────────────────────┐
    │                      │               │ Contoso-PROD-HSM     │
    │   UNAVAILABLE        │               │ (Extended → Active)  │
    │                      │               │                      │
    │                      │               │ Keys: Available      │
    │                      │               │ SQL MI: Promoted     │
    │                      │               │ AD VMs: Active       │
    └──────────────────────┘               └──────────────────────┘

    FAILOVER STEPS:
    ① Detect South India region outage
    ② Central India workloads reference same HSM → keys available ✅
    ③ Promote Central India SQL MI to primary (auto-failover group or manual)
    ④ DNS/traffic switch to Central India resources
    ⑤ Validate all services operational

    KEY POINT: Because it's one HSM with two regional presences,
    failover does NOT require separate HSM provisioning, key creation,
    or RBAC configuration — everything is already there.
```

> ⚠️ **Soft-delete nuance for multi-region:** If the extended region (Central India) is **removed** from the Managed HSM, the HSM presence in that region is **purged immediately** — it is NOT soft-deleted. Keys in the primary region are unaffected, but Central India workloads will lose HSM access. This is different from deleting the entire HSM resource, which does follow the soft-delete retention period. With purge protection enabled, the HSM resource itself cannot be permanently destroyed before the retention period expires.

---

## 10. Key Rotation Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                     KEY ROTATION PROCEDURE                          │
└─────────────────────────────────────────────────────────────────────┘

    ┌───────────┐     ┌───────────────────┐     ┌──────────────────────┐
    │  Crypto   │────►│  Create new key   │────►│  Update service to   │
    │  User     │     │  version in HSM   │     │  use new key version │
    │  triggers │     │                   │     │                      │
    │  rotation │     │  Key name stays   │     │  • SQL MI TDE        │
    │           │     │  same; version    │     │    protector         │
    │  (annual/ │     │  increments       │     │  • DES key URL       │
    │  quarterly│     │                   │     │  • Storage key ref   │
    │  per      │     │  Old version      │     │                      │
    │  policy)  │     │  retained for     │     │  Old data re-encrypts│
    │           │     │  decrypt of       │     │  automatically with  │
    └───────────┘     │  existing data    │     │  new version (for    │
                      │                   │     │  most services)      │
                      │  New version auto-│     │                      │
                      │  replicates to    │     │                      │
                      │  Central India    │     │                      │
                      └───────────────────┘     └──────────────────────┘

    COMMAND:
    az keyvault key rotate \
      --hsm-name Contoso-PROD-HSM \
      --name Contoso-SQLMI-CMK

    AUTOMATIC ROTATION (optional):
    az keyvault key rotation-policy update \
      --hsm-name Contoso-PROD-HSM \
      --name Contoso-SQLMI-CMK \
      --value @rotation-policy.json

    rotation-policy.json:
    {
      "lifetimeActions": [{
        "trigger": { "timeAfterCreate": "P365D" },
        "action": { "type": "rotate" }
      }],
      "attributes": {
        "expiryTime": "P730D"
      }
    }
```

> ⚠️ **ASR Key Rotation Limitation:** If Contoso uses **Azure Site Recovery (ASR)** to replicate VMs that are encrypted with CMK, rotating the CMK key requires **disabling and re-enabling replication** on those VMs. ASR does not automatically pick up new key versions on protected VMs. This must be planned as part of any key rotation maintenance window for ASR-protected workloads.

---

## 11. Complete Resource Inventory

### Managed HSM (Single Resource — Two Regional Presences)

| Resource | Name | Region(s) | Notes |
|---|---|---|---|
| Managed HSM Pool | `Contoso-PROD-HSM` | South India (primary) + Central India (extended) | One resource with multi-region replication. Soft-delete and purge protection enabled. |

### Keys (Created Once — Auto-Replicated)

| Key Name | Purpose | Used By | Replication |
|---|---|---|---|
| `Contoso-SQLMI-CMK` | SQL MI TDE encryption | SQL MI Managed Identity | Auto-replicated to Central India |
| `Contoso-DISK-CMK` | VM Disk encryption | Disk Encryption Set(s) | Auto-replicated to Central India |
| `Contoso-STORAGE-CMK` | Storage encryption (Phase 2) | Storage Account Identity | Auto-replicated to Central India |

> **Note:** All keys are created in the primary region (South India) and automatically replicated to the extended region (Central India) via multi-region replication. There are no separate "-CI" key resources to create or manage.

### RBAC Assignments (Created Once — Auto-Replicated)

| Role | Assignee | Scope | Replication |
|---|---|---|---|
| Managed HSM Administrator | Contoso Security Team (2-3 people) | / | Auto-replicated |
| Managed HSM Crypto Officer | Contoso Governance / Compliance Team | / | Auto-replicated |
| Managed HSM Crypto User | Contoso Key Management Team | /keys | Auto-replicated |
| Managed HSM Crypto Service Encryption User | SQL MI Managed Identity | /keys/Contoso-SQLMI-CMK | Auto-replicated |
| Managed HSM Crypto Service Encryption User | DES Managed Identity (South India) | /keys/Contoso-DISK-CMK | Auto-replicated |
| Managed HSM Crypto Service Encryption User | DES Managed Identity (Central India) | /keys/Contoso-DISK-CMK | Auto-replicated |

> **Note:** RBAC assignments are made once on the primary HSM and automatically replicate. The Central India DES identity requires its own RBAC assignment because it is a separate Azure Compute resource with a different managed identity, but this assignment is still made on the single Managed HSM resource.

### Regional Azure Compute Resources (NOT Replicated — Must Be Created Per Region)

**South India / Chennai (Primary):**

| Resource | Name | Purpose |
|---|---|---|
| Private Endpoint | `pe-contoso-hsm-southindia` | Secure VNet access to HSM |
| Private DNS Zone | `privatelink.managedhsm.azure.net` | DNS resolution for PE |
| Disk Encryption Set | `Contoso-DES-SouthIndia` | Links VM disks to HSM key |
| Jump Box / Bastion | `contoso-bastion-southindia` | Admin access to HSM when public endpoint is disabled |

**Central India / Pune (DR/Extended):**

| Resource | Name | Purpose |
|---|---|---|
| Private Endpoint | `pe-contoso-hsm-centralindia` | Secure VNet access to HSM in extended region |
| Private DNS Zone | `privatelink.managedhsm.azure.net` | DNS resolution for PE |
| Disk Encryption Set | `Contoso-DES-CentralIndia` | Required for ASR-replicated VM disks in DR region |
| Jump Box / Bastion | `contoso-bastion-centralindia` | Admin access to HSM when public endpoint is disabled |

---

## 12. Pricing Summary

| Component | Cost | Notes |
|---|---|---|
| Managed HSM Pool (Standard B1) — Primary | ~$3.20/hour (~$2,304/month) | South India primary pool |
| Multi-region replication (extended region) | ~$3.20/hour (~$2,304/month) | Central India extended pool — **additional charge** |
| Keys (estimated 3 keys) | $3–$15/month | Depends on key type (RSA 2048 vs. 3072/4096) |
| Operations (estimated 500K/month) | ~$1.50/month | Encrypt, decrypt, wrap, unwrap operations |
| Private Endpoints (2) | No additional HSM charge | Standard PE networking costs may apply |
| **Total estimated (two regions)** | **~$4,620–$4,635/month** | Primary pool + extended region pool + key/operation fees |

> **Single-region deployment:** If Contoso does not require multi-region replication, a single Managed HSM pool costs ~$2,304/month with built-in within-region HA (3-partition cluster across fault domains).

> **Purge protection billing impact:** If the HSM is deleted, it enters a soft-deleted state for the configured retention period (7–90 days). With purge protection enabled, the HSM **continues to be billed for the full retention period** because early purging is blocked. This is by design and cannot be reversed.

### Cost Comparison with Legacy Dedicated HSM

| Model | Monthly Cost | HA Included | Regions |
|---|---|---|---|
| Dedicated HSM (HA pair, single region) | ~$6,984/month | Manual (2 devices) | 1 |
| **Managed HSM (multi-region)** | **~$4,620/month** | **Built-in (3-partition + extended region)** | **2** |
| **Savings** | **~$2,364/month (~34%)** | | **Better HA + DR coverage across 2 regions** |

> **Note on comparison basis:** The Dedicated HSM cost above is for an HA pair in a **single region** only — it did not include cross-region DR. Managed HSM at ~$4,620/month provides both within-region HA **and** cross-region DR, which would have required 4 Dedicated HSM devices (~$13,968/month) under the legacy model. On a like-for-like two-region basis, the savings are ~$9,348/month (~67%).
>
> For a single-region comparison: Managed HSM (~$2,304/month) vs. Dedicated HSM HA pair (~$6,984/month) = ~$4,680/month savings (~67%).

> *We would recommend confirming per-key pricing details with your Microsoft account team, as some advanced key type charges may also apply within the Managed HSM pool.*

---

## 13. Questions Still Pending from Contoso

| # | Question | Impact on Architecture |
|---|---|---|
| 1 | **South India / Central India relationship** — Primary/DR, Active-Active, or independent? | Confirms multi-region replication model |
| 2 | **AD server type** — Self-hosted DCs on VMs or Azure AD Domain Services? | Determines if CMK is feasible for AD workloads |
| 3 | **SQL MI count** — How many instances, which region(s)? | Per-key RBAC scoping |
| 4 | **Phase 2 services** — Full list of additional services? | Key planning and timeline |
| 5 | **DR mandate** — Is cross-region key redundancy required by compliance? | Confirms need for multi-region replication |
| 6 | **Key rotation cadence** — Annual, quarterly, or other? | Rotation policy configuration + ASR impact assessment |
| 7 | **VNet peering** — Are South India and Central India VNets peered? | Network and DNS design |
| 8 | **Non-prod environment** — Available for pilot? | Deployment sequencing |
| 9 | **ASR usage** — Are AD or other VMs replicated via Azure Site Recovery? | Determines DES prerequisites and rotation constraints |
| 10 | **Admin access model** — Does Contoso have jump boxes or Bastion in both regions? | Required for HSM admin operations with public endpoint disabled |

---

## References

- [Azure Key Vault Managed HSM Overview](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/overview)
- [Managed HSM Built-in Roles (Crypto Officer vs. Crypto User)](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/built-in-roles)
- [Quickstart: Provision and Activate Managed HSM](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/quick-create-cli)
- [Managed HSM Security Domain](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/security-domain)
- [Managed HSM Multi-Region Replication](https://learn.microsoft.com/en-us/azure/key-vault/managed-hsm/multi-region-replication)
- [SQL MI TDE with CMK](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/transparent-data-encryption-byok-overview)
- [Azure Disk Encryption with CMK](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption-customer-managed-keys)
- [Azure Site Recovery with CMK-encrypted disks](https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-how-to-enable-replication-cmk-disks)
- [Azure Key Vault Pricing](https://azure.microsoft.com/en-us/pricing/details/key-vault/)
- [Azure Regions — South India (No AZ Support)](https://www.azurespeed.com/Information/AzureRegions/SouthIndia)
