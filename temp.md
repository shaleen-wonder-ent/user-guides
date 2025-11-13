# High Availability Architecture for HCL-Customer A Connectivity

## Executive Summary

This document outlines the high availability (HA) and resilience architecture for establishing secure connectivity between HCL's Azure environment and Customer A's infrastructure (both Azure and on-premises). The solution addresses both inbound connectivity (Customer A Azure → HCL Azure) via Private Link and outbound connectivity (HCL Azure → Customer A on-premises) via VPN with NAT.

**Document Date**: November 13, 2025  
**Prepared for**: HCL Architecture Team  
**Author**: shaleent_microsoft  
**Last Updated**: November 13, 2025 11:43:10 UTC

---

## 1. Current Architecture Overview

### 1.1 What We Have
- **Inbound Path**: Customer A Azure → Private Link → HCL Application Gateway (AGW) → HCL AKS
- **Outbound Path**: HCL AKS → VPN Gateway → Customer A On-premises
- **Backend**: Multi-zonal AKS cluster in HCL Azure
- **Connectivity Requirements**: NAT for outbound connections, Private Link for inbound

### 1.2 Key Challenges
- Single point of failure concerns for AGW and VPN Gateway
- Questions about zone-level failures and their impact
- Need for enterprise-grade SLAs (≥99.95%)
- Complexity of managing multiple VPN connections from AGW

---

## 2. High Availability Requirements

### 2.1 Why HA is Critical

**Business Impact of Downtime**:
- **Revenue Loss**: Every minute of downtime impacts business transactions
- **SLA Compliance**: Customer A requires enterprise-grade availability
- **Reputation**: Service disruptions affect customer trust
- **Regulatory**: May have compliance requirements for uptime

**Technical Requirements**:
- **RTO (Recovery Time Objective)**: < 5 minutes
- **RPO (Recovery Point Objective)**: Zero data loss
- **Availability Target**: 99.95% minimum
- **Zone Failure Resilience**: Must survive single zone failures

### 2.2 What Constitutes True HA

| Component | Without HA | With HA | Availability Gain |
|-----------|------------|---------|-------------------|
| Application Gateway | Single zone | Zone-redundant | 99.9% → 99.99% |
| VPN Gateway | Single tunnel | Active-Active | 99.9% → 99.95% |
| Private Link | N/A | Built-in HA | 99.99% (managed) |
| AKS Cluster | Single zone | Multi-zone | 99.9% → 99.95% |

---

## 3. Private Link High Availability Solution

### 3.1 What: Zone-Redundant Application Gateway

**Architecture Components**:
- Azure Application Gateway v2 SKU (Standard_v2 or WAF_v2)
- Deployment across 2-3 availability zones
- Private Link Service configuration
- Multiple frontend IP configurations

### 3.2 Why: Benefits of This Approach

1. **Automatic Failover**: No manual intervention required during zone failures
2. **Zero Downtime Maintenance**: Rolling updates without service interruption
3. **Load Distribution**: Traffic automatically distributed across zones
4. **Cost Efficiency**: Pay for single resource, get multi-zone redundancy

### 3.3 How: Implementation Steps

```yaml
Configuration Requirements:
  Application Gateway:
    SKU: Standard_v2 or WAF_v2
    Zones: [1, 2, 3]  # Deploy across all three zones
    Minimum Instances: 2
    Maximum Instances: 10
    Autoscale: Enabled
    
  Private Link Service:
    Load Balancer: Standard SKU
    Frontend IPs: Multiple for redundancy
    NAT IP Configuration: Static allocation
    Visibility: Restricted to Customer A subscription
```

**Deployment Process**:
1. Create zone-redundant AGW v2 in HCL subscription
2. Configure Private Link Service on AGW
3. Share Private Link Service alias with Customer A
4. Customer A creates Private Endpoint in their VNet
5. Configure DNS for private endpoint resolution
6. Test failover scenarios

---

## 3.4 Understanding Zone-Redundant AGW Deployment

