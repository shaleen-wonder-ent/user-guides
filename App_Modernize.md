# Multi-Tenant SaaS Landing Zone Architecture on Azure

---

## Table of Contents
1. [Introduction](#introduction)
2. [Multi-Tenancy Fundamentals](#multi-tenancy)
3. [Architecture Option 1 – Azure Native (Microsoft Fabric & PaaS)](#option1)
4. [Architecture Option 2 – Cloud-Agnostic (Containers & Kubernetes)](#option2)
5. [Comparative Trade-off Analysis](#tradeoffs)
6. [Multi-Tenant Landing Zone Design](#landing-zone)
7. [Identity & Access Management](#identity)
8. [AI-Ready Data Platform](#ai-fabric)
9. [Reference Architectures & Resources](#references)

---

<a name="introduction"></a>
## 1. Introduction

This document provides a reference architecture for building a **multi-tenant SaaS analytics platform** on Microsoft Azure. It covers:

- Multi-tenancy isolation patterns across compute, storage, and networking
- Two architecture options: Azure Native (Fabric + PaaS) and Cloud-Agnostic (Kubernetes)
- Landing zone design with hub-and-spoke networking
- Identity federation and SSO patterns
- AI-ready data platform design
- Microsoft reference architectures and resources

The guidance is vendor-neutral in its framing and applicable to any ISV or enterprise building a multi-tenant analytics SaaS product.

[↑ Back to top](#)

---

<a name="multi-tenancy"></a>
## 2. Multi-Tenancy Fundamentals

### What is Multi-Tenancy?

Multi-tenancy is a software architecture pattern where a single instance of software serves multiple customers (tenants). Each tenant's data and configuration are isolated, but the underlying infrastructure may be shared to optimise cost and operations.

### Isolation Models

| Model | Isolation Level | Cost | Complexity | Best For |
|---|---|---|---|---|
| **Shared everything** (RLS/RBAC) | Logical | Lowest | Low | Small tenants, low compliance |
| **Separate database / workspace per tenant** | Data-level | Moderate | Moderate | Most SaaS platforms |
| **Separate compute per tenant** | Compute + Data | Higher | Higher | Noisy-neighbor concerns |
| **Separate subscription per tenant** | Full | Highest | Highest | Regulated / enterprise clients |

### Key Design Considerations

1. **Tenant onboarding & offboarding** – How are new tenants provisioned? Can it be automated via IaC?
2. **Data isolation** – Is data physically separated (separate storage) or logically separated (shared storage + RBAC)?
3. **Compute isolation** – Do tenants share compute pools, or do high-tier tenants get dedicated capacity?
4. **Configuration per tenant** – Can business rules, feature flags, and UI branding vary per tenant?
5. **Cost allocation** – Can you meter and charge back per-tenant resource consumption?
6. **Compliance** – Do different tenants have different regulatory requirements (SOC2, HIPAA, data residency)?

### Tenant Tiering Pattern

A common pattern is to define tenant tiers that map to isolation levels:

| Tier | Profile | Isolation Model |
|---|---|---|
| **Standard** | SMB / smaller tenants | Shared infrastructure + logical isolation |
| **Professional** | Mid-market tenants | Dedicated compute capacity + data isolation |
| **Enterprise** | Large / regulated tenants | Dedicated subscription + customer-managed keys + private endpoints |

**Reference:** [Tenancy Models for Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models)

[↑ Back to top](#)

---

<a name="option1"></a>
## 3. Architecture Option 1 – Azure Native (Microsoft Fabric & PaaS)

### Overview

This option leverages Microsoft Fabric as the core analytics platform with Azure PaaS services for identity, API management, secrets, and governance. Multi-tenancy is achieved through Fabric's native workspace isolation model.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     AZURE NATIVE (Per-Tenant Isolation)         │
│                                                                 │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐      │
│  │  Tenant A     │   │  Tenant B     │   │  Tenant C     │      │
│  │  Workspace    │   │  Workspace    │   │  Workspace    │      │
│  │  (Fabric)     │   │  (Fabric)     │   │  (Fabric)     │      │
│  └──────┬────────┘   └──────┬────────┘   └──────┬────────┘      │
│         │                   │                   │               │
│  ┌──────▼───────────────────▼───────────────────▼────────┐      │
│  │              Microsoft Fabric Capacity (F SKU)        │      │
│  │         (Shared Compute, Isolated Workspaces)         │      │
│  └──────────────────────────┬────────────────────────────┘      │
│                             │                                   │
│  ┌──────────────────────────▼────────────────────────────┐      │
│  │                    OneLake (Unified Storage)          │      │
│  │   Tenant A Data │ Tenant B Data │ Tenant C Data       │      │
│  │  (Folder-level isolation + RBAC + sensitivity labels) │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  Supporting PaaS:                                               │
│  Azure API Management │ Microsoft Entra ID │ Azure Key Vault    │
│  Azure Monitor / Purview │ Azure Data Factory │ Event Hub       │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Role | Multi-Tenancy Model |
|---|---|---|
| **Microsoft Fabric Workspaces** | Isolated analytics environment per tenant | One workspace per tenant |
| **OneLake** | Unified storage layer | Folder-level isolation + RBAC |
| **Fabric Capacity (F SKU)** | Shared compute pool | Capacity-level or dedicated per tier |
| **Power BI Embedded / Reports** | Tenant-facing analytics UI | Row-level security (RLS) per tenant |
| **Azure API Management (APIM)** | Unified API gateway | Subscription-per-tenant, rate limiting |
| **Microsoft Entra ID** | Identity & SSO | B2B federation with external IdPs |
| **Azure Key Vault** | Secrets & encryption keys | Customer-managed keys per tenant |
| **Microsoft Purview** | Data governance & lineage | Cross-tenant data catalog |
| **Azure Data Factory** | Data pipeline orchestration | Parameterised pipelines per tenant |
| **Azure Monitor + Log Analytics** | Observability | Workspace-per-tenant or filtered |

### Tenancy Models in Microsoft Fabric

Microsoft supports three main tenancy models in Fabric:

1. **Separate Fabric Workspace per tenant** *(Recommended)*
   - Strong data isolation boundary
   - Independent RBAC, sensitivity labels, and workspace identities
   - Easiest to onboard/offboard tenants
   - Cost: all tenants share one Fabric capacity (F SKU)

2. **Separate Fabric Capacity per tenant** *(Premium / regulated tenants)*
   - Full compute and billing isolation
   - Required for SLA guarantees or cost chargebacks
   - Higher cost but maximum isolation

3. **Shared workspace with RLS** *(Not recommended for most SaaS)*
   - Row-level security in semantic models to filter by tenant
   - Lower isolation, complex to maintain, security risk

### Pros
- Fully managed platform — no VM patching, no container orchestration overhead
- Native integration: Fabric + Power BI + Azure OpenAI + Purview + Entra ID = one ecosystem
- OneLake eliminates data duplication across layers (Lakehouse, Warehouse, Real-Time)
- Built-in compliance (ISO 27001, SOC2, GDPR, HIPAA eligible)
- Fastest time-to-value for analytics-heavy workloads

### Cons
- Azure-specific — not natively portable to other clouds (Azure Arc can bridge this)
- Licensing cost model (F SKUs) can be opaque for initial estimation
- Customisation is constrained to what Fabric exposes in its API surface

[↑ Back to top](#)

---

<a name="option2"></a>
## 4. Architecture Option 2 – Cloud-Agnostic (Containers & Kubernetes)

### Overview

This option uses Kubernetes as the compute platform with open-source analytics tools. It runs on any cloud (AKS, EKS, GKE) or on-premises. Multi-tenancy is achieved through Kubernetes namespace isolation with network policies.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│            CLOUD-AGNOSTIC (Kubernetes / Container-Based)        │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │                  Control Plane / Management            │     │
│  │   Tenant Registry │ Config Service │ Identity Proxy    │     │
│  └──────────────────────────┬─────────────────────────────┘     │
│                             │                                   │
│     ┌───────────────────────┼───────────────────────┐           │
│     ▼                       ▼                       ▼           │
│  ┌──────────┐         ┌──────────┐          ┌──────────┐        │
│  │Namespace │         │Namespace │          │Namespace │        │
│  │ Tenant A │         │ Tenant B │          │ Tenant C │        │
│  │(K8s NS)  │         │(K8s NS)  │          │(K8s NS)  │        │
│  └──────────┘         └──────────┘          └──────────┘        │
│                                                                 │
│  ┌────────────────────────────────────────────────────────┐     │
│  │       Kubernetes Cluster (AKS / GKE / EKS / On-Prem)   │     │
│  │  Network Policies │ Pod Security │ RBAC │ Secrets Mgmt │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  Storage Layer:                                                 │
│  Object Store (Azure Blob / S3 / GCS) – abstracted via CSI      │
│  Metadata DB: PostgreSQL or managed (per-tenant schemas/DBs)    │
│                                                                 │
│  Supporting Services (deployable anywhere):                     │
│  Ingress (NGINX/Traefik) │ Keycloak/OIDC │ Prometheus/Grafana   │
│  Apache Kafka │ dbt / Spark │ OpenMetadata                      │
└─────────────────────────────────────────────────────────────────┘
```

### Tenant Isolation Models in Kubernetes

| Model | Isolation Level | Recommended For |
|---|---|---|
| **Namespace-per-tenant** | Logical (shared cluster) | Most tenants; moderate isolation |
| **Node pool per tenant** | Physical compute isolation | High compliance / noisy-neighbor concerns |
| **Cluster per tenant** | Full isolation | Regulated, large enterprise tenants |

### Pros
- True cloud agnosticism — runs on AKS, EKS, GKE, or on-prem with minimal rework
- Full control over runtime, dependencies, and upgrade cycles
- Tenant onboarding via Helm charts / GitOps (ArgoCD / Flux) = repeatable and auditable
- Can incorporate any open-source stack (Apache Spark, dbt, Trino, OpenMetadata)

### Cons
- Higher operational complexity — the platform team owns the infrastructure layer
- Requires dedicated DevOps/SRE capability to manage at scale
- Analytics capabilities must be built or integrated from scratch
- Longer time-to-value; more bespoke engineering effort
- Security hardening (CIS benchmarks, pod policies, network policies) is entirely on the team

[↑ Back to top](#)

---

<a name="tradeoffs"></a>
## 5. Comparative Trade-off Analysis

### Side-by-Side Comparison

| Dimension | Option 1: Azure Native (Fabric) | Option 2: Cloud-Agnostic (K8s) | Advantage |
|---|---|---|---|
| **Cloud Lock-in** | Azure-first (Azure Arc for non-Azure) | Deploy anywhere | Option 2 |
| **Multi-Tenancy** | Native workspace isolation + OneLake RBAC | Namespace / node pool / cluster isolation | **Option 1** |
| **Time-to-Value** | Fast — Fabric has analytics built-in | Slower — more custom build required | **Option 1** |
| **Operational Complexity** | Low — Microsoft manages infrastructure | High — team manages K8s and all layers | **Option 1** |
| **Configurability per Tenant** | Moderate — Fabric APIs + config DB | High — Helm values, config maps, feature flags | Option 2 |
| **Analytics Depth** | Very High — Lakehouse, Notebooks, Power BI, Copilot | Needs integration (Spark, Trino, Superset, etc.) | **Option 1** |
| **AI/ML Integration** | Native — Azure OpenAI + Fabric Copilot | Bring-your-own (Azure AI, Hugging Face, etc.) | **Option 1** |
| **Data Residency** | Supported via Fabric Multi-Geo | Supported via cloud region selection | Tie |
| **SSO / Identity** | Entra ID + B2B federation | Keycloak / OIDC proxy | **Option 1** |
| **Cost Model** | Predictable — F SKU capacity pricing | Variable — compute + ops labour cost | **Option 1** |
| **Compliance** | Microsoft holds certs (SOC2, ISO 27001, HIPAA) | Must certify per cloud / per deployment | **Option 1** |
| **Disaster Recovery** | Built-in Fabric reliability + Azure SLAs | Requires bespoke DR strategy | **Option 1** |
| **Vendor Risk** | Platform dependency on Microsoft | Open-source stack, portable | Tie |

### Recommended Decision Framework

```
Start with Option 1 (Azure Native / Fabric) as the default.

Can the tenant use Azure?
    YES ──► Option 1 (Fabric) — full platform value, AI, analytics, governance
    NO  ──► Can Azure Arc bridge the gap?
              YES ──► Option 1 + Azure Arc (extend Azure management to any infra)
              NO  ──► Option 2 (K8s) as a constrained fallback for that tenant only
```

### Hybrid Path (Recommended Approach)

The pragmatic approach is **Azure Native first, with Azure Arc for edge cases**:
- **Microsoft Fabric** is the core analytics and data platform for Azure-compatible tenants — delivering multi-tenancy, AI, Power BI, and governance out of the box
- **Azure Arc** extends Azure's management plane to non-Azure environments for tenants that cannot use Azure directly
- **Containerised application services** (on AKS) handle compute-heavy or cloud-portable workloads, complementing Fabric's analytics layer
- Reference: [Azure Arc for Kubernetes](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview)

### Key Trade-off Dimensions Explained

#### Scalability
- **Option 1:** Azure PaaS auto-scaling with no manual intervention. Fabric provides unified compute across ETL, warehousing, reporting, and AI workloads.
- **Option 2:** Kubernetes scaling is portable but must be configured manually (HPA, Cluster Autoscaler). Requires dedicated DevOps expertise.

#### Cost
- **Option 1:** Shifts spend from people (ops engineers) to platform (Azure fees). Platform spend is predictable and scales sub-linearly.
- **Option 2:** Lower platform licensing but higher labour cost. Operational overhead scales linearly with the tenant base.

#### Multi-Tenancy
- **Option 1:** Platform-level primitives — separate workspaces, OneLake folder isolation, sensitivity labels, workspace-scoped RBAC, Fabric REST APIs for automation.
- **Option 2:** Infrastructure-level isolation only — namespaces, network policies. Application-level multi-tenancy must be custom-built.

#### Security & Compliance
- **Option 1:** Pre-certified security foundation — SOC 2, ISO 27001, HIPAA, with built-in encryption, Microsoft Defender, and Purview governance.
- **Option 2:** Maximum flexibility but all security controls must be implemented, tested, and certified independently.

#### Performance & Reliability
- **Option 1:** Enterprise-grade SLAs, automated failover, built-in HA. Unified analytics engine reduces data movement latency.
- **Option 2:** Higher theoretical performance ceiling with specialised tools, but requires significant engineering effort to realise.

#### Cloud Portability
- **Option 1:** Azure-specific, but most SaaS tenants consume the service — not infrastructure. Azure Arc bridges the gap for non-Azure tenants.
- **Option 2:** Fully portable but at the cost of reduced platform capability across all tenants.

[↑ Back to top](#)

---

<a name="landing-zone"></a>
## 6. Multi-Tenant Landing Zone Design

### What is a Landing Zone?

A Landing Zone is the pre-configured, governed Azure environment into which workloads are deployed. For a multi-tenant SaaS platform, the landing zone defines:
- Subscription structure
- Networking topology
- Identity & access management
- Security baselines
- Cost management boundaries

### Recommended Landing Zone Structure

```
Azure Management Group
└── SaaS Platform Management Group
    ├── Platform Subscription (shared services)
    │   ├── Hub VNet (Firewall, Bastion, Private DNS)
    │   ├── Azure API Management (Premium tier)
    │   ├── Azure Key Vault (Platform secrets)
    │   ├── Azure Monitor / Log Analytics Workspace
    │   ├── Microsoft Purview Account
    │   └── CI/CD (Azure DevOps / GitHub Actions)
    │
    ├── Analytics Subscription
    │   ├── Microsoft Fabric Capacity (F64+)
    │   ├── OneLake Storage
    │   └── Per-tenant Fabric Workspaces (logical isolation)
    │
    └── Per-Tenant Spoke Subscriptions (for high-isolation tenants)
        ├── Tenant Spoke VNet (peered to Hub)
        ├── Tenant Azure SQL / Cosmos DB
        └── Tenant Key Vault (CMK)
```

### Hub-and-Spoke Networking

```
┌───────────────────────────────────────────┐
│              HUB VNet                     │
│  Azure Firewall │ Private DNS │ Bastion   │
│  APIM (internal mode) │ Entra Domain Svc  │
└──────────────────┬────────────────────────┘
         ┌─────────┼─────────┐
         ▼         ▼         ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ Spoke A │ │ Spoke B │ │Platform │
    │ Tenant A│ │ Tenant B│ │Shared   │
    │  VNet   │ │  VNet   │ │Services │
    └─────────┘ └─────────┘ └─────────┘
```

### Landing Zone Implementation Steps

**Step 1 – Tenant Classification**

Define tiers that determine isolation level:

| Tier | Profile | Landing Zone Model |
|---|---|---|
| **Standard** | SMB / smaller tenants | Shared Fabric Workspace + shared compute |
| **Professional** | Mid-market tenants | Dedicated Fabric Capacity + spoke VNet |
| **Enterprise** | Large / regulated tenants | Dedicated subscription + CMK + private endpoints |

**Step 2 – Subscription Vending Automation**

For Enterprise-tier tenants: automate subscription provisioning using:
- Azure Bicep templates or Terraform
- CI/CD pipelines triggered on tenant onboarding
- Policy assignments via Azure Management Groups to enforce security baseline
- Reference: [Subscription Vending](https://learn.microsoft.com/en-us/azure/architecture/landing-zones/subscription-vending)

**Step 3 – Network Topology**

- All tenant workloads connect via **Private Endpoints** to avoid public internet exposure
- APIM in internal mode acts as the entry point, forwarding to per-tenant backends
- Azure Firewall in the hub enforces egress rules and inspects cross-tenant traffic

**Step 4 – Identity Plane**

- One **Microsoft Entra ID tenant** for the platform
- Tenant users onboarded as **B2B guest users** (or via External ID if consumer-facing)
- Per-tenant **App Registrations** for OIDC flows
- External IdP federated via SAML/OIDC to Entra ID

**Step 5 – Governance & Compliance**

- Azure Policy enforcements: require private endpoints, deny public IP, enforce tagging
- Microsoft Defender for Cloud enabled on all subscriptions
- Microsoft Purview as the cross-tenant data catalog
- Fabric sensitivity labels tied to Purview classification

[↑ Back to top](#)

---

<a name="identity"></a>
## 7. Identity & Access Management

### Federated SSO Pattern

For SaaS platforms where tenants bring their own identity provider (e.g., any SAML/OIDC-compatible IdP):

```
Tenant User
    │
    ▼
External IdP (corporate) ──SAML 2.0 / OIDC──► Microsoft Entra ID (B2B Federation)
                                                        │
                                                 Entra ID issues token
                                                        │
                                          ┌─────────────┼───────────────┐
                                          ▼             ▼               ▼
                                    Fabric API    App Service      API Management
                                   (workspace     (backend API)   (subscription key
                                    scoped)                        per tenant)
```

- **Microsoft Entra External ID** supports federated SAML/OIDC from external IdPs
- Users authenticate via their corporate IdP; Entra issues claims-enriched tokens with tenant context
- Downstream services use the token's `tid` (tenant ID) claim to enforce data access rules
- For the K8s option, replace Entra with **Keycloak** fronted by the same OIDC flow

### RBAC Design for Multi-Tenancy

| Role | Scope | Example Permissions |
|---|---|---|
| **Platform Admin** | Entire platform | Manage all tenants, billing, infrastructure |
| **Tenant Admin** | Single tenant | Manage users, configuration, data sources within their tenant |
| **Tenant User** | Single tenant | View dashboards, run queries, access data per role |
| **Data Engineer** | Single tenant | Manage pipelines, transformations, data quality |

### Reference Links – Identity

| Resource | Link |
|---|---|
| Microsoft Entra External ID (B2B federation) | [Link](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b) |
| Federation with external IdPs via SAML | [Link](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-saml-single-sign-on) |
| Claims-based identity on Azure | [Link](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens) |
| Conditional Access in Microsoft Fabric | [Link](https://learn.microsoft.com/en-us/fabric/security/security-conditional-access) |
| Identity approaches for multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity) |

[↑ Back to top](#)

---

<a name="ai-fabric"></a>
## 8. AI-Ready Data Platform

### Architecture: AI on Microsoft Fabric

```
                    ┌──────────────────────────────────────┐
                    │        AI-Ready SaaS Platform        │
                    │  "Ask anything about your data"      │
                    └──────────────┬───────────────────────┘
                                   │
           ┌───────────────────────┼───────────────────────────┐
           ▼                       ▼                           ▼
   ┌──────────────┐       ┌──────────────────┐      ┌──────────────────┐
   │   Fabric     │       │  Azure OpenAI    │      │  Real-Time       │
   │   Copilot    │       │  (GPT-4o)        │      │  Intelligence    │
   │ (natural lang│       │  RAG on tenant   │      │  (Event streams  │
   │  queries on  │       │  data via AI     │      │  + KQL queries)  │
   │  Lakehouse)  │       │  Search index)   │      │                  │
   └──────┬───────┘       └────────┬─────────┘      └────────┬─────────┘
          │                       │                          │
          └───────────────────────┼──────────────────────────┘
                                  │
          ┌───────────────────────▼──────────────────────────┐
          │           Microsoft Fabric                       │
          │  ┌────────────────────────────────────────────┐  │
          │  │    OneLake (Delta/Parquet, unified store)  │  │
          │  │  Raw Data │ Cleansed │ Curated │ Serving   │  │
          │  └────────────────────────────────────────────┘  │
          │  Lakehouse  │ Warehouse │ Notebooks │ Pipelines  │
          │  Eventstream│ Real-Time Hub │ Data Activator     │
          └──────────────────────────────────────────────────┘
                                  │
          ┌───────────────────────▼──────────────────────────┐
          │           Source Systems (per tenant)            │
          │  ERP │ CRM │ Files │ APIs │ Databases │ Streams  │
          └──────────────────────────────────────────────────┘
```

### Three AI Pillars

| Pillar | Technology | What It Does | Multi-Tenant Safety |
|---|---|---|---|
| **Fabric Copilot** | Built-in Fabric AI | Natural language queries on structured Lakehouse data (Power BI, Notebooks, SQL) | Copilot only accesses data within the current workspace — tenant-isolated by design |
| **Azure OpenAI + RAG** | Azure OpenAI + Azure AI Search | Retrieval-Augmented Generation — query both structured and unstructured data (documents, contracts, reports) | Per-tenant AI Search index or security-trimmed with tenant ID |
| **Real-Time Intelligence** | Fabric Eventstream + KQL Database + Data Activator | Continuous monitoring of streaming data with automated alerts when thresholds are breached | Eventstreams and KQL databases are workspace-scoped |

### How the Three Pillars Work Together

```
┌────────────────────────────────────────────────────────────────────┐
│                        User Interaction                            │
│                                                                    │
│   "What happened?"          "Why?" / "What if?"      "Alert me"    │
│   ┌─────────────┐          ┌─────────────────┐    ┌──────────────┐ │
│   │   Fabric    │          │  Azure OpenAI   │    │  Real-Time   │ │
│   │   Copilot   │          │  + RAG          │    │  Intelligence│ │
│   │ Structured  │          │ Structured +    │    │  Streaming   │ │
│   │ data Q&A    │          │ Unstructured    │    │  data +      │ │
│   │ (tables,    │          │ data Q&A        │    │  automated   │ │
│   │ reports)    │          │ (docs + tables) │    │  triggers    │ │
│   └──────┬──────┘          └────────┬────────┘    └──────┬───────┘ │
│          └──────────────────────────┼────────────────────┘         │
│                                     │                              │
│                          ┌──────────▼───────────┐                  │
│                          │     OneLake          │                  │
│                          │  (single source of   │                  │
│                          │   truth for all AI)  │                  │
│                          └──────────────────────┘                  │
└────────────────────────────────────────────────────────────────────┘
```

### Key Design Principles for AI in Multi-Tenant SaaS

1. **No data silos** — OneLake means every AI model operates on the same curated, governed data
2. **Governance-first AI** — Purview tracks data access; sensitivity labels prevent AI from surfacing confidential data to the wrong tenant
3. **Incremental AI adoption** — Start with Power BI Copilot (low risk), progress to RAG, then predictive analytics
4. **Native Azure OpenAI integration** — Fabric notebooks call Azure OpenAI via managed identity (no keys in code)
5. **Multi-tenant AI safety** — AI responses are workspace-scoped; Tenant A's Copilot cannot access Tenant B's data

[↑ Back to top](#)

---

<a name="references"></a>
## 9. Reference Architectures & Resources

### Core Multi-Tenancy References

| Resource | Link |
|---|---|
| Architect multitenant solutions on Azure (full series) | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview) |
| Tenancy models for multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models) |
| SaaS & multitenant solution architecture | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/saas-multitenant-solution-architecture/) |
| Multitenant architecture checklist | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/checklist) |
| Multitenant SaaS on Azure SQL – design patterns | [Link](https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns) |
| Noisy Neighbor antipattern | [Link](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/noisy-neighbor/noisy-neighbor) |
| Measure consumption in multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/measure-consumption) |
| Deployment Stamp pattern | [Link](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) |
| Azure SaaS Dev Kit | [Link](https://azure.github.io/azure-saas/) |

### Microsoft Fabric References

| Resource | Link |
|---|---|
| Microsoft Fabric overview | [Link](https://learn.microsoft.com/en-us/fabric/fundamentals/microsoft-fabric-overview) |
| Fabric Workspaces & Roles | [Link](https://learn.microsoft.com/en-us/fabric/fundamentals/workspaces) |
| Fabric Security overview | [Link](https://learn.microsoft.com/en-us/fabric/security/security-overview) |
| OneLake Security | [Link](https://learn.microsoft.com/en-us/fabric/onelake/security/fabric-onelake-security) |
| Fabric Multi-Geo (Data Residency) | [Link](https://learn.microsoft.com/en-us/fabric/admin/service-admin-premium-multi-geo) |
| Row-Level Security in Fabric | [Link](https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security) |
| Fabric Permission Model | [Link](https://learn.microsoft.com/en-us/fabric/security/permission-model) |
| Fabric Licenses & SKUs | [Link](https://learn.microsoft.com/en-us/fabric/enterprise/licenses) |
| Fabric Workspace Identity | [Link](https://learn.microsoft.com/en-us/fabric/security/workspace-identity) |
| Power BI Embedded multi-tenancy | [Link](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy) |
| Power BI Embedded for ISVs | [Link](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embedded-analytics-power-bi) |

### Kubernetes & Cloud-Agnostic References

| Resource | Link |
|---|---|
| AKS multi-tenancy best practices | [Link](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation) |
| AKS cluster isolation (logical vs physical) | [Link](https://learn.microsoft.com/en-us/azure/aks/cluster-multi-tenancy) |
| AKS baseline architecture | [Link](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks) |
| Multi-region AKS | [Link](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-multi-region/aks-multi-cluster) |
| Azure Container Apps | [Link](https://learn.microsoft.com/en-us/azure/container-apps/overview) |
| Azure Arc for Kubernetes | [Link](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview) |
| AKS Workload Identity | [Link](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) |

### Landing Zone & Governance References

| Resource | Link |
|---|---|
| Azure Landing Zone overview | [Link](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/) |
| CAF Enterprise-Scale Landing Zone | [Link](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/enterprise-scale/architecture) |
| Subscription vending | [Link](https://learn.microsoft.com/en-us/azure/architecture/landing-zones/subscription-vending) |
| Hub-Spoke Network Topology | [Link](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke) |
| Azure Policy | [Link](https://learn.microsoft.com/en-us/azure/governance/policy/overview) |
| Microsoft Purview Governance | [Link](https://learn.microsoft.com/en-us/purview/purview) |
| Azure Well-Architected Framework | [Link](https://learn.microsoft.com/en-us/azure/well-architected/) |

### AI & Analytics References

| Resource | Link |
|---|---|
| Fabric AI / Copilot overview | [Link](https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview) |
| Copilot in Power BI | [Link](https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-introduction) |
| Azure OpenAI on your data (RAG) | [Link](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/use-your-data) |
| Implement RAG with Azure OpenAI | [Link](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview) |
| Azure AI Search | [Link](https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search) |
| Fabric Data Activator | [Link](https://learn.microsoft.com/en-us/fabric/data-activator/data-activator-introduction) |
| Fabric Real-Time Intelligence | [Link](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/overview) |
| Fabric Eventstream | [Link](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/event-streams/overview) |
| Fabric Lakehouse overview | [Link](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview) |
| Azure API Management | [Link](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts) |

### Multitenant Approach Guides (by Layer)

| Layer | Resource | Link |
|---|---|---|
| Compute | Compute approaches for multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/compute) |
| Storage & Data | Storage and data approaches | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data) |
| Networking | Networking approaches | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/networking) |
| Identity | Identity approaches | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity) |
| Deployment & Config | Deployment and configuration | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/deployment-configuration) |
| Cost Management | Cost management and allocation | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/cost-management-allocation) |
| Governance | Governance and compliance | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/governance-compliance) |

[↑ Back to top](#)
