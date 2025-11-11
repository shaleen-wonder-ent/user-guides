# Active-Passive DR Architecture for Drupal 10 Website

## Overview
This architecture provides a cost-effective disaster recovery solution with the primary region (Central India) actively serving all traffic, and the secondary region (South India) on standby, ready to take over in case of failure.

<img width=auto height=auto alt="image" src="https://github.com/user-attachments/assets/6a99d04d-cd29-4ab9-9ab5-2bea4d7f2f20" />

## Architecture Components

### 1. Traffic Management Layer
- **Azure Traffic Manager**: Priority-based routing
  - Primary Endpoint: Central India (Priority 1)
  - Secondary Endpoint: South India (Priority 2)
  - Health Check Interval: 30 seconds
  - Automatic failover on endpoint failure

### 2. PRIMARY REGION - Central India (ACTIVE)

#### Edge Security
```yaml
F5 WAF - Central India:
  Status: ACTIVE
  Configuration:
    - Application Security Policies
    - DDoS Protection
    - SSL/TLS Termination
    - Bot Protection
    - Geo-blocking rules

Azure Application Gateway v2:
  Status: ACTIVE
  WAF Policy: Enabled (OWASP 3.2)
  Listeners:
    - HTTP (port 80) → HTTPS redirect
    - HTTPS (port 443)
  Backend Pools: VM Scale Set
```

#### Compute Layer
```yaml
VM Scale Set - PRIMARY:
  Status: ACTIVE
  Configuration:
    Minimum Instances: 2
    Maximum Instances: 10
    Current Running: Based on auto-scale
  
  VM Configuration:
    OS: Ubuntu 20.04 LTS
    Size: Standard_D4s_v3 (or appropriate)
    Apache: 2.4.62
    PHP: 8.3.19
    Drupal: 10.x
    
  Auto-scale Rules:
    Scale Out:
      - CPU > 70% for 5 minutes
      - Memory > 80% for 5 minutes
      - HTTP queue > 100 requests
    Scale In:
      - CPU < 30% for 10 minutes
      - Memory < 50% for 10 minutes

Azure Load Balancer:
  SKU: Standard
  Health Probe: HTTP:80/health-check.php
  Probe Interval: 15 seconds
```

#### Data Layer
```yaml
Azure Database for MySQL:
  Status: PRIMARY (Read/Write)
  Version: 8.0.42-azure
  Tier: General Purpose / Memory Optimized
  Compute: 4-8 vCores
  Storage: 256 GB with auto-grow
  
  High Availability:
    Zone-Redundant: Enabled
    Same-zone Standby: Enabled
  
  Backup:
    Automated Backup: Enabled
    Retention: 35 days
    Geo-Redundant: Enabled
    Point-in-Time Restore: Available
  
  Replication:
    Read Replica: South India (STANDBY)
    Replication Lag: Typically < 5 seconds
    Replication Type: Asynchronous

Azure Storage Account:
  Status: ACTIVE
  Replication: GRS (Geo-Redundant)
  Access Tier: Hot
  Contents:
    - /sites/default/files (Drupal uploads)
    - Media library
    - Backups
  
  GRS Details:
    Primary: Central India
    Secondary: South India (Read-Only)
    RPO: ~15 minutes
```

### 3. SECONDARY REGION - South India (STANDBY/PASSIVE)

#### Edge Security
```yaml
F5 WAF - South India:
  Status: STANDBY (Pre-configured, ready)
  Configuration: Mirror of Central India
  State: Can be activated in < 5 minutes
  Keep-Warm: Optional health check traffic

Azure Application Gateway v2:
  Status: STANDBY (Pre-configured)
  WAF Policy: Enabled (same as primary)
  State: Ready to accept traffic
  Keep-Warm: Minimal backend pool or deallocated
```

