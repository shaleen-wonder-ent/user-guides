# E360 Next-Gen Platform – Architecture Discussion Guide

---

## Table of Contents
0. [E360 Current State & Proposed Vision](#current-state)
1. [Discovery Questions](#discovery)
2. [Architecture Option 1 – Azure Native (Microsoft Fabric & PaaS)](#option1)
3. [Architecture Option 2 – Cloud-Agnostic (Containers on VMs)](#option2)
4. [Comparative Trade-off Discussion](#tradeoffs)
5. [Contoso Requirements Alignment & Open Questions](#Contoso)
6. [Multi-Tenant Landing Zone Architecture](#landing-zone)
7. [AI & Fabric Story](#ai-fabric)
8. [Microsoft Reference Architectures Closest to E360](#closest-refs)

---
> ### Microsoft's Recommendation
> **Azure Native (Microsoft Fabric + Azure PaaS) is the recommended architecture for E360.** Fabric delivers the fastest path to a production-grade, multi-tenant analytics SaaS platform — with built-in AI, governance, security certifications, and enterprise SLAs that would take years to replicate on a self-managed stack. For the ~80% of Contoso's client base that can use Azure, this approach maximises value delivery while minimising operational risk. For the remaining ~20% requiring non-Azure deployment, **Azure Arc** provides a unified management plane to extend Azure services to any infrastructure. The document presents both options for completeness, but **Option 1 (Azure Native) is the strongly recommended path**.

---

<a name="current-state"></a>

## 0. E360 Current State & Proposed Vision

### 0.1 Current Architecture (E360 on other platform)

**Key Observations:**
- **Single-tenant monolith** – All clients share one other platform storage environment with only RBAC for segregation
- **Okta** is the identity provider; no Azure AD / Entra ID today
- **Power BI** already used for portfolio drill-down views (a bridge to Option 1)
- **MuleSoft** handles integration – must be replaced or abstracted in migration
- **No IaC / GitOps** – tenant onboarding is manual
- **Gen AI features** (Playground, Ops Designer, CXO Insights) exist but appear ChatGPT-based, not Azure OpenAI

### 0.2 Proposed Architecture (E360 Plus – AI Enabled)

Contoso's own architects have proposed this layered design:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Layer 6: Presentation & UX                                              │
│  Power BI Premium/Embedded │ Persona Dashboards │ Config UI              │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 5: API & Gateway                                                  │
│  API Gateway │ GraphQL/REST API │ Traffic Mgmt & Auth                    │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 8*: Agentic Intelligence Layer (DE-SCOPED for MVP)                │
│  Azure OpenAI (GPT) │ Semantic Kernel │ Agent framework                  │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 4: Compute & Application Logic                                    │
│  Transformation Engine │ dbt Core │ Composite Scoring Engine             │
│  Workflow Workbench │ Analytics & Correlation                            │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 3: Semantic Configuration Layer                                   │
│  UDAL │ Taxonomy Designer │ Metadata │ Abstraction                       │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 2: Lakehouse-Centric Storage (Iceberg & ClickHouse)               │
│  Iceberg REST Catalog / Hive Metastore                                   │
│  ┌────────┐  dbt  ┌────────┐  dbt  ┌────────┐   ┌──────────────┐         │
│  │ Bronze │ ────► │ Silver │ ────► │  Gold  │──►│Serving Accel.│         │
│  └────────┘       └────────┘       └────────┘   └──────────────┘         │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 1: Data & Integration Layer                                       │
│  Data Ingestion │ Azure Data Factory │ Connectors │ Key Vault            │
├──────────────────────────────────────────────────────────────────────────┤
│  Layer 0: Cloud-Native Infrastructure (Azure PaaS)                       │
│  Azure PaaS │ ADLS Gen2 │ Cloud-Native Infra │ ClickHouse Cloud │ VNet   │
└──────────────────────────────────────────────────────────────────────────┘

Cross-cutting (right side):
  Governance, Security & Observability
  Microsoft Entra ID (RBAC/RLS)
  Azure Monitor & Log Analytics
  dbt Lineage & Data Quality Tests
```

**Key Observations from Contoso's Proposal:**
- **Already hybrid by design:** Uses Azure-native services (ADF, ADLS Gen2, Key Vault, Entra ID) alongside open-source tools (dbt Core, Iceberg, ClickHouse)
- **Lakehouse medallion pattern:** Bronze → Silver → Gold with dbt as the transformation engine
- **Semantic Configuration Layer (UDAL):** Critical for multi-tenancy – this is where per-client metadata, taxonomies, and abstraction live
- **Agentic Intelligence Layer de-scoped for MVP:** AI features are envisioned but not v1 priority – this is our opportunity to tell the Fabric AI story
- **Config UI in Layer 6:** Confirms the need for a self-service admin portal for tenant onboarding
- **No explicit multi-tenancy boundary** shown – the architecture is application-level; tenant isolation must be designed into every layer

---

<a name="discovery"></a>
## 1. Discovery Questions

### 1.1 Questions for Azure Migration & Multi-Tenancy (Section 1)

**Current State & Migration Scope:**

1. What is the current data volume per client, and total across all clients? (GB/TB – needed for Fabric F SKU sizing)
2. How many concurrent clients are on the platform today, and what is the 12-month / 24-month target?
3. What other platform services are currently in use beyond S3? (Lambda, RDS, Glue, Redshift, SageMaker, etc.)
4. Is the current codebase Python-based? What frameworks (Flask, FastAPI, Django)?
5. How are data pipelines currently orchestrated? (Airflow, Step Functions, custom cron?)
6. What is the current deployment model? (EC2 instances, ECS containers, or bare Python scripts?)
7. Is there an existing CI/CD pipeline? (GitHub Actions, Jenkins, CodePipeline?)
8. What data domains exist per client? (The diagram shows Finance, Operations, People, Portfolio, Masters, Misc – is this consistent across all clients or do some clients have subsets?)

**Multi-Tenancy Specifics:**

9. How is tenant isolation currently enforced? (Just RBAC on shared S3 buckets? Separate S3 prefixes? Shared DB with tenant_id column?)
10. Are there any clients today that require **physical data isolation** (separate storage, separate compute)?
11. Do clients bring their own data schemas / custom fields, or is the schema identical across all clients?
12. How does the UDAL (Unified Data Abstraction Layer) in the proposed design handle per-tenant taxonomy differences?
13. What is the Composite Scoring Engine? Is it a custom-built scoring algorithm per client, or a shared model with client-parameterised weights?
14. Is the Benchmarking module cross-tenant (comparing Client A vs. anonymised industry)? If yes, this complicates strict data isolation.

**Identity & Access:**

15. Is Okta the sole IdP, or do some clients use their own IdP (Azure AD, Ping, etc.)?
16. What RBAC roles exist today? (The diagram shows Ops Roles, Functional & Exception Role, Account SPOC, DEFAULT – are these consistent across clients?)
17. Is there a service account / API key model for system-to-system integrations?

**Compliance & Governance:**

18. What compliance certifications does E360 currently hold? (SOC2, ISO 27001, HIPAA?)
19. Are there data residency requirements per client? (e.g., EU data must stay in EU)
20. Is there an existing data catalogue or lineage tool, or is this net-new?

### 1.2 Questions for Hyperscaler Agnosticism (Section 2)

**Requirements Clarification:**

1. What percentage of current/target clients cannot use Azure? (The ~20% figure – is this validated?)
2. For non-Azure clients, what are the specific constraints? (Contractual other platform-only? On-prem mandate? GCP preference?)
3. Is "cloud agnostic" a contractual/RFP requirement from Contoso's clients, or an internal strategic preference?
4. Does "agnostic" mean the **same binary** runs everywhere, or is it acceptable to have cloud-specific adapters behind a common interface?
5. Are there clients who need E360 deployed **in the client's own cloud subscription** (vs. Contoso-managed)?

**Technical Depth:**

6. The proposed architecture uses Azure Data Factory, ADLS Gen2, Key Vault, and Entra ID – these are Azure-specific. What is the abstraction strategy for non-Azure deployments? (e.g., ADF → Airflow, ADLS → S3, Key Vault → HashiCorp Vault, Entra → Keycloak?)
7. Is ClickHouse Cloud the confirmed analytical DB? (It is cloud-agnostic, which supports Option 2)
8. dbt Core is open-source and portable – is the team committed to dbt, or considering dbt Cloud (which adds SaaS dependency)?
9. For the Iceberg table format: is there a dependency on a specific catalog implementation (other platform Glue Catalog, Hive Metastore, Nessie)?
10. Has the team evaluated **Azure Arc** as a way to get Azure management plane portability without full re-architecture?

**Operational Readiness:**

11. Does Contoso have a Kubernetes operations team today, or would this need to be built/hired?
12. What is the experience level with Infrastructure-as-Code? (Terraform, Bicep, Pulumi?)
13. Is there a preference for GitOps tooling? (ArgoCD, Flux, or custom?)
14. What is the target SLA for the platform? (99.9%? 99.95%? Per-client or platform-wide?)

---

<a name="option1"></a>
## 2. Architecture Option 1 – Azure Native Solution (Microsoft Fabric & PaaS)

### High-Level Architecture (Option 1 mapped to E360 Layers)

This architecture maps Contoso's proposed layers onto Azure-native services:

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
│  │                    OneLake (Single Storage)           │      │
│  │   Tenant A Data │ Tenant B Data │ Tenant C Data       │      │
│  │  (Folder-level isolation + RBAC + sensitivity labels) │      │
│  └───────────────────────────────────────────────────────┘      │
│                                                                 │
│  Supporting PaaS:                                               │
│  Azure API Management │ Azure AD B2B/B2C │ Azure Key Vault      │
│  Azure Monitor / Purview │ Azure Data Factory │ Event Hub       │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Role | Multi-Tenancy Model |
|---|---|---|
| **Microsoft Fabric Workspaces** | Isolated environment per client | One workspace per tenant |
| **OneLake** | Unified storage layer | Folder-level isolation + RBAC |
| **Fabric Capacity (F SKU)** | Shared compute pool | Capacity-level or dedicated per tier |
| **Power BI Embedded / Reports** | Client-facing analytics UI | Row-level security (RLS) per tenant |
| **Azure API Management (APIM)** | Unified API gateway | Subscription-per-tenant, rate limiting |
| **Microsoft Entra ID** | Identity & SSO | B2B federation with Contoso/Okta |
| **Azure Key Vault** | Secrets & CMK | Customer-managed keys per tenant |
| **Microsoft Purview** | Data governance & lineage | Cross-tenant data catalog |
| **Azure Data Factory** | Data pipeline orchestration | Parameterised pipelines per tenant |
| **Azure Monitor + Log Analytics** | Observability | Workspace-per-tenant or filtered |

### Tenancy Models in Microsoft Fabric

Microsoft supports three main tenancy models in Fabric:

1. **Separate Fabric Workspace per client** *(Recommended for E360)*
   - Strong data isolation boundary
   - Independent RBAC, sensitivity labels, and workspace identities
   - Easiest to onboard/offboard clients
   - Cost: all tenants share one Fabric capacity (F SKU)

2. **Separate Fabric Capacity per client** *(Premium tier / regulated clients)*
   - Full compute and billing isolation
   - Required if clients need SLA guarantees or cost chargebacks
   - Higher cost but maximum isolation

3. **Shared workspace with RLS** *(Not recommended for E360)*
   - Row-level security in semantic models to filter by tenant
   - Lower isolation, complex to maintain, security risk

### Microsoft Reference Links – Option 1

| Resource |
|---|
| [Microsoft Fabric Overview](https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview) |
| [Fabric Workspaces & Roles](https://learn.microsoft.com/en-us/fabric/fundamentals/workspaces) |
| [Fabric Security Overview](https://learn.microsoft.com/en-us/fabric/security/security-overview) |
| [Fabric Security Fundamentals](https://learn.microsoft.com/en-us/fabric/security/security-fundamentals) |
| [OneLake Security](https://learn.microsoft.com/en-us/fabric/onelake/security/fabric-onelake-security) |
| [Fabric Multi-Geo (Data Residency)](https://learn.microsoft.com/en-us/fabric/admin/service-admin-premium-multi-geo) |
| [Row-Level Security in Fabric](https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security) |
| [Fabric Permission Model](https://learn.microsoft.com/en-us/fabric/security/permission-model) |
| [Fabric Private Links](https://learn.microsoft.com/en-us/fabric/security/security-private-links-use) |
| [Microsoft Fabric Licenses & SKUs](https://learn.microsoft.com/en-us/fabric/enterprise/licenses) |
| [Power BI Embedded Multi-Tenancy](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy) |
| [Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts) |
| [SaaS Multi-Tenant on Azure – Architecture Guide](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview) |
| [Multitenant SaaS – Azure SQL patterns](https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns) |
| [Azure Architecture Center – Architect multitenant solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/overview) |
| [**Tenancy Models for Multitenant Solutions**](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models) |
| [**SaaS & Multitenant Solution Architecture (Architecture Center)**](https://learn.microsoft.com/en-us/azure/architecture/guide/saas-multitenant-solution-architecture/) |
| [**Microsoft Fabric Overview (Official Docs)**](https://learn.microsoft.com/en-us/fabric/fundamentals/microsoft-fabric-overview) |
| [**Azure SaaS Dev Kit**](https://azure.github.io/azure-saas/) |
| [**Power BI Embedded for ISVs**](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embedded-analytics-power-bi) |
| [**Microsoft Purview Governance**](https://learn.microsoft.com/en-us/purview/purview) |
| [**Fabric Workspace-per-Tenant Pattern**](https://learn.microsoft.com/en-us/fabric/security/workspace-identity) |

### Pros
- Fully managed platform – no VM patching, no container orchestration overhead
- Native integration: Fabric + Power BI + Azure OpenAI + Purview + Entra = one ecosystem
- OneLake means no data duplication across layers (Lakehouse, Warehouse, Real-Time)
- Built-in compliance (ISO 27001, SOC2, GDPR, HIPAA eligible)
- Fastest time-to-value for analytics heavy workloads like E360

### Cons
- **Azure lock-in** – Contoso's cloud-agnostic requirement is not met natively
- Fabric is not yet available in all Azure regions
- Licensing cost model (F SKUs) can be opaque for initial estimation
- Customization options are constrained by what Fabric exposes in its API surface

### Technology Mapping: Contoso's Proposed Stack → Azure Native Equivalents

Contoso's proposed E360 Plus architecture includes several open-source technologies. In the Azure Native path (Option 1), these are replaced by Azure-native equivalents that are fully managed, integrated, and supported:

| Contoso's Proposed (Open-Source) | Azure Native Replacement | Why the Azure Native Option is Better |
|---|---|---|
| **Apache Iceberg** (table format) | **Delta Lake** (native in Fabric / OneLake) | Delta Lake is the native table format in Microsoft Fabric. OneLake stores all data as Delta/Parquet by default — no additional catalog setup, no format conversion, no compatibility layer needed. Fabric Lakehouse, Warehouse, Notebooks, and Power BI all read Delta natively. |
| **Iceberg REST Catalog / Hive Metastore** | **OneLake** (built-in unified catalog) + **Microsoft Purview** (governance catalog) | OneLake IS the catalog — every Fabric workspace automatically organises tables, files, and shortcuts with no separate metadata store to deploy or manage. Purview adds cross-tenant data governance, lineage tracking, and sensitivity classification on top. |
| **ClickHouse Cloud** (analytical acceleration) | **Fabric SQL Warehouse** (MPP analytics) + **KQL Database** (time-series / real-time) | Fabric SQL Warehouse provides massively parallel query processing on Lakehouse data. KQL Database (Real-Time Intelligence) handles sub-second time-series queries. Both are workspace-scoped, eliminating the need to manage a separate ClickHouse deployment. ClickHouse can still complement Fabric for specialised query patterns if needed. |
| **dbt Core** (transformations) | **dbt Core on Fabric** (supported) OR **Fabric Notebooks / Dataflows** | dbt Core works natively with Fabric's SQL Warehouse endpoint — no replacement needed. Alternatively, Fabric Notebooks (PySpark) and Dataflows Gen2 provide built-in transformation capabilities with Copilot assistance. |
| **Keycloak** (identity proxy) | **Microsoft Entra ID** (B2B federation) | Enterprise-grade identity with built-in Okta federation (SAML/OIDC), Conditional Access, MFA, and seamless integration with every Azure service. No self-hosted identity server to maintain. |
| **Apache Kafka** (event streaming) | **Fabric Eventstream** + **Azure Event Hubs** | Eventstream is Fabric's native real-time ingestion engine with no-code connectors. Event Hubs provides Kafka-compatible endpoints — existing Kafka producers can publish to Event Hubs without code changes. |
| **Prometheus / Grafana** (monitoring) | **Azure Monitor** + **Log Analytics** + **Power BI** | Unified observability across all Azure services. Per-tenant monitoring with workspace-scoped dashboards. No separate monitoring infrastructure to operate. |
| **OpenMetadata** (data catalog) | **Microsoft Purview** | Enterprise data governance with automatic lineage from Fabric pipelines, sensitivity labels, cross-tenant cataloging, and compliance reporting. |

> **Key insight:** Contoso's proposed architecture already uses Azure Data Factory, ADLS Gen2, Key Vault, and Power BI — all Azure-native. The open-source components (Iceberg, ClickHouse, Keycloak, Kafka) are the exceptions, not the rule. Replacing them with Azure-native equivalents completes the Azure Native vision without losing any capability.

> **Note on Iceberg interoperability:** If Contoso has existing data in Iceberg format, Fabric supports **OneLake shortcuts** and **Iceberg table support** (preview) allowing Fabric to read Iceberg tables without migration. This provides a smooth transition path rather than a hard cutover.

---

<a name="option2"></a>
## 3. Architecture Option 2 – Cloud-Agnostic Solution (Containers on VMs)

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
| **Node pool per tenant** | Physical compute isolation | High compliance/noisy neighbor concern |
| **Cluster per tenant** | Full isolation | Regulated, large enterprise clients |

- For E360, **namespace-per-tenant** is the pragmatic starting point, with node pool isolation available as a premium tier option.

### Microsoft Reference Links – Option 2

| Resource |
|---|
| [AKS Multi-Tenancy Best Practices](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation) |
| [AKS Cluster Isolation (Logical vs Physical)](https://learn.microsoft.com/en-us/azure/aks/cluster-multi-tenancy) |
| [Azure Architecture Center – AKS Baseline](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks) |
| [Azure Architecture Center – Multi-Region AKS](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-multi-region/aks-multi-cluster) |
| [Azure Container Apps (serverless alternative)](https://learn.microsoft.com/en-us/azure/container-apps/overview) |
| [Cloud-Agnostic App Design on Azure](https://learn.microsoft.com/en-us/azure/architecture/guide/design-principles/design-for-self-healing) |
| [Azure Architecture Center – Multitenant SaaS on AKS](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/compute) |
| [Kubernetes Network Policies](https://learn.microsoft.com/en-us/azure/aks/use-network-policies) |
| [AKS + External OIDC / Workload Identity](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) |
| [Azure Arc (unified multi-cloud K8s management)](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview) |
| [Deployment Stamp Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) |

### Pros
- **True cloud agnosticism** – runs on AKS, EKS, GKE, or on-prem with minimal rework
- Full control over runtime, dependencies, and upgrade cycles
- Tenant onboarding via Helm charts / GitOps (ArgoCD / Flux) = repeatable and auditable
- Can incorporate any open-source stack (Apache Spark, dbt, Trino, OpenMetadata)
- Avoids Microsoft Fabric licensing model entirely

### Cons
- Higher operational complexity – you own the platform layer
- Requires dedicated DevOps/SRE capability to manage at scale
- Analytics capabilities must be built or integrated from scratch (vs. Fabric's native Power BI, Notebooks, Lakehouse)
- Longer time-to-value; more bespoke engineering effort
- Security hardening (CIS benchmarks, pod policies, network policies) is entirely on the team

---

<a name="tradeoffs"></a>
## 4. Comparative Trade-off Discussion

### Side-by-Side Comparison

| Dimension | Option 1: Azure Native (Fabric) | Option 2: Cloud-Agnostic (K8s) | Advantage |
|---|---|---|---|
| **Cloud Lock-in** | Azure-first (Azure Arc for non-Azure) | Deploy anywhere | Option 2 |
| **Multi-Tenancy** | Native workspace isolation + OneLake RBAC | Namespace / node pool / cluster isolation | **Option 1** |
| **Time-to-Value** | Fast – Fabric has analytics built-in | Slower – more custom build required | **Option 1** |
| **Operational Complexity** | Low – Microsoft manages infrastructure | High – team manages K8s and all layers | **Option 1** |
| **Configurability per Tenant** | Moderate – Fabric APIs + config DB | High – Helm values, config maps, feature flags | Option 2 |
| **Analytics Depth** | Very High – Lakehouse, Notebooks, Power BI, Copilot | Needs integration (Spark, Trino, Superset etc.) | **Option 1** |
| **AI/ML Integration** | Native – Azure OpenAI + Fabric Copilot | Bring-your-own (Azure AI, Hugging Face, etc.) | **Option 1** |
| **Data Residency / Sovereignty** | Supported via Fabric Multi-Geo | Supported via cloud region selection | Tie |
| **SSO / Identity** | Entra ID + B2B federation (Okta via SAML/OIDC) | Keycloak / OIDC proxy + Okta integration | **Option 1** |
| **Cost Model** | Predictable – F SKU capacity pricing | Variable – compute + ops labor cost | **Option 1** |
| **Compliance / Certifications** | Microsoft holds certs (SOC2, ISO 27001, HIPAA) | Must certify per cloud provider / per deployment | **Option 1** |
| **Disaster Recovery** | Built-in Fabric reliability + Azure SLAs | Requires bespoke DR strategy | **Option 1** |
| **Customization (client-specific logic)** | Config DB + API parameters + Power BI personalisation | Feature flags, per-tenant Helm overrides, custom code branches | Option 2 |
| **Vendor Risk** | Low – Microsoft is strategic partner & heavily invested | Low – open-source stack, portable | Tie |

> **Score: Option 1 wins 9 of 14 dimensions.** Option 2 leads only on cloud portability and per-tenant customisation — both addressable via Azure Arc and configuration patterns respectively.


### Recommended Decision Framework

Use this to guide the Contoso conversation:

```
Start with Option 1 (Azure Native / Fabric) as the default.

Does the client require Azure? (80% of Contoso's clients = YES)
    YES ──► Option 1 (Fabric) – full platform value, AI, analytics, governance
    NO  ──► Can Azure Arc bridge the gap?
              YES ──► Option 1 + Azure Arc (extend Azure management to any infra)
              NO  ──► Option 2 (K8s) as a constrained fallback for that client only
```

> **Key principle:** Do not compromise the platform for 80% of clients to accommodate the 20%. Build the best possible platform on Azure, then provide a lightweight escape hatch for exceptions.

### Hybrid Path (Recommended Approach)
The pragmatic approach is **Azure Native first, with Azure Arc for edge cases**:
- **Microsoft Fabric is the core analytics and data platform** for all Azure-compatible clients (~80%) — delivering multi-tenancy, AI, Power BI, and governance out of the box
- **Azure Arc** extends Azure's management plane to non-Azure environments, enabling the ~20% of clients to run on AWS, GCP, or on-prem while still being managed from a single Azure control plane
- **Containerised application services** (on AKS) handle compute-heavy or cloud-portable workloads, complementing Fabric's analytics layer
- This approach avoids the trap of building the lowest-common-denominator platform that serves no one optimally
- Reference: [Azure Arc for Kubernetes](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview)

### Detailed Pros and Cons by Key Dimension

#### 4.1 Scalability

**Option 1 (Azure-Native):**

- **Pro:** Azure PaaS services provide auto-scaling without manual intervention. Microsoft Fabric offers a unified compute and storage architecture supporting ETL, data warehousing, reporting, real-time analytics, and generative AI workloads within a single SaaS platform. The proposed architecture allows each client to have a separate database managed via a semantic layer, meaning new tenants can be added without reconfiguring the core infrastructure.
- **Con:** Scaling is constrained to Azure's service limits and regional availability. The ~20% of customers who cannot use Azure (due to policy or preference) would need a separate solution entirely, fragmenting the scaling model.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Kubernetes-based scaling is portable: the same cluster configuration and Helm charts can scale workloads on AKS, EKS, GKE, or on-prem infrastructure. No ceiling imposed by a specific cloud vendor's service quotas.
- **Con:** Auto-scaling must be configured manually (Horizontal Pod Autoscalers, Cluster Autoscaler). Capacity planning, node pool management, and performance tuning require dedicated DevOps expertise. As the number of tenants grows, managing isolated deployments (if using per-client instances as an interim approach) becomes operationally expensive.

> **Trade-off:** Option 1 provides effortless vertical and horizontal scalability within Azure — which serves ~80% of Contoso's clients immediately. Option 2 provides universal scalability but demands continuous operational investment that scales linearly with the customer base. **Recommendation: Build on Azure Native (Option 1) for the majority, and use Azure Arc to extend reach to the ~20% who need non-Azure deployment, rather than penalising the majority with Option 2's operational overhead.**

---

#### 4.2 Cost and Pricing Model

**Option 1 (Azure-Native):**

- **Pro:** Fabric's pricing is based on two main components: storage (OneLake) and compute, with calculators available to estimate costs based on data volume and transformation needs. Microsoft cites reference case studies showing cost advantages over other platforms for analytics workloads. The pay-as-you-go model can align costs with actual usage, avoiding upfront capital expenditure.
- **Con:** Azure consumption costs can escalate unpredictably with growing data volumes or compute-intensive analytics workloads. Additionally, Contoso's existing other platform investments (infrastructure, expertise, tooling) are largely non-transferable, creating a sunk-cost write-off during migration.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Leverages existing open-source tooling (Python, dbt Core, Iceberg) and avoids premium PaaS licensing. Infrastructure costs are more predictable (fixed VM/node costs). Contoso retains the option to deploy on whichever cloud offers the best pricing for a given client.
- **Con:** Operational overhead constitutes a significant hidden cost: Contoso must staff and train a team to manage Kubernetes clusters, perform upgrades, handle incident response, and maintain custom tooling — essentially operating as a SaaS provider. These labour costs may offset the savings from avoiding PaaS premiums.

> **Trade-off:** The cost comparison is not straightforward. Option 1 shifts spend from **people** (ops engineers) to **platform** (Azure fees); Option 2 does the reverse. However, platform spend is predictable and scales sub-linearly, while ops labour scales linearly. **Recommendation: For a SaaS business like E360, Option 1's consumption model is structurally more efficient at scale — Contoso should invest in platform capability, not headcount. Microsoft's partnership investment and potential EA pricing further tilts the economics in Option 1's favour.**

---

#### 4.3 Ease of Integration with Existing E360 Presentation Layer

**Option 1 (Azure-Native):**

- **Pro:** Power BI is natively integrated within Fabric, allowing seamless embedding of reports into custom application layers, with APIs available for further integration and automation. Since E360 already uses Power BI for drill-down views, migrating these into Fabric-hosted Power BI would be relatively low-friction. Contoso's proposed architecture already anticipates Power BI Premium/Embedded for persona-based dashboards, suggesting alignment with this path.
- **Con:** Replacing the current direct other platform S3 data access with calls to Fabric or ADLS Gen2 APIs requires backend refactoring. Integration of Contoso's Okta authentication with Microsoft Entra ID adds complexity in a mixed identity environment.

**Option 2 (Cloud-Agnostic):**

- **Pro:** This is the lowest-friction path for integration. The existing backend services run inside containers essentially unchanged, and the E360 front-end (Vue.js) interacts with them exactly as before. No new API paradigms, no new authentication protocols, no new data access patterns need to be introduced immediately.
- **Con:** The legacy architecture is preserved with all its limitations. Any new capabilities (e.g., automated data pipelines, enhanced analytics, admin UI) must be custom-built and integrated from scratch. The E360 portal continues interfacing with a set of self-managed services rather than one unified platform, increasing integration surface area over time.

> **Trade-off:** Option 2 offers a faster initial integration path (weeks), while Option 1 offers the most capable long-term integration (months of refactoring). However, the faster path preserves all of E360's current architectural limitations. **Recommendation: Invest in Option 1's integration now. Contoso's proposed architecture already envisions Power BI Embedded, ADLS Gen2, and Azure PaaS — this is inherently an Azure Native design. Starting with Option 2 risks accumulating technical debt that makes the eventual Azure Native migration harder, not easier. A phased migration to Fabric can begin with the data layer (OneLake, ADF pipelines) while the presentation layer transitions in parallel.**

---

#### 4.4 Performance and Reliability

**Option 1 (Azure-Native):**

- **Pro:** Fabric provides a unified analytics engine supporting ETL, data warehousing, reporting, real-time analytics, and generative AI workloads within a single SaaS architecture. Data doesn't need to move between systems for different analytical tasks, reducing latency. Azure-managed services come with enterprise-grade SLAs, automated failover, and built-in high availability.
- **Con:** Performance tuning options are more constrained — Contoso depends on Microsoft's platform optimisations rather than having direct control over compute allocation, caching strategies, or query optimisation at the infrastructure level.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Contoso retains full control over performance optimisation. The proposed architecture includes ClickHouse Cloud for analytical acceleration, which is a high-performance columnar database suitable for real-time analytics. This could deliver excellent query performance for E360's reporting workloads. Additionally, isolated per-client deployments mean one client's workload spikes cannot impact another's performance.
- **Con:** Without managed services, achieving high availability and disaster recovery requires significant DevOps investment (container health checks, pod disruption budgets, cross-zone replication). Performance benchmarking and optimisation is Contoso's responsibility, consuming engineering bandwidth.

> **Trade-off:** Option 1 provides predictable, enterprise-grade performance backed by Microsoft SLAs, suitable for the vast majority of analytical workloads. Option 2 has a higher theoretical performance ceiling with specialised tools like ClickHouse, but realising that ceiling requires significant engineering effort and ongoing tuning. **Recommendation: Option 1 is the lower-risk, higher-value choice. Fabric's unified engine eliminates data movement latency, and Microsoft continuously optimises the platform — Contoso benefits from these improvements automatically without engineering investment. ClickHouse can complement Fabric as an acceleration layer if specific query patterns demand it, rather than replacing the entire platform.**

---

#### 4.5 Multi-Tenancy Support

> *This is arguably the most critical dimension, given that E360's current system stores all client data in a single storage environment with only RBAC for access control — a model that "limits the ability to provide true data segregation and raises concerns for clients about data privacy and security".*

**Option 1 (Azure-Native):**

- **Pro:** The proposed architecture envisions each client having a separate database with a semantic layer managing metadata, access hierarchies, and data transformations, ensuring strict data segregation and privacy. Microsoft Fabric supports multi-tenant SaaS deployments on Azure, with separate workspaces or lakehouses per tenant providing physical data isolation. Configuration of data pipelines and onboarding of new clients can be managed through the Fabric portal or via API integrations.
- **Con:** Implementing per-tenant workspaces in Fabric still requires significant design work (metadata management, tenant routing, access control mapping). The existing monolithic data model must be decomposed — this is a substantial engineering effort regardless of the platform.

**Option 2 (Cloud-Agnostic):**

- **Pro:** There are well-documented ways to achieve multi-tenancy at the AKS level with Namespace Isolation, Network Policies, resource quotas, and RBAC (see [AKS Multitenancy Guide](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation)). Additionally, the team could bring their as-is architecture on Azure, make it SaaS-ready and add IaC to automate the deployment for any net-new customers — meaning each new client gets an automated fresh deployment via Infrastructure-as-Code.
- **Con:** Kubernetes-level isolation is **not** application-level multi-tenancy. The E360 codebase itself must be modified to support per-tenant data partitioning, configuration, and access control — this development burden falls entirely on Contoso.

> **Trade-off:** Both options require substantial multi-tenancy engineering at the application layer. However, Option 1 provides **platform-level** multi-tenancy primitives (separate workspaces, OneLake folder isolation, sensitivity labels, workspace-scoped RBAC, Fabric REST APIs for automation) that fundamentally accelerate the work. Option 2 provides only infrastructure-level isolation (namespaces, network policies) and leaves the entire application-level multi-tenancy burden on Contoso. **Recommendation: Option 1 is clearly superior for multi-tenancy. Fabric's workspace-per-tenant model directly addresses Contoso's stated concern about data segregation and privacy — each client's data is physically separated at the storage layer, not merely filtered by RBAC. This is the single strongest argument for Azure Native.**

---

#### 4.6 Configurability and Admin UI Potential

**Option 1 (Azure-Native):**

- **Pro:** Configuration of data pipelines and client onboarding can be managed through the Fabric portal or via API integrations, enabling custom UI development for client management within the Contoso-branded application. The E360 team can build an admin interface that programmatically calls Fabric APIs to create new tenant workspaces, configure data sources, and manage access — leveraging existing platform capabilities rather than building from scratch.
- **Con:** Some configuration may only be possible through Azure's own portal rather than via APIs, which would force administrators to leave the E360 interface. The learning curve for Fabric APIs and administration adds upfront development time.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Complete freedom to design the admin experience to match Contoso's exact specifications. The E360 team can build microservices for configuration management, storing tenant settings in a central database and exposing them via the presentation layer — fulfilling the requirement that ETL, Auth, and User Provisioning all be manageable via the Presentation Layer.
- **Con:** Every admin function must be designed and coded from scratch: pipeline configuration, monitoring, user administration, access controls, data source management. This is a significant development investment.

> **Trade-off:** Option 1 provides building blocks (APIs, portal features) that reduce custom development. Option 2 provides unlimited customisation but at the cost of building every admin function from scratch. **Recommendation: Option 1 reduces the backend complexity behind the admin UI significantly. Fabric's REST APIs for workspace provisioning, data pipeline configuration, and access management provide a pre-built foundation that the E360 admin portal can orchestrate — instead of Contoso building and maintaining these capabilities themselves. This directly serves the stated need for a low-code/no-code configuration UI.**

---

#### 4.7 Cloud Portability and Vendor Lock-In

**Option 1 (Azure-Native):**

- **Pro:** For the ~80% of customers subscribing to Contoso's SaaS offering, Azure lock-in is a non-issue — they are consuming E360 as a service, not managing infrastructure. Contoso benefits from Microsoft's partnership support and potential co-investment.
- **Con:** Fabric is not cloud-agnostic and requires Azure for deployment — though it supports interoperability with data residing in other clouds, clients strictly on non-Azure platforms would require alternative solutions. This directly contradicts Contoso's requirement to maintain one codebase across cloud providers. The ~20% of clients needing non-Azure deployments would require a separate solution track.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Fully aligned with Contoso's design principle. Containerised E360 can be deployed on AKS, EKS, or any Kubernetes-compatible environment. Contoso explicitly stated they are "not open for any re-architecture or platform-sticky components", making this the philosophically aligned choice.
- **Con:** Maintaining cloud neutrality imposes constraints: the team must avoid using any cloud-specific managed services (or abstract them behind interfaces), which limits the use of optimised platform features. This can slow development and reduce the sophistication of the analytics layer compared to what Fabric could offer.

> **Trade-off:** This is often framed as the **most polarising dimension**, but it shouldn't be. Option 1 maximises value for the majority (80%) and Azure Arc addresses the minority (20%) — this is not a gap, it's a solved problem. Option 2 serves everyone equally but at a higher operational cost for all — effectively punishing 80% of clients to accommodate 20%. **Recommendation: Azure Native is the right choice. For SaaS clients (~80%), cloud portability is irrelevant — they consume E360 as a service, not infrastructure. For the ~20% requiring non-Azure deployment, Azure Arc + AKS provides a managed path without sacrificing the platform's core capability. The "one codebase everywhere" ideal sounds appealing in theory but delivers a mediocre experience everywhere in practice.**

---

#### 4.8 Operational Complexity and Maintainability

**Option 1 (Azure-Native):**

- **Pro:** Offloads significant operational burden to Microsoft — patching, scaling, backups, uptime monitoring, and security updates for Fabric and Azure PaaS are handled by the platform. Fewer moving parts for Contoso to manage day-to-day.
- **Con:** Contoso's team must acquire new skills in Azure networking and deployment. Until this ramp-up is complete, Contoso is dependent on Microsoft for troubleshooting and optimisation. Providing a list of team members who require Azure portal access for hands-on familiarisation is an identified action item.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Familiar operational model — the same team that manages E360 on other platform today can manage it on AKS with transferable skills. Existing monitoring, logging, and CI/CD practices can be adapted rather than replaced. Using IaC to automate deployment for net-new customers provides a scalable operational model.
- **Con:** Running Kubernetes at production scale is resource-intensive: cluster maintenance, node upgrades, security patching, container image management, and incident response all fall on Contoso's team. As the customer base grows, operational overhead scales with it — Contoso effectively becomes a managed infrastructure provider alongside being a software vendor.

> **Trade-off:** Option 1 trades operational simplicity for a learning curve; Option 2 trades operational autonomy for ongoing labour intensity that grows with the customer base. **Recommendation: Option 1 is clearly superior for operational sustainability. Microsoft manages patching, scaling, backups, and security updates for the entire Fabric platform — Contoso's team focuses on building E360 features, not operating infrastructure. The learning curve is a one-time investment; Kubernetes operational overhead is permanent and compounds. With Microsoft's empanelled partners available for Azure environment setup, the transition risk is manageable.**

---

#### 4.9 Security and Compliance

**Option 1 (Azure-Native):**

- **Pro:** The proposed architecture includes Azure Key Vault for secrets and credential management and Azure PaaS ADLS Gen2 with VNet integration for network-level security. Azure provides built-in encryption, compliance certifications, and identity management via Microsoft Entra ID. Multi-tenant data isolation can be enforced at the platform level (separate storage containers, database-level security). The proposed design includes a data governance layer with metadata management and data quality controls.
- **Con:** Integration of Contoso's existing Okta authentication with Microsoft Entra ID requires careful federation design to avoid gaps in access control during migration. Clients subject to specific data sovereignty regulations may resist having data stored in Microsoft-managed Azure regions.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Contoso retains complete control over security implementation. For clients with strict regulatory requirements, Contoso can deploy E360 within the client's own subscription or on-premises environment, giving the client full control over data residency, network policies, and access controls. Kubernetes Namespace Isolation, Network Policies, and RBAC provide infrastructure-level security boundaries between tenants.
- **Con:** Contoso must implement, test, and certify all security controls themselves. Container image scanning, runtime security monitoring, network segmentation, secret management, and audit logging must all be built or procured separately. The greater number of self-managed components increases the attack surface.

> **Trade-off:** Option 1 provides a pre-certified security foundation suitable for most enterprise requirements, with SOC 2, ISO 27001, HIPAA, and other certifications already held by Microsoft. Option 2 provides maximum flexibility for clients with bespoke security needs. **Recommendation: Option 1 is the stronger security posture for the vast majority of clients. Azure's built-in encryption, Microsoft Defender for Cloud, Purview data governance, and Conditional Access policies provide defense-in-depth that would cost millions to replicate independently. For the rare client requiring deployment within their own security perimeter, Azure Arc + AKS in the client's own subscription provides this capability without abandoning the Azure ecosystem. Contoso should not build and certify a custom security stack when Microsoft already provides one.**

---

#### 4.10 Future-Proofing and Innovation Potential

**Option 1 (Azure-Native):**

- **Pro:** Direct alignment with Microsoft's innovation roadmap, including Fabric's ongoing enhancements for generative AI workloads. Contoso's broader collaboration with Microsoft on Azure OpenAI Service positions E360 to incorporate advanced AI capabilities as they mature. Microsoft Fabric was proposed as the core analytics platform specifically highlighting its integrated ETL, data warehousing, real-time analytics, and Power BI experiences, with the ability to support multi-tenant SaaS deployments on Azure.
- **Con:** E360's technical evolution becomes coupled to Microsoft's product roadmap. Features outside Azure's ecosystem (e.g., new open-source analytics frameworks) become harder to adopt.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Technology independence allows adoption of the best available tools as they emerge. The proposed architecture already includes forward-looking open-source technologies: dbt Core for transformations, Iceberg REST Catalog / Hive Metastore for an open table format with bronze-silver-gold data patterns, and ClickHouse Cloud for analytical acceleration. These are vendor-neutral, community-supported technologies that can evolve independently of any cloud provider.
- **Con:** Contoso's team must independently evaluate, adopt, and integrate new technologies — a resource-intensive process. Without a platform vendor driving innovation, the pace of feature advancement depends entirely on Contoso's engineering investment.

> **Trade-off:** Option 1 provides accelerated innovation through platform leverage; Option 2 provides innovation freedom at the cost of engineering effort. **Recommendation: Option 1 aligns E360 with Microsoft's multi-billion-dollar investment in Fabric, Azure OpenAI, and Copilot — Contoso gets world-class AI and analytics capabilities delivered through platform updates, without dedicating engineering resources to build or integrate them. Notably, Contoso's own proposed architecture already uses Azure-native services (Data Factory, ADLS Gen2, Key Vault, Power BI) alongside open-source tools — this hybrid technology selection is already an Azure Native design with open-source components, not a cloud-agnostic design. Microsoft's innovation roadmap is E360's innovation roadmap.**

---

<a name="Contoso"></a>
## 5. Contoso Requirements Alignment & Open Questions

### Requirement 1: Multi-Tenancy

| Aspect | Option 1 (Fabric) | Option 2 (K8s) |
|---|---|---|
| **Approach** | One Fabric Workspace per client + OneLake folder isolation | Kubernetes Namespace per tenant |
| **Data Isolation** | Workspace-level RBAC, sensitivity labels, optional customer-managed keys | Network policies, K8s secrets, per-namespace service accounts |
| **Onboarding** | Scripted via Fabric REST API or Terraform (AzAPI provider) | Helm chart instantiation, GitOps pipeline |
| **Offboarding** | Delete workspace + revoke RBAC | Delete namespace + PVC cleanup |
| **Gaps** | Fabric API coverage for workspace automation still maturing | Ops overhead; need tenant lifecycle management service |

### Requirement 2: Configurability

| Aspect | Approach |
|---|---|
| **Client-specific logic** | Store tenant configuration in a dedicated **Configuration Database** (Azure SQL or Cosmos DB). Application reads config at runtime – no code forking. |
| **Feature flags** | Use [Azure App Configuration](https://learn.microsoft.com/en-us/azure/azure-app-configuration/overview) with feature filters per tenant ID |
| **UI-driven config** | Build a lightweight **Admin Portal** (React/Next.js on Azure Static Web Apps) backed by a config API |
| **Short-term** | Scripting / runbooks acceptable; flag this as a roadmap item for self-service |
| **Reference** | [Feature flags in .NET](https://learn.microsoft.com/en-us/azure/azure-app-configuration/use-feature-flags-dotnet-core) |

### Requirement 3: Cloud Agnosticism

**Microsoft's Perspective:**
- Cloud agnosticism sounds strategically appealing but **comes at a measurable cost**: lower analytics capability, higher operational complexity, slower time-to-value, and weaker AI integration. The question is whether that cost is justified.
- **For ~80% of Contoso's client base (Azure-compatible SaaS clients):** Cloud agnosticism delivers zero value. These clients consume E360 as a service — they never see or care about the underlying infrastructure. Building a cloud-agnostic platform for them is engineering overhead with no customer benefit.
- **For ~20% requiring non-Azure deployment:** Azure Arc + AKS provides a managed path to run Kubernetes workloads on any infrastructure (AWS, GCP, on-prem) under Azure's unified management plane. This is not a workaround — it is a production-grade Microsoft solution used by enterprises globally.
- **Fabric itself supports data interoperability** with non-Azure storage (OneLake shortcuts to S3, GCS), meaning data residing outside Azure can still be accessed by Fabric without physical migration.
- **Recommendation:** Adopt Azure Native as the core platform. For the minority of clients where Azure is not possible, deploy the application tier via AKS (managed through Azure Arc) with data connectors back to the client's environment. This satisfies the portability requirement without sacrificing platform capability for the majority.

### Authentication / SSO: Contoso Okta + Azure Entra ID

This is likely Contoso's biggest practical question. Here is the recommended flow:

```
Contoso User
    │
    ▼
Okta (IdP) ──SAML 2.0 / OIDC──► Azure Entra ID (via B2B Federation)
                                         │
                                  Entra ID issues token
                                         │
                           ┌─────────────┼───────────────┐
                           ▼             ▼               ▼
                     Fabric API    App Service      API Management
                    (workspace     (backend API)   (subscription key
                     scoped)                        per tenant)
```

- **Microsoft Entra External ID** supports federated SAML/OIDC from Okta
- Users authenticate via Okta; Entra issues claims-enriched tokens with tenant context
- Downstream services use the token's `tid` (tenant ID) claim to enforce data access rules
- For Option 2 (K8s), replace Entra with **Keycloak** fronted by the same Okta OIDC flow

**Reference Links – Identity & SSO:**
- [Microsoft Entra External ID (B2B federation)](https://learn.microsoft.com/en-us/entra/external-id/what-is-b2b)
- [Federation with Okta via SAML in Entra ID](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-saml-single-sign-on)
- [Claims-based identity on Azure](https://learn.microsoft.com/en-us/entra/identity-platform/security-tokens)
- [Conditional Access in Microsoft Fabric](https://learn.microsoft.com/en-us/fabric/security/security-conditional-access)

### Open Questions to Resolve with Contoso

1. **Is cloud agnosticism a contractual requirement, or a preference?** – If contractual, Azure Arc bridges the gap for non-Azure clients. If a preference, it should not override the value of Azure Native for 80% of clients.
2. **What is the timeline for the first 3–5 tenants, and 50+ tenants?** – Affects isolation model choice.
3. **What does "client-specific customization" mean concretely?** – Custom data models? Custom business rules? Custom UI? Each requires a different config pattern.
4. **Which regions does E360 need to operate in?** – Affects Fabric Multi-Geo and data residency compliance.
5. **Does Contoso have an EA with Microsoft?** – Impacts Fabric F-SKU pricing significantly.
6. **What is the existing identity infrastructure?** – Is Okta federated to Azure AD already, or does this need to be set up?
7. **What are the SLA expectations per client?** – Shared capacity vs. dedicated capacity decision.
8. **What audit/compliance standards must be met per client?** – SOC2, HIPAA, ISO 27001?


---

<a name="landing-zone"></a>
## 6. Multi-Tenant Landing Zone

### What is a Landing Zone?

A Landing Zone is the pre-configured, governed Azure environment into which workloads are deployed. For a multi-tenant SaaS platform, the landing zone defines:
- Subscription structure
- Networking topology
- Identity & access management
- Security baselines
- Cost management boundaries

### Recommended Landing Zone Structure for E360

```
Azure Management Group
└── E360 Platform Management Group
    ├── Platform Subscription (shared services)
    │   ├── Hub VNet (Firewall, Bastion, Private DNS)
    │   ├── Azure API Management (Premium tier)
    │   ├── Azure Key Vault (Platform secrets)
    │   ├── Azure Monitor / Log Analytics Workspace
    │   ├── Microsoft Purview Account
    │   └── Azure DevOps / GitHub Actions (CI/CD)
    │
    ├── Fabric / Analytics Subscription
    │   ├── Microsoft Fabric Capacity (F64+)
    │   ├── OneLake Storage
    │   └── Per-tenant Fabric Workspaces (logical isolation)
    │
    └── Per-Tenant Spoke Subscriptions (for high-isolation clients)
        ├── Tenant A Spoke VNet (peered to Hub)
        ├── Tenant A Azure SQL / Cosmos DB
        └── Tenant A Key Vault (CMK)
```

### Hub-and-Spoke Networking for Multi-Tenant SaaS

```
┌───────────────────────────────────────────┐
│              HUB VNet                     │
│  Azure Firewall │ Private DNS │ Bastion   │
│  APIM (internal mode) │ Azure AD DS       │
└──────────────────┬────────────────────────┘
         ┌─────────┼─────────┐
         ▼         ▼         ▼
    ┌─────────┐ ┌─────────┐ ┌─────────┐
    │ Spoke A │ │ Spoke B │ │Platform │
    │ Tenant A│ │ Tenant B│ │Shared   │
    │  VNet   │ │  VNet   │ │Services │
    └─────────┘ └─────────┘ └─────────┘
```

### Microsoft Reference Architectures for Multi-Tenant Landing Zone

| Resource |
|---|
| [**Azure Landing Zone – Official Overview**](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/) |
| [**CAF Enterprise-Scale Landing Zone**](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/enterprise-scale/architecture) |
| [**Multitenant SaaS – Architectural approaches overview**](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview) |
| [**Multitenant – Compute approaches**](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/compute) |
| [**Multitenant – Networking approaches**](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/networking) |
| [**Multitenant – Identity approaches**](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity) |
| [**Multitenant – Deployment & config**](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/deployment-configuration) |
| [**Multitenant – Cost management**](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/cost-management-allocation) |
| [**Multitenant Checklist**](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/checklist) |
| [**SaaS tenancy models (SQL)**](https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns) |
| [**Deployment Stamp pattern**](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) |
| [**Hub-Spoke Network Topology**](https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke) |
| [**AKS Baseline Landing Zone**](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks) |
| [**Azure Policy for Landing Zones**](https://learn.microsoft.com/en-us/azure/governance/policy/overview) |
| [**Subscription vending / tenant onboarding (bicep)**](https://learn.microsoft.com/en-us/azure/architecture/landing-zones/subscription-vending) |

### How to Apply This to E360 Specifically

**Step 1 – Tenant Classification Tier**

Define tiers that determine isolation level:

| Tier | Profile | Landing Zone Model |
|---|---|---|
| **Standard** | SMB / smaller clients | Shared Fabric Workspace + shared compute |
| **Professional** | Mid-market clients | Dedicated Fabric Capacity + spoke VNet |
| **Enterprise** | Large Contoso enterprise clients | Dedicated subscription + CMK + private endpoints |

**Step 2 – Subscription Vending Automation**

For Enterprise-tier tenants: automate subscription provisioning using:
- Azure Bicep templates or Terraform
- Azure DevOps pipelines triggered on tenant onboarding
- Policy assignments via Azure Management Groups to enforce security baseline

**Step 3 – Network Topology**

- All tenant workloads connect via **Private Endpoints** to avoid public internet exposure
- APIM in internal mode acts as the entry point, forwarding to per-tenant backends
- Azure Firewall in the hub enforces egress rules and inspects cross-tenant traffic

**Step 4 – Identity Plane**

- One **Microsoft Entra ID tenant** for the platform (E360)
- Client users onboarded as **B2B guest users** (or via External ID if consumer-facing)
- Per-tenant Azure AD **App Registrations** for OIDC flows
- Okta federated via SAML to Entra ID – single point of trust

**Step 5 – Governance & Compliance**

- Azure Policy enforcements: require private endpoints, deny public IP, enforce tagging
- Microsoft Defender for Cloud enabled on all subscriptions
- Microsoft Purview as the cross-tenant data catalog
- Fabric sensitivity labels tied to Purview classification

---

<a name="ai-fabric"></a>
## 7. AI & Fabric Story

### The Case

> *"E360 is not just an analytics platform — it is a decision intelligence platform. With Microsoft Fabric as the foundation, every client's data is unified in OneLake. Azure OpenAI and Fabric's native Copilot capabilities sit on top of that data — meaning AI is not bolted on, it is built in from day one."*

### Architecture: AI-Ready Data Platform on Microsoft Fabric

```
                    ┌──────────────────────────────────────┐
                    │          E360 AI Platform            │
                    │  "Ask anything about your business"  │
                    └──────────────┬───────────────────────┘
                                   │
           ┌───────────────────────┼───────────────────────────┐
           ▼                       ▼                           ▼
   ┌──────────────┐       ┌──────────────────┐      ┌──────────────────┐
   │   Fabric     │       │  Azure OpenAI    │      │  Real-Time       │
   │   Copilot    │       │  (GPT-4o)        │      │  Intelligence    │
   │ (natural lang│       │  RAG on client   │      │  (Event streams  │
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
          │           Source Systems (per client)            │
          │  ERP │ CRM │ Files │ APIs │ Databases │ Streams  │
          └──────────────────────────────────────────────────┘
```

### The Three AI Pillars – Why Each One & What It Does for E360

The architecture has three distinct AI capabilities sitting on top of Microsoft Fabric. Each one solves a different class of problem for E360's users:

---

**Pillar 1: Fabric Copilot** *(left box)*

| | |
|---|---|
| **What it is** | Microsoft's built-in AI assistant embedded inside every Fabric experience — Power BI, Notebooks, Data Factory pipelines, SQL Warehouse. It is powered by Azure OpenAI under the hood but presented as a native Fabric feature. |
| **Why E360 needs it** | E360's current users (Ops Managers, Account SPOCs, CXO personas) are **not data engineers**. Today they rely on pre-built dashboards and static reports. Copilot lets them ask questions in plain English — "What were Tenant A's top 5 escalation categories last quarter?" — and get answers directly from the Lakehouse data without waiting for a report to be built. |
| **E360 use cases** | - **Power BI Copilot:** A CXO opens an E360 dashboard and asks "Summarise this month's DQI trends across my accounts" → Copilot generates a narrative summary and suggests a chart. <br> - **Notebook Copilot:** An E360 data engineer types "Write a Spark transformation to calculate rolling 30-day SLA compliance per client" → Copilot generates the PySpark code. <br> - **Pipeline Copilot:** The team says "Create a pipeline that loads CSV files from ADLS, cleanses nulls, and writes to the Silver layer" → Copilot scaffolds the Data Factory pipeline. |
| **Multi-tenant safety** | Copilot only accesses data within the **current Fabric workspace**. Since each tenant has its own workspace, Tenant A's Copilot cannot read Tenant B's data. No additional configuration needed — isolation is architectural. |
| **What it replaces** | The current ChatGPT-based "Gen AI Playground" in E360 (which has no access to live client data and just wraps a public LLM). Fabric Copilot is grounded on the actual tenant data. |

---

**Pillar 2: Azure OpenAI + RAG** *(centre box)*

| | |
|---|---|
| **What it is** | A custom-built "ask your data" experience using **Retrieval-Augmented Generation (RAG)**. Azure OpenAI (GPT-4o) is the LLM. **Azure AI Search** is the retrieval layer that indexes the client's documents and structured data, so the LLM answers questions grounded in real facts rather than hallucinating. |
| **Why E360 needs it** | Fabric Copilot works brilliantly for structured data already in the Lakehouse (*tables, columns, metrics*). But E360 clients also have **unstructured knowledge** — SOW documents, SLA contracts, operational runbooks, transition playbooks, audit reports. RAG lets users query across both structured and unstructured data in one conversational interface. |
| **E360 use cases** | - **CXO Insights (next-gen):** "What are the contractual SLA targets for Account X, and how does current performance compare?" → RAG retrieves the SOW document from AI Search, extracts the SLA clause, then queries the Gold-layer DQI table to compute actual performance, and synthesises a grounded answer. <br> - **Ops Designer (next-gen):** "Based on historical transition data, what are the risk factors for this new account onboarding?" → RAG pulls from prior transition documents and operational KPIs. <br> - **Benchmarking Q&A:** "How does this client's delivery quality index compare to the industry average?" → RAG combines structured benchmarking data with published benchmark reports. |
| **How it differs from Copilot** | Copilot is a Microsoft-managed UI embedded in Fabric. RAG is a **custom application** — E360's team builds the ingestion pipeline (documents → AI Search index), the prompt engineering, and the UI. This gives full control over the experience, branding, and what data is exposed. |
| **Multi-tenant safety** | Each tenant's documents are indexed into a **separate AI Search index** (or filtered via security trimming with the tenant ID). The Azure OpenAI call includes only the current tenant's retrieved chunks. No cross-tenant data leakage. |
| **Architecture flow** | User question → E360 API → Azure AI Search (retrieve relevant chunks for this tenant) → Azure OpenAI (generate answer using retrieved context) → Return to E360 UI |

---

**Pillar 3: Real-Time Intelligence** *(right box)*

| | |
|---|---|
| **What it is** | Microsoft Fabric's **Real-Time Intelligence** workload (formerly Real-Time Analytics / KQL Database). It ingests streaming data via **Eventstream**, stores it in a KQL (Kusto) database optimised for time-series and log analytics, and enables sub-second queries using **KQL (Kusto Query Language)**. It also includes **Data Activator** — a no-code trigger engine that monitors data streams and fires alerts or actions when conditions are met. |
| **Why E360 needs it** | E360's current architecture is entirely **batch-based** — data is loaded periodically from other platform S3, processed, and surfaced in static dashboards. There is no real-time operational intelligence. But E360's modules (Ops Hub, SMF Alerts/Escalations, Daily Huddle, Guardrails) are inherently **operational and time-sensitive**. Real-Time Intelligence closes the gap between "something happened" and "the dashboard shows it". |
| **E360 use cases** | - **Ops Hub – Live Alerts:** Instead of polling a database every 15 minutes, Eventstream ingests operational events (ticket created, SLA breach, escalation triggered) and Data Activator fires a Teams notification or email **within seconds**. <br> - **Daily Huddle – Live KPIs:** The Daily Huddle screen shows KPIs that update in real-time during the meeting, not from a stale morning batch run. <br> - **DQI – Anomaly detection:** A KQL query continuously monitors the Delivery Quality Index stream. If a client's DQI drops below threshold, Data Activator triggers an automated escalation workflow. <br> - **Guardrails – Proactive compliance:** Real-time monitoring of operational metrics against guardrail thresholds; instant notification when a metric is trending toward a breach. <br> - **CXO Insights – Live event feed:** CXO dashboards get a "live activity" panel showing real-time operational events across accounts. |
| **How it differs from the other two** | Copilot and RAG are **interactive / on-demand** — a user asks a question and gets an answer. Real-Time Intelligence is **continuous / event-driven** — it monitors data streams 24/7 and acts automatically, even when no one is looking at a dashboard. |
| **Multi-tenant safety** | Eventstreams and KQL databases are created within Fabric workspaces. Per-tenant workspace isolation means each tenant's real-time data is physically separate. Data Activator triggers are workspace-scoped. |

---

### How the Three Pillars Work Together

```
┌────────────────────────────────────────────────────────────────────┐
│                        E360 User Interaction                       │
│                                                                    │
│   "What happened?"          "Why?" / "What if?"      "Alert me"    │
│   ┌─────────────┐          ┌─────────────────┐    ┌──────────────┐ │
│   │   Fabric    │          │  Azure OpenAI   │    │  Real-Time   │ │
│   │   Copilot   │          │  + RAG          │    │  Intelligence│ │
│   │             │          │                 │    │  + Activator │ │
│   │ Structured  │          │ Structured +    │    │  Streaming   │ │
│   │ data Q&A    │          │ Unstructured    │    │  data +      │ │
│   │ (tables,    │          │ data Q&A        │    │  automated   │ │
│   │ reports)    │          │ (docs + tables) │    │  triggers    │ │
│   └──────┬──────┘          └────────┬────────┘    └──────┬───────┘ │
│          │                          │                    │         │
│          └──────────────────────────┼────────────────────┘         │
│                                     │                              │
│                          ┌──────────▼───────────┐                  │
│                          │     OneLake          │                  │
│                          │  (single source of   │                  │
│                          │   truth for all AI)  │                  │
│                          └──────────────────────┘                  │
└────────────────────────────────────────────────────────────────────┘
```

| Question Type | Which Pillar Answers It | Example |
|---|---|---|
| "Show me last quarter's DQI trends" | **Fabric Copilot** (structured data already in Power BI / Lakehouse) | Ops Manager reviewing performance |
| "What does the SLA contract say about penalty clauses for Account X?" | **Azure OpenAI + RAG** (unstructured document retrieval + LLM reasoning) | Account SPOC preparing for a client review |
| "Notify me immediately if any client's DQI drops below 85" | **Real-Time Intelligence + Data Activator** (continuous monitoring + automated trigger) | Guardrails / proactive alerting |
| "Summarise today's escalations and suggest root causes" | **RAG + Copilot together** (RAG retrieves relevant docs; Copilot analyses the structured escalation data) | CXO daily briefing |

### AI Capabilities in the Platform

| Capability | Technology | Description |
|---|---|---|
| **Copilot for Data Engineering** | Fabric Copilot | Generate Spark code, pipeline shapes, data transformations from natural language |
| **Copilot for Power BI** | Fabric Copilot | Auto-generate reports, suggest visualisations, answer questions on datasets |
| **RAG-based Q&A on client data** | Azure OpenAI + Azure AI Search | Index client documents/data; answer business questions with grounded responses |
| **Anomaly Detection & Forecasting** | Fabric Notebooks (Python/ML) + Azure ML | Built into the Lakehouse; no data movement needed |
| **Data Activator (event-driven AI triggers)** | Microsoft Fabric Data Activator | Alert or trigger workflows when data conditions are met (e.g., KPI threshold breach) |
| **AI-enhanced ETL / data quality** | Fabric Data Factory + OpenAI | Use LLM to classify, clean, or enrich data during ingestion |
| **Natural language to KQL/SQL** | Fabric Copilot + Azure OpenAI | Let non-technical users query real-time or historical data with plain English |

### The "Fabric as AI Data Foundation" Story Points

1. **No data silos** – OneLake means every AI model trains on the same curated, governed data. No copying, no stale snapshots.
2. **Governance-first AI** – Microsoft Purview tracks what data AI has access to. Sensitivity labels prevent AI from surfacing confidential data to the wrong tenant.
3. **Incremental AI adoption** – Start with Power BI Copilot (low risk), progress to RAG Q&A, then predictive analytics. No big-bang.
4. **Azure OpenAI integration is native** – No third-party connectors. Fabric notebooks call Azure OpenAI via managed identity (no keys stored in code).
5. **Multi-tenant AI safety** – AI responses are workspace-scoped. Tenant A's Copilot cannot access Tenant B's data by design.

### Reference Links – AI & Fabric

| Resource |
|---|
| [Microsoft Fabric AI Features Overview](https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview) |
| [Fabric Copilot in Power BI](https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-introduction) |
| [Azure OpenAI on your data (RAG)](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/use-your-data) |
| [Fabric Data Activator](https://learn.microsoft.com/en-us/fabric/data-activator/data-activator-introduction) |
| [Fabric Lakehouse Overview](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview) |
| [AI + Analytics reference architecture (Azure)](https://learn.microsoft.com/en-us/azure/architecture/solution-ideas/articles/advanced-analytics-on-big-data) |
| [Responsible AI in Azure OpenAI](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/responsible-ai) |
| [Azure AI Search (for RAG grounding)](https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search) |
| [Implement RAG with Azure OpenAI](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview) |
| [Fabric Real-Time Intelligence Overview](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/overview) |
| [Fabric Eventstream](https://learn.microsoft.com/en-us/fabric/real-time-intelligence/event-streams/overview) |
| [KQL Overview (Kusto Query Language)](https://learn.microsoft.com/en-us/kusto/query/) |


---

<a name="closest-refs"></a>
## 8. Microsoft Reference Architectures Closest to E360

The E360 architecture is a **multi-tenant, analytics-heavy SaaS platform** with a layered Lakehouse pattern, RBAC, embedded BI, and an API gateway. The following Microsoft reference architectures are the closest matches and should be cited when discussing with Contoso.

### 8.1 Exact-Match Reference Architectures

| Architecture | Why It Matches E360 | Link |
|---|---|---|
| **Architect multitenant solutions on Azure** (full series) | The definitive guide. Covers all isolation models, identity, networking, data, compute, and cost for multi-tenant SaaS. | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview) |
| **Multitenant SaaS on Azure SQL – Design Patterns** | E360's per-client database with shared semantic layer maps directly to the "database-per-tenant" or "elastic pool" pattern described here. | [Link](https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns) |
| **Power BI Embedded – Multi-Tenancy** | E360 uses Power BI for persona dashboards. This guide covers workspace-per-tenant vs. RLS approaches in Power BI Embedded — directly applicable. | [Link](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy) |
| **Deployment Stamps pattern** | E360's Option 2 (IaC-automated per-client deployment) is essentially the Deployment Stamp pattern. | [Link](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) |
| **Multitenant – Storage and data approaches** | Covers OneLake folder isolation, shared vs. dedicated databases, sharding — all relevant to E360's Lakehouse layer. | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data) |

### 8.2 Layer-by-Layer Mapping to Microsoft References

| E360 Layer | Contoso's Components | Closest Microsoft Reference | Link |
|---|---|---|---|
| **Layer 0: Infrastructure** | Azure PaaS, ADLS Gen2, ClickHouse, VNet | Azure Landing Zone + Hub-Spoke | [Link](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/) |
| **Layer 1: Data & Integration** | ADF, Connectors, Key Vault | Data integration with ADF in multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data) |
| **Layer 2: Lakehouse Storage** | Delta Lake (Azure Native) / Iceberg (Contoso's proposal), Bronze/Silver/Gold medallion | Fabric Lakehouse architecture | [Link](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview) |
| **Layer 3: Semantic Config** | UDAL, Taxonomy, Metadata | Multitenant configuration & deployment | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/deployment-configuration) |
| **Layer 4: Compute & Logic** | dbt Core, Scoring Engine, Workflow | Compute approaches for multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/compute) |
| **Layer 5: API & Gateway** | API Gateway, GraphQL/REST, Auth | API Management in multitenant architectures | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/api-management) |
| **Layer 6: Presentation** | Power BI Embedded, Dashboards, Config UI | Power BI Embedded multi-tenancy | [Link](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy) |
| **Layer 8: Agentic AI** | Azure OpenAI, Semantic Kernel | Azure OpenAI RAG pattern | [Link](https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview) |
| **Cross-cutting: Identity** | Entra ID, RBAC/RLS | Identity in multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity) |
| **Cross-cutting: Governance** | Purview, dbt Lineage | Governance in multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/governance-compliance) |
| **Cross-cutting: Networking** | VNet, Private Endpoints | Networking in multitenant solutions | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/networking) |

### 8.3 Additional High-Value References

| Resource | Relevance | Link |
|---|---|---|
| **Multitenant architecture checklist** | Comprehensive gating checklist before go-live | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/checklist) |
| **Measure consumption in multitenant solutions** | How to charge-back / meter per-tenant costs (relevant for E360's SaaS pricing) | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/measure-consumption) |
| **Noisy Neighbor antipattern** | E360's shared compute risk — how to mitigate one tenant impacting others | [Link](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/noisy-neighbor/noisy-neighbor) |
| **AKS multi-tenancy operator guide** | For Option 2: namespace isolation, network policies, RBAC in AKS | [Link](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation) |
| **Subscription vending** | Automating per-tenant Azure subscription provisioning via Bicep/Terraform | [Link](https://learn.microsoft.com/en-us/azure/architecture/landing-zones/subscription-vending) |
| **SaaS Technical Foundations (Training)** | Microsoft Learn module covering SaaS architecture foundations | [Link](https://learn.microsoft.com/en-us/training/saas/saas-technical-foundations/) |
| **Fabric Copilot overview** | For the AI story in Layer 8 | [Link](https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview) |
| **Azure Well-Architected Framework** | Cross-cutting quality pillars (reliability, security, cost, ops, perf) | [Link](https://learn.microsoft.com/en-us/azure/well-architected/) |

---

## Quick Reference Card 

| Topic | Key Point | Link |
|---|---|---|
| **Microsoft's Recommendation** | **Azure Native (Fabric + PaaS) — strongly recommended** | *(see Section 4 trade-offs)* |
| Multi-tenancy guide | Full series from Microsoft | [Link](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview) |
| Fabric workspace isolation | One workspace per client | [Link](https://learn.microsoft.com/en-us/fabric/fundamentals/workspaces) |
| AKS multi-tenancy | Namespace isolation | [Link](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation) |
| Okta + Entra federation | SAML 2.0 / OIDC | [Link](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-saml-single-sign-on) |
| Landing zone | CAF Enterprise-Scale | [Link](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/enterprise-scale/architecture) |
| Fabric Copilot / AI | Copilot overview | [Link](https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview) |
| Deployment Stamp pattern | Scale-out model | [Link](https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp) |
| Azure Arc (hybrid K8s) | Multi-cloud K8s | [Link](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview) |
| Power BI Embedded multi-tenant | Workspace-per-tenant | [Link](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy) |
| Multitenant SaaS SQL patterns | DB-per-tenant / elastic pool | [Link](https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns) |
| Azure Well-Architected Framework | Quality pillars | [Link](https://learn.microsoft.com/en-us/azure/well-architected/) |

---
## 9. High level Architecture Diagrams (Mermaid)

---

### 9.1 E360 Plus – Recommended Azure Native Architecture (End-State)

This is the recommended target-state architecture built entirely on Azure Native services — Microsoft Fabric for analytics, Azure PaaS for platform services, and Azure OpenAI for AI capabilities.

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="End-State" 
     src="https://github.com/user-attachments/assets/c14cf103-2806-4377-bb52-cce801132040"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


---

### 9.2 Multi-Tenant Isolation Model

This diagram shows how tenant isolation works across the platform layers.

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Tenant Isolation Model" 
     src="https://github.com/user-attachments/assets/c0da4bbe-0531-4e79-b294-65cac840b7c1"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>

---

### 9.3 AI Data Flow – Three Pillars

This shows the data flow for each AI capability and how they connect to OneLake.

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Three Pillars" 
     src="https://github.com/user-attachments/assets/f31cf1c4-346e-4e07-a17c-c76e66979684"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>

---

### 9.4 Okta → Entra ID → E360 Authentication Flow

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Entra ID → E360 Authentication Flow" 
     src="https://github.com/user-attachments/assets/30ebe4da-2cca-4c22-9ce1-9d8ff60c99fe"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>

---

### 9.5 Landing Zone – Hub & Spoke Topology

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt=" Hub & Spoke Topology" 
     src="https://github.com/user-attachments/assets/56c45296-8060-49f9-8783-8313f1e6225c"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>

---
