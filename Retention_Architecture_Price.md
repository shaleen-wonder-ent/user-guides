# Microsoft Entra ID — Sign-in Log Retention: Cost Estimation

---

## Customer Input Summary

| Parameter                              | Value                                      |
|----------------------------------------|--------------------------------------------|
| Total Users (incl. service accounts)   | 6,50,000+                                  |
| Approx. Sign-in Log Events per Day     | 24–26 Lakhs (~2.5 Million events/day)      |
| Azure Region                           | Central India                              |
| Commitment Tier                        | Not currently in place                     |
| Query Frequency on Historical Data     | Daily to Weekly                            |

---

## Pricing Basis

All unit prices below are derived from the **Azure public pricing page for Central India**
as of 31 March 2026, converted at the **Azure Pricing Calculator rate of ₹90.96 per USD**.

> **Important:** Azure bills Indian subscriptions in INR. The actual billed rate is
> determined by Microsoft using the **London closing spot rate** from the preceding month.
> The INR figures below are therefore **indicative** and may vary slightly on your invoice.
>
> Source: [Azure Monitor Pricing (India)](https://azure.microsoft.com/en-in/pricing/details/monitor/)
> | [Azure Pricing Calculator](https://azure.microsoft.com/en-in/pricing/calculator/)

---

## Volume Estimation

All cost formulas below are built on top of the estimated daily/monthly ingestion volume.

```
Average event size (JSON)         = ~4 KB  (typical range: 3–5 KB)
Events per day                    = 2,500,000  (midpoint of 24–26 Lakhs)

Daily Ingestion Volume            = Events_Per_Day × Avg_Event_Size
                                  = 25,00,000 × 4 KB
                                  = 1,00,00,000 KB
                                  = ~10 GB/day

Monthly Ingestion Volume          = Daily_Volume × 30
                                  = 10 GB × 30
                                  = ~300 GB/month
```

> **Note:** If Non-Interactive Sign-in Logs (`AADNonInteractiveUserSignInLogs`) and
> Service Principal Sign-in Logs are tracked separately and are *not* included in the
> 24–26 Lakh count above, the actual volume could be **1.5×–2× higher**. Please confirm.

---

## Current Public Pricing Reference — Central India (Pay-As-You-Go)

*Source: [Azure Monitor Pricing](https://azure.microsoft.com/en-in/pricing/details/monitor/) and
[Azure Data Lake Storage Pricing](https://azure.microsoft.com/en-in/pricing/details/storage/data-lake/),
accessed 31 March 2026. Converted at ₹90.96/USD.*

| Component                                      | Unit Rate (₹)            |
| ---------------------------------------------- | ------------------------ |
| Analytics Logs — Ingestion                     | **₹209.21 per GB**      |
| Analytics Logs — Retention (first 31 days)     | **₹0.00 (Free)**        |
| Analytics Logs — Interactive Retention (31+ d) | **₹9.10 per GB/month**  |
| Auxiliary Logs — Ingestion                     | **₹4.55 per GB**        |
| Auxiliary Logs — Retention (long-term, 31+ d)  | **₹1.82 per GB/month**  |
| Auxiliary Logs — Query / Search Job            | **₹0.45 per GB scanned**|
| ADLS Gen2 — Storage (Cool Tier, LRS)          | **₹1.18 per GB/month**  |
| ADLS Gen2 — Write Transactions (Cool, per 10K)| **₹5.91 per 10,000 ops**|
| ADLS Gen2 — Read Transactions (Cool, per 10K) | **₹0.59 per 10,000 ops**|
| ADLS Gen2 — Data Retrieval (Cool)             | **₹0.91 per GB**        |

---

# Section 1: Option A — Azure Monitor Auxiliary Logs

**Architecture:** Analytics Logs (Days 1–31) → Auxiliary Logs (Days 31–180)

This option keeps everything **native within Azure Monitor / Log Analytics**. Sign-in logs
are ingested into Analytics Logs for the first 31 days (full KQL query support, alerting,
workbooks), then the same data is also ingested into Auxiliary Logs for long-term retention
up to 180 days at a fraction of the cost.

---

### 1A. Tier 1 — Analytics Logs (Days 1–31)

| Component                        | Unit Price (₹)            | Source                                        |
|----------------------------------|---------------------------|-----------------------------------------------|
| Data Ingestion                   | **₹209.21 per GB**       | [Azure Monitor Pricing](https://azure.microsoft.com/en-in/pricing/details/monitor/) |
| Interactive Retention (≤ 31 days)| **₹0.00 (Free)**         | Included with ingestion                       |

**Formula:**

```
Monthly Ingestion Cost   = Monthly_Volume × Price_Per_GB
                         = 300 GB × ₹209.21
                         = ₹62,763 / month

Retention Cost (31 days) = ₹0.00 (included free)

─────────────────────────────────────────────────
TIER 1 TOTAL             = ₹62,763 / month
─────────────────────────────────────────────────
```

---

### 1B. Tier 2 — Auxiliary Logs (Days 31–180)

| Component                              | Unit Price (₹)            | Source                                        |
|----------------------------------------|---------------------------|-----------------------------------------------|
| Data Ingestion                         | **₹4.55 per GB**         | [Azure Monitor Pricing](https://azure.microsoft.com/en-in/pricing/details/monitor/) |
| Long-term Retention (after 30 days)    | **₹1.82 per GB/month**   | [Azure Monitor Pricing](https://azure.microsoft.com/en-in/pricing/details/monitor/) |
| Query / Search Job (data scanned)      | **₹0.45 per GB scanned** | [Azure Monitor Pricing](https://azure.microsoft.com/en-in/pricing/details/monitor/) |

**Formula — Ingestion:**

```
Monthly Ingestion Cost   = Monthly_Volume × Price_Per_GB
                         = 300 GB × ₹4.55
                         = ₹1,365 / month
```

**Formula — Retention (Steady State):**

At steady state (after 5 months of accumulation), you will have ~5 months × 300 GB = 1,500 GB
stored in long-term retention (days 31–180).

```
Stored Volume (steady state)  = Retention_Months × Monthly_Volume
                              = 5 × 300 GB
                              = 1,500 GB

Monthly Retention Cost        = Stored_Volume × Retention_Price_Per_GB
                              = 1,500 GB × ₹1.82
                              = ₹2,730 / month
```

> **Ramp-up Note:** Retention cost grows linearly over the first 5 months:
>
> | Month | Stored Volume | Retention Cost (₹) |
> |-------|---------------|---------------------|
> | 1     | 300 GB        | ₹546                |
> | 2     | 600 GB        | ₹1,092              |
> | 3     | 900 GB        | ₹1,638              |
> | 4     | 1,200 GB      | ₹2,184              |
> | 5+    | 1,500 GB      | **₹2,730** (steady) |

**Formula — Query Costs:**

Assuming daily queries scanning ~10 GB of data per query execution:

```
Queries per month             = 30  (daily frequency)
Data scanned per query        = ~10 GB  (estimated)

Monthly Query Cost            = Queries × Data_Scanned × Price_Per_GB_Scanned
                              = 30 × 10 GB × ₹0.45
                              = ₹135 / month

(If weekly queries instead: 4 × 10 GB × ₹0.45 = ₹18 / month)
```

**Tier 2 Subtotal (Steady State):**

```
─────────────────────────────────────────────────────────
Ingestion           ₹1,365
Retention           ₹2,730
Query (daily)       ₹135
─────────────────────────────────────────────────────────
TIER 2 TOTAL      = ₹4,230 / month
─────────────────────────────────────────────────────────
```

---

### Option A — Combined Total (Steady State)

```
┌──────────────────────────────────────────────────────────┐
│  Tier 1 (Analytics, 31 days)       =  ₹62,763 / month   │
│  Tier 2 (Auxiliary, 31–180 days)   =   ₹4,230 / month   │
│──────────────────────────────────────────────────────────│
│  OPTION A MONTHLY TOTAL            ≈  ₹66,993 / month   │
│  OPTION A ANNUAL TOTAL             ≈ ₹8,03,916 / year   │
└──────────────────────────────────────────────────────────┘
```

| Component                      | Monthly (₹)  | Annual (₹)    |
| ------------------------------ | ------------- | ------------- |
| Analytics Ingestion (300 GB)   | ₹62,763       | ₹7,53,156     |
| Analytics Retention (31 days)  | ₹0            | ₹0            |
| Auxiliary Ingestion (300 GB)   | ₹1,365        | ₹16,380       |
| Auxiliary Retention (1,500 GB) | ₹2,730        | ₹32,760       |
| Auxiliary Query Cost           | ₹135          | ₹1,620        |
| **TOTAL**                      | **₹66,993**   | **₹8,03,916** |

---

# Section 2: Option B — Azure Data Lake Storage Gen2

**Architecture:** Analytics Logs (Days 1–31) → ADLS Gen2 via Diagnostic Settings (Days 31–180)

This option exports Sign-in Logs to an Azure Data Lake Storage Gen2 account using
**Diagnostic Settings** for long-term retention. Storage costs are lower, but querying
requires **separate tooling** (e.g., Azure Synapse Analytics, Azure Databricks, or
custom scripts). There is **no native KQL query support** on ADLS Gen2.

---

### 2A. Tier 1 — Analytics Logs (Days 1–31)

*Identical to Option A — Tier 1:*

```
─────────────────────────────────────────────────
TIER 1 TOTAL             = ₹62,763 / month
─────────────────────────────────────────────────
```

---

### 2B. Tier 2 — Azure Data Lake Storage Gen2 (Days 31–180, Cool Tier)

| Component                                | Unit Price (₹)                | Source                                              |
|------------------------------------------|-------------------------------|-----------------------------------------------------|
| Export via Diagnostic Settings           | **₹0.00 (no extra charge)**  | Built-in Azure Monitor feature                      |
| Storage — Cool Tier (LRS)               | **₹1.18 per GB/month**       | [ADLS Gen2 Pricing](https://azure.microsoft.com/en-in/pricing/details/storage/data-lake/) |
| Write Transactions (per 10,000 ops)     | **₹5.91**                    | [ADLS Gen2 Pricing](https://azure.microsoft.com/en-in/pricing/details/storage/data-lake/) |
| Read Transactions (per 10,000 ops)      | **₹0.59**                    | [ADLS Gen2 Pricing](https://azure.microsoft.com/en-in/pricing/details/storage/data-lake/) |
| Data Retrieval (Cool)                   | **₹0.91 per GB**             | [ADLS Gen2 Pricing](https://azure.microsoft.com/en-in/pricing/details/storage/data-lake/) |

> **Recommended Tier:** Cool — best suited for data accessed weekly/daily at lower storage cost.

**Formula — Storage (Steady State, Cool Tier):**

```
Stored Volume (steady state)  = Retention_Months × Monthly_Volume
                              = 5 × 300 GB
                              = 1,500 GB

Monthly Storage Cost          = Stored_Volume × Storage_Price_Per_GB
                              = 1,500 GB × ₹1.18
                              = ₹1,770 / month
```

> **Ramp-up Note:**
>
> | Month | Stored Volume | Storage Cost (₹) |
> |-------|---------------|-------------------|
> | 1     | 300 GB        | ₹354              |
> | 2     | 600 GB        | ₹708              |
> | 3     | 900 GB        | ₹1,062            |
> | 4     | 1,200 GB      | ₹1,416            |
> | 5+    | 1,500 GB      | **₹1,770** (steady)|

**Formula — Write Transactions (Ingestion via Diagnostic Settings):**

```
Events per month               = 25,00,000/day × 30 = 7,50,00,000
Write ops (assuming batching)  ≈ 7,50,00,000 / 500 (batch factor) = 1,50,000 ops

Monthly Write Txn Cost         = (Write_Ops / 10,000) × Price_Per_10K
                               = (1,50,000 / 10,000) × ₹5.91
                               = 15 × ₹5.91
                               = ₹88.65 / month  ≈ ₹89 / month
```

> **Note:** Diagnostic Settings batch events before writing. The actual batch factor varies;
> 500 events/write is a reasonable estimate. Actual costs are typically between ₹45–₹180/month.

**Formula — Read Transactions (Querying):**

```
Estimated read ops per month   ≈ 50,000  (daily queries across partitions)

Monthly Read Txn Cost          = (Read_Ops / 10,000) × Price_Per_10K
                               = (50,000 / 10,000) × ₹0.59
                               = 5 × ₹0.59
                               = ₹2.95 / month  ≈ ₹3 / month
```

**Formula — Data Retrieval (Cool Tier):**

```
Data retrieved per month       ≈ 50 GB  (estimated for daily/weekly analysis)

Monthly Retrieval Cost         = Data_Retrieved × ₹0.91/GB
                               = 50 × ₹0.91
                               = ₹45.50 / month  ≈ ₹46 / month
```

**Tier 2 Subtotal (Steady State):**

```
─────────────────────────────────────────────────────────
Storage (Cool)          ₹1,770
Write Transactions      ₹89
Read Transactions       ₹3
Data Retrieval          ₹46
─────────────────────────────────────────────────────────
TIER 2 TOTAL          = ₹1,908 / month
─────────────────────────────────────────────────────────
```

---

### ⚠️ Additional Tooling Cost (Option B Only)

ADLS Gen2 does **not** support native KQL queries. To query sign-in logs stored in
ADLS Gen2 on a daily/weekly basis, you will need one of the following:

| Query Tool               | Approx. Additional Cost (₹)                    |
|--------------------------|-------------------------------------------------|
| Azure Synapse Serverless | ~₹455 per TB scanned (pay-per-query)            |
| Azure Databricks         | Starting ~₹6.37/DBU (depends on cluster config) |
| Microsoft Fabric         | Varies by capacity unit                          |
| Custom scripts (ADF)     | Minimal (pipeline execution costs)               |

**Estimated monthly query tooling cost (Synapse Serverless, daily queries):**

```
Data scanned per query    = ~10 GB
Queries per month         = 30

Monthly Synapse Cost      = (Total_GB_Scanned / 1000) × ₹455
                          = (300 / 1000) × ₹455
                          = ₹136.50 / month  ≈ ₹137 / month
```

---

### Option B — Combined Total (Steady State)

```
┌─────────────────────────────────────────────────────────────┐
│  Tier 1 (Analytics, 31 days)          = ₹62,763 / month    │
│  Tier 2 (ADLS Gen2, 31–180 days)      =  ₹1,908 / month    │
│  Query Tooling (Synapse Serverless)   =    ₹137 / month    │
│─────────────────────────────────────────────────────────────│
│  OPTION B MONTHLY TOTAL               ≈ ₹64,808 / month    │
│  OPTION B ANNUAL TOTAL                ≈ ₹7,77,696 / year   │
└─────────────────────────────────────────────────────────────┘
```

| Component                            | Monthly (₹) | Annual (₹)    |
| ------------------------------------ | ------------ | ------------- |
| Analytics Ingestion (300 GB)         | ₹62,763      | ₹7,53,156     |
| Analytics Retention (31 days)        | ₹0           | ₹0            |
| ADLS Gen2 Storage (1,500 GB Cool)    | ₹1,770       | ₹21,240       |
| ADLS Gen2 Write Transactions         | ₹89          | ₹1,068        |
| ADLS Gen2 Read Transactions          | ₹3           | ₹36           |
| ADLS Gen2 Data Retrieval             | ₹46          | ₹552          |
| Query Tooling (Synapse Serverless)   | ₹137         | ₹1,644        |
| **TOTAL**                            | **₹64,808**  | **₹7,77,696** |

---

# Side-by-Side Comparison

| Aspect                           | Option A (Auxiliary Logs)     | Option B (ADLS Gen2)         |
|----------------------------------|------------------------------|------------------------------|
| **Monthly Cost (steady state)**  | **₹66,993**                  | **₹64,808**                  |
| **Annual Cost**                  | **₹8,03,916**                | **₹7,77,696**                |
| **Cost Difference**              | —                            | ~₹2,185/month cheaper        |
| Native KQL Query Support         | ✅ Yes                        | ❌ No                         |
| Alerting & Workbooks             | ✅ Yes                        | ❌ Requires custom setup      |
| Setup Complexity                 | Low                          | Medium–High                  |
| Additional Tooling Required      | None                         | Synapse / Databricks / Fabric|
| Daily/Weekly Query Support       | ✅ Built-in search jobs       | ⚠️ Requires external tooling  |
| Long-term Compliance (12 yr)     | ✅ Up to 12 years             | ✅ Unlimited                  |
| Operational Overhead             | Low                          | Medium                       |

---

# Key Formulas Summary

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  Daily Volume (GB) = Events/Day × Avg Event Size (KB) ÷ 10,48,576            │
│                                                                              │
│  Monthly Volume (GB) = Daily Volume × 30                                     │
│                                                                              │
│  Analytics Ingestion Cost = Monthly Volume × ₹209.21/GB                      │
│                                                                              │
│  Auxiliary Ingestion Cost = Monthly Volume × ₹4.55/GB                        │
│                                                                              │
│  Auxiliary Retention Cost = Stored Volume (GB) × ₹1.82/GB/month              │
│    where Stored Volume = Monthly Volume × Months Retained (max 5)            │
│                                                                              │
│  Auxiliary Query Cost = Queries/Month × GB Scanned/Query × ₹0.45/GB          │
│                                                                              │
│  ADLS Gen2 Storage Cost = Stored Volume (GB) × ₹1.18/GB/month (Cool)         │
│                                                                              │
│  ADLS Gen2 Write Cost = (Write Ops ÷ 10,000) × ₹5.91                         │
│                                                                              │
│  Total Monthly = Tier 1 (Analytics) + Tier 2 (Auxiliary OR ADLS Gen2)        │
│                                                                              │
│  Total Annual = Total Monthly × 12                                           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

# Recommendation

Given Contoso's requirement for **daily-to-weekly querying** on historical data (days 31–180),
**Option A (Auxiliary Logs)** is recommended despite the ~₹2,185/month premium because:

1. **Zero additional tooling** — KQL search jobs work natively; no Synapse/Databricks
   setup, licensing, or management overhead.
2. **Lower operational overhead** — everything is managed within Azure Monitor from a
   single pane of glass.
3. **Faster incident response** — search jobs can be triggered on-demand without building
   or maintaining ETL pipelines.
4. **Simpler RBAC** — single Log Analytics workspace permissions model.

The ~₹26,220/year difference does **not** justify the additional complexity, licensing,
and operational effort required to maintain a separate ADLS Gen2 + query pipeline.

---

# Important Disclaimers

1. **All prices are indicative** and based on publicly available Azure pricing for the
   **Central India** region as of 31 March 2026 (Pay-As-You-Go). INR amounts are
   converted at the **Azure Pricing Calculator rate of ₹90.96 per USD**.

2. **GST and applicable taxes are NOT included.** India GST on cloud services is
   currently **18%**. To estimate the final billed amount:
   ```
   Final Billed Amount = Estimated Cost × 1.18
   ```
   For example, Option A: ₹66,993 × 1.18 = **~₹79,052/month** (incl. GST).

3. **Exchange rate fluctuation:** Microsoft determines the actual INR billing rate using
   the **London closing spot rate** from the preceding month. The INR figures above may
   vary slightly based on the actual billing month's rate.

4. **Commitment Tiers:** If Contoso's overall Log Analytics ingestion (across all tables)
   exceeds 100 GB/day, a commitment tier would reduce the Analytics Logs ingestion
   rate by 15–30%. This estimate uses Pay-As-You-Go rates.

5. **Log table scope:** This estimate covers `SigninLogs` only. Additional tables
   (`AADNonInteractiveUserSignInLogs`, `AADServicePrincipalSignInLogs`,
   `AADManagedIdentitySignInLogs`) will increase volumes and costs proportionally.
   Apply the scaling formula:
   ```
   Adjusted Cost = Base Cost × (Actual Daily GB ÷ 10 GB)
   ```

6. For official pricing and contractual rates, please consult your **Microsoft Account
   Representative** or use the [Azure Pricing Calculator](https://azure.microsoft.com/en-in/pricing/calculator/).

---

# References

| # | Resource | URL |
|---|----------|-----|
| 1 | Azure Monitor Pricing — India | https://azure.microsoft.com/en-in/pricing/details/monitor/ |
| 2 | Azure Pricing Calculator — India | https://azure.microsoft.com/en-in/pricing/calculator/ |
| 3 | Azure Monitor Logs — Cost Calculations | https://learn.microsoft.com/en-us/azure/azure-monitor/logs/cost-logs |
| 4 | ADLS Gen2 Pricing — India | https://azure.microsoft.com/en-in/pricing/details/storage/data-lake/ |
| 5 | Azure Monitor Auxiliary Logs Docs | https://learn.microsoft.com/en-us/azure/azure-monitor/logs/auxiliary-logs |
| 6 | Microsoft Sentinel Multi-Tier Logging | https://charbelnemnom.com/microsoft-sentinel-multi-tier-logging/ |

---

