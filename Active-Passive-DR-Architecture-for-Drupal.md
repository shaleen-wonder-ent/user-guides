# Active-Passive DR Architecture for Drupal (Single VM with Managed Disk, Managed MySQL, F5 Global WAF)

## 1. Overview

This document describes an **Active-Passive Disaster Recovery (DR) architecture** for a Drupal website with a single VM per region (Central India as PRIMARY, South India as DR Standby), managed Azure Database for MySQL, and F5 WAF at the global level. File storage uses a managed disk attached to the VM; DR is based on Azure Site Recovery (ASR) for the VM and disk, and using Azure Database for MySQL geo-replication for the database.

---

## 2. Architecture Diagram

<img width="826" height="524" alt="image" src="https://github.com/user-attachments/assets/840155bd-059e-4f0d-8090-69e151aa191d" />

---

## 3. Traffic & Failover Flow

- **User** traffic goes to **F5 WAF (global)** for inbound protection.
- **F5** forwards to **Azure Traffic Manager (ATM)**.
- **ATM** directs sessions to **Central India VM** (active).
- **Site Recovery (ASR):** Replicates the web VM and managed disk from CI to SI.
- **Database DR:** Uses native Azure Database for MySQL geo-replica in SI for DR.
- On failover (manual/ATM-detected health fail), ATM routes to the DR VM in South India.
    - Web VM is brought up in SI from ASR replica.
    - DB replica is promoted to primary during failover (manual or automated).

---

## 4. Database DR/Replication (Azure DB for MySQL)

- **Primary (read/write):** Azure DB for MySQL in Central India.
- **Read Replica:** Azure DB for MySQL in South India; created using portal/CLI.
- **Replication type:** Asynchronous geo-replication.
- **Normal operation:** All writes/reads in CI.
- **DR event:** Promote SI replica to be primary (manual/automated).
- **RPO:** < 5 min typical (depends on workload/network; monitor in Azure Portal).
- **RTO:** A few minutes (depends on failover automation and DR procedures).
- **Failback:** After CI recovers, sync SI to CI, reestablish normal primary/replica roles.

---

## 5. Azure Database for MySQL – Cross-Region HA/DR: What’s Supported and Not Supported

### Supported
- **Cross-region Read Replicas**
    - Asynchronous replication
    - Read-only replicas (multiple allowed)
    - Manual promotion of replica to primary

- **Zone-Redundant High Availability (within region)**
    - Automatic failover **within** region/availability zone
    - Protects from AZ failure (NOT regional outage)

###  NOT Supported
- **Automatic failover across regions**
    - No automatic region failover
    - No automated promotion of the replica
    - No automatic redirection of connections

- **Multi-region Active-Active writes**
    - Not supported; only single primary for writes

- **Synchronous replication across regions**
    - Not supported; all cross-region replication is async

### What Happens During Region Failure?

Example:  
- Primary = Central India  
- Replica = South India  
If Central India fails completely:
  1. **You MUST do these manual steps:**
      - Promote the South India read-replica → becomes new primary  
      - Update DB connection string in Drupal/webapp to point to new primary endpoint  
      - Restart/reinitialize Drupal app as necessary  
      - Optionally, create new replica(s) for failback after primary recovers

### What About "Active-Passive"?

- In this model, only the **primary region** (Central India) handles all active workloads and database writes.
- The **secondary (DR) region** (South India) is maintained as a **read-only replica** (for database) and a cold or warm copy (for VM/files).
- **During normal operation:**  
    - All user/database traffic and app writes occur in Central India.
    - The South India resources are on standby (for VM: replicated via ASR; for DB: read replica only).
- **During DR/failover:**  
    - You **manually promote** the South India DB replica to primary.
    - You **activate** the replicated VM/disks in South India via ASR.
    - You update connection strings/traffic manager to point to the new region.
- Once the original region recovers, you can perform a planned failback.

---

## 6. File Storage

- Files reside on managed disk, replicated via ASR.
- After failover, the DR VM in SI has a copy as of the last ASR sync (within RPO window).
- **Note:** Some file loss could occur if disk changes were not yet replicated when failover occurs.

---

## 7. Cutover/Failover Process

- Azure Traffic Manager monitors endpoint (health probe).
- On failover:
    1. Trigger ASR recovery to bring up SI VM and disk.
    2. Promote MySQL replica in SI to primary.
    3. Update application's DB connection details if needed.
    4. ATM redirects new sessions to SI region/application.
- **Failback:** Manual/scheduled after CI is restored; databases may need reconciliation.

---

## 8. Pros & Cons

### Pros

- Simple, reliable DR with cloud-native failover.
- No app code/storage changes.
- Fast recovery, minimal data loss if sync is frequent.

### Cons

- Risk of small RPO (data written just before DR may be lost).
- Cutover time includes bringing up standby VM, DB promotion, & potential DNS/ATM reroute delay.
- Not suitable if you require zero-RPO failover.

---


