# Azure Blob Storage Vaulted Backup Solution - Architectural Strategy


## Summary

This document provides comprehensive technical and commercial guidance for implementing **Azure Vaulted Backup** for Azure Blob Storage. Vaulted backup creates immutable, air-gapped copies of blob data in a separate Azure Backup Vault, providing enhanced protection against ransomware, accidental deletion, and compliance requirements.

### Key Characteristics of Vaulted Backup:
-  **Data copied to separate Backup Vault** (not stored in source account)
-  **Immutable backups** with WORM (Write Once, Read Many) protection
-  **Air-gapped** from source storage account
-  **Snapshot-based** scheduled backups
-  **Cross-region vault** support for geo-redundancy


---

## Table of Contents

1. [Pricing Model & Worked Examples](#1-pricing-model--worked-examples)
2. [Incremental Backup & Data Movement](#2-incremental-backup--data-movement)
3. [Charging for Read Operations](#3-charging-for-read-operations)
4. [Compression, Deduplication, and Encryption](#4-compression-deduplication-and-encryption)
5. [Stop Protection / Retain Data Behaviour](#5-stop-protection--retain-data-behaviour)
6. [Quick answers to the questions](#5-Quick-Answers-to-the-questions:

---

## Architecture Overview

### Vaulted Backup Architecture:

```
┌────────────────────────────────────────────────────────────┐
│  REGION A: Production Environment                          │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Source Storage Account (storageacct01)            │    │
│  │  ┌──────────────────────────────────────────────┐  │    │
│  │  │  Container: production-data                  │  │    │
│  │  │  - Blob Storage (Hot/Cool/Archive Tier)      │  │    │
│  │  │  - Live production data                      │  │    │
│  │  │  - Total Size: 10 TB                         │  │    │
│  │  └──────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────┘    │
│                          │                                 │
│                          │ Scheduled Backup Job            │
│                          │ (Daily/Weekly snapshots)        │
│                          │                                 │
│                          ▼                                 │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Azure Backup Service (Management Plane)           │    │
│  │  - Orchestrates backup jobs                        │    │
│  │  - Reads source blob snapshots                     │    │
│  │  - Copies to Backup Vault                          │    │
│  └────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────┘
                          │
                          │ Copy snapshot data
                          │ (Can be cross-region)
                          ▼
┌────────────────────────────────────────────────────────────┐
│  REGION A (or REGION B): Backup Vault Storage              │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Backup Vault (backupvault01)                      │    │
│  │  ┌──────────────────────────────────────────────┐  │    │
│  │  │  Immutable Backup Storage                    │  │    │
│  │  │    Snapshot 1: Day 1 (10 TB - Full)          │  │    │
│  │  │    Snapshot 2: Day 2 (10 TB - Full)          │  │    │
│  │  │    Snapshot 3: Day 3 (10 TB - Full)          │  │    │
│  │  │  ..                                          │  │    │
│  │  │    Snapshot 30: Day 30 (10 TB - Full)        │  │    │
│  │  │                                              │  │    │
│  │  │  Features:                                   │  │    │
│  │  │  - WORM (immutable)                          │  │    │
│  │  │  - Soft delete protection                    │  │    │
│  │  │  - Storage redundancy (LRS/GRS/ZRS)          │  │    │
│  │  │  - Encryption at rest                        │  │    │
│  │  └──────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────┘
```

### Key Components:

| Component | Purpose | Location |
|-----------|---------|----------|
| **Source Storage Account** | Production blob data | Original region |
| **Backup Vault** | Stores immutable backup copies | Same or different region |
| **Backup Policy** | Defines schedule, retention | Backup vault configuration |
| **Recovery Points** | Point-in-time snapshots | Stored in vault |

---

## 1. Pricing Model & Worked Examples

### Cost Components for Vaulted Backup:

#### A. Protected Instance Fee
- **$10-15 per storage account/month** (region-dependent)
- Charged for each protected storage account
- Billed regardless of data size

#### B. Backup Storage Cost (MAJOR COST DRIVER)
- Charged for **actual snapshot data** stored in Backup Vault
- **Full snapshots** stored (not just deltas)
- Pricing tiers:
  - **Backup Storage (LRS)**: ~$0.05/GB/month
  - **Backup Storage (GRS)**: ~$0.10/GB/month
  - **Backup Storage (ZRS)**: ~$0.0625/GB/month

**Important**: With vaulted backup, each snapshot is a **full copy**, so storage costs accumulate quickly.

#### C. Snapshot Operations Cost
- **Snapshot creation**: ~$0.05 per 10,000 operations
- **List operations**: ~$0.05 per 10,000 operations
- Typically minimal (few dollars/month)

#### D. Data Transfer (Ingress to Vault)
- **Same region**: FREE
- **Cross-region**: ~$0.02-0.087/GB (depends on regions)

#### E. Restore Operations
- **Restore from vault**: FREE
- **Data egress to source**: 
  - Same region: FREE
  - Cross-region: ~$0.02-0.087/GB

#### F. Soft Delete Storage (if enabled)
- Additional 14-day retention after deletion
- Same rate as backup storage (~$0.05/GB/month)

---

### Worked Example 1: Single Region, 30-Day Retention

```
Scenario:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Source Storage Account: 10 TB (10,000 GB)
- Backup Schedule: Daily
- Retention: 30 days
- Redundancy: LRS (Locally Redundant Storage)
- Region: East US
- Change Rate: 5% daily (500 GB actual changes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Monthly Costs Breakdown:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Protected Instance Fee:
   1 storage account × $10/month = $10.00

2. Backup Storage (Full Snapshots):
   CRITICAL: Each snapshot stores FULL 10 TB
   
   Day 1:  10,000 GB × $0.05 = $500.00
   Day 2:  20,000 GB × $0.05 = $1,000.00 (2 snapshots)
   Day 3:  30,000 GB × $0.05 = $1,500.00 (3 snapshots)
   ...
   Day 30: 300,000 GB × $0.05 = $15,000.00 (30 snapshots)
   
   Average monthly cost: ~$7,750.00
   (Average: 15 snapshots × 10 TB × $0.05)

3. Snapshot Operations:
   Daily snapshot creation: 30 snapshots/month
   Cost: ~$0.15/month (negligible)

4. Data Transfer (same region):
   Ingress to vault: FREE

5. Source Account Read Transactions:
   Snapshot read operations: ~100M operations/month
   100M × $0.0004/10K = $400.00

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL MONTHLY COST: ~$8,160.15
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Annual TCO: ~$97,921.80
```

---

### Worked Example 2: Cross-Region GRS, 90-Day Retention

```
Scenario:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Source Storage Account: 10 TB (10,000 GB)
- Backup Schedule: Daily
- Retention: 90 days (compliance requirement)
- Redundancy: GRS (Geo-Redundant Storage)
- Source Region: East US
- Vault Region: West US 2
- Change Rate: 5% daily
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Monthly Costs Breakdown:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Protected Instance Fee:
   1 storage account × $10/month = $10.00

2. Backup Storage (GRS - Full Snapshots):
   90 snapshots at steady state
   900,000 GB × $0.10/GB = $90,000.00/month

3. Snapshot Operations:
   Daily snapshots: 30/month
   Cost: ~$0.15/month

4. Cross-Region Data Transfer (Ingress):
   Daily: 10,000 GB × $0.02 = $200.00
   Monthly: $200 × 30 = $6,000.00

5. Source Account Read Transactions:
   Snapshot operations: ~$400.00/month

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL MONTHLY COST: ~$96,410.15
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Annual TCO: ~$1,156,921.80
```

---

### Worked Example 3: Tiered Retention (Optimized)

```
Scenario (Optimized with Tiered Retention):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Source Storage Account: 10 TB
- Daily backups: Retained for 7 days (7 snapshots)
- Weekly backups: Retained for 4 weeks (4 snapshots)
- Monthly backups: Retained for 12 months (12 snapshots)
- Redundancy: LRS
- Region: Same region
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Total Snapshots at Steady State:
Daily: 7 × 10 TB = 70 TB
Weekly: 4 × 10 TB = 40 TB
Monthly: 12 × 10 TB = 120 TB
Total: 230 TB (23 snapshots)

Monthly Costs:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. Protected Instance: $10.00
2. Backup Storage: 230,000 GB × $0.05 = $11,500.00
3. Snapshot Operations: ~$0.20
4. Source Read Transactions: ~$400.00

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL MONTHLY COST: ~$11,910.20
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Annual TCO: ~$142,922.40

SAVINGS vs. 90-day daily retention: ~87% reduction
```

---

## 2. Incremental Backup & Data Movement

### How Vaulted Backup Works:

#### Backup Process Flow:

```
Step 1: Snapshot Creation at Source
┌────────────────────────────────────────┐
│  Source Storage Account                │
│  ┌──────────────────────────────────┐  │
│  │  Create Blob Snapshot            │  │
│  │  - Incremental at block level    │  │
│  │  - Only changed blocks stored    │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
           │
           │ Snapshot metadata
           ▼
Step 2: Copy to Backup Vault
┌────────────────────────────────────────┐
│  Azure Backup Service                  │
│  - Reads snapshot data                 │
│  - Transfers to vault storage          │
│  - Stores as recovery point            │
└────────────────────────────────────────┘
           │
           │ Full snapshot data
           ▼
Step 3: Store in Vault
┌────────────────────────────────────────┐
│  Backup Vault                          │
│  ┌──────────────────────────────────┐  │
│  │  Recovery Point Created          │  │
│  │  - Full snapshot stored          │  │
│  │  - Immutable (WORM)              │  │
│  │  - Encrypted at rest             │  │
│  └──────────────────────────────────┘  │
└────────────────────────────────────────┘
```

---

### Incremental vs. Full Backup Behavior:

#### At Source Storage Account Level:
```
Block-Level Incremental Storage
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Initial Blob: 1 GB file
Day 1 Snapshot: 1 GB stored
Day 2: 50 MB changed
Day 2 Snapshot: +50 MB stored (incremental blocks)
Day 3: 100 MB changed
Day 3 Snapshot: +100 MB stored (incremental blocks)

Source Account Storage Growth:
Day 1: 1 GB (base) + 1 GB (snapshot) = 2 GB
Day 2: 1 GB (base) + 1 GB + 50 MB = 2.05 GB
Day 3: 1 GB (base) + 1 GB + 50 MB + 100 MB = 2.15 GB

 Efficient incremental storage at source
```

#### At Backup Vault Level:
```
Full Snapshot Storage (Logical, not Physical)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Day 1: Full snapshot appears as 1 GB
Day 2: Full snapshot appears as 1 GB
Day 3: Full snapshot appears as 1 GB

Each recovery point shows complete data state

HOWEVER: Azure may optimize storage behind the scenes
with deduplication, but you're billed for logical size

Vault Storage (billed):
Day 1: 1 GB
Day 2: 2 GB (2 × 1 GB snapshots)
Day 3: 3 GB (3 × 1 GB snapshots)

 Billed for full logical snapshot size
```

---

### Delta Detection Method:

**Vaulted backup uses snapshot-based approach**:

1. **Snapshot Creation** (at source):
   - Azure Blob Storage creates snapshot
   - Snapshots are incremental at block level
   - Only changed blocks consume additional storage

2. **Snapshot Transfer** (to vault):
   - Backup service reads entire snapshot
   - Transfers logical complete snapshot to vault
   - Vault may apply deduplication (Azure-managed)

3. **Delta Detection**:
   - **Automatic** - Azure Blob Storage tracks changed blocks
   - Uses internal block-level change tracking
   - No manual delta calculation needed

---

### Data Movement Architecture:

```
┌─────────────────────────────────────────────────────────┐
│  Source Storage Account (East US)                       │
│                                                         │
│  Blob Container: /production-data/                      │
│  ├── file1.pdf (100 MB)                                 │
│  ├── file2.jpg (50 MB)                                  │
│  └── file3.docx (25 MB)                                 │
│                                                         │
│  Daily Snapshot Created: 11/17/2025 00:00 UTC           │
│  Snapshot ID: snapshot-2025-11-17                       │
└─────────────────────────────────────────────────────────┘
                    │
                    │ Azure Backup Service
                    │ Orchestrated Transfer
                    │
                    │ Transfer Details:
                    │ - Reads snapshot data
                    │ - 175 MB total transfer
                    │ - Encrypted in transit (TLS 1.2)
                    │ - Same region: FREE transfer
                    │ - Cross-region: Egress charges apply
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│  Backup Vault (East US or West US 2)                    │
│                                                         │
│  Recovery Point: 2025-11-17 00:00 UTC                   │
│  ├── Snapshot data: 175 MB                              │
│  ├── Metadata: Policy, retention, tags                  │
│  ├── Immutability: Locked until 2025-12-17              │
│  └── Encryption: AES-256 (vault-managed key)            │
│                                                         │
│  Storage Cost: 175 MB × $0.05/GB = $0.00875/month       │
└─────────────────────────────────────────────────────────┘
```

---

### Performance Characteristics:

| Metric | Value | Notes |
|--------|-------|-------|
| **Backup Window** | 2-6 hours | Depends on data size |
| **RPO (Recovery Point Objective)** | 24 hours (daily) | Based on backup schedule |
| **RTO (Recovery Time Objective)** | 2-4 hours | For 10 TB restore |
| **Data Transfer Rate** | 100-500 MB/s | Varies by region/network |
| **Maximum Snapshot Size** | Up to 500 TB | Per storage account |

---

## 3. Charging for Read Operations

### Source Storage Account Transaction Costs:

#### During Backup Operations:

```
Transaction Types During Vaulted Backup:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Snapshot Creation (at source):
   - Write operation: Create snapshot
   - Cost: $0.05 per 10,000 write operations
   
2. Snapshot Reading (for vault transfer):
   - Read operations: Read all blob data in snapshot
   - Cost: $0.0044 per 10,000 read operations (Hot tier)
   
3. List/Metadata Operations:
   - List blobs in container
   - Read blob metadata
   - Cost: $0.065 per 10,000 list operations

4. Snapshot Storage (at source):
   - Incremental block storage
   - Cost: Same as base tier (Hot/Cool/Archive)
```

---

### Cost Example - Daily Backup of 10 TB:

```
Scenario: 10 TB blob storage, daily backup
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Estimated Operations per Backup:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- List operations: ~100,000 (inventory containers)
- Snapshot create: ~10,000 (create blob snapshots)
- Read operations: ~100,000,000 (read 10 TB of data)
  (Assuming 100 KB average blob size)

Monthly Transaction Costs (30 backups):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
1. List operations:
   100K × 30 days = 3,000,000/month
   3M × $0.065/10K = $19.50

2. Snapshot creation:
   10K × 30 days = 300,000/month
   300K × $0.05/10K = $1.50

3. Read operations (MAJOR COST):
   100M × 30 days = 3,000,000,000/month (3 billion)
   3B × $0.0044/10K = $1,320.00

4. Snapshot storage at source (incremental):
   Assuming 5% daily change × 30 days
   10 TB × 5% × 30 = 15 TB additional
   15,000 GB × $0.018/GB (Hot tier) = $270.00

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TOTAL SOURCE ACCOUNT COSTS: ~$1,611.00/month
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

This is IN ADDITION to vault storage costs!
```

---

### Transaction Cost Comparison by Tier:

| Storage Tier | Read Cost (per 10K) | Monthly Cost (10 TB daily backup) |
|--------------|---------------------|-----------------------------------|
| **Hot** | $0.0044 | $1,320.00 |
| **Cool** | $0.01 | $3,000.00 |
| **Archive** | $5.50 | $1,650,000.00 (!) |

**WARNING**: Backing up from Archive tier is **EXTREMELY expensive** due to rehydration costs!

---

### During Restore Operations:

```
Restore Transaction Costs:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Read from Backup Vault:
   - Read operations: Read snapshot data from vault
   - Cost: Included in vault storage (no additional charge)

2. Write to Source Account:
   - Write operations: Restore blobs to source
   - Cost: $0.05 per 10,000 write operations
   
3. Data Egress (if cross-region):
   - Transfer from vault region to source region
   - Cost: $0.02-0.087 per GB

Example: Restore 10 TB from vault to source (same region)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Read from vault: $0 (included)
- Write to source: 100M operations × $0.05/10K = $500.00
- Data transfer (same region): $0
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total Restore Cost: ~$500.00

Example: Restore 10 TB cross-region (vault in West US, source in East US)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Read from vault: $0
- Write to source: $500.00
- Data egress: 10,000 GB × $0.02 = $200.00
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total Restore Cost: ~$700.00
```

---

## 5. Compression, Deduplication, and Encryption

### Azure Vaulted Backup Optimization Features:

#### A. Compression

```
Native Compression Status: NO explicit compression
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Azure Backup does NOT apply additional compression
- Data stored in vault is same logical size as source
- Already-compressed data (ZIP, GZIP) not re-compressed

Workaround for Better TCO:
- Compress blobs BEFORE backup (application-level)
- Use native compression formats (GZIP, Brotli, LZ4)
- Store compressed objects in blob storage

Example:
Original file: 1 GB log file
GZIP compressed: 100 MB
Vault storage cost: Billed for 100 MB (compressed size)
Savings: 90% reduction
```

---

#### B. Deduplication

```
Deduplication Behavior: LIMITED (Azure-managed)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Snapshot-Level Deduplication:
- Azure may deduplicate identical blocks across snapshots
- NOT guaranteed or documented
- Billing is still based on LOGICAL snapshot size
- Customer cannot control or optimize this

Example Scenario:
┌─────────────────────────────────────────────────────────┐
│  Day 1 Snapshot: 10 TB (1000 blobs)                     │
│  Day 2 Snapshot: 10 TB (998 unchanged, 2 changed)       │
│                                                         │
│  Physical Storage (with Azure dedup):                   │
│  - Day 1: 10 TB                                         │
│  - Day 2: +20 GB (only changed blobs)                   │
│  Total Physical: ~10.02 TB                              │
│                                                         │
│  Billing (logical size):                                │
│  - Day 1: 10 TB × $0.05 = $500                          │
│  - Day 2: 20 TB × $0.05 = $1,000 (billed for both)      │
│  Total Billed: $1,000/month                             │
└─────────────────────────────────────────────────────────┘

 You are billed for LOGICAL size, not physical storage
```

---


### Encryption Details:

#### Encryption at Rest (Vault Storage):

```
┌────────────────────────────────────────────────────────┐
│  Encryption Architecture                               │
├────────────────────────────────────────────────────────┤
│                                                        │
│  1. Platform-Managed Keys (Default):                   │
│     ┌────────────────────────────────────────────┐     │
│     │  Azure Backup Vault                        │     │
│     │  ├─ Encrypted with Microsoft-managed key   │     │
│     │  ├─ AES-256 encryption                     │     │
│     │  ├─ Keys rotated automatically             │     │
│     │  └─ No customer management required        │     │
│     └────────────────────────────────────────────┘     │
│                                                        │
│  2. Customer-Managed Keys (CMK):                       │
│     ┌────────────────────────────────────────────┐     │
│     │  Azure Key Vault (Customer)                │     │
│     │  ├─ Customer controls encryption key       │     │
│     │  ├─ Key rotation managed by customer       │     │
│     │  └─ Can revoke access anytime              │     │
│     └────────────────────────────────────────────┘     │
│                │                                       │
│                ▼                                       │
│     ┌────────────────────────────────────────────┐     │
│     │  Backup Vault                              │     │
│     │  ├─ Data encrypted with CMK                │     │
│     │  ├─ Key reference stored                   │     │
│     │  └─ Decryption requires Key Vault access   │     │
│     └────────────────────────────────────────────┘     │
│                                                        │
└────────────────────────────────────────────────────────┘

Encryption Specifications:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Algorithm: AES-256
Mode: CBC (Cipher Block Chaining)
Key Size: 256-bit
Key Storage: Azure Key Vault
FIPS 140-2 Compliant: Yes
```

---

#### Encryption in Transit:

```
Data Transfer Encryption:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Source → Vault:
- TLS 1.2 or higher
- Perfect Forward Secrecy (PFS)
- SHA-256 signature algorithm

Vault → Restore Destination:
- TLS 1.2 or higher
- Encrypted channel end-to-end

Supported Cipher Suites:
- TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
- TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

---

#### Double Encryption (Infrastructure + Customer):

```
Optional: Infrastructure Encryption
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌──────────────────────────────────────────────┐
│  Layer 1: Service-Level Encryption           │
│  (Platform-managed, 256-bit AES)             │
│         ┌──────────────────────────┐         │
│         │  Encrypted Backup Data   │         │
│         └──────────────────────────┘         │
└──────────────────────────────────────────────┘
                    │
                    ▼
┌──────────────────────────────────────────────┐
│  Layer 2: Infrastructure Encryption          │
│  (Platform-managed, 256-bit AES)             │
│         ┌──────────────────────────┐         │
│         │  Double-Encrypted Data   │         │
│         └──────────────────────────┘         │
└──────────────────────────────────────────────┘

Benefits:
- Defense-in-depth security
- Separate encryption keys for each layer
- Compliance with strict regulatory requirements
- No performance impact
- No additional cost
```

---

## 6. Stop Protection / Retain Data Behaviour

### Stop Protection Options:

#### Option A: Stop Protection with Retain Data

```
┌────────────────────────────────────────────────────────┐
│  Configuration Changes:                                │
│   Backup policy: DISASSOCIATED                         │
│   Scheduled backups: STOPPED                           │
│   Existing recovery points: RETAINED                   │
│   Immutability locks: REMAIN IN EFFECT                 │
│   Restore capability: AVAILABLE                        │
│   Soft delete: STILL ACTIVE (if enabled)               │
└────────────────────────────────────────────────────────┘

Billing Behavior:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

STOPS Charging:
 - Protected Instance Fee ($10/month)
 - Backup job execution costs
 - Source account snapshot creation

CONTINUES Charging:
  - Vault storage for retained recovery points
  - Soft delete storage (if applicable)
  - Vault storage redundancy (LRS/GRS/ZRS)

Example Cost Evolution:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Initial state: 30-day retention, 30 snapshots × 10 TB

Month 0 (Active Protection):
  Protected Instance: $10
  Vault Storage: 300 TB × $0.05 = $15,000
  Total: $15,010/month

Month 1 (Stop Protection - Retain):
  Protected Instance: $0 (stopped)
  Vault Storage: 300 TB × $0.05 = $15,000
  Total: $15,000/month
  (Recovery points aging out: Day 1 snapshot expires)

Month 2:
  Vault Storage: 290 TB × $0.05 = $14,500
  (10 more snapshots expired)
  Total: $14,500/month

Month 3:
  Vault Storage: 200 TB × $0.05 = $10,000
  Total: $10,000/month

Month 4+:
  Vault Storage: 0 TB × $0.05 = $0
  (All snapshots aged out per original 30-day policy)
  Total: $0/month
```

---

#### Option B: Stop Protection with Delete Data

```
┌────────────────────────────────────────────────────────┐
│  Configuration Changes:                                │
│   Backup policy: DISASSOCIATED                         │
│   Scheduled backups: STOPPED                           │
│   Existing recovery points: DELETED IMMEDIATELY        │
│   Immutability locks: OVERRIDDEN (after soft delete)   │
│   Restore capability: NOT AVAILABLE                    │
│  Soft delete: 14-day grace period (if enabled)         │
└────────────────────────────────────────────────────────┘

Soft Delete Behavior:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

If Soft Delete Enabled (Default):
  Day 0: Stop protection → Mark for deletion
  Day 1-14: Soft delete retention period
            - Data still in vault (billed)
            - Can be undeleted/restored
            - Vault storage charges continue
  Day 15: Permanent deletion
            - Data purged from vault
            - Billing stops
            - Cannot be recovered

Billing Timeline with Soft Delete:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Day 0 (Stop + Delete):
  Protected Instance: $0
  Vault Storage: 300 TB × $0.05 = $15,000
  Total: $15,000

Days 1-14 (Soft Delete Period):
  Protected Instance: $0
  Vault Storage: 300 TB × $0.05 = $15,000
  Soft Delete: Included in vault storage
  Total: $15,000/month (prorated for 14 days = ~$7,000)

Day 15+ (Permanent Deletion):
  All charges: $0
```

---

### Lifecycle Management Matrix:

| Scenario | Protected Instance | Vault Storage | Restore | Timeline to $0 |
|----------|-------------------|---------------|---------|----------------|
| **Active Protection** |  Charged |  Charged |  Available | N/A |
| **Stop + Retain** |  Free |  Charged |  Available | 30 days (retention policy) |
| **Stop + Delete (Soft Delete ON)** |  Free |  Charged (14 days) |  14 days only | 14 days |
| **Stop + Delete (Soft Delete OFF)** |  Free |  Free |  Not available | Immediate |

---

### Detailed Cost Example - Stop Protection Scenarios:

```
Starting Scenario:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
- Storage Account: 10 TB
- Vault: 30 recovery points × 10 TB = 300 TB
- Monthly Cost: $15,010
- Retention Policy: 30 days

Scenario 1: Stop Protection + Retain Data
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Month 0 (Before Stop):     $15,010
Month 1 (After Stop):      $15,000 (no instance fee)
  30 snapshots remaining

Day 31 (First snapshot expires):
  Vault: 290 TB × $0.05 = $14,500
  
Day 60 (30th snapshot expires):
  Vault: 0 TB
  Cost: $0
  
Total Cost After Stop: ~$7,250 over 30 days
Recovery capability: Available for 30 days

Scenario 2: Stop Protection + Delete Data (Soft Delete Enabled)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Day 0 (Stop + Delete):
  Vault: 300 TB × $0.05 = $15,000
  Status: Marked for soft deletion
  
Days 1-14 (Soft Delete Period):
  Vault: 300 TB × $0.05 = $15,000/month
  Prorated: $15,000 × (14/30) = $7,000
  Recovery: Can still restore or undelete
  
Day 15+ (Permanent Deletion):
  Vault: 0 TB
  Cost: $0
  Recovery: NOT POSSIBLE
  
Total Cost After Stop: ~$7,000 for 14 days
Recovery capability: 14 days only

Scenario 3: Stop Protection + Delete Data (Soft Delete Disabled)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Day 0 (Stop + Delete):
  Immediate deletion
  Cost: $0 (prorated to hour of deletion)
  Recovery: NOT POSSIBLE
  
Total Cost After Stop: $0-50 (depends on time of day)
```

---

### Immutability and Retention Lock Behavior:

```
Retention Lock Impact on Stop Protection:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

If Backup Policy Has Immutability Lock:
┌─────────────────────────────────────────────────────────┐
│  Locked Retention Policy (e.g., 7-year compliance)      │
│                                                         │
│  Attempting to "Stop Protection + Delete Data":         │
│   BLOCKED - Cannot delete recovery points               │
│   Must wait until retention period expires              │
│   Can only "Stop Protection + Retain Data"              │
│                                                         │
│  Billing Implications:                                  │
│  - Vault storage charges continue for full 7 years      │
│  - Cannot reduce costs until lock expires               │
│  - Protected instance fee stops (saves $10/month)       │
└─────────────────────────────────────────────────────────┘

Cost Impact Example (7-Year Locked Retention):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Current state: 300 TB in vault, locked for 7 years

Stop Protection + Retain (ONLY option):
  Year 1: 300 TB × $0.05 × 12 = $180,000
  Year 2: 300 TB × $0.05 × 12 = $180,000
  ...
  Year 7: 300 TB × $0.05 × 12 = $180,000
  
Total 7-Year Cost: ~$1,260,000

 CRITICAL: Understand immutability implications BEFORE
enabling retention locks!
```

---


### Audit and Monitoring:

```
Azure Backup Reporting & Monitoring:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Azure Monitor Integration:
   ┌────────────────────────────────────────────┐
   │  Backup Vault                              │
   │  ↓                                         │
   │  Diagnostic Logs → Log Analytics Workspace │
   │  ↓                                         │
   │  Metrics & Alerts                          │
   │  ├─ Backup job failures                    │
   │  ├─ Restore job status                     │
   │  ├─ Policy changes                         │
   │  └─ Vault storage consumption              │
   └────────────────────────────────────────────┘

2. Backup Reports (Power BI):
   - Backup job success/failure trends
   - Storage consumption over time
   - Policy compliance dashboards
   - Cost analysis and forecasting

3. Azure Activity Log:
   - All vault configuration changes
   - Policy modifications
   - Protection stop/start events
   - Delete operations (with MUA approval trail)

4. Backup Center:
   - Centralized view across all vaults
   - Multi-subscription monitoring
   - Compliance reporting
   - Restore point inventory
```

---

### Cost Optimization Strategies:

#### 1. Tiered Retention Policy (RECOMMENDED)

```
Instead of: 30 daily backups
Use: 7 daily + 4 weekly + 12 monthly

Cost Comparison:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Original (30 daily):
  30 snapshots × 10 TB × $0.05 = $15,000/month

Optimized (7+4+12):
  23 snapshots × 10 TB × $0.05 = $11,500/month

Monthly Savings: $3,500 (23% reduction)
Annual Savings: $42,000
```

---

#### 2. Application-Level Compression

```
Compress data BEFORE storing in blob storage:

Scenario: Log files (highly compressible)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Original: 10 TB raw logs
GZIP Compressed: 1 TB (90% reduction)

Backup Cost Comparison:
  Uncompressed: 300 TB vault × $0.05 = $15,000/month
  Compressed: 30 TB vault × $0.05 = $1,500/month

Monthly Savings: $13,500 (90% reduction)
Annual Savings: $162,000
```

---

#### 3. Lifecycle Management at Source

```
Strategy: Move older data to Cool/Archive tier BEFORE backup

┌────────────────────────────────────────────────────────┐
│  Data Lifecycle:                                       │
│  Days 0-30: Hot tier (frequently accessed)             │
│  Days 31-90: Cool tier (backup from Cool)              │
│  Days 91+: Archive tier (no vaulted backup)            │
└────────────────────────────────────────────────────────┘

Cost Impact:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Hot tier backup transaction cost: $1,320/month
Cool tier backup transaction cost: $3,000/month (higher!)
   Cool tier has higher transaction costs

Recommendation:
- Keep data in Hot tier if backing up frequently
- OR backup LESS frequently from Cool tier
- Avoid backing up Archive tier (extremely expensive)
```

---

#### 4. Right-Size Backup Scope

```
Selective Backup Strategy:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Instead of backing up entire storage account:
Use container-level or blob prefix filtering

Example:
Storage Account: 10 TB total
├─ /critical-data/ (2 TB) → BACKUP (immutable vault)
├─ /transient-logs/ (5 TB) → NO BACKUP (regenerable)
└─ /archived-reports/ (3 TB) → Blob versioning only

Vault Cost Comparison:
Full account: 300 TB (30 × 10 TB) × $0.05 = $15,000/month
Critical only: 60 TB (30 × 2 TB) × $0.05 = $3,000/month

Monthly Savings: $12,000 (80% reduction)
```

---

#### 5. Cross-Region Strategy

```
When to Use Cross-Region Vaulted Backup:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 Compliance requires geo-redundancy
 Disaster recovery (regional failure)
 Data sovereignty requirements

Cost Implications:
Same-Region (LRS):
  Vault Storage: $0.05/GB
  Data Transfer: $0
  Total: $15,000/month (for 300 TB)

Cross-Region (GRS):
  Vault Storage: $0.10/GB (2× cost)
  Data Transfer: $0.02/GB × 10 TB × 30 = $6,000
  Total: $30,000 + $6,000 = $36,000/month

 Cross-region is 2.4× more expensive

Alternative: Use GRS storage at source + LRS vault
  Source: Enable GRS replication ($0.046/GB)
  Vault: LRS backup ($0.05/GB)
  Total: $15,460/month (cost-effective geo-redundancy)
```

---

### Cost Comparison: Vaulted vs. Alternatives

```
┌──────────────────────────────────────────────────────────────┐
│  10 TB Blob Backup - Monthly Cost Comparison                 │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Option 1: No Backup (Risky)                                 │
│  Cost: $0                                                    │
│  Protection: None                                            │
│  RTO/RPO: N/A (data loss)                                    │
│                                                              │
│  Option 2: Blob Versioning Only                              │
│  Cost: ~$500/month (5% daily change, 30-day retention)       │
│  Protection: Accidental deletion, overwrite                  │
│  RTO/RPO: Seconds / Point-in-time                            │
│   Not air-gapped, no compliance                              │
│                                                              │
│  Option 3: Operational Backup (Versioning + Azure Backup)    │
│  Cost: ~$760/month                                           │
│  Protection: Versioning + policy management                  │
│  RTO/RPO: Minutes / Continuous                               │
│   Data stays in source account (not air-gapped)              │
│                                                              │
│  Option 4: Vaulted Backup (Air-Gapped, Immutable)            │
│  Cost: ~$8,160/month (30-day daily retention)                │
│  Protection: Full compliance, ransomware protection          │
│  RTO/RPO: 2-4 hours / 24 hours                               │
│   Air-gapped, immutable, compliance-ready                    │
│                                                              │
│  Option 5: Vaulted Backup (Optimized Tiered Retention)       │
│  Cost: ~$11,910/month (7D+4W+12M)                            │
│  Protection: Full compliance, extended retention             │
│  RTO/RPO: 2-4 hours / 24 hours                               │
│   Best balance of cost and protection                        │
│                                                              │
│  Option 6: Third-Party (e.g., Commvault, Veeam)              │
│  Cost: ~$15,000-25,000/month (varies by vendor)              │
│  Protection: Advanced features, multi-cloud                  │
│  RTO/RPO: Varies / Varies                                    │
│      Additional licensing and infrastructure costs           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 6. Quick Answers to the questions:

### **1. Pricing Model & Worked Example**  
**Quick Answer:**  
Pay-as-you-use model: protected-instance fee + backup-storage cost (per GB) + chosen redundancy tier; restores and intra-region transfers typically incur no additional charge.

**Azure references:**  
- Azure Backup Pricing Overview  
  https://learn.microsoft.com/en-us/azure/backup/azure-backup-pricing  
- Backup Vault Overview  
  https://learn.microsoft.com/en-us/azure/backup/backup-vault-overview  
- Storage Redundancy Options (LRS/GRS/ZRS)  
  https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy  
- Restore Billing Details  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-restore-files-from-vm  

---

### **2. Incremental Backup & Data Movement**  
**Quick Answer:**  
After the initial full backup, Azure Backup only transfers changed blocks (incremental), comparing block-level deltas against the previous recovery point.

**Azure references:**  
- Backup Architecture (Incremental backup behaviour)  
  https://learn.microsoft.com/en-us/azure/backup/backup-architecture  
- Blob Versioning & Change Tracking  
  https://learn.microsoft.com/en-us/azure/storage/blobs/versioning-overview  
- Data Movement & Change Tracking  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-vms-introduction  

---

### **3. Charging for Read Operations**  
**Quick Answer:**  
Backup operations do **not** incur separate read-transaction costs on the source storage account—charges apply only to backup storage and protected-instance sizing.

**Azure references:**  
- Azure Storage Billing  
  https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-pricing  
- Azure Backup Billing FAQ  
  https://learn.microsoft.com/en-us/azure/backup/azure-backup-faq#billing  

---

### **4. Compression, Deduplication & Encryption**  
**Quick Answer:**  
Azure Backup encrypts all data by default and applies internal compression/deduplication, but Microsoft does not provide a guaranteed compression ratio.

**Azure references:**  
- Backup Encryption  
  https://learn.microsoft.com/en-us/azure/backup/backup-encryption  
- Security Baseline for Azure Backup  
  https://learn.microsoft.com/en-us/security/benchmark/azure/baselines/backup-security-baseline  
- Backup Storage Redundancy  
  https://learn.microsoft.com/en-us/azure/backup/backup-storage-redundancy-overview  

---

### **5. Stop Protection / Retain Data Behaviour**  
**Quick Answer:**  
Stopping protection while retaining data continues to incur storage and protected-instance charges until recovery points are explicitly deleted.

**Azure references:**  
- Stop Protection Behaviour  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-manage-stop-protection  
- Retention & Lifecycle  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-vault-overview#retention  
- Backup Data Deletion  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-manage-vault  

---

## **Additional Recommended References**

### **Storage Transactions & Data Access Costs**  
- Storage Read/Write/Egress Billing  
  https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-pricing  
- Bandwidth (Egress) Pricing  
  https://learn.microsoft.com/en-us/azure/bandwidth-pricing  

### **Restore Operations**  
- File/Folder Restore  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-restore-files-from-vm  
- VM Restore  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-arm-restore-vms  

### **Backup Vault Redundancy**  
- Redundancy Comparison  
  https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy  

### Official Documentation:
- [Azure Backup for Blob Storage Overview](https://learn.microsoft.com/azure/backup/blob-backup-overview)
- [Azure Backup Pricing](https://azure.microsoft.com/pricing/details/backup/)
- [Blob Storage Pricing](https://azure.microsoft.com/pricing/details/storage/blobs/)
- [Azure Backup Security Features](https://learn.microsoft.com/azure/backup/security-overview)
- [Immutable Storage for Azure Blobs](https://learn.microsoft.com/azure/storage/blobs/immutable-storage-overview)



---
