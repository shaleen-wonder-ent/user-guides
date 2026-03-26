# Azure Landing Zone — Contoso Multi-Tenant SaaS Contact Center Platform

## Executive Summary & Reference Guide

---

## 1. Executive Summary

Contoso operates an **agent-based contact center application** for the airline industry, currently serving one customer deployed across two Azure regions (West Europe & North Europe) in an Active-Active configuration.

### The Challenge

As Contoso looks to **onboard new airline customers** in potentially different geographies, the current approach of building infrastructure from scratch for each customer is not sustainable. Contoso needs a **repeatable, scalable, and compliant foundation** that allows rapid tenant onboarding without re-engineering.

### The Solution: Azure Landing Zone + Deployment Stamps

We recommend Contoso adopt the **Azure Landing Zone** framework from Microsoft's Cloud Adoption Framework (CAF), combined with the **Deployment Stamps** pattern for multi-region scale-out and the **Azure Multi-Tenant SaaS** architectural guidance for tenant isolation.

This approach will give Contoso:

| Capability | Benefit |
| - | - |
| **Repeatable Infrastructure** | Deploy a new region in hours, not weeks — same IaC template every time |
| **Multi-Tenant by Design** | Onboard new customers with subdomain routing, per-tenant DB schemas, and dedicated frontends |
| **GDPR & SOC 2 Compliance** | Data residency enforcement, encryption, audit logging, and breach notification built into the platform |
| **Centralized Governance** | Azure Policy guardrails inherited automatically by every new stamp and tenant |
| **Operational Excellence** | Single pane of glass for monitoring, security, and cost management across all tenants and regions |

---

## 2. What is an Azure Landing Zone?

An **Azure Landing Zone** is a prescriptive architectural blueprint from Microsoft that provides the foundation for cloud adoption at enterprise scale. It ensures that when workloads are deployed, they land in an environment that has:

* ✅ **Proper identity & access controls** (who can do what)
* ✅ **Network segmentation & security** (how traffic flows)
* ✅ **Governance guardrails** (what's allowed and what's not)
* ✅ **Monitoring & operations** (visibility into everything)
* ✅ **Cost management** (track spend per tenant, per region)
* ✅ **Compliance alignment** (GDPR, SOC 2, ISO 27001)

It consists of two key components:

| Component | Purpose | Contoso Mapping |
| - | - | - |
| **Platform Landing Zone** | Shared services — identity, networking, monitoring, security | Entra ID, Hub VNets, Azure Firewall, Sentinel, Azure Monitor |
| **Application Landing Zone** | Workload-specific resources in governed subscriptions | Contact Center stamps (frontend, BFF, APIs, databases) per region |

> [What is an Azure Landing Zone? — Microsoft Learn](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)

---

## 3. Why This Matters for Contoso

### Current State at Contoso

| Area | Status | Details |
| - | - | - |
| Multi-Region Deployment | ✅ Done | Active-Active across West EU & North EU |
| Subdomain Tenant Routing | ✅ Done | `tenant1.app.com` pattern via Azure Front Door |
| Per-Tenant DB Schemas | ✅ Done | Shared PostgreSQL with separate schemas per customer |
| Security Tooling | ✅ Done | Sentinel, Defender, WAF, DDoS, Key Vault HSM |
| Network Security | ✅ Done | Hub-Spoke VNets, Azure Firewall, Private Endpoints, NSGs |
| Cross-Region Monitoring | ✅ Done | Azure Monitor, App Insights, Log Analytics |

### What's Missing (The Gaps)

| Area | Gap | Impact |
| - | - | - |
| **Management Group Hierarchy** | No formal MG structure aligned to CAF | Cannot enforce policies consistently across tenants/regions |
| **Azure Policy Guardrails** | Policies not applied at MG level | New deployments may not inherit security/compliance controls |
| **IaC Templates (Stamp)** | No repeatable Bicep/Terraform template | Each new region requires manual setup — slow & error-prone |
| **Tenant Onboarding Automation** | No control plane API for tenant lifecycle | Onboarding a new customer is a manual, multi-step process |
| **GDPR Data Residency Enforcement** | No Azure Policy for allowed locations | Data could inadvertently be stored outside allowed regions |
| **Tenant Definition & SLA Tiering** | No formal definition of what constitutes a "tenant" or how SLAs differ per customer | Cannot differentiate service levels or plan for large-tenant growth |
| **Cost Allocation** | No per-tenant cost tracking | Cannot show each customer their infrastructure cost |
| **Resource Organization Strategy** | No formal decision on shared vs. pooled vs. isolated resource boundaries | Inconsistent resource grouping may lead to governance and billing blind spots |

