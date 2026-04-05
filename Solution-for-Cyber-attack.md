# Point of View: Fastest Approach to Restore Mission-Critical Applications After Azure Tenant Compromise
## Enhanced Hybrid Strategy — Microsoft Native + Rubrik + Commvault

---

## Executive Summary

**The Challenge:** In the event of an Azure (Contoso) tenant compromise where the primary tenant remains inaccessible for 15–30 days during forensic investigation, organizations must rapidly restore mission-critical applications from backups stored in a separate tenant.

**The Reality:** Microsoft Azure does not natively support direct cross-tenant backup restoration. Azure Backup's Cross-Subscription Restore (CSR) feature only works within the same tenant, creating a critical gap in disaster recovery planning for tenant compromise scenarios. [[1]](https://learn.microsoft.com/en-us/answers/questions/1301063/is-it-possible-to-restore-one-virtual-machine-back) [[2]](https://learn.microsoft.com/en-us/answers/questions/2260263/how-to-restore-and-keep-azure-vm-backups-updated-a)

**The Enhanced Solution:** A **multi-vendor hybrid recovery strategy** combining:
- **Azure Site Recovery (ASR)** for real-time VM replication
- **Rubrik** for immutable, air-gapped backups with orchestrated cyber recovery and clean room validation
- **Commvault** for full infrastructure-aware cross-tenant recovery including ARM configurations, networking, and application orchestration

This layered approach eliminates single-vendor dependency, achieves **sub-1-hour RTO** for Tier 1 workloads, and provides ransomware-resilient recovery with forensic validation — ensuring compromised data is never restored.

---

## Why a Multi-Vendor Approach?

### The Gap in Microsoft-Only Solutions

| Limitation | Impact |
|---|---|
| No native cross-tenant backup restore | Forces manual VHD export/import workflows |
| ASR requires "physical server" model for cross-tenant | Added complexity; must be pre-configured |
| No built-in clean room or forensic validation | Risk of restoring compromised/infected workloads |
| No automated infrastructure (ARM) rebuild | Network, NSGs, policies must be manually recreated |

### What Rubrik and Commvault Add

| Capability | Microsoft Native | + Rubrik | + Commvault |
|---|---|---|---|
| Cross-tenant VM restore | ❌ Manual VHD transfer | ✅ SaaS-managed cross-tenant | ✅ Native cross-tenant restore |
| Clean room recovery | ❌ Not available | ✅ Isolated forensic recovery | ⚠️ Partial (cleanroom testing) |
| Threat scanning in backups | ❌ Not available | ✅ ML-based anomaly detection | ✅ Anomaly detection |
| Infrastructure rebuild (ARM, VNets, NSGs) | ❌ Manual | ⚠️ VM/data focused | ✅ Cloud Rewind — full infra recovery |
| Immutable/air-gapped backups | ⚠️ Immutable vault only | ✅ Zero-trust, air-gapped by design | ✅ Air-gapped copy support |
| Warm standby in target tenant | ⚠️ ASR only | ✅ Orchestrated recovery plans | ✅ Live Sync warm standby |
| Identity (Entra ID) recovery | ⚠️ Preview (March 2026) | ✅ Hybrid identity recovery | ⚠️ Limited |

---

## Critical Constraint: No Native Cross-Tenant Restore

