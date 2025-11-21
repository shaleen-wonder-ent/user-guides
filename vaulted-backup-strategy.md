# Azure Blob Storage Vaulted Backup Solution

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
6. [Quick answers to the questions](#6-quick-answers-to-the-questions)
7. [Common Misconceptions](#7-common-misconceptions)

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

> **⚠️ PRICING DISCLAIMER:**  
> - Prices vary by Azure region and are subject to change
> - All pricing examples are approximate and for illustrative purposes
> - Always verify current pricing at https://azure.microsoft.com/pricing/
> - Document last updated: 2025-11-21

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

✅ Efficient incremental storage at source
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

⚠️ Billed for full logical snapshot size
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

## 4. Compression, Deduplication, and Encryption

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

⚠️ You are billed for LOGICAL size, not physical storage
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

## 5. Stop Protection / Retain Data Behaviour

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
✅ - Protected Instance Fee ($10/month)
✅ - Backup job execution costs
✅ - Source account snapshot creation

CONTINUES Charging:
⚠️  - Vault storage for retained recovery points
⚠️  - Soft delete storage (if applicable)
⚠️  - Vault storage redundancy (LRS/GRS/ZRS)

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
| **Active Protection** | ✅ Charged | ✅ Charged | ✅ Available | N/A |
| **Stop + Retain** | ❌ Free | ✅ Charged | ✅ Available | 30 days (retention policy) |
| **Stop + Delete (Soft Delete ON)** | ❌ Free | ⚠️ Charged (14 days) | ⚠️ 14 days only | 14 days |
| **Stop + Delete (Soft Delete OFF)** | ❌ Free | ❌ Free | ❌ Not available | Immediate |

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
│   ❌ BLOCKED - Cannot delete recovery points               │
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

⚠️ CRITICAL: Understand immutability implications BEFORE
enabling retention locks!
```

---

## 6. Quick Answers to the questions:

### **1. Pricing Model & Worked Example**  
**Quick Answer:**  
Azure Vaulted Backup pricing includes: (a) Protected Instance Fee (~$10-15/month per storage account), (b) Backup Storage in vault (major cost - billed for full logical snapshot size at ~$0.05/GB for LRS), (c) Snapshot operations, (d) Data transfer costs (free same-region, charged cross-region), (e) Source account read transaction costs during backup, and (f) Optional soft delete storage. Each recovery point is a full snapshot, so costs accumulate with retention period.

**Refer to:** [Pricing Model & Worked Examples](#1-pricing-model--worked-examples)

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
Azure Vaulted Backup uses a snapshot-based approach. While **snapshots at the source storage account are incremental** (only changed blocks consume additional storage), each **recovery point in the backup vault is stored as a full logical copy**. The backup service reads the entire snapshot and transfers it to the vault. Azure may apply backend deduplication, but billing is based on the logical full snapshot size, not physical storage.

**Refer to:** [Incremental Backup & Data Movement](#2-incremental-backup--data-movement)

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
Backup operations **DO** incur read-transaction costs on the source storage account when reading snapshot data for transfer to the vault. For a 10 TB daily backup from Hot tier, expect ~$1,320/month in read operation charges alone (3 billion read operations × $0.0044 per 10K operations). These costs are **in addition** to vault storage and protected instance fees. Avoid backing up from Archive tier due to extreme rehydration costs.

**Refer to:** [Charging for Read Operations](#3-charging-for-read-operations)

**Azure references:**  
- Azure Storage Billing  
  https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-pricing  
- Azure Backup Billing FAQ  
  https://learn.microsoft.com/en-us/azure/backup/azure-backup-faq#billing  

---

### **4. Compression, Deduplication & Encryption**  
**Quick Answer:**  
Azure Backup encrypts all data by default (AES-256) with platform-managed or customer-managed keys. However, Azure does NOT apply explicit compression—you're billed for the same logical size as source data. Limited deduplication may occur internally, but Microsoft does not guarantee compression ratios and billing is based on logical snapshot size. For better TCO, compress blobs at the application level before backup.

**Refer to:** [Compression, Deduplication, and Encryption](#4-compression-deduplication-and-encryption)

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
Stopping protection while retaining data **continues to incur vault storage charges only** (protected instance fee stops immediately). Recovery points age out according to the original retention policy. With "Stop + Delete" and soft delete enabled, vault storage charges continue for 14 days before permanent deletion. With immutability locks, you cannot delete recovery points and must wait for the retention period to expire, incurring storage costs throughout.

**Refer to:** [Stop Protection / Retain Data Behaviour](#5-stop-protection--retain-data-behaviour)

**Azure references:**  
- Stop Protection Behaviour  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-manage-stop-protection  
- Retention & Lifecycle  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-vault-overview#retention  
- Backup Data Deletion  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-manage-vault  

---

## 7. Common Misconceptions

### ❌ Misconception #1: "Vaulted backup only stores incremental data"
**Reality:** While snapshots at the source are incremental (block-level), each recovery point in the vault is a **full logical snapshot**. With 30-day retention, you're storing 30 full copies of your data, not just deltas.

**Impact:** For 10 TB source data with 30-day retention, you pay for 300 TB vault storage, not 10 TB + deltas.

---

### ❌ Misconception #2: "Backup doesn't cost anything on the source storage account"
**Reality:** Reading snapshot data from the source storage account **incurs read transaction charges**. For large datasets, this can be significant.

**Impact:** For 10 TB daily backups from Hot tier: ~$1,320/month in read operations alone.

---

### ❌ Misconception #3: "Stopping backup stops all charges immediately"
**Reality:** 
- "Stop + Retain" stops the protected instance fee but **vault storage charges continue** until retention expires
- "Stop + Delete" with soft delete enabled continues charging for 14 days
- With immutability locks, you **cannot delete data** and must pay storage costs until the lock period expires

**Impact:** With 300 TB in vault, you'll pay ~$15,000/month in storage costs even after stopping protection.

---

### ❌ Misconception #4: "Azure automatically compresses and deduplicates my backups"
**Reality:** Azure does NOT apply guaranteed compression. You're billed for the logical snapshot size. While Azure may deduplicate behind the scenes, this is not documented or guaranteed.

**Impact:** If you backup 10 TB, you're billed for 10 TB per snapshot, regardless of actual changes or duplication.

---

### ❌ Misconception #5: "Cross-region backup is just a little more expensive"
**Reality:** Cross-region backup adds:
- Data transfer costs (~$0.02/GB = $200/day for 10 TB)
- Higher vault storage costs (GRS vs LRS)
- Cross-region restore costs

**Impact:** For 10 TB with 90-day retention: ~$96,410/month vs ~$8,160/month for same-region LRS.

---

### ❌ Misconception #6: "I can backup Archive tier blobs cheaply"
**Reality:** Archive tier read/rehydration costs are **extremely expensive** (~$5.50 per 10K operations).

**Impact:** 10 TB daily backup from Archive tier: ~$1,650,000/month in read operations alone!

---

### 💡 Best Practices to Avoid Costly Surprises:

1. **Choose retention carefully** - Use tiered retention (daily/weekly/monthly) instead of long daily retention
2. **Same-region vaults** - Only use cross-region for true DR requirements
3. **Compress before backup** - Application-level compression reduces all costs proportionally
4. **Monitor source tier** - Keep backups on Hot/Cool tier, never Archive
5. **Understand immutability** - Don't enable retention locks unless legally required
6. **Test stop protection** - Verify soft delete behavior and cost implications
7. **Calculate TCO** - Include ALL costs: instance fee, vault storage, read ops, transfers

---

## **Additional Recommended References**

### **Storage Transactions & Data Access Costs**  
- Storage Read/Write/Egress Billing  
  https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-pricing  
- Transaction Optimization  
  https://learn.microsoft.com/en-us/azure/storage/common/storage-plan-manage-costs  

### **Compliance & Governance**  
- Azure Backup Compliance Offerings  
  https://learn.microsoft.com/en-us/azure/backup/compliance-offerings  
- Immutability Policies  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-immutable-vault  

### **Cost Management**  
- Azure Cost Management + Billing  
  https://learn.microsoft.com/en-us/azure/cost-management-billing/  
- Backup Cost Optimization  
  https://learn.microsoft.com/en-us/azure/backup/backup-azure-cost-management  

### **Architecture & Design**  
- Blob Backup Architecture  
  https://learn.microsoft.com/en-us/azure/backup/blob-backup-overview  
- Backup Center (multi-vault management)  
  https://learn.microsoft.com/en-us/azure/backup/backup-center-overview  

---

## Document Change Log

| Date | Version | Changes |
|------|---------|---------|
| 2025-11-21 | 1.1 | Fixed Section 2, 3, 5 quick answers; Added Common Misconceptions section; Added pricing disclaimer |
| 2025-11-17 | 1.0 | Initial document creation |

---

**Document Prepared By:** Technical Solutions Team  
**Last Reviewed:** 2025-11-21  
**Next Review Date:** 2026-02-21

---

**Disclaimer:** This document provides general guidance and approximate pricing based on publicly available Azure pricing as of November 2025. Actual costs may vary based on region, specific configuration, usage patterns, and Azure pricing changes. Always verify current pricing and test in your environment before production deployment.