### What Does "Deploy AGW in All 3 Zones" Mean?

**CRITICAL CLARIFICATION**: When we say "deploy AGW in all 3 zones," we mean deploying **ONE Application Gateway resource** that automatically distributes its compute instances across multiple availability zones, NOT three separate AGW resources.

#### Technical Implementation

```yaml
What You Create:
  - ONE Application Gateway resource (e.g., agw-hcl-prod)
  - Single management plane
  - Single configuration set
  - One public/private IP address

What Azure Does Behind the Scenes:
  - Creates multiple compute instances
  - Distributes these instances across zones 1, 2, and 3
  - Manages health monitoring and failover
  - Synchronizes configuration automatically
  - Handles traffic distribution transparently
```

#### Visual Representation

```
                    Single AGW Resource (agw-hcl-prod)
                    Management View in Azure Portal
                              │
                              │ Azure Manages Distribution
                              ▼
    ┌──────────────────────────────────────────────────────┐
    │                 Availability Zones                     │
    │                                                        │
    │  Zone 1              Zone 2              Zone 3        │
    │  ┌────────┐          ┌────────┐          ┌────────┐  │
    │  │Instance│          │Instance│          │Instance│  │
    │  │   #1   │          │   #2   │          │   #3   │  │
    │  └────────┘          └────────┘          └────────┘  │
    │                                                        │
    │  All instances share:                                 │
    │  - Same configuration (rules, backends, probes)       │
    │  - Same public/private IP (traffic distributed)       │
    │  - Same backend pools and routing rules               │
    │  - Synchronized state and session data                 │
    └──────────────────────────────────────────────────────┘
```

#### Deployment Example

**Azure Portal Configuration**:
1. Create Application Gateway
2. Choose "Standard_v2" or "WAF_v2" SKU
3. In "Availability zone" section, select zones 1, 2, and 3
4. Set autoscale minimum to 3 (ensures at least 1 instance per zone)

**ARM Template Configuration**:
```json
{
  "name": "agw-hcl-prod",
  "type": "Microsoft.Network/applicationGateways",
  "location": "eastus",
  "zones": ["1", "2", "3"],  // This single parameter enables zone redundancy
  "properties": {
    "sku": {
      "name": "Standard_v2",
      "tier": "Standard_v2"
    },
    "autoscaleConfiguration": {
      "minCapacity": 3,  // Minimum 1 instance per zone
      "maxCapacity": 10  // Can scale up as needed
    }
  }
}
```

**Azure CLI Command**:
```bash
# This creates ONE AGW that spans THREE zones
az network application-gateway create \
    --name "agw-hcl-prod" \
    --resource-group "rg-hcl-network" \
    --location "eastus" \
    --zones 1 2 3 \
    --sku Standard_v2 \
    --capacity 3 \
    --public-ip-address "pip-agw-hcl" \
    --vnet-name "vnet-hcl-hub" \
    --subnet "snet-agw"
```

#### How Zone Failover Works

| Scenario | What Happens | User Impact |
|----------|--------------|-------------|
| **Normal Operation** | Traffic distributed across all 3 zones | Optimal performance |
| **Zone 2 Fails** | Traffic automatically redirects to Zone 1 & 3 | No downtime, slight capacity reduction |
| **Zones 2 & 3 Fail** | All traffic handled by Zone 1 | No downtime, reduced capacity |
| **Zone 2 Recovers** | Traffic automatically rebalances | Performance improves |

#### Common Misconceptions Clarified

| ❌ Incorrect Understanding | ✅ Correct Understanding |
|---------------------------|-------------------------|
| "I need 3 separate AGW resources" | "I need 1 AGW resource configured for 3 zones" |
| "Each zone has different configuration" | "All zones share the same configuration" |
| "I must manually manage failover" | "Azure handles failover automatically" |
| "Each zone needs its own IP" | "Single IP address serves all zones" |
| "Zone deployment is complex" | "It's just a configuration parameter" |

#### Cost Implications of Zone Redundancy