#### Compute Layer
```yaml
VM Scale Set - SECONDARY:
  Status: STANDBY
  Configuration:
    Minimum Instances: 0 (or 1 for warm standby)
    Maximum Instances: 10
    Current Running: 0 instances
  
  Deployment State:
    Image: Same as primary (golden image)
    Configuration: Identical to primary
    Start-up Time: 3-5 minutes
    
  Options:
    Cold Standby: 0 instances, start on failover
    Warm Standby: 1-2 instances always running
    Hot Standby: Same as primary (higher cost)

Azure Load Balancer:
  SKU: Standard
  Status: Configured, ready
  Health Probe: Pre-configured
```

#### Data Layer
```yaml
Azure Database for MySQL:
  Status: READ REPLICA (Read-Only)
  Version: 8.0.42-azure
  Tier: Same as primary
  Compute: Same as primary
  
  Replication:
    Source: Central India primary
    Type: Asynchronous
    Lag Monitoring: Enabled
    Target Lag: < 5 seconds
  
  Promotion Process:
    Manual or Automated: Configurable
    Promotion Time: 1-2 minutes
    Result: Becomes read-write primary
  
  Note: Point-in-time restore available from primary backups

Azure Storage Account:
  Status: GRS SECONDARY (Read-Access)
  Replication: Automatic from Central India
  Access: Read-only until failover
  
  Failover Process:
    Type: Account failover
    Time: 1-2 hours for complete failover
    Alternative: RA-GRS for read access
```

### 4. Shared Services (Active in Both Regions)

```yaml
Azure Cache for Redis:
  Tier: Premium
  Geo-Replication: Enabled
  Primary: Central India (write)
  Secondary: South India (read replica)
  Failover: Can be manually initiated
  
  Usage:
    - Drupal page cache
    - Session storage (with persistence)
    - Object cache

Azure CDN / Front Door:
  Status: Active globally
  Origin: Primary region
  Failback Origin: Secondary region
  Benefits:
    - Reduced origin load
    - Static content caching
    - Global performance

Azure Key Vault:
  Primary: Central India
  Replication: Automatic to paired region
  Contents:
    - Database credentials
    - API keys
    - SSL certificates
    - Application secrets

Azure Backup:
  VM Backup:
    Frequency: Daily
    Retention: 30 days
    Cross-Region Restore: Enabled
  
  Database Backup:
    Frequency: Automated continuous
    Retention: 35 days
    Geo-redundant: Enabled

Azure Site Recovery (ASR):
  Purpose: VM-level replication
  Source: Central India VMs
  Target: South India
  RPO: 15 minutes
  RTO: 30-60 minutes
  Use Case: Additional protection layer
```

### 5. Network Architecture

```yaml
Central India VNet:
  Address Space: 10.227.0.0/16
  Subnets:
    App Subnet: 10.227.1.0/24
    DB Subnet: 10.227.2.0/24 (Private Link)
    Gateway Subnet: 10.227.255.0/27
    AzureBastionSubnet: 10.227.254.0/27
  
  NSG Rules (App Subnet):
    Inbound:
      - Allow 443 from Application Gateway
      - Allow 22 from Bastion
      - Deny all other
    Outbound:
      - Allow 3306 to DB subnet
      - Allow 443 to Azure services
      - Allow 6379 to Redis

South India VNet:
  Address Space: 10.227.64.0/16
  Subnets:
    App Subnet: 10.227.65.0/24
    DB Subnet: 10.227.66.0/24 (Private Link)
    Gateway Subnet: 10.227.127.0/27
    AzureBastionSubnet: 10.227.126.0/27

VNet Connectivity:
  Type: Global VNet Peering
  Or: Site-to-Site VPN (Active-Active)
  Purpose:
    - Data replication
    - Management access
    - Cross-region communication
```

### 6. Monitoring & Disaster Recovery Automation