---

## 4. The Eight Design Areas

Microsoft defines **eight design areas** that every Landing Zone must address. Here's how each applies to Contoso:

| # | Design Area | What It Covers |
| - | - | - |
| 1 | **Billing & Entra Tenant** | Subscription structure, billing model, Entra ID tenant setup |
| 2 | **Identity & Access Management** | Entra ID, SSO, Conditional Access, PIM, RBAC |
| 3 | **Resource Organization** | Management Groups, Subscriptions, Resource Groups, naming & tagging |
| 4 | **Network Topology & Connectivity** | Hub-Spoke, Azure Firewall, Private Endpoints, DNS, Front Door |
| 5 | **Security** | Defender for Cloud, Sentinel, Key Vault, WAF, DDoS |
| 6 | **Management** | Azure Monitor, Log Analytics, App Insights, Automation |
| 7 | **Governance** | Azure Policy, Blueprints, Cost Management, tagging standards |
| 8 | **Platform Automation & DevOps** | IaC (Bicep/Terraform), CI/CD pipelines, GitOps |

> [Azure Landing Zone Design Areas — Microsoft Learn](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-areas)

---

## 5. Recommended Architecture Approach

Based on Contoso's current state and goals, we recommend the following patterns:

### 5.1 Deployment Model: Pure SaaS (Model A)

Contoso centrally owns and operates all infrastructure. Customers (tenants) access the platform via subdomain routing. No customer-deployed resources.

> [ISV Considerations for Azure Landing Zones — Microsoft Learn](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/isv-landing-zone)

### 5.2 Tenancy Model: Partially Shared

* **Dedicated** — Tenant UIs (frontend portals)
* **Shared** — BFF orchestration layer, API Management, Azure OpenAI
* **Logically Isolated** — Database (per-tenant schemas + Row-Level Security), Redis (prefixed keys), App Config (prefixed settings)

This is Microsoft's recommended balance between cost efficiency and tenant isolation for most SaaS ISVs.

> [Tenancy Models for Multitenant Solutions — Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models)

### 5.3 Scale-Out Pattern: Deployment Stamps

Each Azure region gets an identical "stamp" of infrastructure (frontend, app layer, data layer, networking) deployed from the same IaC template. New regions = new stamps. Tenants are routed to their stamp via Azure Front Door.

> [Deployment Stamps Pattern — Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp)

### 5.4 Multi-Region: Multinational Landing Zone

For customers in different regulatory jurisdictions (EU, UK, APAC), Contoso can create region-specific Management Groups with sovereignty-appropriate Azure Policies — all inheriting from a global governance baseline.

> [Multinational Landing Zone — Microsoft Learn](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/landing-zone-multinational)

---

## 6. Key Decisions Contoso Must Make

Before finalizing the architecture, Contoso should formally resolve these foundational questions. The guidance column links directly to the Microsoft documentation that provides the framework to answer each:

| # | Decision | Question to Answer | Guidance |
| - | - | - | - |
| 1 | **Tenant Definition** | What is a "tenant"? Is it a customer? A customer's regional division? A department? | [SaaS & Multitenant Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/saas-multitenant-solution-architecture/), [Architectural Considerations](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/overview) |
| 2 | **SLA Tiering** | Will all customers receive the same SLA, or can premium customers get higher availability/performance guarantees? | [Architectural Considerations](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/overview) |
| 3 | **Resource Organization** | For each Azure service, should the resource be shared, pooled, or isolated per tenant? What's the Resource Group strategy per tenant? | [Architectural Approaches Overview](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview) |
| 4 | **Cost Model** | How will per-customer costs be measured and allocated? Tagging-based? Metering-based? Flat rate per tenant? | [Architectural Approaches Overview](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview) |
| 5 | **Noisy Neighbor** | If one customer generates 10x the traffic, how do you prevent performance degradation for others? Rate limiting? Dedicated compute? | [Architectural Approaches Overview](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview), [Architectural Considerations](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/overview) |
| 6 | **Growth Planning** | What happens when a tenant scales significantly? At what threshold do they get dedicated resources? | [Architectural Considerations](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/overview) |
| 7 | **Data Residency** | Which regions are acceptable for each customer's data? Is EU-only sufficient, or do some customers require country-specific residency? | [Governance & Compliance in Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/governance-compliance), [Azure GDPR Guidance](https://learn.microsoft.com/en-us/compliance/regulatory/gdpr) |
| 8 | **Onboarding Automation** | What is the target time to onboard a new customer? What manual steps exist today that should be automated? | [Deployment & Configuration](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/deployment-configuration), [Automate Landing Zones Across Tenants](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/multi-tenant/automation) |

---

## 7. Compliance Summary

### 7.1 GDPR

| Requirement | How It's Addressed |
| - | - |
| **Data Residency** (Art. 44–49) | Azure Policy restricts resource deployment to allowed regions only |
| **Right to Erasure** (Art. 17) | Per-tenant schema deletion APIs |
| **Data Portability** (Art. 20) | Tenant-scoped data export APIs |
| **Encryption** (Art. 25) | Key Vault Premium HSM (at rest) + TLS 1.2+ (in transit) |
| **Breach Notification** (Art. 33) | Sentinel alerts → Logic Apps → automated 72-hour notification |
| **Records of Processing** (Art. 30) | Azure Activity Logs + Log Analytics retention |
| **Access Controls** (Art. 32) | Entra ID RBAC + Conditional Access + PIM |

> [Azure GDPR Guidance — Microsoft Learn](https://learn.microsoft.com/en-us/compliance/regulatory/gdpr)

### 7.2 SOC 2

| Trust Criteria | Azure Controls |
| - | - |
| **Security** | Azure Firewall, NSGs, DDoS, WAF, Defender for Cloud |
| **Availability** | Active-Active multi-region, SLA-backed PaaS |
| **Confidentiality** | Key Vault (CMK), Private Endpoints, encryption |
| **Processing Integrity** | Application Insights, distributed tracing |
| **Privacy** | Entra ID RBAC, audit logging, data classification |

---

## 8. Microsoft Reference Architecture Links

### Landing Zone Foundations

| # | Resource | Why It Matters for Contoso | Link |
| - | - | - | - |
| 1 | **What is an Azure Landing Zone?** | Core concepts — Management Groups, subscriptions, Platform vs. Application LZ. Starting point for all Landing Zone design. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/) |
| 2 | **Landing Zone Design Areas** | The eight pillars (billing, identity, networking, security, etc.) that every LZ must address. Use as a checklist for Contoso. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-areas) |
| 3 | **Management Groups in Landing Zones** | How to structure the MG hierarchy — directly applicable to Contoso's Platform/LZ/Sandbox structure. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-management-groups) |
| 4 | **ISV Considerations for Landing Zones** ⭐ | **Most relevant for Contoso.** Covers deployment models (Pure SaaS vs. customer-deployed), subscription design, and control plane patterns for ISVs building SaaS. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/isv-landing-zone) |
| 5 | **Landing Zone Regions** | Guidance for expanding Landing Zones to new Azure regions — essential for Contoso's multi-region stamp strategy. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/considerations/regions) |
| 6 | **Multinational Landing Zone** | How to modify LZ architecture for different countries/regulatory zones (EU, UK, APAC). Covers data sovereignty and regional policies. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/landing-zone-multinational) |

### Multi-Tenant SaaS Architecture