```yaml
Cost Breakdown:
  Single Zone AGW:
    - Instances: 1 (minimum)
    - Cost: ~$0.25/hour
    - Monthly: ~$180
    
  Zone-Redundant AGW (3 zones):
    - Instances: 3 (minimum, 1 per zone)
    - Cost: ~$0.75/hour
    - Monthly: ~$540
    - Additional cost: ~$360/month
    - Benefit: 99.99% SLA vs 99.95%
```

#### Verification and Monitoring

**To Verify Zone-Redundant Deployment**:

```powershell
# PowerShell
$agw = Get-AzApplicationGateway -Name "agw-hcl-prod" -ResourceGroupName "rg-hcl-network"
Write-Host "Zones configured: $($agw.Zones -join ', ')"  # Should show: 1, 2, 3
Write-Host "Current capacity: $($agw.Sku.Capacity)"
Write-Host "Autoscale min: $($agw.AutoscaleConfiguration.MinCapacity)"
```

**Monitor Zone Health**:
```kql
// KQL Query for Azure Monitor
AzureMetrics
| where ResourceId contains "agw-hcl-prod"
| where MetricName == "HealthyHostCount"
| summarize HealthyHosts = avg(Total) by bin(TimeGenerated, 5m)
| render timechart
```

#### Key Takeaways

1. **Single Resource**: You manage ONE AGW, not three
2. **Automatic Distribution**: Azure handles instance placement
3. **Transparent Failover**: No manual intervention needed
4. **Unified Configuration**: One set of rules for all zones
5. **Higher SLA**: 99.99% availability with zone redundancy
6. **Increased Cost**: Approximately 3x the single-zone cost for 3-zone redundancy

---

## 3.5 Regional Considerations for AGW

### Zone Redundancy vs Regional Redundancy

