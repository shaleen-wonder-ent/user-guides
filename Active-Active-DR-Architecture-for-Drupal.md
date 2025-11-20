
# Active-Active DR Architecture for Drupal (Single VM with Managed Disk, Managed MySQL, F5 Global WAF)

## 1. Overview

This document describes an **Active-Active Disaster Recovery (DR) architecture** for a Drupal application that is hosted as a single VM in each of two Azure regions (Central India, South India), where user files are stored on attached managed disks (not Azure Files/Blob), the database is Azure Database for MySQL (managed PaaS), and global web traffic is protected and distributed by an F5 WAF before reaching Azure Traffic Manager.

> **Key characteristics:**  
> - One VM per region for the web tier  
> - **Managed MySQL (Azure Database for MySQL) in each region**  
> - User files (PDF, JPEG, etc.) stored on managed disk  
> - F5 WAF at global level  
> - No Redis/session management needed  
> - Manual or scripted file sync between VMs for user files  
> - Database replication relies on Azure Database for MySQL geo-replication

---

## 2. Architecture Diagram

<img width="785" height="523" alt="image" src="https://github.com/user-attachments/assets/54c17c16-ea14-46c6-a439-a3565c773a07" />

---

## 3. Traffic & Security Flow

- **User** traffic enters via **F5 WAF Global** for WAF & DDoS.
- **F5 WAF** forwards to **Azure Traffic Manager (ATM)**.
- **ATM** uses performance/geo-routing to direct sessions to either **Central India** or **South India**.
- Each region’s **VM** hosts the Drupal site, with a managed disk for files and connects to local Azure Database for MySQL in its region.
- No Azure Application Gateway or Redis layer is required.

---

## 4. File and Database Synchronization

### File Sync (Managed Disk)

- **Managed Disk cannot be natively replicated across regions or shared.**
- **Manual/Scheduled File Sync Needed:**
    - Use `rsync`, `robocopy`, or a 3rd-party file sync agent (e.g., Resilio, Syncthing).
    - Set up secure connections (such as VPN or ExpressRoute between VMs).
    - Schedule sync jobs to run every few minutes/hours.
    - Handle file conflicts (naming/timestamps/locks).
    - Example:
      ```bash
      # On CI VM, sync to SI VM:
      rsync -avz /var/www/drupal/sites/default/files/ user@SIVM:/var/www/drupal/sites/default/files/
      ```
- **Note:** In A-A, if users write to both regions, file conflicts can occur—app/app-admin should define rules: Single-writer or conflict resolution.

### Database Sync (Azure Database for MySQL)

- **Central India DB** operates as Primary; **South India DB** as a Read Replica.
- **Geo-Replication is asynchronous:**
    - All writes occur in the Primary (CI), replicated to Replica (SI).
    - Reads can occur in both, but only Primary supports writes.
    - Failover: Promote the Read Replica in SI as Primary during disaster recovery (manual operation).
    - Typically, application should only write to one region at a time to avoid split-brain (Active-Active with true multi-write is not supported by Azure DB for MySQL).
- **Failover/Promotion:**
    - On CI failure, manually (or scripted) promote SI replica to be the new primary and point app traffic there.

#### Key Points:
- **Replication Lag (RPO):** Seconds-to-minutes depending on workload and network (monitor using Azure metrics).
- **RTO:** Failover time is typically a few minutes (manual or scripted).
- **No automatic write conflict, since multi-write is not allowed.**  
  True bidirectional master-master is not supported—classic A-A is "read in both, write in one" using native service.

#### Example Setup:
- Create MySQL server in CI (primary), deploy DB.
- Create MySQL server in SI (read replica), using Azure Portal/CLI.
- Monitor the health and lag.
- Write-traffic always lands on CI DB as long as possible.

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

### NOT Supported
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

### What About "Active-Active"?

- Even in Active-Active, only one region acts as the **writer (primary)**  
- The second region is a **read-only replica**  
- If writer fails → manual promotion required  
- So "Active-Active" for managed MySQL is actually "Active-ReadOnly-Active" (both regions active for reads, only one for writes)

---

## 6. Recommendations

- Use the DB replica in SI **only for read** (reporting, etc.) in normal operation.
- For file sync, try to minimize file changes in both regions and have a defined sync frequency.
- For full multi-active writes at DB and file level, use purpose-built distributed filesystems and DBs.

---

## 7. Pros & Cons

### Pros
- Both regions can serve user traffic (read and write, but writing only to primary region).
- Quick failover with manual or scripted replica promotion.
- Simple to roll back/fail back after recovery.

### Cons
- True write-active/active is not supported for managed DB or file tier.
- Manual intervention needed for DB failover/promotion.
- Manual/scheduled file sync solutions are not as robust as cloud-native shared storage.

---

