# Active-Active DR Architecture for Drupal 10 Website

## Overview
This architecture provides a fully active disaster recovery solution with both regions (Central India and South India) serving live traffic simultaneously.

<img width="1160" height="569" alt="image" src="https://github.com/user-attachments/assets/057de936-cfa2-4d6e-9631-dc21c725ba7a" />

## Architecture Components

### 1. Traffic Management Layer
- **Azure Traffic Manager**: Performance-based routing distributing traffic across both regions
- **Azure DNS**: Global DNS resolution with health checks
- **Distribution**: 50-50 split or performance-based routing

### 2. Edge Security Layer (Both Regions)
- **F5 WAF**: Active in both regions
  - DDoS protection
  - Application layer security
  - SSL/TLS termination
  - Rate limiting
- **Azure Application Gateway v2**:
  - WAF v2 enabled
  - Path-based routing
  - Session affinity

### 3. Compute Layer (Both Regions)
**VM Scale Sets Configuration**:
```yaml
Drupal Application Servers:
  OS: Ubuntu 20.04 LTS or RHEL 8
  Web Server: Apache 2.4.62
  PHP: 8.3.19
  Framework: Drupal 10
  Scale Set:
    Minimum Instances: 2
    Maximum Instances: 10
    Auto-scale Rules:
      - CPU > 70% for 5 min: Scale out
      - CPU < 30% for 10 min: Scale in
      - Memory > 80%: Scale out
  
Load Balancer:
  Type: Azure Load Balancer (Standard SKU)
  Health Probes: HTTP on port 80, 443
  Load Distribution: 5-tuple hash
```
### 4. Data Layer

#### Database Configuration
```yaml
Azure Database for MySQL:
  Version: 8.0.42-azure
  Primary Region: Central India
  Secondary Region: South India
  Replication:
    Type: Active-Active (Bidirectional)
    Sync Mode: Asynchronous
    Conflict Resolution: Last-write-wins or custom
  Performance Tier: General Purpose or Memory Optimized
  Compute: 4-16 vCores
  Storage: 100-1000 GB with auto-grow
  Backup:
    Automated: 35 days retention
    Geo-redundant: Enabled
```

#### Storage Configuration
```yaml
Azure Storage Account:
  Replication: GRS (Geo-Redundant Storage)
  Access Tier: Hot
  Usage: 
    - Drupal file uploads
    - Media assets
    - Static content
  CDN: 
    Enabled: Yes
    Provider: Azure CDN / Front Door
```

#### Cache Layer
```yaml
Azure Cache for Redis:
  Tier: Premium
  Replication: Active Geo-Replication
  Usage:
    - Drupal page cache
    - Session storage
    - Object cache
  Size: P1-P4 based on workload
```

### 5. Network Architecture

```yaml
Central India VNet:
  Address Space: 10.227.0.0/16
  Subnets:
    - App Subnet: 10.227.1.0/24
    - DB Subnet: 10.227.2.0/24
    - Gateway Subnet: 10.227.255.0/27

South India VNet:
  Address Space: 10.227.64.0/16
  Subnets:
    - App Subnet: 10.227.65.0/24
    - DB Subnet: 10.227.66.0/24
    - Gateway Subnet: 10.227.127.0/27

Connectivity:
  VNet Peering: Global VNet Peering
  Or: VPN Gateway (Active-Active)
  NSG Rules: Strict inbound/outbound filtering
```

### 6. Monitoring & Operations

```yaml
Azure Monitor:
  - Application Insights for Drupal
  - VM metrics and logs
  - Database performance metrics
  - Network performance monitor
  
Log Analytics:
  - Centralized logging
  - Query and analysis
  - Custom alerts
  
Alerts:
  - Response time > 3 seconds
  - Error rate > 5%
  - CPU > 80% for 10 minutes
  - Database connection failures
  - Storage capacity > 85%
```

## Data Synchronization Strategy

### Database Sync
1. **Active-Active MySQL Replication**:
   - Use Azure MySQL Flexible Server with read replicas promoted to write-capable
   - Implement application-level conflict resolution
   - Monitor replication lag (target: < 5 seconds)

2. **Alternative - Application-Level Routing**:
   - Write operations to primary region (Central India)
   - Async replication to secondary
   - Read operations from both regions

### File Synchronization
- **GRS Storage**: Automatic replication (RPO: ~15 minutes)
- **Azure File Sync**: Real-time sync for critical files
- **CDN**: Cache static content globally