**Important Distinction**:
- **Zone Redundancy** (what we're implementing): Protects against failures within a region
- **Regional Redundancy** (future consideration): Protects against entire region failures

```yaml
Current Zone-Redundant Setup:
  Protection Against:
    ✅ Single availability zone failure
    ✅ Multiple zone failures (if at least 1 zone survives)
    ✅ Zone-level maintenance and updates
    ❌ Complete regional failure
    ❌ Region-wide Azure outage
    
  Scope: Single Region (e.g., East US)
  SLA: 99.99% within the region
```

### When to Consider Multi-Region

| Scenario | Zone-Redundant Sufficient? | Need Multi-Region? |
|----------|---------------------------|-------------------|
| Zone maintenance | ✅ Yes | No |
| Data center failure | ✅ Yes | No |
| Regional disaster | ❌ No | ✅ Yes |
| Compliance requirements | Depends | Maybe |
| Global user base | Partially | ✅ Yes |

**Recommendation**: Start with zone redundancy (current plan), evaluate regional redundancy based on:
- Customer A's disaster recovery requirements
- Budget constraints (multi-region doubles infrastructure costs)
- Regulatory compliance needs
- Actual risk assessment for regional failures

---

## 4. VPN High Availability Solution

### 4.1 What: Active-Active VPN Gateway Configuration

**Architecture Components**:
- Azure VPN Gateway (VpnGw2-5 SKU for zone redundancy)
- Dual public IP addresses
- Redundant IPsec tunnels
- BGP routing for dynamic path selection

### 4.2 Why: Advantages of Active-Active VPN

1. **Continuous Availability**: Both tunnels active simultaneously
2. **Automatic Failover**: Sub-second convergence with BGP
3. **Load Balancing**: Distribute traffic across tunnels
4. **Maintenance Windows**: Update one tunnel while other handles traffic
5. **SLA Guarantee**: Microsoft-backed 99.95% availability

### 4.3 How: Implementation Architecture

```yaml
VPN Gateway Configuration:
  Gateway:
    SKU: VpnGw2AZ or higher
    Type: RouteBased
    VPN Type: Active-Active
    Zones: [1, 2, 3]
    
  Connections:
    Tunnel1:
      Public IP: pip-vpn-primary
      BGP: Enabled
      ASN: 65001
      Peer IP: Customer-A-Firewall-1
      
    Tunnel2:
      Public IP: pip-vpn-secondary
      BGP: Enabled
      ASN: 65001
      Peer IP: Customer-A-Firewall-2
      
  NAT Rules:
    Type: Static
    Mode: EgressSnat
    Internal Subnet: 10.0.0.0/16
    External Mapping: 192.168.1.0/24
```

---

## 5. Recommended End-to-End Architecture

### 5.1 Hub-Spoke Topology with Gateway Transit

```
┌─────────────────────────────────────────────────────────────┐
│                     Customer A Azure                         │
│                           │                                  │
│                    Private Endpoint                          │
│                           │                                  │
└───────────────────────┬───────────────────────────────────┘
                        │ Private Link
                        ▼
┌─────────────────────────────────────────────────────────────┐
│                    HCL Hub VNet                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   Zone-Redundant Application Gateway v2              │    │
│  │   Zones: 1, 2, 3 | Autoscale: 2-10 instances       │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   Azure Firewall (Optional)                          │    │
│  │   Zones: 1, 2, 3 | For additional security         │    │
│  └─────────────────────────────────────────────────────┘    │
│                           │                                  │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   Active-Active VPN Gateway                          │    │
│  │   2 Public IPs | BGP Enabled | Zone-redundant      │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────┬──────────────────────────────────┘
                           │ Dual IPsec Tunnels
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              Customer A On-Premises                          │
│         Redundant Firewalls/VPN Devices                     │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   HCL Spoke VNets                            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │   Multi-Zone AKS Cluster                             │    │
│  │   Node Pools across Zones 1, 2, 3                   │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Traffic Flow Patterns

**Inbound Flow (Customer A → HCL)**:
1. Customer A application → Private Endpoint
2. Private Link connection → HCL AGW
3. AGW (with zone failover) → Backend AKS pods
4. Response follows reverse path

**Outbound Flow (HCL → Customer A On-Prem)**:
1. AKS pods → Hub VNet (via peering)
2. NAT translation at VPN Gateway
3. Active tunnel selection via BGP
4. Encrypted traffic → Customer A on-premises

---

## 6. Failover Scenarios and Recovery

### 6.1 Zone Failure Scenarios

| Failure Type | Impact | Recovery Time | Automatic? |
|--------------|---------|---------------|------------|
| AGW Zone Failure | Traffic redirects to healthy zones | < 30 seconds | Yes |
| VPN Gateway Failure | Traffic switches to secondary tunnel | < 60 seconds | Yes |
| AKS Node Failure | Pods rescheduled to healthy nodes | < 2 minutes | Yes |
| Private Link Failure | Managed by Microsoft | < 30 seconds | Yes |

### 6.2 Testing Procedures

**Monthly HA Validation Tests**:
1. **Zone Failure Simulation**
   - Disable one availability zone
   - Verify traffic continues flowing
   - Measure failover time

2. **VPN Tunnel Failure**
   - Disconnect primary tunnel
   - Verify BGP reconvergence
   - Check NAT translations

3. **Load Testing**
   - Simulate peak traffic conditions
   - Verify autoscaling triggers
   - Monitor performance metrics

---

## 7. Monitoring and Alerting Strategy

### 7.1 Key Metrics to Monitor

```yaml
Monitoring Configuration:
  Application Gateway:
    - Healthy Host Count
    - Failed Request Count
    - Response Time (P95, P99)
    - Throughput
    - Connection Count
    
  VPN Gateway:
    - Tunnel Connectivity
    - BGP Peer Status
    - Bandwidth Utilization
    - Packet Drop Rate
    - IPsec SA Count
    
  Private Link:
    - Endpoint Connectivity
    - Data Path Availability
    - Connection State
    
  AKS Cluster:
    - Node Health
    - Pod Availability
    - Cluster Autoscaler Status
    - Resource Utilization
```

### 7.2 Alert Configuration

| Alert | Threshold | Action |
|-------|-----------|--------|
| AGW Unhealthy Hosts | > 0 | Investigate immediately |
| VPN Tunnel Down | Any tunnel | Check BGP status |
| High Latency | > 100ms P99 | Scale out AGW |
| Certificate Expiry | < 30 days | Renew certificates |

---

## 8. Cost Optimization

### 8.1 Estimated Monthly Costs

| Component | Configuration | Estimated Cost |
|-----------|--------------|----------------|
| AGW v2 (Zone-redundant) | 2-10 instances, autoscale | $400-1,200 |
| VPN Gateway (Active-Active) | VpnGw2AZ | $400 |
| Private Link | Data processing charges | $50-200 |
| Monitoring | Log Analytics, Metrics | $100-300 |
| **Total Estimated** | **With HA** | **$950-2,100** |

### 8.2 Cost Optimization Tips

1. **Right-size AGW**: Start with minimum instances, use autoscale
2. **VPN Gateway SKU**: Choose based on bandwidth needs
3. **Reserved Instances**: 1-3 year reservations save 30-50%
4. **Monitoring Retention**: Adjust log retention based on compliance needs

---

## 9. Implementation Roadmap

### Phase 1: Foundation (Week 1-2)
- [ ] Deploy zone-redundant AGW v2
- [ ] Configure Private Link Service
- [ ] Set up basic monitoring

### Phase 2: VPN HA (Week 3-4)
- [ ] Deploy Active-Active VPN Gateway
- [ ] Configure BGP routing
- [ ] Implement NAT rules
- [ ] Test tunnel failover

### Phase 3: Integration (Week 5-6)
- [ ] Connect Customer A via Private Link
- [ ] Establish VPN tunnels to on-premises
- [ ] Configure end-to-end monitoring
- [ ] Perform integration testing

### Phase 4: Validation (Week 7-8)
- [ ] Conduct failover tests
- [ ] Performance testing
- [ ] Documentation completion
- [ ] Handover to operations

---

## 10. Service Level Agreements (SLAs)

### 10.1 Composite SLA Calculation

**For Serial Dependencies (worst case)**:
- Private Link: 99.99%
- AGW (zone-redundant): 99.99%
- VPN Gateway (Active-Active): 99.95%
- AKS (multi-zone): 99.95%

**Composite SLA = 0.9999 × 0.9999 × 0.9995 × 0.9995 = 99.88%**

### 10.2 Achieving Higher SLA

To reach 99.95%+ composite SLA:
1. Implement multi-region failover (Active-Passive)
2. Use Traffic Manager for DNS-based routing
3. Deploy redundant Private Link endpoints
4. Consider Azure Front Door for global load balancing

---

## 11. Security Considerations

### 11.1 Network Security

- **Network Segmentation**: Use NSGs and ASGs
- **DDoS Protection**: Enable Standard DDoS protection
- **WAF Rules**: Configure OWASP rules on AGW
- **Private Endpoints**: No public IP exposure
- **Encryption**: IPsec for VPN, TLS for HTTPS

### 11.2 Access Control

- **RBAC**: Implement least-privilege access
- **Managed Identities**: For service-to-service auth
- **Key Vault**: Store secrets and certificates
- **Conditional Access**: For administrative access

---

## 12. Operational Runbooks

### 12.1 Incident Response Procedures

**AGW Degradation**:
1. Check Application Insights for errors
2. Review AGW metrics in Azure Monitor
3. Verify backend health probes
4. Scale out if needed
5. Engage Microsoft Support if required

**VPN Connectivity Loss**:
1. Verify both tunnel status
2. Check BGP peer connectivity
3. Review IPsec SA establishment
4. Verify on-premises device status
5. Check NAT rule configuration

### 12.2 Maintenance Procedures

**Planned Maintenance Windows**:
- Schedule during low-traffic periods
- Notify Customer A 72 hours in advance
- Test in non-production first
- Have rollback plan ready
- Monitor closely post-change

---

## 13. Conclusion and Recommendations

### 13.1 Key Recommendations

1. **Immediate Actions**:
   - Deploy zone-redundant AGW v2 (single resource across 3 zones)
   - Implement Active-Active VPN Gateway
   - Enable comprehensive monitoring

2. **Short-term (3 months)**:
   - Conduct monthly failover tests
   - Optimize autoscaling policies
   - Review and adjust alert thresholds

3. **Long-term (6-12 months)**:
   - Consider multi-region deployment
   - Evaluate Azure Virtual WAN
   - Implement automated remediation

### 13.2 Success Criteria

- ✅ Achieve 99.95% availability
- ✅ Zero unplanned outages due to zone failures
- ✅ Sub-minute failover times
- ✅ Successful monthly DR tests
- ✅ Customer A satisfaction with performance

---

## Appendix A: Configuration Scripts

### A.1 AGW Deployment (Terraform snippet)

```hcl
resource "azurerm_application_gateway" "main" {
  name                = "agw-hcl-prod-001"
  location            = "eastus"
  resource_group_name = azurerm_resource_group.main.name
  zones               = ["1", "2", "3"]  # Single AGW across 3 zones
  
  sku {
    name     = "Standard_v2"
    tier     = "Standard_v2"
  }
  
  autoscale_configuration {
    min_capacity = 3  # Ensures at least 1 instance per zone
    max_capacity = 10
  }
  
  # Additional configuration...
}
```

### A.2 VPN Gateway Active-Active Setup

```powershell
# PowerShell script for Active-Active VPN
$GW = Get-AzVirtualNetworkGateway -Name "vgw-hcl-prod" -ResourceGroupName "rg-network"
$GW.ActiveActive = $true
$GW.Sku.Name = "VpnGw2AZ"
$GW.EnableBgp = $true
Set-AzVirtualNetworkGateway -VirtualNetworkGateway $GW
```

---

## Appendix B: Monitoring Queries

### B.1 Application Gateway Health Query (KQL)

```kql
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS"
| where TimeGenerated > ago(1h)
| summarize UnhealthyHosts = countif(host_s == "Unhealthy"),
            TotalHosts = count() by bin(TimeGenerated, 5m)
| where UnhealthyHosts > 0
```

### B.2 VPN Connection Status Query

```kql
AzureDiagnostics
| where Category == "TunnelDiagnosticLog"
| where TimeGenerated > ago(1h)
| project TimeGenerated, Resource, OperationName, ResultType, Message
| where ResultType != "Success"
```

---

## Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-11-13 | shaleent_microsoft | Initial draft |
| 1.1 | 2025-11-13 11:43:10 UTC | shaleent_microsoft | Added detailed explanation of zone-redundant AGW deployment (Section 3.4 and 3.5) |

**Next Review Date**: February 13, 2026  
**Document Owner**: HCL Architecture Team  
**Distribution**: HCL Technical Team, Customer A Architecture Team

---

# Understanding VPN Gateway High Availability in Azure  
### (Why You Don’t Need Two Gateways in a VNet)

This document explains a common confusion:  
**“How can we achieve VPN High Availability—do we need two VPN gateways in one VNet?”**

The short and correct answer is:

> **You only deploy ONE Azure VPN Gateway.  
> Azure creates TWO gateway instances behind it when Active-Active mode is enabled.  
> This gives full HA without needing two separate gateways.**

---

# 1. The Source of Confusion
Many engineers assume that HA requires:
- Two VPN Gateways  
- Two Gateway Subnets  
- Two VNets  

**But Azure does not allow multiple VPN Gateways in a single VNet.**  
Instead, the HA is provided *inside* the gateway resource itself.

---

# 2. How Azure Actually Provides HA

## ✅ One Logical Gateway → Two Physical Instances

When you deploy **one VPN Gateway** and enable **Active-Active mode**, Azure automatically creates:

- **Gateway Instance #1** → Public IP #1  
- **Gateway Instance #2** → Public IP #2  

Both exist **inside the same GatewaySubnet** in **one VNet**.