```yaml
Azure Monitor:
  Metrics Collection:
    - Application performance (Application Insights)
    - VM performance metrics
    - Database performance and replication lag
    - Network metrics
    - Storage account metrics
  
  Availability Tests:
    - Endpoint monitoring from 5+ global locations
    - Frequency: Every 5 minutes
    - Alerts on: 2 consecutive failures

Log Analytics Workspace:
  Data Collection:
    - Application logs
    - System logs (Syslog)
    - IIS/Apache logs
    - Database query logs
    - NSG flow logs
  
  Queries:
    - Error rate trending
    - Performance analysis
    - Security event detection

Azure Alerts:
  Critical Alerts (Trigger Failover):
    - Primary region unavailable (2+ monitors)
    - Database primary unreachable for > 2 minutes
    - Application error rate > 50% for 5 minutes
    - F5 WAF health check failure
  
  Warning Alerts:
    - Replication lag > 30 seconds
    - CPU > 80% for 10 minutes
    - Storage > 85% capacity
    - SSL certificate expiring in 30 days

Azure Automation:
  Failover Runbook:
    Trigger: Manual or automatic (alert-based)
    
    Steps:
      1. Verify primary region failure
      2. Promote MySQL read replica to primary
      3. Scale up secondary VM scale set
      4. Update Traffic Manager priority
      5. Initiate storage account failover (if needed)
      6. Update application configuration
      7. Verify secondary region health
      8. Send notifications
    
    Estimated Execution Time: 5-10 minutes
  
  Failback Runbook:
    Trigger: Manual after primary region recovery
    
    Steps:
      1. Verify primary region health
      2. Sync data from secondary to primary
      3. Promote primary database
      4. Update Traffic Manager priority
      5. Scale down secondary region
      6. Send notifications
    
    Estimated Execution Time: 15-30 minutes
```

## Disaster Recovery Procedures

### Failover Process (Primary to Secondary)

#### Automatic Failover
```yaml
Trigger Conditions:
  - Traffic Manager health check failures (3 consecutive)
  - Azure Monitor alert rules met
  - Azure Service Health impairment in primary region

Automated Steps:
  1. Alert Detection (0 minutes)
     - Azure Monitor detects failure
     - Trigger automation runbook
  
  2. Validation (0-2 minutes)
     - Confirm primary region failure
     - Check secondary region health
     - Verify data replication status
  
  3. Database Promotion (2-4 minutes)
     - Stop replication from primary
     - Promote read replica to read-write
     - Verify write capability
  
  4. Compute Activation (4-7 minutes)
     - Scale VM Scale Set to minimum instances
     - Wait for instances to be healthy
     - Load balancer health probes pass
  
  5. Traffic Redirection (7-8 minutes)
     - Update Traffic Manager endpoint priority
     - DNS propagation begins
     - Monitor traffic shift
  
  6. Verification (8-10 minutes)
     - End-to-end application testing
     - Database write operations confirmed
     - User traffic flowing correctly
  
  Total RTO: 10-15 minutes
```

#### Manual Failover
```yaml
Use Cases:
  - Planned maintenance in primary region
  - Testing DR procedures
  - Primary region degradation (not complete failure)

Process:
  1. Pre-Failover Checks
     - Verify secondary region readiness
     - Check replication lag (should be minimal)
     - Notify stakeholders
     - Create backup of current state
  
  2. Initiate Failover
     - Run Azure Automation runbook
     - Or execute manual steps from runbook
  
  3. Monitor Progress
     - Track each step completion
     - Verify no errors
  
  4. Post-Failover Validation
     - Application functionality testing
     - Database write operations
     - File upload/download
     - User authentication
     - Performance benchmarks
  
  Total RTO: 15-30 minutes (with validation)
```

### Failback Process (Secondary to Primary)

