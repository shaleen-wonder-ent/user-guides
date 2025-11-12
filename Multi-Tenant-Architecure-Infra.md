# Multi-Tenant Application Architecture on Azure VMs

**Scope:**  
This document defines the complete **multi-tenant architecture** for a web and API-based solution hosted on **Azure Virtual Machines**.  
It covers all layers except the SQL Server database, focusing on the **application**, **API**, **infrastructure**, and **operations** components for both **Active–Active** and **Active–Passive** designs.

---

## 1. Architecture Overview

### Current Setup
- All compute resources are hosted on **Azure Virtual Machines**.
- Application stack includes **Web**, **API**, **SSIS**, **SSRS**, and **File Services**.
- Database: **SQL Server 2019** (handled separately).
- Deployment Regions: **East US** and **West US**.

### Goals
- Multi-tenancy for multiple clients on shared infrastructure.
- High availability and scalability across regions.
- Disaster recovery and performance optimization.
- Maintain compliance, security, and network resiliency.

---

## 2. High-Level Architecture Summary

- **Front Door + Traffic Manager:** Global routing, WAF, and DR failover.
- **Regional App Gateways:** TLS termination, layer-7 routing, and load balancing.
- **VM Scale Sets:** Host web and API workloads per region.
- **SSIS/SSRS VMs:** For ETL and reporting workloads.
- **SQL Server Always On AG:** Provides HA/DR (handled separately).
- **File Servers:** SMB shares for local file access and third-party tools.
- **Monitoring, Key Vault, and Bastion:** Centralized operations and secure management.

### Active–Passive Mode
- East US serves as the **active** region.
- West US remains in **standby** mode with asynchronous data replication.
- Failover via **Traffic Manager** or **Front Door health probes**.

### Active–Active Mode
- Both regions handle live user traffic.
- Front Door distributes traffic based on performance or proximity.
- Application layers remain synchronized, and data replication is asynchronous between regions.

---

## 3. Multi-Tenancy at the Application Layer

### Option A: Shared Instance with Tenant-Aware Logic (Recommended)
- A single codebase and deployment shared across tenants.
- Each tenant identified via a **Tenant ID** in token, header, or subdomain.
- Application resolves tenant context dynamically.

**Advantages**
- Simpler deployment and upgrades.
- Shared compute = lower cost.
- Centralized monitoring.

**Challenges**
- Strong data isolation must be enforced.
- Performance impact possible under heavy multi-tenant load.

### Option B: Tenant-Scoped Application Instances
- Separate App Pools or VMSS instances per tenant group.
- Routing via subdomain (e.g., `tenantA.example.com → TenantA VMSS`).

**Advantages**
- Better performance and fault isolation per tenant.
- Custom scaling possible.

**Challenges**
- Increased management overhead.
- More complex DR and CI/CD handling.

### Option C: Dedicated Environments (VIP Clients)
- Full isolation: dedicated VMs, App Gateway, and supporting services.
- Automated provisioning via CI/CD templates.

**Advantages**
- Strongest isolation and compliance posture.
- Tailored SLAs per tenant.

**Challenges**
- Highest operational and infrastructure cost.

---

## 4. Multi-Tenancy at the API Layer

### Core Design Principles
<img width="500" height="251" alt="image" src="https://github.com/user-attachments/assets/f3c24329-c509-4a4f-9d22-8a414f62905a" />


### Tenant Context Middleware Example
```csharp
public class TenantMiddleware {
    public async Task Invoke(HttpContext context) {
        var tenantId = context.User.FindFirst("tenant_id")?.Value 
                        ?? context.Request.Headers["X-Tenant-ID"];
        context.Items["TenantId"] = tenantId;
        await _next(context);
    }
}
```

### API Deployment Models
1. **Single Shared API VMSS** – All tenants share the same API backend.
2. **Per-Tenant API Pools** – Isolated APIs for high-volume or VIP tenants.

---

## 5. Infrastructure and Networking Design

### Hub–Spoke Model per Region
- **Hub VNet:** Firewall, Bastion, AD DS, DDoS Protection, and Monitoring.
- **Spoke VNets:** Host web, API, and file workloads.
- **Peering:** Spokes communicate only via Hub VNet.

**Example**
```
EastUS-Hub
│
├─ Spoke-Shared
│   ├─ Web VMSS
│   ├─ API VMSS
│   ├─ Redis / Cache
│
├─ Spoke-TenantA (optional)
│   ├─ Custom VMSS
│
└─ Spoke-TenantB (optional)
```

