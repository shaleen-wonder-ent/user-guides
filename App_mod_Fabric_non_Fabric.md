# Multi-Tenant SaaS Landing Zone Architecture on Azure

---

## Table of Contents
1. [Introduction](#introduction)
2. [Multi-Tenancy Fundamentals](#multi-tenancy)
3. [Option 1 – Azure Native with Microsoft Fabric](#option1)
   - [Architecture Overview](#option1-arch)
   - [Complete Landing Zone](#option1-lz)
   - [Identity & Access](#option1-identity)
   - [AI & Analytics](#option1-ai)
4. [Option 2 – Azure Native without Fabric](#option2)
   - [Architecture Overview](#option2-arch)
   - [Complete Landing Zone](#option2-lz)
   - [Identity & Access](#option2-identity)
   - [AI & Analytics](#option2-ai)
5. [Side-by-Side Comparison](#comparison)
6. [Reference Architectures & Resources](#references)

---

<a name="introduction"></a>
## 1. Introduction

This document provides a reference architecture for building a **multi-tenant SaaS analytics platform** on Microsoft Azure. It presents two **Azure-native** paths:

| | Option 1 | Option 2 |
|---|---|---|
| **Platform** | Microsoft Fabric + Azure PaaS | Azure PaaS (Synapse, AKS, ADLS, SQL) — no Fabric |
| **Cloud** | Azure-native | Azure-native |
| **Multi-tenancy** | Fabric workspace isolation | Azure subscription / resource-level isolation |
| **Analytics** | OneLake + Lakehouse + Power BI Embedded | ADLS Gen2 + Synapse + Power BI Embedded |
| **AI** | Fabric Copilot + Azure OpenAI | Azure OpenAI + Azure AI Search |

Both options are fully Azure-native. The choice is whether to adopt Microsoft Fabric as the unified analytics layer, or build the equivalent capability from individual Azure PaaS services.

[↑ Back to top](#)

---

<a name="multi-tenancy"></a>
## 2. Multi-Tenancy Fundamentals

### What is Multi-Tenancy?

Multi-tenancy is a software architecture pattern where a single platform serves multiple customers (tenants). Each tenant's data and configuration are isolated, but the underlying infrastructure may be shared to optimise cost and operations.

### Isolation Models

| Model | Isolation Level | Cost | Complexity | Best For |
|---|---|---|---|---|
| **Shared everything** (RLS/RBAC) | Logical | Lowest | Low | Small tenants, low compliance |
| **Separate database / workspace per tenant** | Data-level | Moderate | Moderate | Most SaaS platforms |
| **Separate compute per tenant** | Compute + Data | Higher | Higher | Noisy-neighbor concerns |
| **Separate subscription per tenant** | Full | Highest | Highest | Regulated / enterprise tenants |

### Tenant Tiering Pattern

| Tier | Profile | Isolation Model |
|---|---|---|
| **Standard** | SMB / smaller tenants | Shared infrastructure + logical isolation (RBAC / RLS) |
| **Professional** | Mid-market tenants | Dedicated compute capacity + data-level isolation |
| **Enterprise** | Large / regulated tenants | Dedicated subscription + customer-managed keys + private endpoints |

### Key Design Considerations

1. **Tenant onboarding & offboarding** — automated via IaC (Bicep / Terraform)
2. **Data isolation** — physical separation vs. logical separation (RBAC + RLS)
3. **Compute isolation** — shared capacity pools vs. dedicated per tenant
4. **Configuration per tenant** — feature flags, business rules, UI branding
5. **Cost allocation** — metering and charge-back per tenant
6. **Compliance** — SOC2, HIPAA, ISO 27001, data residency per tenant

**Reference:** [Tenancy Models for Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models)

[↑ Back to top](#)

---

<a name="option1"></a>
## 3. Option 1 – Azure Native with Microsoft Fabric

<a name="option1-arch"></a>
### 3.1 Architecture Overview

This option uses **Microsoft Fabric** as the unified analytics and data platform. Fabric consolidates data engineering, data warehousing, real-time analytics, data science, and Power BI into a single SaaS experience backed by **OneLake** (unified storage).

Multi-tenancy is achieved through **Fabric workspace isolation** — each tenant gets a dedicated Fabric workspace with its own data, pipelines, reports, and RBAC.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    OPTION 1: AZURE NATIVE WITH FABRIC                │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                  Presentation Layer                          │    │
│  │  Power BI Embedded │ Custom SaaS Portal (App Service)        │    │
│  │  Admin UI (Static Web Apps) │ Azure API Management           │    │
│  └────────────────────────────────┬─────────────────────────────┘    │
│                                   │                                  │
│  ┌────────────────────────────────▼─────────────────────────────┐    │
│  │                  Application / API Layer                     │    │
│  │  Azure App Service / Azure Functions                         │    │
│  │  Tenant Config DB (Azure SQL / Cosmos DB)                    │    │
│  │  Azure App Configuration (feature flags per tenant)          │    │
│  └────────────────────────────────┬─────────────────────────────┘    │
│                                   │                                  │
│  ┌────────────────────────────────▼─────────────────────────────┐    │
│  │                  Microsoft Fabric (Analytics Platform)        │    │
│  │                                                              │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │    │
│  │  │ Workspace A │  │ Workspace B │  │ Workspace C │          │    │
│  │  │ (Tenant A)  │  │ (Tenant B)  │  │ (Tenant C)  │          │    │
│  │  │ Lakehouse   │  │ Lakehouse   │  │ Lakehouse   │          │    │
│  │  │ Warehouse   │  │ Warehouse   │  │ Warehouse   │          │    │
│  │  │ Notebooks   │  │ Notebooks   │  │ Notebooks   │          │    │
│  │  │ Pipelines   │  │ Pipelines   │  │ Pipelines   │          │    │
│  │  │ Reports     │  │ Reports     │  │ Reports     │          │    │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘          │    │
│  │         │                │                │                  │    │
│  │  ┌──────▼────────────────▼────────────────▼──────────┐      │    │
│  │  │        Fabric Capacity (F SKU – shared compute)    │      │    │
│  │  └──────────────────────┬─────────────────────────────┘      │    │
│  │                         │                                    │    │
│  │  ┌──────────────────────▼─────────────────────────────┐      │    │
│  │  │                   OneLake                          │      │    │
│  │  │  Tenant A Folder │ Tenant B Folder │ Tenant C      │      │    │
│  │  │  (RBAC + sensitivity labels + folder isolation)    │      │    │
│  │  └────────────────────────────────────────────────────┘      │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                  AI Layer                                    │    │
│  │  Fabric Copilot │ Azure OpenAI │ Azure AI Search (RAG)       │    │
│  │  Fabric Real-Time Intelligence │ Data Activator              │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Cross-Cutting Services:                                             │
│  Microsoft Entra ID │ Azure Key Vault │ Microsoft Purview            │
│  Azure Monitor / Log Analytics │ Azure Policy │ Microsoft Defender   │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Role | Multi-Tenancy Model |
|---|---|---|
| **Fabric Workspaces** | Isolated analytics environment per tenant | One workspace per tenant |
| **OneLake** | Unified storage (Delta/Parquet) | Folder-level isolation + RBAC |
| **Fabric Capacity (F SKU)** | Shared compute pool | Shared across tenants, or dedicated per tier |
| **Fabric Lakehouse** | Bronze → Silver → Gold medallion data layers | Per-workspace (per-tenant) |
| **Fabric SQL Warehouse** | MPP analytical queries | Per-workspace (per-tenant) |
| **Fabric Pipelines** | Data ingestion & orchestration | Parameterised per tenant |
| **Power BI Embedded** | Tenant-facing analytics UI | Row-level security + workspace isolation |
| **Azure API Management** | API gateway | Subscription-per-tenant, rate limiting |
| **Microsoft Entra ID** | Identity & SSO | B2B federation with tenant IdPs |
| **Azure Key Vault** | Secrets & CMK | Customer-managed keys per tenant |
| **Microsoft Purview** | Data governance & lineage | Cross-tenant catalog |
| **Azure App Configuration** | Feature flags & tenant config | Per-tenant feature filters |

### Tenancy Models in Fabric

| Model | Isolation | Cost | When to Use |
|---|---|---|---|
| **Workspace per tenant** *(Recommended)* | Strong — separate data, RBAC, sensitivity labels | Moderate — shared F SKU | Default for most tenants |
| **Capacity per tenant** | Full compute + billing isolation | High — dedicated F SKU | SLA guarantees, regulated tenants |
| **Shared workspace + RLS** | Logical only | Lowest | Not recommended for SaaS |

---

<a name="option1-lz"></a>
### 3.2 Complete Landing Zone (With Fabric)

```
Azure Management Group
└── SaaS Platform Management Group
    │
    ├── Platform Subscription (shared services)
    │   ├── Hub VNet
    │   │   ├── Azure Firewall (egress filtering)
    │   │   ├── Azure Bastion (secure admin access)
    │   │   ├── Private DNS Zones
    │   │   └── APIM (internal mode – API gateway)
    │   ├── Azure Key Vault (platform-level secrets)
    │   ├── Azure Monitor + Log Analytics Workspace
    │   ├── Microsoft Purview Account
    │   ├── Microsoft Defender for Cloud
    │   ├── Azure App Configuration (feature flags)
    │   ├── Tenant Configuration DB (Azure SQL or Cosmos DB)
    │   └── CI/CD (Azure DevOps / GitHub Actions)
    │
    ├── Fabric / Analytics Subscription
    │   ├── Microsoft Fabric Capacity (F64 or higher)
    │   ├── OneLake Storage (unified, auto-provisioned)
    │   ├── Per-Tenant Fabric Workspaces
    │   │   ├── Workspace: Tenant A (Lakehouse, Warehouse, Reports, Pipelines)
    │   │   ├── Workspace: Tenant B
    │   │   └── Workspace: Tenant N...
    │   └── Shared Workspace (platform-level: templates, shared datasets)
    │
    ├── Application Subscription
    │   ├── Spoke VNet (peered to Hub)
    │   ├── Azure App Service / Functions (backend APIs)
    │   ├── Azure Static Web Apps (admin portal / SaaS frontend)
    │   ├── Azure OpenAI Service
    │   ├── Azure AI Search (RAG index per tenant)
    │   └── Power BI Embedded Capacity
    │
    └── Per-Tenant Spoke Subscriptions (Enterprise tier only)
        ├── Tenant Spoke VNet (peered to Hub)
        ├── Dedicated Fabric Capacity (if required)
        ├── Tenant Key Vault (customer-managed keys)
        └── Private Endpoints to shared platform services
```

#### Hub-and-Spoke Network – With Fabric

```
┌──────────────────────────────────────────────────────┐
│                     HUB VNet                         │
│  Azure Firewall │ Private DNS │ Bastion │ APIM       │
└───────────────────────┬──────────────────────────────┘
          ┌─────────────┼──────────────┐
          ▼             ▼              ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │ App Spoke  │ │ Enterprise │ │ Platform   │
   │ VNet       │ │ Tenant     │ │ Shared     │
   │ (App Svc,  │ │ Spoke VNet │ │ Services   │
   │  Functions,│ │ (Dedicated │ │ (Monitor,  │
   │  OpenAI,   │ │  Fabric    │ │  Purview,  │
   │  AI Search)│ │  Capacity) │ │  Key Vault)│
   └────────────┘ └────────────┘ └────────────┘
          │             │              │
          └─────────────┼──────────────┘
                        ▼
              Microsoft Fabric
              (Managed VNet / Private Links)
              OneLake + Workspaces
```

#### Tenant Onboarding Flow (With Fabric)

```
New Tenant Request
       │
       ▼
CI/CD Pipeline (Bicep / Terraform)
       │
       ├──► Create Fabric Workspace (via Fabric REST API)
       ├──► Configure OneLake folders & RBAC
       ├──► Deploy parameterised Fabric Pipelines
       ├──► Create Power BI reports (from templates)
       ├──► Register tenant in Config DB (Azure SQL / Cosmos DB)
       ├──► Set feature flags in Azure App Configuration
       ├──► Create APIM subscription (API key per tenant)
       ├──► Create AI Search index (for RAG, if enabled)
       └──► Federate tenant IdP in Entra ID (B2B)
```

#### Data Flow – With Fabric

```
Tenant Source Systems (ERP, CRM, Files, APIs, DBs)
       │
       ▼
Azure Data Factory / Fabric Pipelines (parameterised per tenant)
       │
       ▼
┌──────────────────────────────────────┐
│          Fabric Lakehouse            │
│  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │ Bronze │─►│ Silver │─►│  Gold  │ │
│  │ (raw)  │  │(clean) │  │(curated│ │
│  └────────┘  └────────┘  └────────┘ │
│          OneLake (Delta/Parquet)      │
└──────────────┬───────────────────────┘
               │
       ┌───────┼──────────┐
       ▼       ▼          ▼
  Power BI  Fabric SQL   Fabric
  Reports   Warehouse    Notebooks
            (ad-hoc      (data science
             queries)     / ML)
```

---

<a name="option1-identity"></a>
### 3.3 Identity & Access (With Fabric)

```
Tenant User
    │
    ▼
Tenant's Corporate IdP ──SAML 2.0 / OIDC──► Microsoft Entra ID (B2B Federation)
                                                      │
                                               Entra ID issues token
                                               (with tenant context)
                                                      │
                                        ┌─────────────┼───────────────┐
                                        ▼             ▼               ▼
                                  Fabric API    App Service      APIM Gateway
                                 (workspace     (backend API)   (subscription
                                  scoped)                        per tenant)
```

| Layer | Service | Tenant Isolation Mechanism |
|---|---|---|
| **Identity** | Microsoft Entra ID | B2B federation per tenant IdP; `tid` claim in tokens |
| **API Gateway** | Azure API Management | Subscription key per tenant; rate limiting; policies |
| **Analytics** | Fabric Workspaces | Workspace RBAC; sensitivity labels; workspace identity |
| **Storage** | OneLake | Folder-level RBAC; data isolation per workspace |
| **Reports** | Power BI Embedded | Row-level security + workspace-scoped embedding |
| **Secrets** | Azure Key Vault | Customer-managed keys (Enterprise tier) |
| **Config** | Azure App Configuration | Feature filters keyed by tenant ID |

---

<a name="option1-ai"></a>
### 3.4 AI & Analytics (With Fabric)

Fabric provides three built-in AI pillars:

| Pillar | Technology | What It Does | Multi-Tenant Safety |
|---|---|---|---|
| **Fabric Copilot** | Built-in Fabric AI | Natural language queries on Lakehouse data, auto-generate reports, write Spark code | Workspace-scoped — Tenant A's Copilot cannot access Tenant B's data |
| **Azure OpenAI + RAG** | Azure OpenAI + Azure AI Search | Retrieval-Augmented Generation on structured and unstructured tenant data | Per-tenant AI Search index or security-trimmed with tenant ID |
| **Real-Time Intelligence** | Fabric Eventstream + KQL DB + Data Activator | Streaming ingestion, sub-second queries, automated alerts on threshold breaches | Eventstreams and KQL databases are workspace-scoped |

```
┌─────────────────────────────────────────────────────────────┐
│                     AI on Fabric                            │
│                                                             │
│  "What happened?"      "Why?"           "Alert me"          │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │ Fabric       │  │ Azure OpenAI  │  │ Real-Time       │  │
│  │ Copilot      │  │ + RAG         │  │ Intelligence    │  │
│  │ (structured) │  │ (structured + │  │ (streaming +    │  │
│  │              │  │  unstructured)│  │  triggers)      │  │
│  └──────┬───────┘  └──────┬────────┘  └──────┬──────────┘  │
│         └─────────────────┼──────────────────┘              │
│                           ▼                                 │
│                       OneLake                               │
│                 (single source of truth)                     │
└─────────────────────────────────────────────────────────────┘
```

### Pros — Option 1

- **Unified platform** — data engineering, warehousing, BI, real-time, and AI in one SaaS experience
- **OneLake** eliminates data duplication — all workloads read from the same storage
- **Workspace-per-tenant** is a first-class multi-tenancy primitive with built-in RBAC
- **Fabric Copilot** delivers AI with zero additional infrastructure
- **Built-in compliance** — ISO 27001, SOC2, GDPR, HIPAA eligible
- **Fastest time-to-value** — less custom engineering, more platform leverage
- **Tenant onboarding** automatable via Fabric REST APIs

### Cons — Option 1

- Fabric is relatively new — some APIs and features are still maturing
- F SKU pricing can be opaque for initial estimation
- Customisation is limited to what Fabric exposes
- Fabric is Azure-only (no native multi-cloud; Azure Arc can bridge for edge cases)

[↑ Back to top](#)

---

<a name="option2"></a>
## 4. Option 2 – Azure Native without Fabric

<a name="option2-arch"></a>
### 4.1 Architecture Overview

This option builds the same multi-tenant analytics SaaS using **individual Azure PaaS services** — without Microsoft Fabric. The data platform is assembled from Azure Data Lake Storage Gen2, Azure Synapse Analytics, Azure Data Factory, Azure SQL, and AKS for application workloads. Power BI Embedded remains the analytics front-end.

Multi-tenancy is achieved through **Azure resource-level isolation** — separate storage containers, separate Synapse workspaces or dedicated SQL pools, and AKS namespace isolation for application services.

```
┌──────────────────────────────────────────────────────────────────────┐
│                  OPTION 2: AZURE NATIVE WITHOUT FABRIC               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                  Presentation Layer                          │    │
│  │  Power BI Embedded │ Custom SaaS Portal (App Service)        │    │
│  │  Admin UI (Static Web Apps) │ Azure API Management           │    │
│  └────────────────────────────────┬─────────────────────────────┘    │
│                                   │                                  │
│  ┌────────────────────────────────▼─────────────────────────────┐    │
│  │                  Application / API Layer                     │    │
│  │  Azure Kubernetes Service (AKS) – microservices              │    │
│  │  Azure App Service / Functions (lightweight APIs)            │    │
│  │  Tenant Config DB (Azure SQL / Cosmos DB)                    │    │
│  │  Azure App Configuration (feature flags per tenant)          │    │
│  └────────────────────────────────┬─────────────────────────────┘    │
│                                   │                                  │
│  ┌────────────────────────────────▼─────────────────────────────┐    │
│  │                  Data Platform (Assembled PaaS)               │    │
│  │                                                              │    │
│  │  ┌──────────────────────────────────────────────────────┐    │    │
│  │  │         Azure Data Lake Storage Gen2 (ADLS)          │    │    │
│  │  │  Tenant A Container │ Tenant B Container │ Tenant C  │    │    │
│  │  │  (Storage account-level or container-level isolation) │    │    │
│  │  └──────────────────────────┬───────────────────────────┘    │    │
│  │                             │                                │    │
│  │  ┌──────────────────────────▼───────────────────────────┐    │    │
│  │  │            Azure Synapse Analytics                   │    │    │
│  │  │  Synapse Workspace (shared or per-tenant)            │    │    │
│  │  │  Dedicated SQL Pool │ Serverless SQL │ Spark Pools   │    │    │
│  │  └──────────────────────────────────────────────────────┘    │    │
│  │                                                              │    │
│  │  Azure Data Factory (orchestration, parameterised per tenant)│    │
│  │  Azure SQL Database (tenant metadata, config, operational)   │    │
│  │  Azure Event Hubs (streaming ingestion)                      │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                  AI Layer                                    │    │
│  │  Azure OpenAI │ Azure AI Search (RAG) │ Azure Machine Learning│   │
│  │  Azure Stream Analytics (real-time) │ Azure Data Explorer     │   │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Cross-Cutting Services:                                             │
│  Microsoft Entra ID │ Azure Key Vault │ Microsoft Purview            │
│  Azure Monitor / Log Analytics │ Azure Policy │ Microsoft Defender   │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Role | Multi-Tenancy Model |
|---|---|---|
| **ADLS Gen2** | Data lake storage (Bronze/Silver/Gold) | Separate storage account or container per tenant |
| **Azure Synapse Analytics** | SQL analytics, Spark processing | Workspace per tenant or shared + schema isolation |
| **Azure Data Factory** | Pipeline orchestration | Parameterised pipelines per tenant |
| **Azure SQL Database** | Operational / metadata DB | Database-per-tenant or elastic pool |
| **Azure Kubernetes Service** | Application microservices | Namespace-per-tenant with network policies |
| **Azure Event Hubs** | Streaming data ingestion | Consumer group per tenant or dedicated namespace |
| **Azure Data Explorer (ADX)** | Time-series / real-time analytics | Database-per-tenant within cluster |
| **Power BI Embedded** | Tenant-facing analytics | Workspace-per-tenant + RLS |
| **Azure API Management** | API gateway | Subscription-per-tenant, rate limiting |
| **Microsoft Entra ID** | Identity & SSO | B2B federation with tenant IdPs |
| **Azure Key Vault** | Secrets & CMK | Per-tenant vaults (Enterprise tier) |
| **Microsoft Purview** | Data governance & lineage | Cross-tenant catalog |
| **Azure App Configuration** | Feature flags & tenant config | Per-tenant feature filters |

### Tenancy Models (Without Fabric)

| Layer | Isolation Options |
|---|---|
| **Storage (ADLS Gen2)** | Separate storage account per tenant *(strongest)* / Separate container per tenant *(moderate)* / Shared container + folder ACLs *(weakest)* |
| **Analytics (Synapse)** | Separate Synapse workspace per tenant / Shared workspace + schema-per-tenant / Dedicated SQL pool per tenant |
| **Compute (AKS)** | Namespace-per-tenant *(default)* / Node pool per tenant *(high isolation)* / Cluster per tenant *(max isolation)* |
| **Operational DB (Azure SQL)** | Database-per-tenant / Elastic pool with per-tenant DBs / Shared DB + tenant_id column |
| **Streaming (Event Hubs)** | Dedicated namespace per tenant / Shared namespace + consumer groups |

---

<a name="option2-lz"></a>
### 4.2 Complete Landing Zone (Without Fabric)

```
Azure Management Group
└── SaaS Platform Management Group
    │
    ├── Platform Subscription (shared services)
    │   ├── Hub VNet
    │   │   ├── Azure Firewall (egress filtering)
    │   │   ├── Azure Bastion (secure admin access)
    │   │   ├── Private DNS Zones
    │   │   └── APIM (internal mode – API gateway)
    │   ├── Azure Key Vault (platform-level secrets)
    │   ├── Azure Monitor + Log Analytics Workspace
    │   ├── Microsoft Purview Account
    │   ├── Microsoft Defender for Cloud
    │   ├── Azure App Configuration (feature flags)
    │   ├── Tenant Configuration DB (Azure SQL)
    │   └── CI/CD (Azure DevOps / GitHub Actions)
    │
    ├── Data Platform Subscription
    │   ├── Spoke VNet (peered to Hub)
    │   ├── ADLS Gen2 Storage Account(s)
    │   │   ├── Container: Tenant A (Bronze / Silver / Gold folders)
    │   │   ├── Container: Tenant B
    │   │   └── Container: Tenant N...
    │   ├── Azure Synapse Analytics Workspace
    │   │   ├── Serverless SQL Pool (cross-tenant ad-hoc queries)
    │   │   ├── Dedicated SQL Pool(s) (high-performance tenants)
    │   │   └── Spark Pool (data engineering / ML)
    │   ├── Azure Data Factory (parameterised pipelines)
    │   ├── Azure Event Hubs Namespace (streaming ingestion)
    │   ├── Azure Data Explorer Cluster (real-time analytics)
    │   │   ├── Database: Tenant A
    │   │   ├── Database: Tenant B
    │   │   └── Database: Tenant N...
    │   └── Private Endpoints to all data services
    │
    ├── Application Subscription
    │   ├── Spoke VNet (peered to Hub)
    │   ├── Azure Kubernetes Service (application microservices)
    │   │   ├── Namespace: Tenant A
    │   │   ├── Namespace: Tenant B
    │   │   └── Namespace: shared-services
    │   ├── Azure App Service / Functions (lightweight APIs)
    │   ├── Azure Static Web Apps (admin portal / SaaS frontend)
    │   ├── Azure OpenAI Service
    │   ├── Azure AI Search (RAG index per tenant)
    │   ├── Azure Machine Learning Workspace
    │   └── Power BI Embedded Capacity
    │
    └── Per-Tenant Spoke Subscriptions (Enterprise tier only)
        ├── Tenant Spoke VNet (peered to Hub)
        ├── Dedicated ADLS Gen2 Storage Account
        ├── Dedicated Synapse Workspace / SQL Pool
        ├── Dedicated ADX Database
        ├── Tenant Key Vault (customer-managed keys)
        └── Private Endpoints to shared platform services
```

#### Hub-and-Spoke Network – Without Fabric

```
┌──────────────────────────────────────────────────────┐
│                     HUB VNet                         │
│  Azure Firewall │ Private DNS │ Bastion │ APIM       │
└───────────────────────┬──────────────────────────────┘
          ┌─────────────┼──────────────────┐
          ▼             ▼                  ▼
   ┌────────────┐ ┌──────────────┐  ┌────────────┐
   │ Data       │ │ App Spoke    │  │ Enterprise │
   │ Platform   │ │ VNet         │  │ Tenant     │
   │ Spoke VNet │ │ (AKS, App    │  │ Spoke VNet │
   │ (ADLS,     │ │  Service,    │  │ (Dedicated │
   │  Synapse,  │ │  Functions,  │  │  storage,  │
   │  ADF, ADX, │ │  OpenAI,     │  │  Synapse,  │
   │  Event Hub)│ │  AI Search)  │  │  ADX, KV)  │
   └────────────┘ └──────────────┘  └────────────┘
```

#### Tenant Onboarding Flow (Without Fabric)

```
New Tenant Request
       │
       ▼
CI/CD Pipeline (Bicep / Terraform)
       │
       ├──► Create ADLS Gen2 container + folder structure (Bronze/Silver/Gold)
       ├──► Configure ADLS ACLs & RBAC
       ├──► Create Synapse schema / linked service for tenant
       ├──► Deploy ADF pipeline (parameterised for tenant)
       ├──► Create AKS namespace + network policies + resource quotas
       ├──► Create ADX database for tenant (if real-time needed)
       ├──► Create Power BI workspace + deploy reports from templates
       ├──► Register tenant in Config DB (Azure SQL)
       ├──► Set feature flags in Azure App Configuration
       ├──► Create APIM subscription (API key per tenant)
       ├──► Create AI Search index (for RAG, if enabled)
       └──► Federate tenant IdP in Entra ID (B2B)
```

#### Data Flow – Without Fabric

```
Tenant Source Systems (ERP, CRM, Files, APIs, DBs)
       │
       ▼
Azure Data Factory (parameterised per tenant)
       │
       ▼
┌──────────────────────────────────────────┐
│        ADLS Gen2 (per-tenant container)  │
│  ┌────────┐  ┌────────┐  ┌────────┐     │
│  │ Bronze │─►│ Silver │─►│  Gold  │     │
│  │ (raw)  │  │(clean) │  │(curated│     │
│  └────────┘  └────────┘  └────────┘     │
│          Delta / Parquet format          │
└──────────────┬───────────────────────────┘
               │
       ┌───────┼──────────────┐
       ▼       ▼              ▼
  Power BI  Synapse SQL    Synapse Spark
  Reports   (ad-hoc        (data engineering
             queries)       / ML)
```

---

<a name="option2-identity"></a>
### 4.3 Identity & Access (Without Fabric)

```
Tenant User
    │
    ▼
Tenant's Corporate IdP ──SAML 2.0 / OIDC──► Microsoft Entra ID (B2B Federation)
                                                      │
                                               Entra ID issues token
                                               (with tenant context)
                                                      │
                                        ┌─────────────┼───────────────┐
                                        ▼             ▼               ▼
                                  AKS / App Svc  Synapse SQL     APIM Gateway
                                 (namespace      (schema          (subscription
                                  scoped)         scoped)          per tenant)
```

| Layer | Service | Tenant Isolation Mechanism |
|---|---|---|
| **Identity** | Microsoft Entra ID | B2B federation per tenant IdP; `tid` claim in tokens |
| **API Gateway** | Azure API Management | Subscription key per tenant; rate limiting; policies |
| **Compute** | AKS | Namespace isolation + network policies + resource quotas |
| **Storage** | ADLS Gen2 | Container-level or storage account-level isolation + ACLs |
| **Analytics** | Azure Synapse | Workspace-per-tenant or schema-per-tenant |
| **Real-Time** | Azure Data Explorer | Database-per-tenant within shared cluster |
| **Reports** | Power BI Embedded | Workspace-per-tenant + row-level security |
| **Secrets** | Azure Key Vault | Customer-managed keys (Enterprise tier) |
| **Config** | Azure App Configuration | Feature filters keyed by tenant ID |

---

<a name="option2-ai"></a>
### 4.4 AI & Analytics (Without Fabric)

Without Fabric, AI capabilities are assembled from individual Azure AI services:

| Capability | Azure Service | What It Does | Multi-Tenant Safety |
|---|---|---|---|
| **LLM / Generative AI** | Azure OpenAI Service | GPT-based natural language queries, summarisation, content generation | API-level isolation; tenant context injected per request |
| **RAG (Retrieval-Augmented Generation)** | Azure OpenAI + Azure AI Search | Ground LLM answers in tenant-specific data (documents, tables) | Per-tenant AI Search index or security-trimmed filters |
| **Real-Time Analytics** | Azure Data Explorer (ADX) + Azure Stream Analytics | Sub-second time-series queries, anomaly detection, streaming ETL | ADX database-per-tenant; Stream Analytics job per tenant or shared with routing |
| **Machine Learning** | Azure Machine Learning | Custom ML models, batch inference, model registry | Workspace-per-tenant or shared with experiment isolation |
| **Automated Alerts** | Azure Monitor Alerts + Azure Logic Apps | Trigger notifications when metrics breach thresholds | Alert rules scoped per tenant resource; Logic App per tenant |

```
┌─────────────────────────────────────────────────────────────┐
│                  AI without Fabric                          │
│                                                             │
│  "What happened?"      "Why?"           "Alert me"          │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐  │
│  │ Power BI     │  │ Azure OpenAI  │  │ Azure Data      │  │
│  │ Q&A /        │  │ + RAG         │  │ Explorer +      │  │
│  │ Synapse SQL  │  │ (structured + │  │ Stream Analytics│  │
│  │ (structured) │  │  unstructured)│  │ + Monitor Alerts│  │
│  └──────┬───────┘  └──────┬────────┘  └──────┬──────────┘  │
│         └─────────────────┼──────────────────┘              │
│                           ▼                                 │
│                   ADLS Gen2 + Synapse                       │
│                 (data platform layer)                        │
└─────────────────────────────────────────────────────────────┘
```

### Pros — Option 2

- **Full control** over every component — choose compute, storage, and analytics independently
- **Mature services** — ADLS Gen2, Synapse, ADF, AKS are all GA with established SLAs
- **Granular cost control** — scale each service independently; stop what you don't need
- **AKS flexibility** — run custom application logic, microservices, open-source tools alongside Azure PaaS
- **Azure Data Explorer** — purpose-built for real-time time-series, potentially higher performance for streaming use cases
- **No Fabric licensing dependency** — use existing Azure commitments / EA

### Cons — Option 2

- **More services to integrate** — data flows across ADLS, Synapse, ADF, AKS, ADX, Power BI (vs. Fabric's unified model)
- **Higher operational complexity** — AKS cluster management, Synapse pool sizing, ADX cluster scaling
- **Data duplication risk** — data may need to be copied between services (vs. OneLake's single-copy model)
- **No built-in Copilot** — must build AI experiences from individual Azure AI services
- **Tenant onboarding is more complex** — more resources to provision per tenant
- **Governance requires more wiring** — Purview integration must be manually configured per service

[↑ Back to top](#)

---

<a name="comparison"></a>
## 5. Side-by-Side Comparison

### Architecture Components Mapping

| Capability | Option 1 (With Fabric) | Option 2 (Without Fabric) |
|---|---|---|
| **Unified Storage** | OneLake (auto-provisioned) | ADLS Gen2 (manually provisioned per tenant) |
| **Data Lakehouse** | Fabric Lakehouse (built-in) | ADLS Gen2 + Delta format + Synapse Spark |
| **SQL Analytics** | Fabric SQL Warehouse | Synapse Dedicated SQL Pool / Serverless SQL |
| **Data Pipelines** | Fabric Pipelines + ADF | Azure Data Factory |
| **Data Engineering** | Fabric Notebooks (Spark) | Synapse Spark Pools |
| **Real-Time Analytics** | Fabric Eventstream + KQL DB + Data Activator | Azure Event Hubs + Azure Data Explorer + Stream Analytics |
| **BI / Reporting** | Power BI (native in Fabric) | Power BI Embedded (separate service) |
| **AI Assistant** | Fabric Copilot (built-in) | Not available — must build custom |
| **Generative AI** | Azure OpenAI (native integration) | Azure OpenAI (manual integration) |
| **RAG** | Azure AI Search + OpenAI | Azure AI Search + OpenAI |
| **Application Compute** | Azure App Service / Functions | AKS + App Service / Functions |
| **Identity** | Microsoft Entra ID | Microsoft Entra ID |
| **API Gateway** | Azure API Management | Azure API Management |
| **Data Governance** | Microsoft Purview (auto-wired) | Microsoft Purview (manual configuration) |
| **Secrets** | Azure Key Vault | Azure Key Vault |
| **Monitoring** | Azure Monitor + Fabric metrics | Azure Monitor + per-service metrics |

### Trade-off Summary

| Dimension | Option 1 (With Fabric) | Option 2 (Without Fabric) | Advantage |
|---|---|---|---|
| **Unified Experience** | Single pane for data + analytics + AI | Multiple services to compose | **Option 1** |
| **Time-to-Value** | Faster — fewer moving parts | Slower — more integration work | **Option 1** |
| **Operational Complexity** | Lower — Fabric is SaaS-managed | Higher — must manage AKS, Synapse pools, ADX clusters | **Option 1** |
| **Multi-Tenancy (Data)** | Workspace-per-tenant (native) | Container / schema / DB per tenant (manual) | **Option 1** |
| **Multi-Tenancy (Compute)** | Shared F SKU or dedicated capacity | AKS namespaces + Synapse pool sizing | Tie |
| **Customisation / Control** | Constrained to Fabric APIs | Full control over each service | **Option 2** |
| **AI Copilot (Built-In)** | Yes — Fabric Copilot | No — must build or go without | **Option 1** |
| **Real-Time Analytics** | Fabric Eventstream + KQL | ADX + Event Hubs + Stream Analytics | Tie |
| **Cost Visibility** | Single F SKU (capacity model) | Per-service billing (granular) | **Option 2** |
| **Service Maturity** | Fabric is newer; some APIs in preview | All services are GA with established SLAs | **Option 2** |
| **Data Duplication** | Eliminated (OneLake) | Possible (data moves between services) | **Option 1** |
| **Compliance** | Fabric inherits Azure certifications | Each service inherits Azure certifications | Tie |
| **Open-Source Tooling** | Limited (within Fabric's boundaries) | Full flexibility (dbt, Spark, Trino on AKS) | **Option 2** |
| **Tenant Onboarding** | Simpler — fewer resources per tenant | More complex — more resources to provision | **Option 1** |

### Decision Framework

```
Does the organisation want a unified analytics SaaS platform with built-in AI?
    │
    YES ──► Is Fabric SKU pricing acceptable?
    │           YES ──► Option 1 (With Fabric)
    │           NO  ──► Option 2 (Without Fabric) — use existing Azure EA
    │
    NO ──► Does the team need full control over each component?
               YES ──► Option 2 (Without Fabric)
               NO  ──► Option 1 (With Fabric) — reduces engineering burden
```

> **Both options are Azure-native.** The choice is platform leverage (Fabric) vs. component control (assembled PaaS). There is no wrong answer — only a trade-off between speed-to-value and architectural flexibility.

[↑ Back to top](#)

---

<a name="references"></a>
## 6. Reference Architectures & Resources

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

### Azure PaaS (Non-Fabric) References

| Resource | Link |
|---|---|
| Azure Data Lake Storage Gen2 | [Link](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction) |
| Azure Synapse Analytics overview | [Link](https://learn.microsoft.com/en-us/azure/synapse-analytics/overview-what-is) |
| Azure Data Factory overview | [Link](https://learn.microsoft.com/en-us/azure/data-factory/introduction) |
| Azure Data Explorer overview | [Link](https://learn.microsoft.com/en-us/azure/data-explorer/data-explorer-overview) |
| Azure Event Hubs overview | [Link](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-about) |
| Azure Stream Analytics | [Link](https://learn.microsoft.com/en-us/azure/stream-analytics/stream-analytics-introduction) |
| AKS multi-tenancy best practices | [Link](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation) |
| AKS baseline architecture | [Link](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks) |
| Azure SQL elastic pools | [Link](https://learn.microsoft.com/en-us/azure/azure-sql/database/elastic-pool-overview) |
| Azure Machine Learning | [Link](https://learn.microsoft.com/en-us/azure/machine-learning/overview-what-is-azure-machine-learning) |

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

### Identity References

| Resource | Link |
|---|---|
| Microsoft Entra External ID (B2B federation) | [Link](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b) |
| Federation with external IdPs via SAML | [Link](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-saml-single-sign-on) |
| Claims-based identity on Azure | [Link](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens) |
| Identity approaches for multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity) |

### AI & Analytics References

| Resource | Link |
|---|---|
| Fabric AI / Copilot overview | [Link](https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview) |
| Copilot in Power BI | [Link](https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-introduction) |
| Azure OpenAI on your data (RAG) | [Link](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/use-your-data) |
| Implement RAG with Azure OpenAI | [Link](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview) |
| Azure AI Search | [Link](https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search) |

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