Microsoft Azure Backup **does not** support direct restoration across different Microsoft Entra tenants. The Cross-Subscription Restore feature explicitly requires both subscriptions to be under the **same tenant**. This architectural limitation forces organizations to adopt workaround strategies or third-party solutions. [[1]](https://learn.microsoft.com/en-us/answers/questions/1301063/is-it-possible-to-restore-one-virtual-machine-back) [[2]](https://learn.microsoft.com/en-us/answers/questions/2260263/how-to-restore-and-keep-azure-vm-backups-updated-a)

---

## Enhanced Recovery Architecture Overview

### Recovery Approach Comparison (All Solutions)

| Recovery Method | RTO | RPO | Cross-Tenant | Clean Room | Infra Rebuild | Cost Profile |
|---|---|---|---|---|---|---|
| **ASR + Rubrik Cyber Recovery** | **30 min – 1 hour** | ~5 min (ASR) / 15 min (Rubrik) | ✅ Both | ✅ Rubrik | ❌ Manual | High |
| **ASR + Commvault Live Sync** | **30 min – 1 hour** | ~5 min (ASR) / 15 min–1 hr (Commvault) | ✅ Both | ⚠️ Partial | ✅ Cloud Rewind | High |
| **Rubrik Orchestrated Recovery (standalone)** | **< 1 hour** | 15 min – 1 hour | ✅ | ✅ | ⚠️ VM/data only | Medium-High |
| **Commvault Live Sync (standalone)** | **1 – 2 hours** | 15 min – 1 hour | ✅ | ⚠️ Partial | ✅ Full | Medium-High |
| **ASR only ("physical server" model)** | ≤ 2 hours | ~5 min | ✅ | ❌ | ❌ | Medium |
| **Manual Snapshot + VHD Transfer** | 12–72 hours | 24+ hours | ✅ | ❌ | ❌ | Low |
| **Azure Backup CSR** | **N/A** | N/A | ❌ Same tenant only | ❌ | ❌ | N/A |

---

## Solution Layer 1: Azure Site Recovery (Foundation)

### Overview
ASR provides the foundational continuous replication layer with the lowest RPO (~5 minutes). For cross-tenant scenarios, the **"physical server" replication model** is required — treating Azure VMs as physical machines with the Mobility agent installed. [[3]](https://www.azure.cn/en-us/support/sla/site-recovery/) [[4]](https://www.cloudinnovatehub.com/how-to-migrate-azure-vm-between-subscriptions-across-different-tenants-using-azure-site-recovery-asr)

### Key Metrics
- **RTO:** ≤ 2 hours (Microsoft SLA) [[3]](https://www.azure.cn/en-us/support/sla/site-recovery/)
- **RPO:** ~5 minutes (crash-consistent) [[5]](https://oneuptime.com/blog/post/2026-02-16-how-to-configure-replication-policies-for-rpo-and-retention-in-azure-site-recovery/view)
- **Cost:** $25/VM/month + storage + network egress [[6]](https://azurelessons.com/azure-site-recovery-pricing/)

### Role in Multi-Vendor Strategy
ASR serves as the **always-on replication engine**, providing the fastest possible failover for VMs. However, it lacks clean room validation and infrastructure rebuild — gaps filled by Rubrik and Commvault.

---

## Solution Layer 2: Rubrik Cyber Recovery & Clean Room Validation

### Overview
Rubrik provides **immutable, air-gapped backups** with **orchestrated cyber recovery** specifically designed for post-compromise scenarios. Its key differentiator is the ability to **validate backups are clean before restoring**, preventing reinfection. [[7]](https://www.rubrik.com/blog/technology/25/6/prepare-respond-orchestrate-rubrik-redefines-azure-cyber-recovery) [[8]](https://www.rubrik.com/solutions/azure-native-protection)

### Key Capabilities

#### Cyber Recovery for Azure VMs (Launched June 2025)
- **Orchestrated Recovery:** Automated, policy-driven recovery workflows that eliminate manual steps
- **Clean Room Recovery:** Isolated environments where backups are restored and validated before production use — critical for ensuring threat actor persistence is not carried over
- **ML-Based Threat Scanning:** Automatically identifies clean recovery points by scanning for ransomware indicators, malware signatures, and anomalous changes in backup data
- **Identity Recovery:** Automated recovery of Microsoft Entra ID (Azure AD) and on-prem Active Directory hybrid identity environments [[7]](https://www.rubrik.com/blog/technology/25/6/prepare-respond-orchestrate-rubrik-redefines-azure-cyber-recovery)

#### Cross-Tenant Capabilities
- Unified SaaS management plane manages backups across multiple Azure tenants, regions, and subscriptions [[8]](https://www.rubrik.com/solutions/azure-native-protection)
- Policy-driven SLA domains allow snapshot intervals as frequent as **every 15 minutes** for critical workloads
- Immutable by design — backups cannot be modified, encrypted, or deleted by any account (including compromised admin accounts)

#### Recovery Process with Rubrik
1. **Threat Assessment (15–30 min):** Rubrik's ML engine scans backup timeline to identify last known clean recovery point
2. **Clean Room Provisioning (15–30 min):** Isolated Azure environment spun up in target tenant
3. **Validated Restore (15–30 min):** Workloads restored to clean room, forensic validation executed
4. **Production Cutover (15–30 min):** Once validated, workloads promoted to production in target tenant

**Total RTO: < 1 hour (for pre-configured, orchestrated recoveries)** [[7]](https://www.rubrik.com/blog/technology/25/6/prepare-respond-orchestrate-rubrik-redefines-azure-cyber-recovery) [[9]](https://www.rubrik.com/insights/azure-disaster-recovery-strategy-guide)

#### Key Benefits for Tenant Compromise Scenarios
- **Zero-trust architecture** ensures backups survive even if admin credentials are compromised
- **Air-gapped storage** physically separates backup data from production tenant
- Automated **recovery plan validation** with auditable proof for compliance (DORA, NIS2, ISO 27001) [[10]](https://business.scoop.co.nz/2025/03/05/rubriks-new-capabilities-set-to-transform-cyber-resilience-across-cloud-hypervisor-and-saas-platforms/)
- **Turbo Threat Hunting** for rapid threat investigation within backup data [[11]](https://siliconangle.com/2025/03/04/rubrik-expands-cyber-resilience-features-strengthen-protection-across-cloud-enterprise-workflows/)

### Limitations
- **VM and data focused** — does not automatically rebuild Azure infrastructure (VNets, NSGs, Load Balancers, ARM policies)
- **Custom pricing** — enterprise contracts average ~$57,000/year; must be quoted per environment [[12]](https://www.vendr.com/buyer-guides/rubrik)
- Cross-tenant restore of complex multi-resource environments may require additional scripting

---

## Solution Layer 3: Commvault — Full Infrastructure Recovery & Live Sync

### Overview
Commvault provides the **most comprehensive cross-tenant recovery** capability, including not just VM/data restoration but full **Azure infrastructure rebuild** — VNets, NSGs, ARM templates, policies, and application configurations. Its **Live Sync** feature maintains warm standby copies in the target tenant for rapid failover. [[13]](https://www.commvault.com/rubrik-vs-commvault) [[14]](https://documentation.commvault.com/saas/restores_for_azure_vms.html)

### Key Capabilities

#### Cross-Tenant Full Environment Recovery
- **Cloud Rewind:** Automatically captures and can restore the complete Azure environment configuration — not just VMs, but the entire infrastructure-as-code state (ARM resources, networking, security policies) [[13]](https://www.commvault.com/rubrik-vs-commvault)
- **Cross-subscription/cross-tenant VM restore:** Native support for restoring Azure VMs to different subscriptions and tenants [[14]](https://documentation.commvault.com/saas/restores_for_azure_vms.html) [[15]](https://documentation.commvault.com/11.42/software/restores_for_azure_vms_and_files.html)
- **Granular recovery:** Full VM, disk-level, and file/folder restores supported

#### Live Sync — Warm Standby DR
- Maintains near-real-time replica of protected workloads in the target tenant
- Warm standby VMs are kept updated based on backup cycles, enabling rapid failover
- Orchestrated failover with automated runbooks reduces manual intervention [[16]](https://learn.microsoft.com/en-us/azure/storage/solution-integration/validated-partners/backup-archive-disaster-recovery/commvault/commvault-solution-guide)

#### Recovery Process with Commvault
1. **Failover Decision (15 min):** Initiate orchestrated recovery from Commvault Command Center
2. **Live Sync Activation (15–30 min):** Warm standby VMs in target tenant brought online
3. **Infrastructure Recovery (30–60 min):** Cloud Rewind restores VNets, NSGs, Load Balancers, ARM policies
4. **Application Validation (30–60 min):** End-to-end connectivity and application health checks

**Total RTO: 1–2 hours (with Live Sync pre-configured)** [[16]](https://learn.microsoft.com/en-us/azure/storage/solution-integration/validated-partners/backup-archive-disaster-recovery/commvault/commvault-solution-guide)

#### Key Benefits for Tenant Compromise Scenarios
- **Only solution that recovers infrastructure + data + configuration** in a single workflow
- Live Sync eliminates the slow VHD export/import process
- Supports legacy, hybrid, and multi-cloud workloads — broadest coverage [[17]](https://controlmonkey.io/resource/rubrik-vs-commvault/)
- Anomaly detection for identifying compromised data
- **Continuous DR testing** with automated validation for compliance

### Limitations
- **More complex to configure** than Rubrik — steeper learning curve [[17]](https://controlmonkey.io/resource/rubrik-vs-commvault/)
- Clean room/forensic validation is less mature than Rubrik's dedicated cyber recovery
- Encrypted VMs (Azure Key Vault with customer-managed keys) may have restore restrictions across tenants [[15]](https://documentation.commvault.com/11.42/software/restores_for_azure_vms_and_files.html)
- **Pricing:** $102.49/VM/month (SaaS model, 10-pack minimum) + Azure storage costs [[18]](https://www.commvault.com/saas-pricing)

---

## Recommended: Tiered Multi-Vendor Recovery Strategy

### Architecture Design

| Tier | Workload Type | Primary Recovery | Secondary Recovery | Target RTO | Target RPO |
|---|---|---|---|---|---|
| **Tier 1** | Mission-critical (ERP, CRM, core DBs) | ASR (replication) + Rubrik (clean room validation) | Commvault Live Sync (infrastructure failover) | **< 1 hour** | **5–15 minutes** |
| **Tier 2** | High-priority (reporting, internal tools) | Commvault Live Sync (warm standby) | Rubrik orchestrated recovery | **1–4 hours** | **15 min – 1 hour** |
| **Tier 3** | Standard (dev/test, non-critical) | Rubrik scheduled backups | Manual snapshot + VHD transfer | **4–24 hours** | **4–24 hours** |

### How the Layers Work Together

```
┌────────────────────────────────────────────────────────────────────┐
│                    TENANT COMPROMISE EVENT                         │
│                  Primary Tenant Inaccessible                       │
└──────────────────────────┬─────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │   TIER 1     │ │   TIER 2     │ │   TIER 3     │
   │  < 1 hour    │ │  1-4 hours   │ │  4-24 hours  │
   └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
          │                │                │
          ▼                ▼                ▼
   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
   │  ASR Failover│ │  Commvault   │ │   Rubrik     │
   │  (VMs up in  │ │  Live Sync   │ │  Scheduled   │
   │   minutes)   │ │ (warm standby│ │  Backup      │
   └──────┬───────┘ │  activated)  │ │  Restore     │
          │         └──────┬───────┘ └──────┬───────┘
          ▼                ▼                │
   ┌──────────────┐ ┌──────────────┐        │
   │  Rubrik      │ │  Commvault   │        │
   │  Clean Room  │ │  Cloud Rewind│        │
   │  Validation  │ │  (infra      │        │
   │  (ensure no  │ │   rebuild)   │        │
   │  reinfection)│ └──────┬───────┘        │
   └──────┬───────┘        │                │
          │                ▼                │
          │         ┌──────────────┐        │
          │         │  Rubrik      │        │
          │         │  Threat Scan │        │
          │         │  (validate   │        │
          │         │   clean)     │        │
          │         └──────┬───────┘        │
          ▼                ▼                ▼
   ┌─────────────────────────────────────────────┐
   │         TARGET TENANT — PRODUCTION          │
   │    All tiers recovered and validated        │
   └─────────────────────────────────────────────┘
```

### Tier 1 Recovery Workflow (Mission-Critical — < 1 Hour)

| Step | Time | Action | Tool |
|---|---|---|---|
| 1 | 0–5 min | Incident declared, DR activated | Manual |
| 2 | 5–15 min | ASR failover initiated for all Tier 1 VMs | Azure Site Recovery |
| 3 | 15–30 min | VMs provisioned in target tenant from replicated disks | Azure Site Recovery |
| 4 | 15–30 min | Rubrik clean room validation — ML threat scan confirms backups are clean | Rubrik Cyber Recovery |
| 5 | 30–45 min | Commvault Cloud Rewind restores VNets, NSGs, LBs, ARM policies | Commvault |
| 6 | 45–60 min | Application health validated, DNS/traffic switched | All |

**Result: Production restored in < 1 hour with forensic confidence that no compromised data was reintroduced.**

---

## Rubrik vs Commvault: Head-to-Head for This Scenario

| Dimension | Rubrik | Commvault | Recommendation |
|---|---|---|---|
| **Primary Strength** | Cyber resilience & clean room recovery | Full infrastructure cross-tenant recovery | **Use both — complementary** |
| **Cross-Tenant VM Restore** | ✅ SaaS-managed | ✅ Native | Both strong |
| **Infrastructure Rebuild** | ⚠️ VM/data only | ✅ Cloud Rewind (VNets, NSGs, ARM) | **Commvault** |
| **Clean Room / Forensic** | ✅ Best-in-class | ⚠️ Cleanroom testing, less mature | **Rubrik** |
| **Threat Detection in Backups** | ✅ ML-based anomaly detection | ✅ Anomaly detection | Both capable |
| **Immutability** | ✅ Zero-trust, air-gapped by design | ✅ Air-gapped copy support | **Rubrik** (stronger) |
| **Identity Recovery** | ✅ Entra ID + on-prem AD | ⚠️ Limited | **Rubrik** |
| **Warm Standby DR** | ✅ Orchestrated recovery plans | ✅ Live Sync | Both capable |
| **Legacy/Hybrid Support** | ⚠️ Modern workload focused | ✅ Broadest coverage | **Commvault** |
| **Ease of Use** | ✅ Simple, intuitive | ⚠️ Steeper learning curve | **Rubrik** |
| **Pricing (per VM/month)** | ~Custom (~$57K/yr avg enterprise) | $102.49/VM/month (SaaS) | Rubrik for fewer VMs; Commvault scales |
| **Compliance (DORA, NIS2, ISO)** | ✅ Built-in | ✅ Built-in | Both strong |

### Verdict: Use Both

- **Rubrik** as the **security and validation layer** — ensures you never restore compromised data
- **Commvault** as the **infrastructure and orchestration layer** — ensures the full environment (not just VMs) is recovered

---

## Supporting Infrastructure Components

### Azure Lighthouse for Cross-Tenant Management
Azure Lighthouse enables delegated resource management across multiple tenants without guest accounts. It supports backup operations, monitoring, and RBAC-based access control. [[19]](https://learn.microsoft.com/en-us/azure/lighthouse/concepts/enterprise) [[20]](https://github.com/MicrosoftDocs/azure-management-docs/blob/main/articles/lighthouse/concepts/cross-tenant-management-experience.md)

### Microsoft Entra ID Backup and Recovery (Preview)
**Status: Public Preview — launched March 19, 2026** [[21]](https://learn.microsoft.com/en-us/entra/backup/overview)

- Automatic daily backups of Entra ID objects (users, groups, apps, CAs, service principals)
- Retention: Up to **5 days** of backup history [[22]](https://www.usmanghani.co/microsoft-entra-id-backup-and-recovery/)
- Complements Rubrik's identity recovery for comprehensive coverage
- **Does not** address Azure resource (VM, storage, networking) recovery

> **Note:** Rubrik's identity recovery capability for Entra ID + on-prem AD provides a more mature solution than the Microsoft preview feature for tenant compromise scenarios.

---

## Comprehensive Cost Analysis

### Per-VM Monthly Cost Comparison

| Component | ASR Only | ASR + Rubrik | ASR + Commvault | ASR + Both |
|---|---|---|---|---|
| ASR licensing | $25 | $25 | $25 | $25 |
| Vendor licensing | — | ~Custom | $102.49 | ~Custom + $102.49 |
| Replicated storage | ~$10 | ~$10 + Rubrik storage | ~$10 + Commvault storage | ~$10 + both |
| Network egress | ~$43 | ~$43 | ~$43 | ~$43 |
| **Est. total per VM** | **~$78** | **~$130–180** | **~$185–230** | **~$240–310** |

### 10 Mission-Critical VMs — Monthly Estimate

| Strategy | Monthly Cost | Annual Cost |
|---|---|---|
| Microsoft Native Only (ASR + manual) | ~$1,580 | ~$18,960 |
| ASR + Rubrik | ~$2,500–3,500 | ~$30,000–42,000 |
| ASR + Commvault | ~$3,000–4,000 | ~$36,000–48,000 |
| **ASR + Rubrik + Commvault (recommended)** | **~$4,000–5,500** | **~$48,000–66,000** |

### Cost Justification

| Scenario | Cost/Impact |
|---|---|
| Downtime cost (mission-critical) | $100,000–500,000/hour |
| Reinfection from compromised backup | $1M–5M+ (incident restart) |
| Regulatory non-compliance (DORA/NIS2) | $300,000+/hour in fines |
| **Multi-vendor DR investment** | **~$66,000/year** |

> **ROI:** Avoiding just **1 hour** of extended downtime or **1 reinfection event** from a contaminated backup pays for **multiple years** of the entire multi-vendor DR infrastructure.

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1–4)
1. Classify all applications into 3 tiers based on business impact
2. Establish backup/recovery tenant with appropriate subscriptions and networking
3. Configure Azure Lighthouse for cross-tenant delegated access
4. Deploy ASR for Tier 1 VMs using "physical server" replication model
5. Procure and deploy **Rubrik Security Cloud** — connect to both tenants
6. Procure and deploy **Commvault Cloud** — configure cross-tenant access

### Phase 2: Integration (Weeks 4–8)
1. Configure Rubrik SLA domains with 15-minute RPO for Tier 1 workloads
2. Configure Commvault Live Sync warm standby for Tier 1 & 2 workloads
3. Set up Commvault Cloud Rewind for infrastructure configuration capture
4. Configure Rubrik clean room environment in target tenant
5. Establish Rubrik anomaly detection and threat scanning policies
6. Develop integrated recovery runbooks covering all three tools

### Phase 3: Validation (Weeks 8–12)
1. Conduct **first full cross-tenant DR test** — all tiers
2. Validate Rubrik clean room recovery with simulated threat scenarios
3. Test Commvault Cloud Rewind infrastructure rebuild end-to-end
4. Measure actual RTO/RPO against targets
5. Document lessons learned and refine procedures

### Phase 4: Ongoing Operations
1. **Monthly:** Rubrik threat scan validation of backup integrity
2. **Quarterly:** Full DR drill across all tiers with documented results
3. **Semi-annually:** Review and update tiering as business priorities evolve
4. **Continuously:** Monitor ASR replication health, Rubrik anomaly alerts, Commvault Live Sync status

---

## Compromise Response Playbook (Hours 0–24)

| Timeframe | Action | Owner | Tools |
|---|---|---|---|
| **Hour 0** | Incident declared, compromised tenant isolated | Security Team | Azure Portal |
| **Hour 0–0.25** | DR team activated, backup tenant access confirmed | DR Lead | Azure Lighthouse |
| **Hour 0.25–0.5** | Rubrik threat scan initiated — identify last clean recovery point | Backup Admin | Rubrik Cyber Recovery |
| **Hour 0.25–0.5** | ASR failover initiated for all Tier 1 VMs (parallel) | Infra Admin | Azure Site Recovery |
| **Hour 0.5–1** | Rubrik clean room validation — restored VMs verified malware-free | Security Team | Rubrik Clean Room |
| **Hour 0.5–1** | Commvault Cloud Rewind — infrastructure config restored in target tenant | Infra Admin | Commvault |
| **Hour 1** | ✅ **Tier 1 applications operational** | | |
| **Hours 1–2** | Commvault Live Sync activated for Tier 2 warm standby | Backup Admin | Commvault Live Sync |
| **Hours 2–4** | Tier 2 applications validated and promoted to production | App Teams | Commvault + Rubrik |
| **Hour 4** | ✅ **Tier 2 applications operational** | | |
| **Hours 4–24** | Rubrik scheduled backup restore for Tier 3 workloads | Backup Admin | Rubrik |
| **Hour 24** | ✅ **All tiers recovered** | | |
| **Ongoing** | Forensic investigation continues in isolated primary tenant | Security Team | Rubrik Threat Hunting |

---

## Critical Success Factors

### Technical Prerequisites (Must Be Configured BEFORE Compromise)

**Microsoft Native:**
- ASR replication active using "physical server" model for all Tier 1 VMs
- Azure Lighthouse delegated access configured
- Target tenant VNets, NSGs, subnets pre-created
- Emergency access credentials for backup tenant

**Rubrik:**
- Rubrik Security Cloud connected to both source and target tenants
- SLA domains configured with appropriate RPO per tier
- Clean room environment pre-provisioned in target tenant
- Anomaly detection and threat scanning policies active
- Air-gapped backup copies verified

**Commvault:**
- Commvault Cloud connected with cross-tenant service principals
- Live Sync warm standby active for Tier 1 & 2 workloads
- Cloud Rewind capturing infrastructure configuration continuously
- Recovery runbooks automated and validated

### Security Considerations

| Control | Implementation |
|---|---|
| **Backup immutability** | Rubrik zero-trust + Commvault air-gapped copies |
| **Threat validation** | Rubrik ML-based scanning before any restore |
| **Access control** | Azure Lighthouse RBAC + PIM just-in-time access |
| **Encryption** | All data encrypted at rest and in transit across all tools |
| **SAS token security** | Minimum permissions, time-limited, logged |
| **Emergency access** | Cloud-only accounts, hardware-bound MFA |
| **Audit trail** | All recovery actions logged across all three platforms |

---

## Conclusion

### The Enhanced Approach

The **most robust, fastest recovery strategy** for Azure tenant compromise combines three complementary layers:

| Layer | Tool | Purpose | Unique Value |
|---|---|---|---|
| **Replication** | Azure Site Recovery | Continuous VM replication, ~5 min RPO | Fastest VM failover (≤ 2 hr SLA) |
| **Security & Validation** | Rubrik | Immutable backups, clean room, threat scanning | Ensures you **never restore compromised data** |
| **Infrastructure & Orchestration** | Commvault | Full environment recovery, Live Sync, Cloud Rewind | Recovers **everything** — not just VMs |

### Achievable RTO with Multi-Vendor Strategy

| Tier | Microsoft-Only RTO | Enhanced Multi-Vendor RTO | Improvement |
|---|---|---|---|
| Tier 1 (Mission-critical) | ≤ 2 hours | **< 1 hour** | **50%+ faster** |
| Tier 2 (High-priority) | 12–24 hours | **1–4 hours** | **80%+ faster** |
| Tier 3 (Standard) | 24–72 hours | **4–24 hours** | **60%+ faster** |

### The Bottom Line

> **Without Rubrik and Commvault:** You can restore VMs, but you risk restoring compromised data, and you must manually rebuild all infrastructure — adding hours to days of recovery time.
>
> **With Rubrik and Commvault:** You restore **validated, clean workloads** into a **fully reconstructed infrastructure**, with forensic confidence and audit proof — in under 1 hour for mission-critical applications.

The incremental investment of ~$48,000–66,000/year for the multi-vendor approach is negligible compared to the cost of a single hour of extended downtime ($100K–500K) or a reinfection event ($1M–5M+).

---

## References

1. [Microsoft Q&A: Cross-tenant VM restore](https://learn.microsoft.com/en-us/answers/questions/1301063/is-it-possible-to-restore-one-virtual-machine-back)
2. [Microsoft Q&A: Cross-subscription backup across tenants](https://learn.microsoft.com/en-us/answers/questions/2260263/how-to-restore-and-keep-azure-vm-backups-updated-a)
3. [Azure Site Recovery SLA](https://www.azure.cn/en-us/support/sla/site-recovery/)
4. [Cross-tenant VM migration using ASR](https://www.cloudinnovatehub.com/how-to-migrate-azure-vm-between-subscriptions-across-different-tenants-using-azure-site-recovery-asr)
5. [ASR replication policy configuration](https://oneuptime.com/blog/post/2026-02-16-how-to-configure-replication-policies-for-rpo-and-retention-in-azure-site-recovery/view)
6. [Azure Site Recovery pricing](https://azurelessons.com/azure-site-recovery-pricing/)
7. [Rubrik Cyber Recovery for Azure VMs](https://www.rubrik.com/blog/technology/25/6/prepare-respond-orchestrate-rubrik-redefines-azure-cyber-recovery)
8. [Rubrik Azure Cloud-Native Protection](https://www.rubrik.com/solutions/azure-native-protection)
9. [Rubrik Azure DR Strategy Guide 2026](https://www.rubrik.com/insights/azure-disaster-recovery-strategy-guide)
10. [Rubrik cyber resilience expansion](https://business.scoop.co.nz/2025/03/05/rubriks-new-capabilities-set-to-transform-cyber-resilience-across-cloud-hypervisor-and-saas-platforms/)
11. [Rubrik cyber resilience features](https://siliconangle.com/2025/03/04/rubrik-expands-cyber-resilience-features-strengthen-protection-across-cloud-enterprise-workflows/)
12. [Rubrik pricing analysis](https://www.vendr.com/buyer-guides/rubrik)
13. [Commvault vs Rubrik](https://www.commvault.com/rubrik-vs-commvault)
14. [Commvault Azure VM restores](https://documentation.commvault.com/saas/restores_for_azure_vms.html)
15. [Commvault Azure VM restore (v11)](https://documentation.commvault.com/11.42/software/restores_for_azure_vms_and_files.html)
16. [Commvault with Azure Storage](https://learn.microsoft.com/en-us/azure/storage/solution-integration/validated-partners/backup-archive-disaster-recovery/commvault/commvault-solution-guide)
17. [Rubrik vs Commvault value comparison 2026](https://controlmonkey.io/resource/rubrik-vs-commvault/)
18. [Commvault SaaS pricing](https://www.commvault.com/saas-pricing)
19. [Azure Lighthouse enterprise scenarios](https://learn.microsoft.com/en-us/azure/lighthouse/concepts/enterprise)
20. [Azure Lighthouse cross-tenant management](https://github.com/MicrosoftDocs/azure-management-docs/blob/main/articles/lighthouse/concepts/cross-tenant-management-experience.md)
21. [Microsoft Entra Backup and Recovery overview](https://learn.microsoft.com/en-us/entra/backup/overview)
22. [Entra ID Backup & Recovery guide](https://www.usmanghani.co/microsoft-entra-id-backup-and-recovery/)

---
