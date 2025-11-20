# Azure Front Door High Availability Architecture Guide
**Response to Azure Front Door Incident (YKYN-BWZ) - October 29-30, 2025**

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Incident Overview](#incident-overview)
3. [Customer Concerns](#customer-concerns)
4. [Cost Optimization for Redundant Architecture](#cost-optimization-for-redundant-architecture)
5. [Certificate Validation and Failover Process](#certificate-validation-and-failover-process)
6. [Industry Best Practices for High Availability](#industry-best-practices-for-high-availability)
7. [Recommended Architecture](#recommended-architecture)

---

## Executive Summary

Following the Azure Front Door global outage on October 29-30, 2025 (Tracking ID: YKYN-BWZ), this document provides comprehensive guidance on implementing a highly available, cost-optimized architecture for customers running 30-40 App Services behind Azure Front Door.

### Key Recommendations:

1. **Cost Optimization**: Implement Azure Front Door Premium with origin groups to reduce costs by ~25-30% compared to separate Traffic Manager + Application Gateway approach
2. **Certificate Management**: Consolidate from 40 certificates to 3-5 wildcard/SAN certificates with pre-deployment across all platforms
3. **High Availability**: Multi-region deployment with automated failover achieving <1 minute RTO

### Expected Outcomes:

- **Monthly Cost**: $800-1,200 (vs. $1,500-2,000 for full parallel infrastructure)
- **Recovery Time Objective (RTO)**: <1 minute
- **Recovery Point Objective (RPO)**: Zero data loss
- **Certificate Failover Delay**: Zero (pre-validated certificates)

---

## Incident Overview

### What Happened?

**Incident Date**: October 29-30, 2025  
**Duration**: 15:41 UTC (Oct 29) to 00:05 UTC (Oct 30) - ~8.5 hours  
**Impact**: Global Azure Front Door and Azure CDN connectivity issues

### Root Cause:

A sequence of customer configuration changes across two different control plane build versions resulted in incompatible metadata that exposed a latent bug in the data plane. 
The configuration passed through validation safeguards because the crash occurred asynchronously (~5 minutes after deployment), allowing the problematic configuration to propagate globally.

### Affected Services:

- Azure Front Door & CDN (primary impact)
- Azure Portal, Azure Active Directory B2C
- Azure App Service, Azure Static Web Apps
- Microsoft 365, Dynamics 365
- 30+ other Azure and Microsoft services

### Key Learnings:

1. **Configuration propagation risk** is inherent to global CDN platforms
2. **Asynchronous processing** can bypass validation safeguards
3. **Active-active with failaway** is now Microsoft's standard for critical infrastructure
4. **Configuration isolation** prevents cascading failures

---

## Customer Concerns

### Current Customer Environment:

- **App Services**: 30-40 production applications
- **Current Setup**: Azure Front Door (single layer)
- **Concern**: Single point of failure demonstrated by October 29 incident

### Proposed Architecture:

```
Layer 1 (DNS):        Traffic Manager (orchestrator)
                              |
                    ┌─────────┴─────────┐
                    |                   |
Layer 2 (CDN/WAF):  AFD          Application Gateway
                    |                   |
                    └─────────┬─────────┘
                              |
Layer 3 (Origin):      Backend Apps (30-40 App Services)
```

### Three Primary Concerns:

1. **Cost Optimization**: How to minimize costs when running AFD and Application Gateway in parallel
2. **Certificate Management**: How to pre-configure SSL certificates for 30-40 apps to enable seamless failover
3. **Best Practices**: Industry validation of proposed architecture

---

## Cost Optimization for Redundant Architecture

### Current Architecture Cost Analysis

#### Proposed Architecture Components:

| Component | Monthly Cost approx (USD) | Notes |
|-----------|-------------------|-------|
| Traffic Manager | $35-50 | Based on 10M DNS queries/month |
| Azure Front Door Standard | $550-800 | 30-40 routing rules, 500GB outbound |
| Application Gateway v2 (Standby) | $250-350 | 2 minimum instances, minimal traffic |
| Private Link | $15-30 | 30-40 Private Endpoints @ $0.01/hour |
| **Total (Proposed)** | **$850-1,230** | Full redundancy |

---

### Cost Optimization Options

### **Option 1: Active-Standby with Minimal Application Gateway Capacity**

#### Architecture:

```
Traffic Manager (Priority Routing)
├── Priority 1: Azure Front Door (active)
└── Priority 2: Application Gateway (warm standby - minimum capacity)
    └── Auto-scaling: min=2, max=10 instances
```

#### Cost Breakdown approx:

- **Traffic Manager**: $35-50/month
- **AFD Standard**: $550-800/month
- **App Gateway (2 instances minimum)**: $250-350/month
- **Total**: **$835-1,200/month**

#### Pros & Cons:

 **Pros**:
- Immediate failover (<1 minute)
- Application Gateway is "warm" and tested
- Predictable costs

 **Cons**:
- Paying for unused Application Gateway capacity
- Management overhead for two parallel systems
- Still ~50% cost increase

---

### **Option 2: Infrastructure-as-Code Rapid Deployment (Cost-Optimized)**

#### Architecture:

```
Traffic Manager (Priority Routing)
├── Priority 1: Azure Front Door (active)
└── Priority 2: Application Gateway (deployed on-demand via automation)

```

#### Cost Breakdown approx:

- **Traffic Manager**: $35-50/month (pre-configured)
- **AFD Standard**: $550-800/month
- **App Gateway**: $0/month (deployed only during incidents)
- **Emergency Deployment Cost**: ~$8-12/hour when activated
- **Total (Normal Operation)**: **$585-850/month**

#### Pros & Cons:

 **Pros**:
- Significant cost savings ($250-350/month)
- App Gateway only costs during actual outages
- IaC ensures consistent deployment

 **Cons**:
- 10-15 minute recovery time (not immediate)
- Requires mature DevOps practices
- Risk of deployment failures during incident
- DNS TTL delays (Traffic Manager still points to non-existent endpoint)

#### **Recommendation**: Only suitable if 10-15 minute RTO is acceptable

---

### **Option 3: Azure Front Door Premium with Origin Groups**  **RECOMMENDED**

#### Architecture:

```
Azure Front Door Premium (No Traffic Manager needed)
├── Origin Group 1 (Priority 1): App Services via Private Link
└── Origin Group 2 (Priority 2): Application Gateway (standby) → App Services
    └── Automatic health-based failover
    └── Application Gateway auto-scales based on traffic
```

#### Why This is Better:

1. **Eliminates Traffic Manager** - AFD Premium has built-in multi-origin failover
2. **Private Link to origins** - Enhanced security, included in Premium
3. **Better health probes** - AFD probes both origin groups, automatic failover
4. **Faster failover** - No DNS TTL delays (happens at AFD edge)
5. **Auto-scaling App Gateway** - Only pay for capacity when actively serving traffic

#### Cost Breakdown approx:

- **AFD Premium**: $330/month (base) + ~$550/month (routing rules, data) = $880/month
- **Application Gateway**: $0-350/month (auto-scales from 0 to 10 based on active traffic)
  - During normal operation: ~$50-100/month (minimal health probe traffic)
  - During failover: $250-350/month (full capacity)
- **Private Link**: Included in AFD Premium for origin connections
- **Total (Normal Operation)**: **$930-980/month**
- **Total (During Failover)**: **$1,130-1,230/month**

#### Pros & Cons:

 **Pros**:
- **No Traffic Manager needed** - Simpler architecture
- **Faster failover** - No DNS TTL delays (~10-30 seconds vs. 60+ seconds)
- **Better security** - Private Link to App Services included
- **Auto-scaling App Gateway** - Pay only for what you use
- **Built-in WAF Premium** - Enhanced security rules
- **Microsoft's own recommendation** (from RCA - active-active approach)

 **Cons**:
- AFD Premium has higher base cost ($330/month vs. Standard $35/month)
- Slightly more complex origin group configuration

#### **Cost Comparison vs. Original Proposal**:

| Scenario | Option 1 (TM + AFD + AppGW) | Option 3 (AFD Premium) | Savings |
|----------|----------------------------|------------------------|---------|
| Normal Operation | $835-1,200 | $930-980 | **-$95 to +$220** |
| During Failover | $835-1,200 | $1,130-1,230 | **Similar** |

**Verdict**: Option 3 provides **comparable cost** with **significant operational benefits**

---

### **Option 4: Multi-CDN Strategy (AFD + Akamai)**

#### Architecture:

```
Traffic Manager (Priority Routing)
├── Priority 1: Azure Front Door → App Services
└── Priority 2: Akamai CDN → App Services
```

#### Configuration Details:

- Both CDNs point to same App Services backends
- Requires duplicate certificate and configuration management
- DNS failover via Traffic Manager

#### Cost Breakdown approx:

- **Traffic Manager**: $35-50/month
- **AFD Standard**: $550-800/month
- **Akamai**: Variable (typically $500-2,000+/month depending on contract)
- **Total**: **$1,085-2,850+/month**

#### Pros & Cons:

 **Pros**:
- True multi-vendor redundancy (Azure incident doesn't affect Akamai)
- Proven at enterprise scale
- Geographic optimization (Akamai has extensive edge presence)

 **Cons**:
- **Significantly higher cost** (potentially double)
- Complex certificate management (need to sync across vendors)
- Different WAF rule formats (requires parallel maintenance)
- Requires expertise in both platforms
- Potential vendor lock-in with Akamai

#### **Recommendation**: Only for mission-critical workloads with >$10K/month CDN budgets

---

### Cost Optimization Recommendation Summary

| Option | Monthly Cost | RTO | Complexity | Recommendation |
|--------|-------------|-----|------------|----------------|
| 1. TM + AFD + AppGW (warm) | $835-1,200 | <1 min | Medium | Good for immediate failover needs |
| **2. AFD Premium + Origin Groups** | **$930-980** | **<30 sec** | **Low-Medium** | ** RECOMMENDED** |
| 3. IaC Rapid Deployment | $585-850 | 10-15 min | Medium-High | Only if RTO allows |
| 4. AFD + Akamai | $1,085-2,850+ | <1 min | High | Mission-critical only |

### **Final Recommendation: Option 3 - Azure Front Door Premium with Origin Groups**

**Rationale**:
1.  Comparable total cost to Traffic Manager approach
2.  Faster failover (no DNS delays)
3.  Simpler architecture (fewer moving parts)
4.  Aligns with Microsoft's post-RCA architecture (active-active with failaway)
5.  Auto-scaling App Gateway minimizes standby costs
6.  Private Link security included

---

## Certificate Validation and Failover Process

### Challenge Statement

**Customer Environment**:
- 30-40 App Services, each with custom domain
- Currently: 30-40 individual SSL certificates
- Concern: Certificate validation delays during failover to secondary CDN/gateway

**Problem**:
- SSL/TLS certificates require domain validation (DV)
- Validation methods:
  - DNS TXT record validation: 5-60 minutes
  - HTTP file validation: 5-30 minutes (requires service to be accessible)
- **During failover**: If certificates aren't pre-validated, 30-60 minute delay before secondary platform can serve HTTPS traffic
- **Impact**: Unacceptable downtime extension

---

### Solution Architecture: Zero-Delay Certificate Failover

### **Step 1: Certificate Consolidation Strategy**

#### Current State:
```
App1.contoso.com → Certificate 1
App2.contoso.com → Certificate 2
App3.contoso.com → Certificate 3
...
App40.contoso.com → Certificate 40
```
**Total**: 40 certificates to manage, renew, and sync

#### Target State:

**Option A: Wildcard Certificates (Recommended if all apps use subdomains)**

```
*.contoso.com → Certificate 1 (covers all subdomains)
*.internal.contoso.com → Certificate 2 (if using second-level subdomains)
```
**Total**: 1-2 certificates

**Option B: Subject Alternative Name (SAN) Certificates**

```
Certificate 1: app1.contoso.com, app2.contoso.com, ..., app10.contoso.com (10 SANs)
Certificate 2: app11.contoso.com, app12.contoso.com, ..., app20.contoso.com (10 SANs)
Certificate 3: app21.contoso.com, ..., app30.contoso.com
Certificate 4: app31.contoso.com, ..., app40.contoso.com
```
**Total**: 4 certificates (10 domains per cert)

**Option C: Mixed Approach**

```
Certificate 1: *.contoso.com (wildcard for most apps)
Certificate 2: special-app.differentdomain.com (SAN for apps on different domains)
Certificate 3: *.partner.contoso.com (separate subdomain space)
```
**Total**: 3-5 certificates

#### Recommendation:

```
Recommended Consolidation:
  Total Certificates: 3-5 (from 40)
  
  Certificate 1: *.contoso.com (wildcard)
    - Covers: app1.contoso.com, app2.contoso.com, ..., app35.contoso.com
    - Provider: DigiCert or Let's Encrypt (if acceptable for production)
  
  Certificate 2: SAN certificate for special cases
    - Covers: 5-10 apps with different domain patterns
  
  Benefits:
    - 90% reduction in certificate management overhead
    - Single certificate to sync across platforms
    - Easier renewal automation
    - Cost savings (1 wildcard cert vs. 40 individual certs)
```

---

### **Step 2: Centralized Certificate Management with Azure Key Vault**

#### Architecture:

```
Azure Key Vault (Central Source of Truth)
├── Certificate: wildcard-contoso-com
├── Certificate: san-special-apps
└── Certificate: wildcard-internal-contoso-com

Synchronized to ↓

├── Azure Front Door Premium
│   ├── Custom Domain: app1.contoso.com (uses wildcard-contoso-com)
│   ├── Custom Domain: app2.contoso.com (uses wildcard-contoso-com)
│   └── ... (all 40 domains configured)
│
├── Application Gateway
│   ├── Listener: app1.contoso.com (references Key Vault certificate)
│   ├── Listener: app2.contoso.com (references Key Vault certificate)
│   └── ... (all 40 listeners configured)
│
└── Akamai (if using Option 4)
    └── Certificates uploaded via API/manually
```

#### Implementation:

```

1. Create Azure Key Vault
2. Upload/Import Certificate to Key Vault
3. Grant AFD Premium access to Key Vault
4. Grant Application Gateway access to Key Vault

```

---

### **Step 3: Pre-Deploy Certificates to All Platforms**

#### Azure Front Door Premium Configuration:

```
# For each of the 30-40 custom domains:
for domain in app1 app2 app3 ... app40; do
  
   Create custom domain in AFD    
   Associate with route
  
done
```

#### Application Gateway Configuration:

```
 Reference Key Vault certificate
 Create listeners for each app
   Create backend pool (points to App Service)
   Create rule  
```

---

### **Step 4: DNS Validation Strategy (Critical for Zero-Delay Failover)**

#### The Problem:
- Certificate Authorities (Let's Encrypt, DigiCert, etc.) require domain ownership validation
- Validation happens during initial certificate issuance AND renewal
- If validation records are removed, re-validating during failover adds delay

#### The Solution: Permanent DNS Validation Records

```dns
; Keep ALL validation records permanently in DNS
; This allows pre-validated certificates on all platforms

; Azure Front Door validation
_acme-challenge.contoso.com.     TXT  "afd-validation-token-12345"
_dnsauth.contoso.com.            TXT  "afd-validation-token-67890"

; Application Gateway / Let's Encrypt validation
_acme-challenge.contoso.com.     TXT  "letsencrypt-validation-abc123"

; Akamai validation (if used)
_acme-challenge.contoso.com.     TXT  "akamai-validation-xyz789"

; Note: Multiple TXT records for same name are allowed per RFC
```

#### Azure DNS Configuration:

```
 Add permanent validation records
```

**Result**: All platforms can validate certificates at ANY time without DNS changes

---

### **Step 5: Automated Certificate Renewal and Synchronization**

#### Challenge:
- Certificates expire (90 days for Let's Encrypt, 1 year for commercial CAs)
- Renewal must happen on all platforms simultaneously
- Manual sync is error-prone with 3-5 certificates across 3 platforms

#### Solution: Azure Function for Automated Certificate Sync


#### Deployment:

```
  Create Function App
  Grant permissions
  Deploy function code
```

---

### Certificate Management Summary

#### Consolidation:
- **Before**: 40 individual certificates
- **After**: 3-5 wildcard/SAN certificates
- **Reduction**: 85-92% fewer certificates

#### Pre-Deployment:
-  All certificates in Azure Key Vault
-  All certificates deployed to AFD Premium (40 custom domains)
-  All certificates deployed to Application Gateway (40 listeners)
-  All certificates deployed to Akamai (if used)
-  DNS validation records permanent

#### Automation:
-  Azure Function for automated renewal and sync
-  Weekly certificate expiry checks
-  Automatic sync across all platforms
-  Monitoring and alerting

#### Failover Impact:
- **Certificate Validation Delay**: **ZERO** (all pre-validated)
- **HTTPS Service Availability**: **Immediate** (certificates already configured)
- **Total Failover Time**: <1 minute (DNS/health probe only)


---

## Industry Best Practices for High Availability

### Microsoft's Post-RCA Recommendations

Based on the October 29-30 Azure Front Door incident, Microsoft documented the following best practices:

#### 1. **Mission-Critical Global HTTP Ingress Architecture**

**Reference**: https://learn.microsoft.com/azure/architecture/guide/networking/global-web-applications/mission-critical-global-http-ingress

**Key Principles**:
-  Active-active deployment across multiple regions
-  Automated health monitoring and failover
-  Configuration isolation (prevent single config change from global impact)
-  Gradual rollout with health validation
-  Quick rollback capability

#### 2. **Microsoft's Own Infrastructure Changes** (Completed Post-Incident)

```
Microsoft migrated critical first-party services to active-active with fail-away:
    Azure Portal
    Azure Communication Services
    Azure Marketplace
    Linux Software Repository for Microsoft Products
    Support ticket creation system

Architecture Pattern:
  Primary: Azure Front Door (global edge)
  Secondary: Independent infrastructure (different control plane)
  Failover: Automated based on health signals
  Recovery Time: < 5 minutes
```

**Lesson**: If Microsoft itself now uses multi-platform failover for critical services, customers should too

---

### Industry-Standard Patterns

### **1. Defense in Depth for Global HTTP Ingress**

#### Layer-Based Architecture:

```
Layer 1 - Global Traffic Distribution (DNS):
  Purpose: Route users to nearest/healthiest entry point
  Options:
    - Azure Traffic Manager (recommended for Azure-centric)
    - AWS Route 53 with health checks
    - Cloudflare Load Balancing
    - NS1 Managed DNS
  
  Configuration:
    - Health Probe Interval: 10-30 seconds
    - Failover Detection: 3 consecutive failures
    - TTL: 60 seconds (balance between failover speed and DNS load)
    - Multiple endpoints: 2-3 geographically diverse

Layer 2 - Edge Security and Acceleration (CDN/WAF):
  Purpose: DDoS protection, WAF, SSL termination, caching
  Primary Options:
    - Azure Front Door Premium (recommended)
    - Cloudflare
    - Akamai
  
  Secondary Options (for redundancy):
    - Azure Application Gateway
    - AWS CloudFront
    - Different CDN provider
  
  Configuration:
    - WAF: OWASP Top 10 protection
    - DDoS: L3/L4 (network) + L7 (application)
    - Bot Protection: Challenge/rate limiting
    - Caching: Based on Cache-Control headers
    - Private Link: Direct connection to origins (no public internet)

Layer 3 - Regional Load Balancing (Optional):
  Purpose: Distribute within region, handle origin failures
  Options:
    - Azure Load Balancer
    - Application Gateway (if not in Layer 2)
  
  Configuration:
    - Health probes to backend instances
    - Session affinity if stateful

Layer 4 - Origin (Application Tier):
  Purpose: Actual application serving
  Configuration:
    - Multi-region deployment (2+ regions)
    - Zone-redundant within region
    - Auto-scaling based on demand
    - Private endpoints (no public internet access)
    - Accept traffic ONLY from Layer 2/3 (restrict by source IP/service tag)
```

---

### **2. Multi-Region Deployment Pattern**

#### For 30-40 App Services:

```
Recommended Regional Distribution:

Primary Region: East US 2
  - App Services: 20 apps (50% capacity)
  - Configuration: Zone-redundant, Premium v3
  - Private Endpoints: Connected to AFD Premium

Secondary Region: West US 2
  - App Services: 20 apps (50% capacity)
  - Configuration: Zone-redundant, Premium v3
  - Private Endpoints: Connected to AFD Premium

Routing Strategy:
  Normal Operation: 
    - AFD uses latency-based routing
    - Users routed to nearest region
    - Both regions active (true active-active)
  
  Regional Failure:
    - AFD automatically detects unhealthy region (health probes)
    - Routes 100% traffic to healthy region
    - Auto-scaling handles increased load

Capacity Planning:
  - Each region sized for 60-70% of total traffic (N+1 redundancy)
  - Auto-scale max: 150% of normal peak capacity
  - Ensure sufficient quota in both regions
```

---

## Recommended Architecture

Based on all considerations (cost, complexity, reliability), here's the recommended architecture:

### **Architecture Diagram:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Azure DNS (contoso.com)                      │
│                  - A records point to AFD                       │
│                  - Permanent TXT validation records             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│              Azure Front Door Premium (Global)                  │
│  - Origin Group 1 (Priority 1): App Services via Private Link   │
│  - Origin Group 2 (Priority 2): Application Gateway             │
│  - WAF Premium: OWASP 3.2, Bot Protection                       │
│  - 40 Custom Domains, Wildcard Certificates                     │
└───────────────────┬────────────────────┬────────────────────────┘
                    │                    │
        ┌───────────┴──────┐   ┌─────────┴──────────┐
        │                  │   │                     │
        ▼                  ▼   ▼                     ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐
│  East US 2   │   │  West US 2   │   │  Application Gateway     │
│  (Primary)   │   │  (Secondary) │   │  (East US 2)             │
│              │   │              │   │  - Auto-scale 0-10       │
│  App Service │   │  App Service │   │  - Standby failover      │
│  20 apps     │   │  20 apps     │   │  - WAF enabled           │
│  Zone-redun. │   │  Zone-redun. │   │  - 40 backend pools      │
│  Private EP  │   │  Private EP  │   └──────────┬───────────────┘
└──────┬───────┘   └──────┬───────┘              │
       │                  │                      │
       └──────────────────┴──────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Azure Key Vault     │
              │   - 3-5 Certificates  │
              │   - Automated Renewal │
              └───────────────────────┘
```

### **Architecture Specifications:**

```
Layer 1 - Global Edge (Azure Front Door Premium):
  SKU: Premium_AzureFrontDoor
  Endpoints: 1 global endpoint
  Custom Domains: 40 domains
  Certificates: 3-5 wildcard/SAN certs from Key Vault
  
  Origin Group 1 (Primary - Direct to App Services):
    Priority: 1
    Origins: 40 App Services (20 in East US 2, 20 in West US 2)
    Connection: Private Link
    Health Probe: /health every 30 seconds
    Load Balancing: Latency-based (route to nearest region)
  
  Origin Group 2 (Secondary - Application Gateway):
    Priority: 2
    Origins: 1 Application Gateway (East US 2)
    Health Probe: /health every 30 seconds
    Activation: Only if Origin Group 1 unhealthy
  
  WAF:
    Mode: Prevention
    Ruleset: Microsoft_DefaultRuleSet_2.1 + Microsoft_BotManagerRuleSet_1.0
    Custom Rules: Rate limiting (1000 req/min/IP)
  
  Caching:
    Policy: Cache static assets (images, CSS, JS) for 24 hours
    Query String: Include in cache key for API responses
  
  Routing:
    Rules: 40 routes (one per app/domain)
    HTTPS Redirect: Enabled
    HTTP/2: Enabled

Layer 2 - Regional Compute (App Services):
  Primary Region: East US 2
    App Service Plans: 4 plans (10 apps each)
    SKU: P1v3 (2 vCPU, 8 GB RAM)
    Zone Redundancy: Enabled
    Auto-Scale: Min 1, Max 5 instances per plan
    Deployment Slots: 1 staging slot per app
  
  Secondary Region: West US 2
    Configuration: Identical to primary
    Data Sync: Continuous (shared database/storage)
  
  Security:
    Public Access: Disabled
    Private Endpoints: Enabled
    VNet Integration: Enabled
    Managed Identity: Enabled

Layer 3 - Standby Failover (Application Gateway):
  Location: East US 2 (could be any region)
  SKU: Standard_v2
  Auto-Scale: Min 0, Max 10 (saves cost during normal operation)
  
  Configuration:
    Listeners: 40 HTTPS listeners (one per app)
    Backend Pools: 40 pools (pointing to App Services)
    Health Probes: HTTPS /health every 30 seconds
    SSL Certificates: From Key Vault (same as AFD)
  
  WAF:
    SKU: WAF_v2
    Ruleset: OWASP 3.2 (matching AFD)
  
  Activation:
    Trigger: AFD detects Origin Group 1 unhealthy
    Action: AFD automatically routes to Origin Group 2
    Scale: Application Gateway auto-scales from 0 to needed capacity

Layer 4 - Certificate Management (Azure Key Vault):
  Certificates:
    1. wildcard-contoso-com (*.contoso.com)
    2. wildcard-internal-contoso-com (*.internal.contoso.com)
    3. san-special-apps (5-10 special case domains)
  
  Auto-Renewal:
    Azure Function: Checks weekly, renews at 30 days before expiry
    Sync: Automatically updates AFD and App Gateway (via managed identity reference)
  
  Access:
    AFD Premium: Key Vault Secrets User (read certificates)
    Application Gateway: Key Vault Secrets User (read certificates)
    Azure Function: Key Vault Certificates Officer (renew certificates)

Layer 5 - Monitoring (Azure Monitor):
  Application Insights: One per App Service (40 instances)
  Log Analytics Workspace: Centralized logs from all resources
  
  Availability Tests: 40 tests (one per app from 5 global locations)
  
  Alerts:
    Critical: AFD origin health, certificate expiry <7 days
    Warning: Increased latency, CPU >80%
    Info: Deployment events, scaling events
```

### **Expected Performance Characteristics:**

```
Availability:
  Target SLA: 99.95% (composite SLA)
  - AFD Premium SLA: 99.99%
  - App Services (Zone-redundant) SLA: 99.95%
  - Application Gateway SLA: 99.95%
  
  Expected Downtime: ~4.3 hours/year maximum
  Actual Expected: <1 hour/year (with proper failover)

Performance:
  Latency (P95):
    - Global users → AFD edge: 20-50ms
    - AFD → App Service (via Private Link): 5-15ms
    - Total E2E latency: 100-200ms (application dependent)
  
  Throughput:
    - AFD: Unlimited (global scale)
    - Single App Service instance: ~2,000 req/sec
    - Total capacity: 80,000+ req/sec (40 apps × 2,000 req/sec)

Failover Characteristics:
  AFD Origin Group Failover (Primary → Secondary):
    - Detection time: 30-90 seconds (3 failed health probes @ 30s interval)
    - Route update: <5 seconds (AFD edge updates)
    - Total RTO: <2 minutes
  
  Regional Failover (East US 2 → West US 2):
    - Detection time: 30-90 seconds
    - AFD routes to healthy region automatically
    - Total RTO: <2 minutes
  
  Certificate Failover:
    - Validation delay: 0 seconds (pre-validated)
    - HTTPS availability: Immediate

Scalability:
  Auto-Scale Triggers:
    - CPU > 70%: Scale out
    - HTTP Queue Length > 100: Scale out
    - CPU < 30% for 10 minutes: Scale in
  
  Scale Limits:
    - App Service: 1-5 instances per plan (40 plans = max 200 instances)
    - Application Gateway: 0-10 instances
    - Total concurrent users: 500,000+ (with caching)

Cost (Monthly):
  Normal Operation: $930-980
  During Failover: $1,130-1,230
  Per-App Cost: ~$23-25/app/month
```

### **Disaster Recovery Specifications:**

```
Recovery Time Objective (RTO):
  Scenario 1: AFD Complete Outage
    - Failover to Application Gateway
    - RTO: <2 minutes (automated)
  
  Scenario 2: Single Region Failure (East US 2)
    - Failover to West US 2
    - RTO: <2 minutes (automated via AFD)
  
  Scenario 3: Complete Azure Outage (hypothetical)
    - Manual failover to Akamai (if implemented)
    - RTO: 5-10 minutes (DNS TTL + manual activation)
  
  Scenario 4: Certificate Expiration
    - Automated renewal + sync
    - RTO: 0 (no downtime)

Recovery Point Objective (RPO):
  - Application data: 0 (active-active, shared database)
  - Configuration: 0 (IaC in Git)
  - Logs: <5 minutes (Log Analytics ingestion delay)

Backup Strategy:
  Infrastructure:
    - Bicep/Terraform code in Git (versioned)
    - Daily automated deployment to staging (validation)
  
  Certificates:
    - Stored in Azure Key Vault (geo-redundant)
    - Exported to secure storage monthly (offline backup)
  
  Configuration:
    - AFD/App Gateway configuration exported daily
    - Stored in Azure Storage (GRS - geo-redundant)
  
  Application Data:
    - Database: Automated backups per Azure SQL SLA
    - File storage: GRS or ZRS based on criticality
```

---

## Conclusion

This architecture provides:

 **High Availability**: 99.95%+ SLA with multi-region, multi-platform redundancy  
 **Fast Failover**: <2 minute RTO with automated health-based routing  
 **Zero Certificate Delay**: Pre-validated certificates across all platforms  
 **Cost Optimized**: approx $930-980/month ($24/app) with auto-scaling  
 **Security Hardened**: Zero-trust with Private Link, WAF, managed identities  
 **Operationally Mature**: Full monitoring, alerting, IaC, runbooks  


---


**END OF DOCUMENT**
