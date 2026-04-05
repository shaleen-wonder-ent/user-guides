# Multi-Tenant SaaS Landing Zone Architecture

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
5. [Option 3 – Portable (OSS + Azure SaaS)](#option3)
   - [Architecture Overview](#option3-arch)
   - [Complete Landing Zone](#option3-lz)
   - [Identity & Access](#option3-identity)
   - [AI & Analytics](#option3-ai)
6. [Side-by-Side Comparison](#comparison)
7. [Regional Landing Zone Strategy](#regional-lz)
   - [Co-Located Tenants (Same Region)](#colocated-tenants)
   - [Data-Residency Tenants (Sovereign Region)](#data-residency-tenants)
8. [Reference Architectures & Resources](#references)

---

<a name="introduction"></a>
## 1. Introduction

This document provides a reference architecture for building a **multi-tenant SaaS analytics platform**. It presents three paths — two Azure-native and one portable:

| | Option 1 | Option 2 | Option 3 |
|---|---|---|---|
| **Platform** | Microsoft Fabric + Azure PaaS | Azure PaaS (Databricks, AKS, ADLS, SQL) — no Fabric | Portable (Kubernetes + OSS + Azure SaaS services) |
| **Cloud** | Azure-native | Azure-native | Portable (runs on Azure, AWS, GCP, or on-prem — uses Azure SaaS services that are cloud-agnostic) |
| **Multi-tenancy** | Fabric workspace isolation | Azure subscription / resource-level isolation | Kubernetes namespace + database-level isolation |
| **Analytics** | OneLake + Lakehouse + Power BI Embedded | ADLS Gen2 + Azure Databricks + Power BI Embedded | Azure Blob Storage + Apache Spark + Power BI Embedded |
| **AI** | Fabric Copilot + Azure OpenAI | Azure OpenAI + Azure AI Search | LLM-agnostic (OpenAI / Anthropic / OSS) + OpenSearch |

Options 1 and 2 are fully Azure-native. Option 3 uses open-source and portable technologies combined with **Azure SaaS services that are cloud-agnostic** (Power BI Embedded, GitHub Actions, Azure Blob Storage) — services that work from any cloud without architectural lock-in. The compute and analytics layers remain portable, while Azure SaaS services provide enterprise-grade capabilities consumable from any hyperscaler. The choice depends on whether the organisation prioritises platform leverage, component control, or multi-cloud portability.

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
│  │                  Microsoft Fabric (Analytics Platform)       │    │
│  │                                                              │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │    │
│  │  │ Workspace A │  │ Workspace B │  │ Workspace C │           │    │
│  │  │ (Tenant A)  │  │ (Tenant B)  │  │ (Tenant C)  │           │    │
│  │  │ Lakehouse   │  │ Lakehouse   │  │ Lakehouse   │           │    │
│  │  │ Warehouse   │  │ Warehouse   │  │ Warehouse   │           │    │
│  │  │ Notebooks   │  │ Notebooks   │  │ Notebooks   │           │    │
│  │  │ Pipelines   │  │ Pipelines   │  │ Pipelines   │           │    │
│  │  │ Reports     │  │ Reports     │  │ Reports     │           │    │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘           │    │
│  │         │                │                │                  │    │
│  │  ┌──────▼────────────────▼────────────────▼───────────┐      │    │
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
│  │  Fabric Copilot / Azure OpenAI / Azure AI Search (RAG)       │    │
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
| **Power BI Embedded** (Part of Fabric)| Tenant-facing analytics UI | Row-level security + workspace isolation |
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
### 3.2 Complete Landing Zone

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
    │   └── Power BI Embedded Capacity
    │
    ├── Application Subscription
    │   ├── Spoke VNet (peered to Hub)
    │   ├── Azure Kubernetes Service / Azure App Service / Functions (backend APIs) - depends what we choose
    │   ├── Azure Static Web Apps (admin portal / SaaS frontend)
    │   ├── Azure OpenAI Service
    │   └── Azure AI Search (RAG index per tenant)
    │   
    │
    └── Per-Tenant Spoke Subscriptions (Enterprise tier only)
        ├── Tenant Spoke VNet (peered to Hub)
        ├── Dedicated Fabric Capacity (if required)
        ├── Tenant Key Vault (customer-managed keys)
        └── Private Endpoints to shared platform services
```

#### Hub-and-Spoke Network 

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


#### Data Flow 

```
Tenant Source Systems (ERP, CRM, Files, APIs, DBs)
       │
       ▼
Azure Data Factory / Fabric Pipelines (parameterised per tenant)
       │
       ▼
┌──────────────────────────────────────┐
│          Fabric Lakehouse            │
│  ┌────────┐  ┌────────┐  ┌────────┐  │
│  │ Bronze │─►│ Silver │─►│  Gold  │  │
│  │ (raw)  │  │(clean) │  │(curated│  │
│  └────────┘  └────────┘  └────────┘  │
│          (Delta/Parquet)             │
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
### 3.3 Identity & Access 

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
| **Reports** | Power BI Embedded (Part of Fabric) | Row-level security + workspace-scoped embedding |
| **Secrets** | Azure Key Vault | Customer-managed keys (Enterprise tier) |
| **Config** | Azure App Configuration | Feature filters keyed by tenant ID |

---

<a name="option1-ai"></a>
### 3.4 AI & Analytics 

With Fabric as the data foundation, the AI layer benefits from **OneLake as a single source of truth** — every AI capability (Copilot, Agents, RAG, ML) reads from the same governed, tenant-isolated data without duplication.

#### Complete Azure AI Architecture (With Fabric)

```
┌───────────────────────────────────────────────────────────────────────┐
│                    AI PLATFORM ON AZURE (WITH FABRIC)                 │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │                     AI Application Layer                     │     │
│  │                                                              │     │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐ │     │
│  │  │ Fabric       │  │  Fabric Data │  │  AI Agents          │ │     │
│  │  │ Copilot      │  │  Agent       │  │  (Azure AI Foundry  │ │     │
│  │  │              │  │  Chat / Q&A  │  │   Agent Service)    │ │     │
│  │  │ (built-in)   │  │  (RAG-based) │  │                     │ │     │
│  │  └──────────────┘  └──────────────┘  └─────────────────────┘ │     │
│  └──────────────────────────────┬───────────────────────────────┘     │
│                                 │                                     │
│  ┌──────────────────────────────▼───────────────────────────────┐     │
│  │                     Orchestration Layer                      │     │
│  │  Semantic Kernel │ Azure AI Foundry │ Prompt Flow| Copilot   │     │
│  │  (agent orchestration, tool calling, multi-step reasoning)   │     │
│  └──────────────────────────────┬───────────────────────────────┘     │
│                                 |──────────────────────────────────┐  │                                  
│  ┌──────────────────────────────▼───────────────────────────────┐  │  │
│  │                     AI Models & Services                     │  │  │
│  │  Azure OpenAI (Models) │ Azure AI Search (RAG index)         │  │  │
│  │  Azure AI Document Intelligence │ Azure AI Content Safety    │  │  │
│  │  Azure Machine Learning (custom models, fine-tuning)         │  │  │
│  └──────────────────────────────────────────────────────────────┘  │  │
│                                 ┌──────────────────────────────────┘  │
│  ┌──────────────────────────────▼───────────────────────────────┐     │
│  │                     Data Foundation                          │     │
│  │  (Delta/Parquet) │ Fabric Lakehouse (Bronze/Silver/Gold)     │     │
│  │  Fabric Real-Time Intelligence (Eventstream + KQL)           │     │
│  │  Per-tenant workspace isolation                              │     │
│  └──────────────────────────────────────────────────────────────┘     │
└───────────────────────────────────────────────────────────────────────┘
```

#### AI Capabilities Breakdown

| Capability | Azure Service | Role in Multi-Tenant SaaS | Multi-Tenant Isolation |
|---|---|---|---|
| **Fabric Copilot** | Built-in Fabric AI | Natural language queries on Lakehouse data, auto-generate Power BI reports, write Spark code | Workspace-scoped — Tenant A's Copilot cannot access Tenant B's data |
| **Azure AI Foundry** | Azure AI Foundry (portal + SDK) | Centralised platform to build, evaluate, and deploy AI models and agents. Model catalog, prompt engineering, evaluation, and deployment management | Project-per-tenant or shared project with tenant-scoped deployments |
| **AI Agents** | Azure AI Foundry Agent Service | Autonomous agents that can reason, plan, use tools, and take actions on behalf of users. Agents call functions, query data, and execute multi-step workflows | Agent instances scoped per tenant; tool access restricted to tenant data |
| **RAG (Retrieval-Augmented Generation)** | Azure OpenAI + Azure AI Search | Ground LLM answers in tenant-specific data — documents, tables, knowledge bases. Eliminates hallucination by retrieving real data before generation | Per-tenant AI Search index or security-trimmed with tenant ID field |
| **Semantic Kernel** | Open-source SDK (.NET / Python) | Orchestration framework for building AI agents and plugins. Connects LLMs to your code, data, and APIs with built-in planning and tool calling | Tenant context passed via kernel arguments; plugins scoped per tenant |
| **Document Intelligence** | Azure AI Document Intelligence | Extract structured data from documents (invoices, contracts, forms, reports). Feeds extracted data into the Lakehouse or directly into RAG pipelines | Per-tenant processing; output stored in tenant's OneLake folder |
| **Content Safety** | Azure AI Content Safety | Filter harmful, biased, or inappropriate content from AI inputs and outputs. Jailbreak detection and prompt shield | Applied globally; audit logs per tenant |
| **Real-Time Intelligence** | Fabric Eventstream + KQL DB + Data Activator | Streaming ingestion, sub-second queries, automated alerts on threshold breaches | Eventstreams and KQL databases are workspace-scoped |
| **Machine Learning** | Azure ML + Fabric Notebooks | Custom model training, batch inference, MLOps. Train on tenant data within Fabric Notebooks (PySpark) | Models trained per-tenant or shared with tenant-parameterised inference |

[↑ Back to top](#)

---

<a name="option2"></a>
## 4. Option 2 – Azure Native without Fabric

<a name="option2-arch"></a>
### 4.1 Architecture Overview

This option builds the same multi-tenant analytics SaaS using **individual Azure PaaS services** — without Microsoft Fabric. The data platform is assembled from Azure Data Lake Storage Gen2, **Azure Databricks** (as the unified analytics engine), Azure Data Factory, Azure SQL, and AKS for application workloads. Power BI Embedded remains the analytics front-end.

This follows the **[Modern Analytics Architecture with Azure Databricks](https://learn.microsoft.com/en-us/azure/architecture/solution-ideas/articles/azure-databricks-modern-analytics-architecture)** reference pattern — where Databricks serves as the central tool for data ingestion, processing, and serving, with a medallion architecture (Bronze/Silver/Gold) on ADLS Gen2 using Delta Lake.

Multi-tenancy is achieved through **Azure resource-level isolation** — separate storage containers, separate Databricks workspaces or Unity Catalog schema isolation, and AKS namespace isolation for application services.

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
│  │                  Data Platform (Azure Databricks + ADLS)     │    │
│  │                                                              │    │
│  │  ┌──────────────────────────────────────────────────────┐    │    │
│  │  │         Azure Data Lake Storage Gen2 (ADLS)          │    │    │
│  │  │  Tenant A Container │ Tenant B Container │ Tenant C  │    │    │
│  │  │ (Storage account-level or container-level isolation) │    │    │
│  │  └──────────────────────────┬───────────────────────────┘    │    │
│  │                             │                                │    │
│  │  ┌──────────────────────────▼───────────────────────────┐    │    │
│  │  │            Azure Databricks                          │    │    │
│  │  │  Unity Catalog (governance & access control)         │    │    │
│  │  │ SQL Warehouses │ Delta Live Tables │ Spark Clusters  │    │    │
│  │  │  Medallion: Bronze → Silver → Gold (Delta Lake)      │    │    │
│  │  └──────────────────────────────────────────────────────┘    │    │
│  │                                                              │    │
│  │  Azure Data Factory (orchestration, parameterised per tenant)│    │
│  │  Azure SQL Database (tenant metadata, config, operational)   │    │
│  │  Azure Event Hubs (streaming ingestion)                      │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                  AI Layer                                    │    │
│  │ Azure OpenAI │ Azure AI Search (RAG) │ Azure Machine Learning│    │
│  │  Azure Data Explorer (real-time analytics)                   │    │
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
| **Azure Databricks** | Unified analytics engine — SQL warehouses, Spark, Delta Live Tables, MLflow | Unity Catalog with catalog-per-tenant or schema-per-tenant isolation |
| **Azure Databricks Unity Catalog** | Centralised governance — access control, auditing, lineage, data discovery | Catalog-per-tenant or schema-per-tenant within shared workspace |
| **Delta Lake** | Open table format on ADLS Gen2 — ACID, time travel, schema evolution | Per-tenant tables within the medallion architecture |
| **Azure Data Factory** | Pipeline orchestration (batch ingestion) | Parameterised pipelines per tenant |
| **Azure SQL Database** | Operational / metadata DB | Database-per-tenant or elastic pool |
| **Azure Kubernetes Service** | Application microservices | Namespace-per-tenant with network policies |
| **Azure Event Hubs** | Streaming data ingestion | Consumer group per tenant or dedicated namespace |
| **Azure Data Explorer (ADX)** | Time-series / real-time analytics | Database-per-tenant within cluster |
| **Power BI Embedded** | Tenant-facing analytics | Workspace-per-tenant + RLS |
| **Azure API Management** | API gateway | Subscription-per-tenant, rate limiting |
| **Microsoft Entra ID** | Identity & SSO | B2B federation with tenant IdPs |
| **Azure Key Vault** | Secrets & CMK | Per-tenant vaults (Enterprise tier) |
| **Microsoft Purview** | Data governance & lineage (complements Unity Catalog) | Cross-tenant catalog |
| **Azure App Configuration** | Feature flags & tenant config | Per-tenant feature filters |

### How Tenancy Models happens for these components (They are not interrelated,we are explaing how these services follows tenancy models)

| Layer | Isolation Options |
|---|---|
| **Storage (ADLS Gen2)** | Separate storage account per tenant *(strongest)* / Separate container per tenant *(moderate)* / Shared container + folder ACLs *(weakest)* |
| **Analytics (Databricks)** | Separate Unity Catalog per tenant / Shared catalog + schema-per-tenant / Shared schema + row-level security via dynamic views |
| **Operational DB (Azure SQL)** | Database-per-tenant / Elastic pool with per-tenant DBs / Shared DB + tenant_id column |
| **Streaming (Event Hubs)** | Dedicated namespace per tenant / Shared namespace + consumer groups |
| **Compute (AKS)** | Namespace-per-tenant *(default)* / Node pool per tenant *(high isolation)* / Cluster per tenant *(max isolation)* |

---

<a name="option2-lz"></a>
### 4.2 Complete Landing Zone

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
    │   ├── Azure Databricks Workspace
    │   │   ├── Unity Catalog (centralised governance)
    │   │   ├── SQL Warehouses (SQL analytics, Power BI connectivity)
    │   │   ├── Delta Live Tables Pipelines (per-tenant ETL)
    │   │   └── Spark Clusters (data engineering / ML)
    │   ├── Azure Data Factory (parameterised pipelines — batch ingestion)
    │   ├── Azure Event Hubs Namespace (streaming ingestion)
    │   ├── Azure Data Explorer Cluster (real-time analytics)
    │   │   ├── Database: Tenant A
    │   │   ├── Database: Tenant B
    │   │   └── Database: Tenant N...
    │   └── Private Endpoints to all data services
    │   └── Power BI Embedded Capacity
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
    │   └── Azure Machine Learning Workspace
    │
    └── Per-Tenant Spoke Subscriptions (Enterprise tier only)
        ├── Tenant Spoke VNet (peered to Hub)
        ├── Dedicated ADLS Gen2 Storage Account
        ├── Dedicated Databricks Workspace + Unity Catalog
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
   │  Databricks│ │  Functions,  │  │  storage,  │
   │  ADF, ADX, │ │  OpenAI,     │  │  Databricks│
   │  Event Hub)│ │  AI Search)  │  │  ADX, KV)  │
   └────────────┘ └──────────────┘  └────────────┘
```

#### Data Flow

```
Tenant Source Systems (ERP, CRM, Files, APIs, DBs)
       │
       ├── Batch ──► Azure Data Factory (parameterised per tenant)
       │
       ├── Streaming ──► Azure Event Hubs ──► Databricks Delta Live Tables
       │
       ▼
┌─────────────────────────────────────────┐
│       ADLS Gen2 (per-tenant container)  │
│  ┌────────┐  ┌────────┐  ┌────────┐     │
│  │ Bronze │─►│ Silver │─►│  Gold  │     │
│  │ (raw)  │  │(clean) │  │(curated│     │
│  └────────┘  └────────┘  └────────┘     │
│          Delta Lake format              │
└──────────────┬──────────────────────────┘
               │
       ┌───────┼──────────────┐
       ▼       ▼              ▼
  Power BI  Databricks     Databricks
  (via      SQL Warehouse  Spark / Notebooks
  connector) (ad-hoc SQL)  (data engineering / ML / MLflow)
```

---

<a name="option2-identity"></a>
### 4.3 Identity & Access 

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
                                  AKS / App Svc  Databricks      APIM Gateway
                                 (namespace      SQL Warehouse   (subscription
                                  scoped)        (Unity Catalog    per tenant)
                                                  scoped)
```

| Layer | Service | Tenant Isolation Mechanism |
|---|---|---|
| **Identity** | Microsoft Entra ID | B2B federation per tenant IdP; `tid` claim in tokens |
| **API Gateway** | Azure API Management | Subscription key per tenant; rate limiting; policies |
| **Compute** | AKS | Namespace isolation + network policies + resource quotas |
| **Storage** | ADLS Gen2 | Container-level or storage account-level isolation + ACLs |
| **Analytics** | Azure Databricks | Unity Catalog — catalog-per-tenant or schema-per-tenant; row/column-level permissions |
| **Real-Time** | Azure Data Explorer | Database-per-tenant within shared cluster |
| **Reports** | Power BI Embedded | Workspace-per-tenant + row-level security |
| **Secrets** | Azure Key Vault | Customer-managed keys (Enterprise tier) |
| **Config** | Azure App Configuration | Feature filters keyed by tenant ID |

[↑ Back to top](#)

---

<a name="option3"></a>
## 5. Option 3 – Portable (OSS + Azure SaaS)

<a name="option3-arch"></a>
### 5.1 Architecture Overview

This option builds the multi-tenant SaaS analytics platform using **portable, open-source technologies** combined with **Azure SaaS services that are cloud-agnostic** — services that can be consumed from any hyperscaler without architectural lock-in. The platform runs on Kubernetes as the universal compute layer, uses Azure Blob Storage (which exposes an S3-compatible API for portability), PostgreSQL for operational data, Apache Spark for analytics, and open-source AI tooling for LLM and RAG capabilities.

The core principle: **the compute, analytics, and orchestration layers can run on Azure, AWS, GCP, or on-premises** without re-architecture. Where Azure services are **SaaS / API-consumable from any cloud** (Power BI Embedded, GitHub Actions, Azure Blob Storage with S3-compatible endpoints), they are used because they add enterprise value without creating architectural lock-in. If the solution moves to AWS or GCP tomorrow, these SaaS services continue to work — only the infrastructure layer (K8s cluster, managed database) needs to be re-provisioned.

Multi-tenancy is achieved through **Kubernetes namespace isolation**, **database-per-tenant** (or schema-per-tenant), and **application-level tenant routing**.

```
┌──────────────────────────────────────────────────────────────────────┐
│              OPTION 3: PORTABLE (OSS + AZURE SaaS)                   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                  Presentation Layer                          │    │
│  │  Power BI Embedded (SaaS) │ Custom SaaS Portal (React/Vue)   │    │
│  │  Admin UI (SPA) │ API Gateway (Kong / Traefik / NGINX)       │    │
│  └────────────────────────────────┬─────────────────────────────┘    │
│                                   │                                  │
│  ┌────────────────────────────────▼─────────────────────────────┐    │
│  │             Application / API Layer (Kubernetes)             │    │
│  │  Microservices (Go / Python / Java / .NET)                   │    │
│  │  Tenant Config DB (PostgreSQL)                               │    │
│  │  Feature Flags (Unleash / LaunchDarkly / Flagsmith)          │    │
│  │  Service Mesh (Istio / Linkerd) — mTLS, traffic policies     │    │
│  └────────────────────────────────┬─────────────────────────────┘    │
│                                   │                                  │
│  ┌────────────────────────────────▼─────────────────────────────┐    │
│  │                  Data Platform (Open-Source)                 │    │
│  │                                                              │    │
│  │  ┌──────────────────────────────────────────────────────┐    │    │
│  │  │      Azure Blob Storage (S3-compatible API)          │    │    │
│  │  │  Tenant A Container │ Tenant B Container │ Tenant C  │    │    │
│  │  │  (container-per-tenant or prefix-per-tenant + RBAC)  │    │    │
│  │  └──────────────────────────┬───────────────────────────┘    │    │
│  │                             │                                │    │
│  │  ┌──────────────────────────▼───────────────────────────┐    │    │
│  │  │          Analytics Engine                            │    │    │
│  │  │  Apache Spark (Databricks / self-hosted / EMR)       │    │    │
│  │  │  Trino / Presto (federated SQL queries)              │    │    │
│  │  │  dbt (data transformation layer)                     │    │    │
│  │  └──────────────────────────────────────────────────────┘    │    │
│  │                                                              │    │
│  │  PostgreSQL / CockroachDB (operational DB, tenant metadata)  │    │
│  │  Apache Kafka / Redpanda (streaming ingestion)               │    │
│  │  Apache Airflow / Dagster (pipeline orchestration)           │    │
│  │  Apache Iceberg / Delta Lake (open table format)             │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                  AI Layer (LLM-Agnostic)                     │    │
│  │  LLM Gateway (LiteLLM / custom) │ OpenSearch (RAG)           │    │
│  │  LangChain / LlamaIndex │ vLLM / Ollama (self-hosted LLMs)   │    │
│  │  Apache Flink (real-time processing)                         │    │
│  └──────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Cross-Cutting Services:                                             │
│  Keycloak / Auth0 (identity) │ HashiCorp Vault (secrets)             │
│  Prometheus + Grafana (observability) │ OPA (policy engine)          │
│  Cert-Manager (TLS) │ GitHub Actions (CI/CD)                         │
└──────────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Role | Multi-Tenancy Model | Portable? |
|---|---|---|---|
| **Kubernetes** | Universal compute platform | Namespace-per-tenant with network policies + resource quotas | Yes — AKS / EKS / GKE / on-prem |
| **Azure Blob Storage** | Data lake (Bronze/Silver/Gold) | Container-per-tenant or prefix isolation + RBAC | Yes — S3-compatible API; swap to AWS S3 / GCS / MinIO by changing endpoint |
| **Apache Spark** | Distributed analytics engine | Spark jobs parameterised per tenant | Yes — Databricks / EMR / Dataproc / self-hosted |
| **Trino / Presto** | Federated SQL query engine | Catalog-per-tenant or schema isolation | Yes — runs on any K8s cluster |
| **dbt** | Data transformation (SQL-based) | dbt project per tenant or parameterised models | Yes — dbt Core is OSS |
| **Apache Iceberg / Delta Lake** | Open table format | Table-per-tenant with catalog-level isolation | Yes — open format, any engine |
| **PostgreSQL** | Operational / metadata DB | Database-per-tenant or schema-per-tenant | Yes — managed on any cloud or self-hosted |
| **Apache Kafka / Redpanda** | Streaming data ingestion | Topic-per-tenant with ACLs | Yes — Confluent Cloud, MSK, self-hosted |
| **Apache Airflow / Dagster** | Pipeline orchestration | DAGs parameterised per tenant | Yes — runs on any K8s cluster |
| **Power BI Embedded** | BI / reporting (SaaS) | Workspace-per-tenant + row-level security | Yes — SaaS API, consumable from any cloud; embed in any web app |
| **Kong / Traefik** | API gateway | Route-per-tenant, rate limiting, API key isolation | Yes — runs on any K8s cluster |
| **Keycloak / Auth0** | Identity & SSO | Realm-per-tenant or organisation isolation | Yes — Keycloak is OSS; Auth0 is SaaS |
| **HashiCorp Vault** | Secrets & CMK | Namespace-per-tenant, policy-based access | Yes — works on any infrastructure |
| **Prometheus + Grafana** | Observability | Label-based tenant isolation, multi-tenant Grafana orgs | Yes — CNCF standard |
| **OPA (Open Policy Agent)** | Policy enforcement | Policies scoped per tenant | Yes — CNCF standard |
| **GitHub Actions** | CI/CD | Workflows per tenant onboarding, deployment, and testing | Yes — platform-agnostic; works with any cloud provider |

### Technology Selection Rationale

| Concern | Azure-Native Answer | Portable Answer | Why Portable |
|---|---|---|---|
| **Compute** | Azure App Service / Functions | Kubernetes (K8s) | K8s runs identically on AKS, EKS, GKE, or bare metal |
| **Object Storage** | ADLS Gen2 | Azure Blob Storage (S3-compatible API) | S3-compatible endpoint means code works against AWS S3 / GCS / MinIO unchanged; Azure Blob is the primary target |
| **SQL Analytics** | Databricks SQL Warehouses | Trino / Presto / Spark SQL | No proprietary query engine lock-in |
| **Data Lake Format** | Parquet on ADLS | Apache Iceberg on Azure Blob Storage | Iceberg provides ACID, schema evolution, time travel — engine-independent; portable across clouds |
| **ETL / Pipelines** | Azure Data Factory | Apache Airflow / Dagster | OSS schedulers with portable operators |
| **Streaming** | Azure Event Hubs | Apache Kafka / Redpanda | Kafka protocol is supported by every cloud and on-prem |
| **BI / Reporting** | Power BI Embedded | Power BI Embedded (SaaS) | Power BI Embedded is a SaaS API — consumable from any cloud; same service regardless of where compute runs |
| **Identity** | Microsoft Entra ID | Keycloak / Auth0 | Keycloak supports OIDC/SAML federation on any infra |
| **Secrets** | Azure Key Vault | HashiCorp Vault | Works on every cloud and on-prem |
| **Monitoring** | Azure Monitor | Prometheus + Grafana + OpenTelemetry | CNCF-standard observability stack |
| **IaC** | Bicep (Azure-only) | Terraform / Pulumi / Crossplane | Multi-cloud IaC with a single codebase |
| **Policy** | Azure Policy | OPA (Open Policy Agent) | Portable admission control and policy |
| **CI/CD** | Azure DevOps | GitHub Actions | GitHub Actions is platform-agnostic — deploys to any cloud; GitHub is the industry-standard code platform |

### Tenancy Models (Portable)

| Layer | Isolation Options |
|---|---|
| **Compute (Kubernetes)** | Namespace-per-tenant *(default)* / Virtual cluster per tenant (vCluster) *(high isolation)* / Dedicated cluster per tenant *(max isolation)* |
| **Storage (Azure Blob)** | Container-per-tenant *(strongest)* / Prefix-per-tenant + RBAC policies *(moderate)* / Shared container + application-enforced ACL *(weakest)* |
| **Analytics (Spark / Trino)** | Catalog-per-tenant / Schema-per-tenant / Shared schema + row-level filtering |
| **Operational DB (PostgreSQL)** | Database-per-tenant / Schema-per-tenant / Shared schema + `tenant_id` column |
| **Streaming (Kafka)** | Topic-per-tenant with ACLs / Shared topic with tenant-keyed partitions |
| **BI (Power BI Embedded)** | Workspace-per-tenant + row-level security / Shared workspace + RLS |

---

<a name="option3-lz"></a>
### 5.2 Complete Landing Zone (Portable)

The landing zone follows the same hub-and-spoke model but uses portable networking and Kubernetes as the core compute layer, combined with Azure SaaS services (Power BI Embedded, GitHub Actions) that are consumable from any cloud. When deployed on Azure, managed services (AKS, Azure Database for PostgreSQL, Azure Blob) are used — but the compute and analytics layers are re-deployable on any cloud.

```
Cloud Provider (Azure / AWS / GCP / On-Prem)
└── Platform Account / Subscription
    │
    ├── Shared Infrastructure
    │   ├── Kubernetes Cluster (AKS / EKS / GKE)
    │   │   ├── Namespace: platform-services
    │   │   │   ├── API Gateway (Kong / Traefik)
    │   │   │   ├── Keycloak (identity provider)
    │   │   │   ├── HashiCorp Vault (secrets management)
    │   │   │   ├── Prometheus + Grafana (observability)
    │   │   │   ├── Cert-Manager (TLS automation)
    │   │   │   └── OPA Gatekeeper (policy enforcement)
    │   │   ├── Namespace: data-platform
    │   │   │   ├── Apache Airflow / Dagster (orchestration)
    │   │   │   ├── Trino (federated SQL)
    │   │   │   └── LLM Gateway (LiteLLM)
    │   │   └── Namespace: ingestion
    │   │       └── Kafka Connect / custom connectors
    │   │
    │   ├── Managed PostgreSQL (Azure DB for PostgreSQL / RDS / CloudSQL)
    │   │   ├── Database: platform_config (tenant metadata, feature flags)
    │   │   ├── Database: tenant_a
    │   │   ├── Database: tenant_b
    │   │   └── Database: tenant_n...
    │   │
    │   ├── Apache Kafka Cluster (Confluent / MSK / Event Hubs Kafka / self-hosted)
    │   │   ├── Topics: tenant-a-events, tenant-b-events, ...
    │   │   └── Schema Registry
    │   │
    │   └── CI/CD: GitHub Actions (platform-agnostic, deploys to any cloud)
    │
    ├── Data Lake: Azure Blob Storage (S3-compatible API)
    │   ├── Container: tenant-a (Bronze / Silver / Gold)
    │   ├── Container: tenant-b
    │   ├── Container: tenant-n...
    │   └── Container: platform-shared (templates, reference data)
    │   (Format: Apache Iceberg tables on Parquet files)
    │   (Portable: swap to AWS S3 / GCS / MinIO by changing endpoint config)
    │
    ├── Analytics Compute
    │   ├── Apache Spark Cluster (Databricks / EMR / Dataproc / K8s Spark Operator)
    │   └── dbt Project (transformations per tenant)
    │
    ├── BI / Reporting: Power BI Embedded (SaaS — consumable from any cloud)
    │   ├── Workspace per tenant
    │   └── Row-level security + embedded reports in SaaS portal
    │
    ├── AI Services
    │   ├── LLM Provider (OpenAI API / Azure OpenAI / Anthropic / self-hosted vLLM)
    │   ├── Vector Store / Search (OpenSearch / Weaviate / Qdrant / Milvus)
    │   └── Document Processing (Apache Tika / Unstructured.io)
    │
    └── Per-Tenant Dedicated Infrastructure (Enterprise tier only)
        ├── Dedicated K8s cluster or vCluster
        ├── Dedicated storage container + encryption keys
        ├── Dedicated PostgreSQL instance
        └── Private network / VPC peering
```

#### Network Architecture (Portable)

```
┌─────────────────────────────────────────────────────────────────┐
│                  NETWORK LAYER                                  │
│  (VPC / VNet — cloud-provider managed)                          │
│                                                                 │
│  ┌────────────────────────────────────────────────────┐         │
│  │  Ingress Layer                                     │         │
│  │  Cloud Load Balancer → Ingress Controller          │         │
│  │  (NGINX / Traefik) → Service Mesh (Istio/Linkerd)  │         │
│  └─────────────────────┬──────────────────────────────┘         │
│                        │                                        │
│  ┌─────────────────────▼──────────────────────────────────────┐ │
│  │  Kubernetes Cluster                                        │ │
│  │  ┌──────────────┐  ┌──────────────┐       ┌──────────────┐ │ │
│  │  │ Namespace:   │  │ Namespace:   │  ...  │ Namespace:   │ │ │
│  │  │ tenant-a     │  │ tenant-b     │       │ services     │ │ │
│  │  │ (app pods,   │  │ (app pods,   │       │ (Kong, Vault,│ │ │
│  │  │  network     │  │  network     │       │  Keycloak,   │ │ │
│  │  │  policies,   │  │  policies,   │       │  Prometheus) │ │ │
│  │  │  resource    │  │  resource    │       └──────────────┘ │ │ 
│  │  │  quotas)     │  │  quotas)     │                        │ │
│  │  └──────────────┘  └──────────────┘                        │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                 │
│  Private Subnets: PostgreSQL, Kafka, Object Storage             │
│  (accessible only from K8s cluster via private endpoints)       │
└─────────────────────────────────────────────────────────────────┘
```

#### Data Flow 

```
Tenant Source Systems (ERP, CRM, Files, APIs, DBs)
       │
       ▼
Apache Kafka (event streaming) / Airflow (batch ingestion)
       │
       ▼
┌──────────────────────────────────────────┐
│  Azure Blob Storage (S3-compatible API)  │
│  per-tenant container                    │
│  ┌────────┐  ┌────────┐  ┌────────┐      │
│  │ Bronze │─►│ Silver │─►│  Gold  │      │
│  │ (raw)  │  │(clean) │  │(curated│      │
│  └────────┘  └────────┘  └────────┘      │
│  Apache Iceberg tables (Parquet files)   │
└──────────────┬───────────────────────────┘
               │
       ┌───────┼──────────────┐
       ▼       ▼              ▼
  Power BI   Trino           Spark
  Embedded   (ad-hoc SQL)    (data engineering / ML)
  (SaaS)                      
```

---

<a name="option3-identity"></a>
### 5.3 Identity & Access 

The identity layer uses **Keycloak** (open-source) or **Auth0** (SaaS) as the identity platform. Keycloak supports multi-realm architecture — each tenant gets its own realm with federated SSO to their corporate IdP.

**How Power BI Embedded works with Keycloak (not Entra ID for users):**

Power BI Embedded only understands Entra ID — but end users never authenticate to Entra directly. The solution uses the **"App Owns Data"** embedding pattern (the standard ISV pattern):

1. **Users authenticate to the SaaS app via Keycloak** — Keycloak issues a JWT with tenant context (realm, roles, `tenant_id`)
2. **The app backend holds a service principal** registered in Microsoft Entra ID — this is a server-side credential stored in Vault, not user-facing
3. **The app backend calls the Power BI REST API** using the service principal to generate an **embed token** — the embed token includes `EffectiveIdentity` with the tenant's RLS roles and `tenant_id`
4. **The embedded PBI iframe in the browser** uses the embed token — the end user never sees an Entra login prompt

This means Keycloak handles all user-facing authentication, while a single Entra ID service principal handles the PBI backend plumbing. They coexist cleanly.

```
Tenant User
    │
    ▼
Tenant's Corporate IdP ──SAML 2.0 / OIDC──► Keycloak (Realm per Tenant)
                                                      │
                                               Keycloak issues JWT
                                        (with tenant context: realm, roles, tenant_id)
                                                            │
                                        ┌───────────────────┼──────────────────────┐
                                        ▼                   ▼                      ▼
                                  K8s Services           Trino / Spark          Kong Gateway
                                (namespace scoped)      (catalog scoped)       (route per tenant)                                                        
                                        │
                                        ▼
                                  App Backend
                                  (extracts tenant_id from Keycloak JWT)
                                        │
                                        ▼
                                  Power BI Embedded API
                                  (service principal in Entra ID generates embed token with
                                   EffectiveIdentity = tenant_id + RLS roles)
                                        │
                                        ▼
                                  Embedded PBI iframe
                                  (user sees tenant-scoped reports — no Entra login)
```

| Layer | Service | Tenant Isolation Mechanism |
|---|---|---|
| **Identity** | Keycloak / Auth0 | Realm-per-tenant (Keycloak) or organisation-per-tenant (Auth0); OIDC/SAML federation with tenant IdPs |
| **API Gateway** | Kong / Traefik | Route-per-tenant; API key + rate limiting; JWT validation plugin |
| **Compute** | Kubernetes | Namespace isolation + NetworkPolicy + ResourceQuota + Istio AuthorizationPolicy |
| **Storage** | Azure Blob Storage | Container-per-tenant with RBAC policies; S3-compatible API for portability |
| **Analytics** | Trino / Spark | Catalog-per-tenant or schema-per-tenant; Spark job submitted with tenant context |
| **BI / Reporting** | Power BI Embedded (SaaS) | "App Owns Data" pattern — Entra ID service principal (server-side) generates embed tokens with `EffectiveIdentity` per tenant; users authenticate via Keycloak, not Entra |
| **Secrets** | HashiCorp Vault | Namespace-per-tenant; policy-based access; dynamic secrets for databases |
| **Config** | Unleash / Flagsmith | Feature flags scoped by tenant ID |
| **Policy** | OPA Gatekeeper | Admission control policies enforce namespace isolation, resource limits, image policies |

#### Kubernetes Multi-Tenancy Model

Kubernetes is the compute foundation. Multi-tenancy is enforced at multiple layers:

| K8s Mechanism | Purpose |
|---|---|
| **Namespace** | Logical isolation boundary per tenant; all tenant pods, services, and configs live within their namespace |
| **NetworkPolicy** | Controls pod-to-pod traffic; tenants cannot reach other tenants' pods |
| **ResourceQuota** | Limits CPU, memory, storage per tenant namespace — prevents noisy neighbors |
| **LimitRange** | Sets default and maximum resource limits per pod within a tenant namespace |
| **Istio AuthorizationPolicy** | Service mesh policies — mTLS between services, JWT-based auth at the mesh level |
| **OPA Gatekeeper** | Admission controller enforcing policies (e.g., tenants can only pull approved images, cannot use hostNetwork) |
| **vCluster** (Enterprise tier) | Virtual cluster inside the physical cluster — strongest namespace-level isolation without a dedicated cluster |


---

<a name="regional-lz"></a>
## 7. Regional Landing Zone Strategy

Multi-tenant SaaS platforms must handle two fundamentally different tenant profiles from a regional perspective:

1. **Co-located tenants** — customers with no data-residency requirements who can be onboarded into the platform's primary region alongside existing tenants.
2. **Data-residency tenants** — customers (often in regulated industries or jurisdictions like the EU, UK, Australia, or Middle East) who require all data, compute, and processing to remain within a specific geographic region.

This section defines the Landing Zone strategy and deployment approach for both scenarios. The patterns apply to **all three options** — Option 1 (Fabric), Option 2 (Azure PaaS), and Option 3 (Portable). For Option 3, the regional stamp uses Kubernetes clusters and OSS services in the target region/cloud instead of Azure-specific resources, while Azure SaaS services (Power BI Embedded, GitHub Actions) continue to work from the target region.

---

<a name="colocated-tenants"></a>
### 6.1 Co-Located Tenants (Same Region — No Data Residency Constraints)

#### Strategy

When a new tenant has **no data-residency requirements**, they are onboarded into the platform's **primary regional stamp** — the existing Landing Zone where the shared platform services, hub network, and analytics infrastructure already run.

This is the **default, lowest-cost, and fastest onboarding path**. No new regional infrastructure is deployed. The tenant's resources are provisioned as additional logical partitions within the existing stamp.

---

### What is the Deployment Stamp Pattern?

The **Deployment Stamp** (also called **Scale Unit** or **Service Unit**) pattern deploys multiple independent copies of your entire application infrastructure — each copy is called a **stamp**. Every stamp is a self-contained, fully functional unit that can serve one or more tenants independently.

Think of it like a franchise model: each stamp is an identical "restaurant" deployed from the same blueprint, but operating independently in its own location.

**Why it matters for multi-tenant SaaS:**

- **Horizontal scale** — when a single stamp reaches capacity (compute, storage, or connection limits), you deploy a new stamp rather than scaling up the existing one
- **Blast radius isolation** — a failure in Stamp 1 does not affect tenants in Stamp 2
- **Data residency** — deploy regional stamps to keep tenant data within a specific geography (e.g., EU stamp, AU stamp)
- **Tenant tiering** — Standard tenants share a stamp; Enterprise tenants get a dedicated stamp
- **Consistent deployments** — every stamp is provisioned from the same IaC template (Bicep / Terraform), parameterised by region and stamp ID

**How it works:**

```
                    Global Router
                  (Front Door / APIM)
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
      ┌─────────┐ ┌─────────┐ ┌─────────┐
      │ Stamp 1 │ │ Stamp 2 │ │ Stamp 3 │
      │ (AU)    │ │ (AU)    │ │ (EU)    │
      │         │ │         │ │         │
      │ Tenants │ │ Tenants │ │ Tenants │
      │ A, B, C │ │ D, E, F │ │ G, H    │
      └─────────┘ └─────────┘ └─────────┘
      Each stamp is identical infrastructure
      deployed from the same IaC template
```

A **Tenant Config DB** (e.g., Cosmos DB with geo-replication) maps each tenant to its assigned stamp. The global router looks up this mapping and directs traffic accordingly.

**Reference:** [Deployment Stamp pattern — Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp)

#### Landing Zone Model

```
┌────────────────────────────────────────────────────────────────────────┐
│              PRIMARY REGION (e.g. Australia East)                      │
│                                                                        │
│  Platform Subscription (shared)                                        │
│  ├── Hub VNet (Firewall, Bastion, Private DNS, APIM)                   │
│  ├── Azure Monitor / Log Analytics                                     │
│  ├── Microsoft Purview                                                 │
│  ├── Azure Key Vault (platform-level)                                  │
│  ├── Tenant Config DB (Azure SQL / Cosmos DB)                          │
│  ├── Azure App Configuration                                           │
│  └── CI/CD Pipelines (Azure DevOps / GitHub Actions)                   │
│                                                                        │
│  Analytics / Fabric Subscription                                       │
│  ├── [Fabric] Fabric Capacity → Workspace per tenant                   │
│  │   ├── Workspace: Tenant A (existing)                                │
│  │   ├── Workspace: Tenant B (existing)                                │
│  │   └── Workspace: Tenant N (NEW — onboarded here)  ◄── new tenant    │
│  │                                                                     │
│  ├── [Non-Fabric] ADLS Gen2 → Container per tenant                     │
│  │   ├── Container: tenant-a (existing)                                │
│  │   ├── Container: tenant-b (existing)                                │
│  │   └── Container: tenant-n (NEW)  ◄── new tenant                     │
│  │                                                                     │
│  └── [Non Fabric] Databricks / ADX — Unity Catalog schema or DB/tenant │
│                                                                        │
│  Application Subscription                                              │
│  ├── App Service / AKS (new namespace / slot for tenant)               │
│  ├── Azure OpenAI (shared instance, tenant-scoped requests)            │
│  ├── Azure AI Search (new index or security-trimmed shared index)      │
│  └── Power BI Embedded (new workspace for tenant)                      │
└────────────────────────────────────────────────────────────────────────┘
```

#### Onboarding Steps (Co-Located Tenant)

```
New Customer Request (no data residency requirement)
       │
       ▼
1. Validate tenant tier (Standard / Professional / Enterprise)
       │
       ▼
2. Provision tenant resources WITHIN the existing regional stamp:
       │
       ├──► [Fabric] Create Fabric Workspace in existing capacity
       │    OR
       ├──► [Non-Fabric] Create ADLS container + Databricks schema / ADX DB
       │
       ├──► Configure RBAC / ACLs scoped to the tenant
       ├──► Deploy parameterised data pipelines (Fabric Pipelines / ADF)
       ├──► Create Power BI workspace + deploy report templates
       ├──► Create AI Search index (or add tenant to shared index)
       ├──► Register tenant in Config DB (region = primary)
       ├──► Set feature flags in Azure App Configuration
       ├──► Create APIM subscription (API key + rate limit policy)
       └──► Federate tenant IdP in Entra ID (B2B)
       │
       ▼
3. Smoke test — validate data flow, API access, BI reports
       │
       ▼
4. Tenant is live (same region as all other co-located tenants)
```

#### Key Design Points

| Aspect | Approach |
|---|---|
| **Infrastructure** | No new infrastructure deployed — reuse the existing regional stamp |
| **Compute** | Shared Fabric capacity / shared AKS cluster / shared App Service plan |
| **Storage** | New logical partition (workspace, container, schema) in existing storage |
| **Networking** | Tenant traffic flows through the existing Hub VNet and APIM gateway |
| **Cost** | Marginal cost only — additional storage, compute units consumed, API calls |
| **Onboarding time** | Minutes to hours (fully automated via CI/CD + IaC) |
| **Scaling** | Vertical: scale up Fabric capacity / AKS node pool / App Service plan. Horizontal: use the Deployment Stamp pattern when a single stamp reaches capacity limits |

#### When a Stamp Reaches Capacity

Even co-located tenants eventually hit the limits of a single regional deployment (Fabric capacity ceiling, AKS node limits, APIM throughput, storage IOPS). When this happens, deploy a **new stamp in the same region**:

```
PRIMARY REGION (e.g. Australia East)
├── Stamp 1 (Tenants A–M)   ◄── existing
├── Stamp 2 (Tenants N–Z)   ◄── new stamp, same region
└── Shared Platform Services (Hub VNet, Monitor, Purview, Config DB)
```

The **Deployment Stamp pattern** treats each stamp as an independent, self-contained unit of scale. A **tenant router** (in APIM or the application layer) directs traffic to the correct stamp based on the tenant's registration in the Config DB.

---

<a name="data-residency-tenants"></a>
### 6.2 Data-Residency Tenants (Sovereign Region Deployment)

#### Strategy

When a tenant has **data-residency requirements** (e.g., all data must remain within the EU, or within a specific country), the platform deploys a **regional stamp** in the required Azure region. Everything — compute, storage, analytics, AI, and networking — is provisioned in that region so that tenant data never leaves the geographic boundary.

This is the **higher-cost, higher-complexity path**, but it is non-negotiable for regulated industries (finance, healthcare, government) and jurisdictions with strict data-sovereignty laws (GDPR, PDPA, LGPD, etc.).

#### Landing Zone Model

```
┌────────────────────────────────────────────────────────────────────────┐
│                      GLOBAL MANAGEMENT PLANE                           │
│                                                                        │
│  Management Group Hierarchy                                            │
│  ├── SaaS Platform MG                                                  │
│  │   ├── Global Shared Services (identity, DNS, CI/CD, governance)     │
│  │   ├── Primary Region Stamp(s) (e.g. Australia East)                 │
│  │   ├── EU Region Stamp (e.g. West Europe / France Central)           │
│  │   ├── UK Region Stamp (e.g. UK South)                               │
│  │   └── [Future] APAC / MEA / Americas stamps                         │
│  └── Azure Policies (applied globally, enforce regional compliance)    │
└────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────┐     ┌────────────────────────────┐
│  PRIMARY STAMP             │     │  EU STAMP                  │
│  (Australia East)          │     │  (West Europe)             │
│                            │     │                            │
│  Hub VNet                  │     │  Hub VNet                  │
│  ├── Firewall              │     │  ├── Firewall              │
│  ├── Bastion               │     │  ├── Bastion               │
│  ├── APIM                  │     │  ├── APIM                  │
│  └── Private DNS           │     │  └── Private DNS           │
│                            │     │                            │
│  Analytics / Data          │     │  Analytics / Data          │
│  ├── Fabric Capacity       │     │  ├── Fabric Capacity       │
│  │   (or ADLS + Databricks)│     │  │   (or ADLS + Databricks)│
│  ├── Tenant Workspaces     │     │  ├── EU Tenant Workspaces  │
│  ├── AI Search indexes     │     │  ├──  AI Search indexes    │
│  └── Power BI Embedded     │     │  └── Power BI Embedded     │
│                            │     │                            │
│  Application               │     │  Application               │
│  ├── App Service / AKS     │     │  ├── App Service / AKS     │
│  └── Azure OpenAI          │     │  └── Azure OpenAI          │
│                            │     │                            │
│  Local Observability       │     │  Local Observability       │
│  ├── Log Analytics WS      │     │  ├── Log Analytics WS      │
│  └── Azure Monitor         │     │  └── Azure Monitor         │
└────────────────────────────┘     └────────────────────────────┘
          │                                    │
          └─────────┬──────────────────────────┘
                    ▼
        ┌─────────────────────┐
        │  GLOBAL SERVICES    │
        │  (not region-bound) │
        │                     │
        │  Microsoft Entra ID │
        │  Azure Front Door   │
        │  (global routing)   │
        │  Global Config DB   │
        │  (geo-replicated    │
        │   Cosmos DB)        │
        │  CI/CD Pipelines    │
        │  Microsoft Purview  │
        │  Azure Policy       │
        │  Defender for Cloud │
        └─────────────────────┘
```

#### What Gets Deployed Per Regional Stamp

Each regional stamp is a **self-contained, fully functional replica** of the platform within that Azure region. The following resources are deployed per stamp:

| Layer | Resources Deployed in the Regional Stamp |
|---|---|
| **Networking** | Hub VNet, Azure Firewall, Bastion, Private DNS Zones, VNet peering (if connected to global hub) or isolated |
| **API Gateway** | Azure API Management instance (or APIM multi-region deployment) |
| **Analytics (Fabric)** | Fabric Capacity (F SKU) provisioned in the target region + per-tenant workspaces |
| **Analytics (Non-Fabric)** | ADLS Gen2 storage account, Azure Databricks workspace, ADX cluster — all in the target region |
| **Application** | App Service / AKS cluster / Azure Functions — deployed in the target region |
| **AI Services** | Azure OpenAI (regional deployment), Azure AI Search (regional index), Document Intelligence |
| **BI** | Power BI Embedded capacity in the target region |
| **Secrets** | Azure Key Vault (regional, for stamp-level and tenant-level secrets) |
| **Observability** | Log Analytics workspace + Azure Monitor in the target region (logs stay local) |
| **Data Pipelines** | Fabric Pipelines or ADF (regional instance, parameterised per tenant) |

#### What Remains Global (Shared Across All Stamps)

| Service | Why Global |
|---|---|
| **Microsoft Entra ID** | Identity is global by design — B2B federation, tenant IdPs, and RBAC are managed centrally |
| **Azure Front Door** | Global load balancer and traffic router — directs tenants to their regional stamp based on routing rules |
| **Tenant Config DB (Cosmos DB)** | Geo-replicated database storing tenant metadata, including the tenant's assigned region/stamp. Read replicas in each region for low-latency lookups |
| **CI/CD Pipelines** | Deployment pipelines are global — they target specific regional stamps via parameterised deployments |
| **Microsoft Purview** | Data governance catalog spans all regions — provides cross-region lineage and classification |
| **Azure Policy** | Policies are applied at the Management Group level — enforced across all regional subscriptions |
| **Microsoft Defender for Cloud** | Security posture management is global — aggregates findings across all stamps |

#### Traffic Routing — How Tenants Reach Their Regional Stamp

```
Tenant User (EU)
     │
     ▼
Azure Front Door (global entry point)
     │
     ├── Routing Rule: tenant-id → lookup Config DB
     │   Config DB returns: region = "westeurope", stamp = "eu-stamp-1"
     │
     ▼
Route to EU Stamp APIM endpoint
     │
     ▼
EU Stamp: APIM → App Service / AKS → Fabric / Databricks (all in West Europe)
     │
     ▼
Data never leaves the EU region
```

**Routing strategies:**
- **Custom domain per region** — `eu.platform.com`, `au.platform.com` — simplest, but requires DNS management
- **Header-based routing** — tenant ID in the request header, Front Door routes based on lookup
- **Path-based routing** — `/eu/api/...`, `/au/api/...` — works but exposes region in URL
- **Config DB lookup** — most flexible; Front Door or APIM policy queries the tenant Config DB to resolve the target stamp

#### Onboarding Steps (Data-Residency Tenant)

```
New Customer Request (data residency = EU)
       │
       ▼
1. Check if an EU regional stamp already exists
       │
       ├── YES ──► Proceed to step 3 (provision tenant in existing EU stamp)
       │
       ├── NO ──► Step 2: Deploy a new EU regional stamp
       │            │
       │            ├──► Deploy Hub VNet + Firewall + Bastion in EU region
       │            ├──► Deploy APIM instance (or extend multi-region APIM)
       │            ├──► Deploy Fabric Capacity in EU (or ADLS + Databricks + ADX)
       │            ├──► Deploy App Service / AKS in EU region
       │            ├──► Deploy Azure OpenAI + AI Search in EU region
       │            ├──► Deploy Power BI Embedded capacity in EU
       │            ├──► Deploy Log Analytics workspace in EU
       │            ├──► Deploy Azure Key Vault in EU
       │            ├──► Configure Azure Front Door routing for EU stamp
       │            ├──► Configure Cosmos DB read replica in EU (for Config DB)
       │            ├──► Apply Azure Policies (data residency enforcement)
       │            └──► Validate stamp health + connectivity
       │
       ▼
3. Provision tenant within the EU stamp:
       │
       ├──► [Fabric] Create Fabric Workspace in EU capacity
       │    OR
       ├──► [Non-Fabric] Create ADLS container + Databricks schema in EU
       │
       ├──► Configure RBAC / ACLs
       ├──► Deploy parameterised data pipelines (EU region)
       ├──► Create Power BI workspace + reports
       ├──► Create AI Search index in EU
       ├──► Register tenant in Config DB (region = "westeurope", stamp = "eu-stamp-1")
       ├──► Set feature flags in Azure App Configuration
       ├──► Create APIM subscription on EU APIM instance
       └──► Federate tenant IdP in Entra ID (B2B)
       │
       ▼
4. Configure Azure Policy to enforce data residency:
       │
       ├──► Policy: "Allowed Locations" = westeurope / northeurope only
       ├──► Policy: Deny resource creation outside EU for this subscription
       └──► Policy: Enforce encryption (CMK stored in EU Key Vault)
       │
       ▼
5. Smoke test — validate data flow stays within EU, API routing, BI reports
       │
       ▼
6. Tenant is live (all data and compute in EU)
```

#### Azure Policy for Data Residency Enforcement

Azure Policy is the **enforcement mechanism** that guarantees data does not leave the designated region — even if a misconfiguration or human error occurs.

| Policy | Purpose |
|---|---|
| **Allowed Locations** | Restricts resource creation to the tenant's designated Azure region(s) only |
| **Allowed Locations for Resource Groups** | Ensures even resource group metadata stays in the correct geography |
| **Deny Public Network Access** | Forces private endpoints for all data services — prevents data exposure over the internet |
| **Require Encryption (CMK)** | Mandates customer-managed keys stored in a regional Key Vault |
| **Deny Cross-Region Replication** | Prevents storage or database replication to regions outside the compliance boundary |
| **Audit Diagnostic Settings** | Ensures all logs and metrics are stored in the regional Log Analytics workspace (not a cross-region one) |

#### Operational Considerations for Multi-Region

| Concern | Strategy |
|---|---|
| **Stamp consistency** | Use a single IaC template (Bicep / Terraform) parameterised by region. Every stamp is identical except for `location` and region-specific SKU availability |
| **Deployment pipeline** | CI/CD pipeline takes `region` and `stamp-id` as parameters. Same pipeline deploys to any stamp |
| **Monitoring** | Each stamp has its own Log Analytics workspace (data stays local). A **global Azure Monitor Workbook** or **Grafana dashboard** aggregates metrics across stamps for the platform team — using cross-workspace queries (metadata only, no raw data leaves the region) |
| **Disaster Recovery** | Within-region redundancy (availability zones). Cross-region DR for data-residency tenants is constrained to the **same compliance boundary** (e.g., West Europe ↔ North Europe for EU tenants) |
| **Cost** | Each regional stamp incurs base infrastructure cost. Justify stamp creation when tenant revenue covers the incremental cost. Use reserved instances and savings plans per region |
| **Service availability** | Not all Azure services are available in every region. Validate service availability in the target region **before** committing to a stamp deployment (e.g., Azure OpenAI regional availability, Fabric capacity regions) |
| **Tenant migration** | If a tenant's data-residency requirements change (e.g., they expand to require EU), a migration path must exist: export data → re-ingest into the new regional stamp → update Config DB routing → decommission old tenant resources |

[↑ Back to top](#)

---

<a name="references"></a>
## 8. Reference Architectures & Resources

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
| Azure Databricks modern analytics architecture | [Link](https://learn.microsoft.com/en-us/azure/architecture/solution-ideas/articles/azure-databricks-modern-analytics-architecture) |
| Azure Databricks documentation | [Link](https://learn.microsoft.com/en-us/azure/databricks/) |
| Azure Databricks Unity Catalog | [Link](https://learn.microsoft.com/en-us/azure/databricks/data-governance/unity-catalog/) |
| Azure Databricks SQL Warehouses | [Link](https://learn.microsoft.com/en-us/azure/databricks/sql/) |
| Delta Live Tables | [Link](https://learn.microsoft.com/en-us/azure/databricks/delta-live-tables/) |
| Delta Lake documentation | [Link](https://learn.microsoft.com/en-us/azure/databricks/delta/) |
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
| Deployment Stamp pattern | [Link](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) |
| Geode pattern (geo-distributed) | [Link](https://learn.microsoft.com/en-us/azure/architecture/patterns/geodes) |
| Azure Front Door overview | [Link](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview) |
| Azure regions & data residency | [Link](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/azure-setup-guide/regions) |
| Azure Policy built-in: Allowed Locations | [Link](https://learn.microsoft.com/en-us/azure/governance/policy/samples/built-in-policies#general) |
| Cosmos DB multi-region distribution | [Link](https://learn.microsoft.com/en-us/azure/cosmos-db/distribute-data-globally) |
| Fabric Multi-Geo (data residency) | [Link](https://learn.microsoft.com/en-us/fabric/admin/service-admin-premium-multi-geo) |

### Identity References

| Resource | Link |
|---|---|
| Microsoft Entra External ID (B2B federation) | [Link](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b) |
| Federation with external IdPs via SAML | [Link](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/add-application-portal-setup-sso) |
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
| **Azure AI Foundry overview** | [Link](https://learn.microsoft.com/en-us/azure/ai-foundry/what-is-azure-ai-foundry) |
| **Azure AI Foundry Agent Service** | [Link](https://learn.microsoft.com/en-us/azure/ai-services/agents/overview) |
| **Semantic Kernel overview** | [Link](https://learn.microsoft.com/en-us/semantic-kernel/overview/) |
| **Semantic Kernel — Agents** | [Link](https://learn.microsoft.com/en-us/semantic-kernel/frameworks/agent/) |
| **Azure AI Document Intelligence** | [Link](https://learn.microsoft.com/en-us/azure/ai-services/document-intelligence/overview) |
| **Azure AI Content Safety** | [Link](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview) |
| **Prompt Flow in Azure AI Foundry** | [Link](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/prompt-flow) |
| **Azure OpenAI models (GPT-4o, GPT-4.1)** | [Link](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/models) |
| **Responsible AI in Azure** | [Link](https://learn.microsoft.com/en-us/azure/ai-services/responsible-use-of-ai-overview) |
| **Azure Machine Learning** | [Link](https://learn.microsoft.com/en-us/azure/machine-learning/overview-what-is-azure-machine-learning) |
| **RAG solution architecture** | [Link](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/architecture/baseline-openai-e2e-chat) |

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