| # | Resource | Why It Matters for Contoso | Link |
| - | - | - | - |
| 7 | **SaaS and Multitenant Solution Architecture** ⭐ | **Start here for SaaS fundamentals.** Comprehensive guide that distinguishes the SaaS business model from the multitenancy architecture pattern — a critical distinction for Contoso. Covers B2B SaaS scenarios (Contoso's model), tenant definition, isolation strategies, and includes a multitenancy checklist plus service-specific guidance for individual Azure services (PostgreSQL, Functions, APIM, etc.). Also addresses anti-patterns like noisy neighbors and provides code samples and sample architectures. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/saas-multitenant-solution-architecture/) |
| 8 | **Architect Multitenant Solutions on Azure** ⭐ | The primary multi-tenancy guide — covers tenant definition, isolation strategies, shared vs. dedicated resources. Directly maps to Contoso's approach. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview) |
| 9 | **Tenancy Models** | Spectrum from fully shared to fully isolated. Contoso sits at "Partially Shared" — this page validates that choice and explains trade-offs. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models) |
| 10 | **Architectural Approaches Overview** ⭐ | **Key for resource organization & cost management.** Covers three resource models (shared, pooled, isolated) and their trade-offs. Includes critical guidance on **per-tenant cost allocation via tagging**, usage metering for billing, and how resource organization choices impact security, scalability, and operational complexity. Directly informs how Contoso should structure Resource Groups per tenant and track per-customer costs. Also covers the Deployment Stamps pattern which applies to Contoso's multi-region model. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview) |
| 11 | **Deployment & Configuration** | How to automate tenant onboarding, stamp provisioning, and configuration management. Key for Contoso's "no-scratch" expansion goal. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/deployment-configuration) |
| 12 | **Identity in Multitenant Solutions** | SSO federation, per-tenant identity providers, Entra ID app registrations. Relevant for Contoso's CRM SSO and per-customer access control. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity) |

### Governance & Compliance

| # | Resource | Why It Matters for Contoso | Link |
| - | - | - | - |
| 13 | **Governance & Compliance in Multitenant Solutions** | How to enforce data residency, encryption, tagging, and audit requirements across tenants. Directly relevant for GDPR and SOC 2. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/governance-compliance) |
| 14 | **Multi-Tenant Landing Zone Considerations** | Specific considerations when operating Azure Landing Zones across multiple Entra tenants or organizational boundaries. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/multi-tenant/considerations-recommendations) |
| 15 | **Automate Landing Zones Across Tenants** | How to use IaC and CI/CD to automate LZ deployment across multiple tenants — directly applicable to Contoso's stamp automation goal. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-area/multi-tenant/automation) |
| 16 | **Azure GDPR Guidance** | Official Microsoft guidance on GDPR compliance in Azure — data subject rights, processing records, breach notification. | [learn.microsoft.com](https://learn.microsoft.com/en-us/compliance/regulatory/gdpr) |

### Deployment Patterns

| # | Resource | Why It Matters for Contoso | Link |
| - | - | - | - |
| 17 | **Deployment Stamps Pattern** ⭐ | The core pattern for Contoso's multi-region expansion. Each stamp = identical infrastructure per region, deployed via IaC. Microsoft's recommended approach for SaaS scale-out. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) |
| 18 | **Geode Pattern** | Complementary to Deployment Stamps — focuses on routing requests to the nearest regional deployment for low latency. Applies to Contoso's Front Door + Traffic Manager setup. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/patterns/geodes) |
| 19 | **Multitenant Architectural Considerations** ⭐ | **Use as a design review checklist.** Covers tenant definition ("who is a tenant?"), commercial/pricing model impact on architecture, per-tenant SLA differentiation, noisy neighbor prevention, and growth planning for large tenants. Contoso should use this to formally define: Is a tenant = a customer? Can customers have different SLA tiers? What happens when one customer scales 10x? Essential reading before finalizing the architecture. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/overview) |
| 20 | **Related Resources for Multitenancy** | Additional cloud design patterns (federated identity, sharding, throttling) and anti-patterns to avoid. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/related-resources) |

### Contact Center & AI Specific