### Session Management
- **Redis Geo-Replication**: Active in both regions
- **Sticky Sessions**: Optional with Application Gateway
- **Session Persistence**: Stored in Redis for cross-region access

## RTO & RPO

```yaml
Recovery Objectives:
  RTO (Recovery Time Objective): ~0 seconds
    - Both regions always active
    - Automatic failover via Traffic Manager
    - No manual intervention required
  
  RPO (Recovery Point Objective): ~5-15 seconds
    - Database: Near real-time replication
    - Storage: GRS async replication (~15 min)
    - Cache: Real-time geo-replication
```

## Traffic Flow
1. Users resolve the site via **Traffic Manager (Performance)** which directs them to the **lowest‑latency healthy** region.
2. Regional **F5 WAF** enforces Layer‑7 security. Health probes target an `/healthz` endpoint from both regions.
3. App tier is **stateless**; sessions and page cache are kept in **Redis** per region.
4. Media is served from **Azure Storage** with **G**eo‑**R**edundant **S**torage (GRS) or RA‑GRS; Drupal uses the same container name in both regions.
5. Writes go to **MySQL Primary in Central India**. South India MySQL is a **read replica**; app in both regions directs writes to primary (via app config/secret).

## Failover (Regional Outage)
- **Trigger:** Health probe failure at Traffic Manager and F5 WAF for Central India.
- **Action:** 
  - **Promote South India MySQL read‑replica** to primary. 
  - Switch **Drupal DB connection strings** (Key Vault + App settings) to local server.
  - **Traffic Manager** continues routing only to healthy South India endpoints. 
- **RTO driver:** DB promotion + app recycle; DNS/TM convergence is seconds to a few minutes.

## Failback
1. Recreate **read‑replica** in Central India from South India primary.
2. Re‑point app write endpoint back to Central India after replica catches up.
3. Revert Traffic Manager to dual‑site serving once both pass health checks.

## Security & Connectivity
- **WAF:** Reuse **F5 WAF** policies in both regions; keep rulesets identical.
- **Network:** Place app subnets in existing VNETs; peer to shared services as needed.
- **Secrets:** Azure Key Vault per region with **Managed Identity** access for apps.
- **Inbound:** Route through existing **Palo Alto** and **F5** as per enterprise standard.
   
## Advantages of Active-Active
- **Maximum Availability**: Both regions serving traffic
- **Load Distribution**: Better resource utilization
- **Performance**: Users routed to nearest region
- **No Failover Delay**: Instant redundancy
- **Testing**: Both environments constantly validated

## Challenges of Active-Active
- **Complexity**: More complex to manage
- **Data Consistency**: Requires conflict resolution
- **Cost**: Double the running resources
- **Synchronization**: Replication lag monitoring needed
- **Session Management**: Cross-region session handling

  ---
  

# Active-Active DR Pricing Structure (DR costing only for the website component)

## Common (Shared Across Both Regions)
These are global or once-per-deployment resources.

**Common Resources:**
- C1 = Traffic Manager (or Azure Front Door)
- C2 = DNS Zone
- C3 = Monitoring (Log Analytics, Alerts)
- C4 = Shared Key Vault (optional)

**Chargeable (Common):**
- C1 + C2 + C3 + C4 → counted once

---

## Region 1 (Central India)
**Components:**
- A1 = App Tier (VMSS)
- B1 = MySQL Server (Primary)
- D1 = Redis Cache (Primary)
- E1 = Blob Storage (GRS)
- F1 = WAF / Firewall
- G1 = VNET + NSG + Public IP

---

## Region 2 (South India)
**Components:**
- A2 = App Tier (VMSS)
- B2 = MySQL Server (Read Replica)
- D2 = Redis Cache (Replica)
- E2 = Blob Storage (RA-GRS)
- F2 = WAF / Firewall
- G2 = VNET + NSG + Public IP

---

## Detailed Breakdown

```
Common:
  - Traffic Manager (1)
  - DNS Zone (1)
  - Monitoring Workspace (1)

Region 1:
  - App Tier (1)
  - MySQL Primary (1)
  - Redis Primary (1)
  - Storage GRS (1)
  - WAF (1)
  - VNET + NSG (1)

Region 2:
  - App Tier (1)
  - MySQL Read Replica (1)
  - Redis Replica (1)
  - Storage RA-GRS (1)
  - WAF (1)
  - VNET + NSG (1)

Chargeable Multipliers:
  - Common: 1x
  - App Tier: 2x
  - Database: 2x
  - Cache: 2x
  - Storage: 2x
  - WAF: 2x
  - Networking: 2x
```

---