### Isolation Controls
- **NSGs and UDRs:** Control lateral movement.
- **Azure Firewall:** Enforce centralized policies.
- **Key Vault:** Manage per-tenant secrets.
- **File Shares:** NTFS ACLs ensure folder-level tenant isolation.
- **Monitoring:** Logs and telemetry tagged with `TenantId`.

---

## 6. Tenant Lifecycle Management

| Stage | Mechanism |
|--------|------------|
| **Provisioning** | CI/CD pipeline deploys app, config, and registers tenant |
| **Configuration** | Adds tenant metadata to registry or Key Vault |
| **Isolation Controls** | Creates storage, folders, secrets, and RBAC roles |
| **Monitoring** | TenantId tagging for observability |
| **Deprovisioning** | Automated cleanup of tenant resources and keys |

Automation ensures consistent onboarding and safe removal of tenants with no manual intervention.

---

## 7. Request Flow (End-to-End)

```
Client (tenantA.example.com)
   ↓
Azure Front Door (Global WAF)
   ↓
Regional App Gateway (East/West)
   ↓
Web VMSS (Tenant-Aware App)
   ↓
API VMSS (Tenant Context Middleware)
   ↓
SQL AG Listener (Tenant-Scoped Query)
   ↓
Redis Cache (Key Prefix = TenantId)
   ↓
Telemetry (App Insights Tagged by Tenant)
```

---

## 8. Security and Governance

- **Identity:** Azure Entra multi-tenant registration (per-tenant directories).
- **Access Control:** Tenant-specific RBAC and claims-based auth.
- **Secrets Management:** Key Vault per tenant or per group.
- **Network Security:** Private endpoints, NSGs, and Firewall policies.
- **Audit Logging:** Tenant-specific logs in App Insights or Log Analytics.
- **Compliance:** Tenant-level audit and retention policies.

---

## 9. Monitoring and Observability

| Component | Function |
|------------|-----------|
| **Azure Monitor** | VM and app performance metrics |
| **Log Analytics** | Centralized logging, tagged with TenantId |
| **Application Insights** | Telemetry, request traces, dependency maps |
| **Defender for Cloud** | Security and compliance posture |
| **Sentinel (Optional)** | Advanced SIEM and threat detection |

Use **custom dimensions** such as `TenantId`, `Region`, and `Environment` in telemetry to isolate tenant metrics.

---

## 10. CI/CD and Deployment Automation

- **Infrastructure as Code (IaC):** Bicep or Terraform templates per tenant.
- **Pipelines:** Azure DevOps or GitHub Actions handle provisioning, scaling, and teardown.
- **Stages:**
  1. Build and Package
  2. Deploy App/VMSS
  3. Register Tenant (App + Config)
  4. Verify Health
- **Config Management:** AppSettings or environment variables per tenant.
- **Rollback:** Automated via versioned deployments.

---

## 11. High Availability (HA) and Disaster Recovery (DR)

| Model | Description | Active Region(s) | Failover Mechanism |
|--------|-------------|------------------|--------------------|
| **Active–Active** | Both regions serve live traffic | East + West | Front Door rerouting |
| **Active–Passive** | Primary handles load; secondary standby | East | Traffic Manager or manual trigger |

**Key Considerations:**
- AG replication is asynchronous cross-region.
- File sync via DFS-R or robocopy.
- VM-level DR via Azure Site Recovery (ASR).
- Health probes for automatic regional failover.

---

## 12. Summary and Recommendations

### Multi-Tenancy Strategy
| Layer | Approach | Isolation | Scaling |
|-------|-----------|------------|----------|
| **Web App** | Shared, tenant-aware logic | Logical | Autoscale VMSS |
| **API** | Shared or per-tenant | Logical / Medium | Autoscale VMSS |
| **Infrastructure** | Hub-Spoke VNets | Network | Per spoke |
| **Storage** | Shared SMB with ACLs | Logical | Per folder/share |
| **Monitoring** | Tenant-tagged logs | Logical | Centralized |
| **CI/CD** | Parameterized templates | Config-based | Automated |

### Recommendations
- Start with a **shared tenant-aware model** for simplicity.
- Use **Row-Level Security (RLS)** in SQL for DB-level isolation.
- Employ **Key Vault** and **Azure Policy** for secure configuration.
- Tag telemetry and metrics with `TenantId`.
- Automate tenant onboarding and deprovisioning via CI/CD pipelines.
- Consider **dedicated environments** only for high-value or compliance-bound tenants.

---