| # | Resource | Why It Matters for Contoso | Link |
| - | - | - | - |
| 21 | **Contact Centers with Azure Communication Services** | Reference architecture for building contact centers on Azure — multi-modal (voice, chat, video), bot-to-human escalation, CCaaS models. Relevant for Contoso's contact center domain. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/communication-services/tutorials/contact-center) |
| 22 | **AI Chat Reference Architecture in Landing Zone** | How to deploy Azure OpenAI workloads within a Landing Zone — network isolation, governance, identity. Directly relevant for Contoso's AI Agent Assist feature. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-landing-zone) |
| 23 | **Baseline AI Chat Reference Architecture** | Foundational architecture for conversational AI on Azure — OpenAI, App Service, data layer. The starting point for Contoso's AI agent capabilities. | [learn.microsoft.com](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-microsoft-foundry-chat) |

### Implementation Accelerators (GitHub)

| # | Resource | What It Provides | Link |
| - | - | - | - |
| 24 | **Azure Landing Zones (Enterprise-Scale)** | Ready-to-deploy IaC (Bicep/ARM) for the full Landing Zone — MG hierarchy, policies, connectivity. The official accelerator. | [github.com/Azure/Enterprise-Scale](https://github.com/Azure/Enterprise-Scale) |
| 25 | **Azure Landing Zones Documentation** | Supplementary docs, Bicep/Terraform modules, architecture patterns for ALZ. | [github.com/Azure/Azure-Landing-Zones](https://github.com/Azure/Azure-Landing-Zones) |
| 26 | **ALZ Bicep Modules** | Modular Bicep components for deploying each piece of the Landing Zone independently. | [github.com/Azure/ALZ-Bicep](https://github.com/Azure/ALZ-Bicep) |
| 27 | **ALZ Terraform Modules** | Terraform-native implementation of the Enterprise-Scale Landing Zone. | [github.com/Azure/terraform-azurerm-caf-enterprise-scale](https://github.com/Azure/terraform-azurerm-caf-enterprise-scale) |
| 28 | **Multi-Tenant Capacity Management** | Quota & capacity planning guidance for multi-tenant deployments — addresses Azure resource limits. | [github.com/microsoft/azcapman](https://github.com/microsoft/azcapman) |
| 29 | **Deploying ALZ — Step-by-Step Wiki** | Walkthrough guide for deploying Azure Landing Zones from scratch using the accelerator. | [Enterprise-Scale Wiki](https://github.com/Azure/Enterprise-Scale/wiki/Deploying-ALZ) |

---

## 9. Recommended Reading Order

For Contoso team members new to Azure Landing Zones, we recommend reading the resources in this order:

| Step | Read This | Time | Purpose |
| - | - | - | - |
| 1️⃣ | [What is an Azure Landing Zone?](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/)  | Understand core concepts |
| 2️⃣ | [Landing Zone Design Areas](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/design-areas)  | Learn the 8 pillars to address |
| 3️⃣ | [SaaS and Multitenant Solution Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/saas-multitenant-solution-architecture/) ⭐  | Understand SaaS vs. multitenancy, B2B patterns, and the full decision framework |
| 4️⃣ | [ISV Considerations](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/isv-landing-zone)  | Understand ISV/SaaS-specific guidance |
| 5️⃣ | [Multitenant Architectural Considerations](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/overview) ⭐  | Define tenants, SLA tiers, growth planning, and noisy neighbor strategy |
| 6️⃣ | [Architectural Approaches Overview](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview) ⭐  | Decide resource organization model and per-tenant cost management strategy |
| 7️⃣ | [Architect Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview)  | Deep dive into multi-tenancy patterns |
| 8️⃣ | [Tenancy Models](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models)  | Validate Contoso's "Partially Shared" approach |
| 9️⃣ | [Deployment Stamps Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp)  | Understand the multi-region scale pattern |
| 🔟 | [Multinational Landing Zone](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/landing-zone-multinational)  | Plan for cross-border expansion |
| 1️⃣1️⃣ | [GDPR Guidance](https://learn.microsoft.com/en-us/compliance/regulatory/gdpr) | Compliance alignment |

> 💡 **Note:** Steps 3, 5, and 6 (marked ⭐) were specifically highlighted by multiple Microsoft CSAs & Experts as essential reading for Contoso's use case.

---