```yaml
Prerequisites:
  - Primary region fully recovered
  - Root cause identified and resolved
  - Approval from change management

Process:
  1. Preparation (Day 1)
     - Assess primary region health
     - Plan failback window
     - Communicate to stakeholders
  
  2. Data Synchronization (Day 1-2)
     - Sync database from secondary to primary
     - Verify data consistency
     - Sync storage accounts
  
  3. Infrastructure Verification (Day 2)
     - Test all primary region services
     - Verify network connectivity
     - Validate security configurations
  
  4. Execute Failback (Scheduled Window)
     - Enable replication from secondary to primary
     - Wait for sync completion
     - Promote primary database
     - Update Traffic Manager
     - Scale down secondary
  
  5. Post-Failback Monitoring (Day 2-7)
     - Monitor primary region closely
     - Keep secondary in warm standby
     - Review incident and improve
  
  Total RTO: Planned, typically 1-3 hours
```

## RTO & RPO

```yaml
Recovery Objectives:
  
  RTO (Recovery Time Objective):
    Automated Failover: 10-15 minutes
    Manual Failover: 15-30 minutes
    
    Breakdown:
      - Detection: 2-3 minutes
      - Database promotion: 2-3 minutes
      - VM startup (cold standby): 3-5 minutes
      - VM startup (warm standby): 1-2 minutes
      - Traffic switchover: 2-3 minutes
      - Validation: 2-5 minutes
  
  RPO (Recovery Point Objective):
    Database: 5-30 seconds
      - Asynchronous replication
      - Typically < 5 seconds lag
      - Maximum: ~30 seconds
    
    Storage (GRS): ~15 minutes
      - Files uploaded in last 15 minutes may need recovery
      - Consider Azure File Sync for critical data
    
    Cache: 0-5 seconds
      - Redis geo-replication
      - Minimal data loss
  
  Data Loss Scenarios:
    Best Case: < 5 seconds (normal replication lag)
    Worst Case: 15 minutes (GRS storage)
    Mitigation: Implement more frequent file sync
```
### Cost Comparison: Standby Options

```yaml
Cold Standby (Lowest Cost):
  VM Scale Set: 0 instances ($0/month)
  Database: Read replica (~50% of primary)
  Storage: GRS (minimal extra cost)
  Other Services: Minimal standby resources
  Total Savings: ~75% vs. active-active
  RTO: 10-15 minutes

Warm Standby (Balanced):
  VM Scale Set: 1-2 instances (~20% of primary)
  Database: Read replica (~50% of primary)
  Storage: GRS (minimal extra cost)
  Other Services: Some services running
  Total Savings: ~60% vs. active-active
  RTO: 5-10 minutes

Hot Standby (Highest Cost):
  VM Scale Set: Same as primary (~100%)
  Database: Read replica (~50% of primary)
  Storage: GRS (minimal extra cost)
  Other Services: All services running
  Total Savings: ~30% vs. active-active
  RTO: 2-5 minutes
```

### Recommended Configuration (Warm Standby)

```yaml
Primary Region - Central India:
  VM Scale Set: 2-10 instances (auto-scale)
  Database: GP_Gen5_4 (4 vCores)
  Storage: GRS, 256 GB
  Application Gateway: WAF_v2 (2 instances)
  Redis: Premium P1
  Estimated Cost: $2,000-3,500/month

Secondary Region - South India:
  VM Scale Set: 1 instance (warm standby)
  Database: Read replica (GP_Gen5_4)
  Storage: GRS secondary (included)
  Application Gateway: WAF_v2 (1 instance, can be deallocated)
  Redis: Geo-replica (included with Premium)
  Estimated Cost: $800-1,200/month

Total DR Cost: ~30-40% of primary region cost
```


## Advantages of Active-Passive

- **Cost-Effective**: ~60-70% less than active-active
- **Simpler Management**: One active region to manage
- **Data Consistency**: No conflict resolution needed
- **Proven Pattern**: Well-understood architecture
- **Flexible RTO**: Choose cold/warm/hot based on budget

## Considerations for Active-Passive

- **Failover Time**: 10-30 minutes RTO (vs. instant)
- **Testing Required**: Regular DR drills essential
- **Potential Data Loss**: RPO of 5 seconds to 15 minutes
- **Unused Capacity**: Secondary region mostly idle
- **Manual Steps**: Some failover steps may require human intervention
