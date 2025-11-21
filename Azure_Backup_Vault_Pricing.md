# Vault Backup Pricing Explanation

## Which Components Are Chargeable?

### 1. Source Storage Account
- **Chargeable:** Yes
- **Why:** Standard blob storage + incremental snapshot block storage.

### 2. Azure Backup Service (Protected Instance Fee)
- **Chargeable:** Yes
- **Why:** A flat monthly fee per protected storage account.

### 3. Blob Snapshot (Source)
- **Chargeable:** Yes
- **Why:** Stored as incremental changed blocks in the source storage account.

### 4. Snapshot Data Transfer
- **Chargeable:** Yes (only if cross‑region)
- **Why:** Cross‑region egress applies if the vault is in a different region.

### 5. Backup Vault Storage
- **Chargeable:** Yes — **main cost component**
- **Why:** Each Recovery Point stored at **full logical size** of the protected dataset.

### 6. Recovery Points in Vault
- **Chargeable:** Yes
- **Why:** Each Recovery Point is billed for every day it exists.

---

## Key Billing Rule

Azure Backup Vault bills **per Recovery Point**, based on the **logical size** of the protected storage account at the time of that backup.

```
Vault Storage Cost = Logical Size (GB) × Rate (₹/GB‑month) × (Days Stored / 30)
```

No flat fee for vault storage.  
Only the Protected Instance has a flat monthly fee.

---

## Example Scenario (Just for Valut)

**Storage Account Growth**
- Day 1 → 10 TB (10,240 GB)
- Day 2 → 12 TB (12,288 GB)
- Day 3 → 14 TB (14,336 GB)

Vault rate example: **₹1.80 per GB-month**

### Day‑wise Vault Cost

| Day | Logical Size | Days Billed | Monthly Cost | Pro‑rated Cost |
|-----|--------------|-------------|--------------|----------------|
| Day 1 | 10 TB | 30 days | ₹18,432 | ₹18,432 |
| Day 2 | 12 TB | 29 days | ₹22,118.40 | ₹21,381.12 |
| Day 3 | 14 TB | 28 days | ₹25,804.80 | ₹24,084.48 |

### Total Monthly Vault Cost
```
₹18,432 + ₹21,381.12 + ₹24,084.48 = ₹63,897.60
```

---


# Example Scenario (3 Backups, 30 day retention - complete)

## Assumptions
- Vault storage rate = ₹1.9758 per GB-month  
- Source snapshot rate = ₹1.6230 per GB-month  
- Protected instance fee = ₹1234.90 per month  
- 1 TB = 1024 GB  
- Month = 30 days  

## Recovery points
- Day 1 → 10 TB = 10,240 GB  
- Day 2 → 12 TB = 12,288 GB  
- Day 3 → 14 TB = 14,336 GB  

## Vault cost calculations (digit by digit)

| Recovery Point | Size (GB) | Full month cost (₹) | Days billed | Pro-rated cost (₹) |
|----------------|-----------|----------------------|-------------|---------------------|
| RP1 | 10,240 | 20,232.68 | 30 | 20,232.68 |
| RP2 | 12,288 | 24,279.22 | 29 | 23,469.91 |
| RP3 | 14,336 | 28,325.76 | 28 | 26,437.37 |

**Vault subtotal (sum of pro-rated RP costs) = ₹70,139.97**

---

## Source snapshot (incremental) cost calculations

Assumed incremental data stored at source:
- Day 1 incremental = 10 TB = 10,240 GB  
- Day 2 incremental = 2 TB = 2,048 GB  
- Day 3 incremental = 2 TB = 2,048 GB  

| Day | Incremental GB | Full month cost (₹) | Days billed | Pro-rated cost (₹) |
|-----|----------------|----------------------|-------------|---------------------|
| Day1 | 10,240 | 16,619.70 | 30 | 16,619.70 |
| Day2 | 2,048 | 3,323.94 | 29 | 3,213.14 |
| Day3 | 2,048 | 3,323.94 | 28 | 3,102.34 |

**Source snapshot subtotal = ₹22,935.19**

---

## Protected instance fee
Flat monthly protected instance fee = **₹1234.90**

# Total Monthly Cost (Vault + Source snapshot + Protected instance)

- Vault subtotal = ₹70,139.97  
- Source snapshot subtotal = ₹22,935.19  
- Protected instance fee = ₹1234.90  

# **Total monthly cost = ₹94,310.07**



---

## Notes and Recommendations

- Vault is the dominant cost because each recovery point is billed at the full logical size.
- To reduce cost consider:
  - Reduce daily recovery point count (e.g., weekly for older data)
  - Tier retention (daily -> weekly -> monthly)
  - Move rarely changed data to cheaper storage pre-protection
  - Use archive or tiering if supported for backup vault
- Replace the example rates above with your actual Azure region rates for precise quotes.

## Final Summary

- Vault storage cost is **based entirely on Recovery Point logical size**.
- Every backup run creates a **new Recovery Point**.
- Each Recovery Point is billed **for the number of days it exists**.
- **No flat vault fee** — only logical GB‑based metering.
- Protected Instance Fee is separate and flat per month.

---

## One-Line Formula

```
Vault Storage Fee = Logical Size (GB) × (Vault Rate / 30) × Days Stored
```

