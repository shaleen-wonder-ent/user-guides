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

<a name="current-state"></a>
## 0. E360 Current State & Proposed Vision

### 0.1 Current Architecture (E360 on AWS)

```
┌────────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                              │
│                                                                        │
│  ┌──────────┐ ┌─────────────┐ ┌──────────────┐ ┌──────────┐ ┌────────┐ │
│  │   RBAC   │ │  Modules    │ │    SMF       │ │ Ops Hub  │ │Portfol.│ │
│  │ Ops Roles│ │ OpsHub      │ │ Alerts/Esc.  │ │ Issues   │ │Risk/   │ │
│  │ Func.Role│ │ Portfolio   │ │ OpsRituals   │ │ Tasks    │ │Compli. │ │
│  │ Acct SPOC│ │ SMF/Insight │ │ Risks        │ │ Customer │ │Top 10  │ │
│  │ Default  │ │ Guardrails  │ │ Notifications│ │ Ops      │ │Transi. │ │
│  │          │ │ Benchmarking│ │ 1-on-1       │ │ People   │ │        │ │
│  │          │ │ DQI         │ │ Daily Huddle │ │ Ops Mod. │ │        │ │
│  │          │ │ Citizen Dev │ │ Guardrails   │ │ Ops Des. │ │        │ │
│  │          │ │ ADMIN       │ │ Bench Upload │ │ CXO Ins. │ │        │ │
│  │          │ │ Gen AI/Ins. │ │              │ │          │ │        │ │
│  └──────────┘ └─────────────┘ └──────────────┘ └──────────┘ └────────┘ │
│                                                                        │
│  ┌─────────────┐  ┌───────────────┐                                    │
│  │   Gen AI    │  │     DQI       │                                    │
│  │ Playground  │  │ DQ Index      │                                    │
│  │ Ops Designer│  │ Acct Landscape│                                    │
│  │ CXO Insights│  │               │                                    │
│  └─────────────┘  └───────────────┘                                    │
└────────────────────────────────────────────────────────────────────────┘
                              │
┌──────────────────────────────────────────────────────────────────────────┐
│       INTEGRATION LAYER (APIs – SOAP & REST, Third Party via MuleSoft)   │
└──────────────────────────────────────────────────────────────────────────┘
                              │
┌──────────────────────────────────────────────────────────────────────────┐
│                          DATA LAYER (AWS)                                │
│                                                                          │
│  Data Domains: Finance│Operations│People│Portfolio│Masters│Misc          │
│            ▼                                                             │
│  AWS S3 + AWS Data Services (single storage, shared across tenants)      │
│                                                                          │
│  Cross-cutting: Security│Meta Data│Data Catalogue│Data Governance        │
│                 Data Quality│AI/ML                                       │
└──────────────────────────────────────────────────────────────────────────┘

Auth: Okta  │  External: ServiceNow, ChatGPT, Power BI Portfolio
```

**Key Observations:**
- **Single-tenant monolith** – All clients share one AWS storage environment with only RBAC for segregation
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
3. What AWS services are currently in use beyond S3? (Lambda, RDS, Glue, Redshift, SageMaker, etc.)
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
2. For non-Azure clients, what are the specific constraints? (Contractual AWS-only? On-prem mandate? GCP preference?)
3. Is "cloud agnostic" a contractual/RFP requirement from Contoso's clients, or an internal strategic preference?
4. Does "agnostic" mean the **same binary** runs everywhere, or is it acceptable to have cloud-specific adapters behind a common interface?
5. Are there clients who need E360 deployed **in the client's own cloud subscription** (vs. Contoso-managed)?

**Technical Depth:**

6. The proposed architecture uses Azure Data Factory, ADLS Gen2, Key Vault, and Entra ID – these are Azure-specific. What is the abstraction strategy for non-Azure deployments? (e.g., ADF → Airflow, ADLS → S3, Key Vault → HashiCorp Vault, Entra → Keycloak?)
7. Is ClickHouse Cloud the confirmed analytical DB? (It is cloud-agnostic, which supports Option 2)
8. dbt Core is open-source and portable – is the team committed to dbt, or considering dbt Cloud (which adds SaaS dependency)?
9. For the Iceberg table format: is there a dependency on a specific catalog implementation (AWS Glue Catalog, Hive Metastore, Nessie)?
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

| Resource | URL |
|---|---|
| Microsoft Fabric Overview | https://learn.microsoft.com/en-us/fabric/get-started/microsoft-fabric-overview |
| Fabric Workspaces & Roles | https://learn.microsoft.com/en-us/fabric/fundamentals/workspaces |
| Fabric Security Overview | https://learn.microsoft.com/en-us/fabric/security/security-overview |
| Fabric Security Fundamentals | https://learn.microsoft.com/en-us/fabric/security/security-fundamentals |
| OneLake Security | https://learn.microsoft.com/en-us/fabric/onelake/security/fabric-onelake-security |
| Fabric Multi-Geo (Data Residency) | https://learn.microsoft.com/en-us/fabric/admin/service-admin-premium-multi-geo |
| Row-Level Security in Fabric | https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security |
| Fabric Permission Model | https://learn.microsoft.com/en-us/fabric/security/permission-model |
| Fabric Private Links | https://learn.microsoft.com/en-us/fabric/security/security-private-links-use |
| Microsoft Fabric Licenses & SKUs | https://learn.microsoft.com/en-us/fabric/enterprise/licenses |
| Power BI Embedded Multi-Tenancy | https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy |
| Azure API Management | https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts |
| SaaS Multi-Tenant on Azure – Architecture Guide | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview |
| Multitenant SaaS – Azure SQL patterns | https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns |
| Azure Architecture Center – Architect multitenant solutions | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/overview |
| **Tenancy Models for Multitenant Solutions** | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models |
| **SaaS & Multitenant Solution Architecture (Architecture Center)** | https://learn.microsoft.com/en-us/azure/architecture/guide/saas-multitenant-solution-architecture/ |
| **Microsoft Fabric Overview (Official Docs)** | https://learn.microsoft.com/en-us/fabric/fundamentals/microsoft-fabric-overview |
| **Azure SaaS Dev Kit** | https://azure.github.io/azure-saas/ |
| **Power BI Embedded for ISVs** | https://learn.microsoft.com/en-us/power-bi/developer/embedded/embedded-analytics-power-bi |
| **Microsoft Purview Governance** | https://learn.microsoft.com/en-us/purview/purview |
| **Fabric Workspace-per-Tenant Pattern** | https://learn.microsoft.com/en-us/fabric/security/workspace-identity |

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

| Resource | URL |
|---|---|
| AKS Multi-Tenancy Best Practices | https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation |
| AKS Cluster Isolation (Logical vs Physical) | https://learn.microsoft.com/en-us/azure/aks/cluster-multi-tenancy |
| Azure Architecture Center – AKS Baseline | https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks |
| Azure Architecture Center – Multi-Region AKS | https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-multi-region/aks-multi-cluster |
| Azure Container Apps (serverless alternative) | https://learn.microsoft.com/en-us/azure/container-apps/overview |
| Cloud-Agnostic App Design on Azure | https://learn.microsoft.com/en-us/azure/architecture/guide/design-principles/design-for-self-healing |
| Azure Architecture Center – Multitenant SaaS on AKS | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/compute |
| Kubernetes Network Policies | https://learn.microsoft.com/en-us/azure/aks/use-network-policies |
| AKS + External OIDC / Workload Identity | https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview |
| Azure Arc (unified multi-cloud K8s management) | https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview |
| Deployment Stamp Pattern | https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp |

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

| Dimension | Option 1: Azure Native (Fabric) | Option 2: Cloud-Agnostic (K8s) |
|---|---|---|
| **Cloud Lock-in** | High – Microsoft Fabric is Azure-only | Low – deploy anywhere |
| **Multi-Tenancy** | Native workspace isolation + OneLake RBAC | Namespace / node pool / cluster isolation |
| **Time-to-Value** | Fast – Fabric has analytics built-in | Slower – more custom build required |
| **Operational Complexity** | Low – Microsoft manages infrastructure | High – team manages K8s and all layers |
| **Configurability per Tenant** | Moderate – Fabric APIs + config DB | High – Helm values, config maps, feature flags |
| **Analytics Depth** | Very High – Lakehouse, Notebooks, Power BI, Copilot | Needs integration (Spark, Trino, Superset etc.) |
| **AI/ML Integration** | Native – Azure OpenAI + Fabric Copilot | Bring-your-own (Azure AI, Hugging Face, etc.) |
| **Data Residency / Sovereignty** | Supported via Fabric Multi-Geo | Supported via cloud region selection |
| **SSO / Identity** | Entra ID + B2B federation (Okta via SAML/OIDC) | Keycloak / OIDC proxy + Okta integration |
| **Cost Model** | Predictable – F SKU capacity pricing | Variable – compute + ops labor cost |
| **Compliance / Certifications** | Microsoft holds certs (SOC2, ISO 27001, HIPAA) | Must certify per cloud provider / per deployment |
| **Disaster Recovery** | Built-in Fabric reliability + Azure SLAs | Requires bespoke DR strategy |
| **Customization (client-specific logic)** | Config DB + API parameters + Power BI personalisation | Feature flags, per-tenant Helm overrides, custom code branches |
| **Vendor Risk** | Medium – Microsoft Fabric is strategic & heavily invested | Low – open-source stack, portable |

### Recommended Decision Framework

Use this to guide the Contoso conversation:

```
Is cloud agnosticism a HARD requirement?
    YES ──► Option 2 (K8s) is the only viable path
    NO  ──► Is Microsoft Azure the primary cloud?
              YES ──► Option 1 (Fabric) for analytics-heavy workloads
              MAYBE ──► Hybrid: Option 1 core + Option 2 escape hatch (Azure Arc)
```

### Hybrid Path (Worth Discussing)
A pragmatic middle ground:
- **Azure Arc** enables running AKS workloads anywhere (on-prem, GCP, AWS) under a single Azure management plane
- Fabric handles analytics; containerised services handle compute-heavy or cloud-neutral workloads
- Reference: [Azure Arc for Kubernetes](https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview)

### Detailed Pros and Cons by Key Dimension

#### 4.1 Scalability

**Option 1 (Azure-Native):**

- **Pro:** Azure PaaS services provide auto-scaling without manual intervention. Microsoft Fabric offers a unified compute and storage architecture supporting ETL, data warehousing, reporting, real-time analytics, and generative AI workloads within a single SaaS platform. The proposed architecture allows each client to have a separate database managed via a semantic layer, meaning new tenants can be added without reconfiguring the core infrastructure.
- **Con:** Scaling is constrained to Azure's service limits and regional availability. The ~20% of customers who cannot use Azure (due to policy or preference) would need a separate solution entirely, fragmenting the scaling model.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Kubernetes-based scaling is portable: the same cluster configuration and Helm charts can scale workloads on AKS, EKS, GKE, or on-prem infrastructure. No ceiling imposed by a specific cloud vendor's service quotas.
- **Con:** Auto-scaling must be configured manually (Horizontal Pod Autoscalers, Cluster Autoscaler). Capacity planning, node pool management, and performance tuning require dedicated DevOps expertise. As the number of tenants grows, managing isolated deployments (if using per-client instances as an interim approach) becomes operationally expensive.

> **Trade-off:** Option 1 provides effortless vertical and horizontal scalability within Azure but is inoperative outside it. Option 2 provides universal scalability but demands continuous operational investment. For Contoso's mixed client base (~80% Azure-compatible, ~20% not), a single scaling strategy cannot serve all clients — **a hybrid model is necessary**.

---

#### 4.2 Cost and Pricing Model

**Option 1 (Azure-Native):**

- **Pro:** Fabric's pricing is based on two main components: storage (OneLake) and compute, with calculators available to estimate costs based on data volume and transformation needs. Microsoft cites reference case studies showing cost advantages over other platforms for analytics workloads. The pay-as-you-go model can align costs with actual usage, avoiding upfront capital expenditure.
- **Con:** Azure consumption costs can escalate unpredictably with growing data volumes or compute-intensive analytics workloads. Additionally, Contoso's existing AWS investments (infrastructure, expertise, tooling) are largely non-transferable, creating a sunk-cost write-off during migration.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Leverages existing open-source tooling (Python, dbt Core, Iceberg) and avoids premium PaaS licensing. Infrastructure costs are more predictable (fixed VM/node costs). Contoso retains the option to deploy on whichever cloud offers the best pricing for a given client.
- **Con:** Operational overhead constitutes a significant hidden cost: Contoso must staff and train a team to manage Kubernetes clusters, perform upgrades, handle incident response, and maintain custom tooling — essentially operating as a SaaS provider. These labour costs may offset the savings from avoiding PaaS premiums.

> **Trade-off:** The cost comparison is not straightforward. Option 1 shifts spend from **people** (ops engineers) to **platform** (Azure fees); Option 2 does the reverse. The optimal choice depends on Contoso's internal cost structure and client pricing model. For clients subscribing to a Contoso-managed SaaS, Option 1's consumption model may be more efficient at scale. For clients requiring self-hosted deployments, Option 2 avoids double-billing for cloud services the client already pays for.

---

#### 4.3 Ease of Integration with Existing E360 Presentation Layer

**Option 1 (Azure-Native):**

- **Pro:** Power BI is natively integrated within Fabric, allowing seamless embedding of reports into custom application layers, with APIs available for further integration and automation. Since E360 already uses Power BI for drill-down views, migrating these into Fabric-hosted Power BI would be relatively low-friction. Contoso's proposed architecture already anticipates Power BI Premium/Embedded for persona-based dashboards, suggesting alignment with this path.
- **Con:** Replacing the current direct AWS S3 data access with calls to Fabric or ADLS Gen2 APIs requires backend refactoring. Integration of Contoso's Okta authentication with Microsoft Entra ID adds complexity in a mixed identity environment.

**Option 2 (Cloud-Agnostic):**

- **Pro:** This is the lowest-friction path for integration. The existing backend services run inside containers essentially unchanged, and the E360 front-end (Vue.js) interacts with them exactly as before. No new API paradigms, no new authentication protocols, no new data access patterns need to be introduced immediately.
- **Con:** The legacy architecture is preserved with all its limitations. Any new capabilities (e.g., automated data pipelines, enhanced analytics, admin UI) must be custom-built and integrated from scratch. The E360 portal continues interfacing with a set of self-managed services rather than one unified platform, increasing integration surface area over time.

> **Trade-off:** Option 2 offers the fastest integration path (weeks), while Option 1 offers the most capable long-term integration but requires months of refactoring. **Given the June 1, 2026 deadline, starting with Option 2's minimal-change approach and gradually introducing Option 1 components later is the pragmatic path.**

---

#### 4.4 Performance and Reliability

**Option 1 (Azure-Native):**

- **Pro:** Fabric provides a unified analytics engine supporting ETL, data warehousing, reporting, real-time analytics, and generative AI workloads within a single SaaS architecture. Data doesn't need to move between systems for different analytical tasks, reducing latency. Azure-managed services come with enterprise-grade SLAs, automated failover, and built-in high availability.
- **Con:** Performance tuning options are more constrained — Contoso depends on Microsoft's platform optimisations rather than having direct control over compute allocation, caching strategies, or query optimisation at the infrastructure level.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Contoso retains full control over performance optimisation. The proposed architecture includes ClickHouse Cloud for analytical acceleration, which is a high-performance columnar database suitable for real-time analytics. This could deliver excellent query performance for E360's reporting workloads. Additionally, isolated per-client deployments mean one client's workload spikes cannot impact another's performance.
- **Con:** Without managed services, achieving high availability and disaster recovery requires significant DevOps investment (container health checks, pod disruption budgets, cross-zone replication). Performance benchmarking and optimisation is Contoso's responsibility, consuming engineering bandwidth.

> **Trade-off:** Option 1 provides predictable, out-of-the-box performance suitable for most analytical workloads. Option 2 has a higher performance ceiling (especially with specialised tools like ClickHouse) but requires more engineering effort to realise. For E360's initial v1 launch, Option 2's existing performance profile is likely adequate; Option 1's benefits become more compelling as data volumes and analytical complexity grow post-launch.

---

#### 4.5 Multi-Tenancy Support

> *This is arguably the most critical dimension, given that E360's current system stores all client data in a single storage environment with only RBAC for access control — a model that "limits the ability to provide true data segregation and raises concerns for clients about data privacy and security".*

**Option 1 (Azure-Native):**

- **Pro:** The proposed architecture envisions each client having a separate database with a semantic layer managing metadata, access hierarchies, and data transformations, ensuring strict data segregation and privacy. Microsoft Fabric supports multi-tenant SaaS deployments on Azure, with separate workspaces or lakehouses per tenant providing physical data isolation. Configuration of data pipelines and onboarding of new clients can be managed through the Fabric portal or via API integrations.
- **Con:** Implementing per-tenant workspaces in Fabric still requires significant design work (metadata management, tenant routing, access control mapping). The existing monolithic data model must be decomposed — this is a substantial engineering effort regardless of the platform.

**Option 2 (Cloud-Agnostic):**

- **Pro:** There are well-documented ways to achieve multi-tenancy at the AKS level with Namespace Isolation, Network Policies, resource quotas, and RBAC (see [AKS Multitenancy Guide](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation)). Additionally, the team could bring their as-is architecture on Azure, make it SaaS-ready and add IaC to automate the deployment for any net-new customers — meaning each new client gets an automated fresh deployment via Infrastructure-as-Code.
- **Con:** Kubernetes-level isolation is **not** application-level multi-tenancy. The E360 codebase itself must be modified to support per-tenant data partitioning, configuration, and access control — this development burden falls entirely on Contoso.

> **Trade-off:** Both options require substantial multi-tenancy engineering at the application layer. Option 1 provides platform-level support (separate databases, workspaces, APIs for tenant management) that accelerates the work. Option 2 provides infrastructure-level isolation (namespaces, network policies) as a simpler starting point but leaves the harder application-level work to Contoso. **Practical implication: Option 2's per-client deployment model (IaC-automated) can serve as an interim multi-tenancy mechanism while true in-app multi-tenancy is developed iteratively.**

---

#### 4.6 Configurability and Admin UI Potential

**Option 1 (Azure-Native):**

- **Pro:** Configuration of data pipelines and client onboarding can be managed through the Fabric portal or via API integrations, enabling custom UI development for client management within the Contoso-branded application. The E360 team can build an admin interface that programmatically calls Fabric APIs to create new tenant workspaces, configure data sources, and manage access — leveraging existing platform capabilities rather than building from scratch.
- **Con:** Some configuration may only be possible through Azure's own portal rather than via APIs, which would force administrators to leave the E360 interface. The learning curve for Fabric APIs and administration adds upfront development time.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Complete freedom to design the admin experience to match Contoso's exact specifications. The E360 team can build microservices for configuration management, storing tenant settings in a central database and exposing them via the presentation layer — fulfilling the requirement that ETL, Auth, and User Provisioning all be manageable via the Presentation Layer.
- **Con:** Every admin function must be designed and coded from scratch: pipeline configuration, monitoring, user administration, access controls, data source management. This is a significant development investment.

> **Trade-off:** Option 1 provides building blocks (APIs, portal features) that reduce custom development. Option 2 provides unlimited customisation at the cost of higher development effort. Given the stated need for a low-code/no-code configuration UI to enable rapid client onboarding and reduce dependency on development resources, the E360 team will need substantial custom UI work regardless of which option is chosen. **Option 1 may reduce the backend complexity behind that UI.**

---

#### 4.7 Cloud Portability and Vendor Lock-In

**Option 1 (Azure-Native):**

- **Pro:** For the ~80% of customers subscribing to Contoso's SaaS offering, Azure lock-in is a non-issue — they are consuming E360 as a service, not managing infrastructure. Contoso benefits from Microsoft's partnership support and potential co-investment.
- **Con:** Fabric is not cloud-agnostic and requires Azure for deployment — though it supports interoperability with data residing in other clouds, clients strictly on non-Azure platforms would require alternative solutions. This directly contradicts Contoso's requirement to maintain one codebase across cloud providers. The ~20% of clients needing non-Azure deployments would require a separate solution track.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Fully aligned with Contoso's design principle. Containerised E360 can be deployed on AKS, EKS (AWS), or any Kubernetes-compatible environment. Contoso explicitly stated they are "not open for any re-architecture or platform-sticky components", making this the philosophically aligned choice.
- **Con:** Maintaining cloud neutrality imposes constraints: the team must avoid using any cloud-specific managed services (or abstract them behind interfaces), which limits the use of optimised platform features. This can slow development and reduce the sophistication of the analytics layer compared to what Fabric could offer.

> **Trade-off:** This is the **most polarising dimension**. Option 1 maximises value for the majority (80%) at the expense of the minority (20%). Option 2 serves everyone equally but at a higher operational cost for all. **The hybrid approach resolves this: use containerised E360 as the universal base, and offer Fabric-enhanced analytics as a value-add for Azure-hosted customers.**

---

#### 4.8 Operational Complexity and Maintainability

**Option 1 (Azure-Native):**

- **Pro:** Offloads significant operational burden to Microsoft — patching, scaling, backups, uptime monitoring, and security updates for Fabric and Azure PaaS are handled by the platform. Fewer moving parts for Contoso to manage day-to-day.
- **Con:** Contoso's team must acquire new skills in Azure networking and deployment. Until this ramp-up is complete, Contoso is dependent on Microsoft for troubleshooting and optimisation. Providing a list of team members who require Azure portal access for hands-on familiarisation is an identified action item.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Familiar operational model — the same team that manages E360 on AWS today can manage it on AKS with transferable skills. Existing monitoring, logging, and CI/CD practices can be adapted rather than replaced. Using IaC to automate deployment for net-new customers provides a scalable operational model.
- **Con:** Running Kubernetes at production scale is resource-intensive: cluster maintenance, node upgrades, security patching, container image management, and incident response all fall on Contoso's team. As the customer base grows, operational overhead scales with it — Contoso effectively becomes a managed infrastructure provider alongside being a software vendor.

> **Trade-off:** Option 1 trades operational simplicity for platform dependency and a learning curve. Option 2 trades operational autonomy for ongoing labour intensity. **Engaging an empanelled partner to assist with Azure environment setup partially mitigates both risks by providing expert support during the transition.**

---

#### 4.9 Security and Compliance

**Option 1 (Azure-Native):**

- **Pro:** The proposed architecture includes Azure Key Vault for secrets and credential management and Azure PaaS ADLS Gen2 with VNet integration for network-level security. Azure provides built-in encryption, compliance certifications, and identity management via Microsoft Entra ID. Multi-tenant data isolation can be enforced at the platform level (separate storage containers, database-level security). The proposed design includes a data governance layer with metadata management and data quality controls.
- **Con:** Integration of Contoso's existing Okta authentication with Microsoft Entra ID requires careful federation design to avoid gaps in access control during migration. Clients subject to specific data sovereignty regulations may resist having data stored in Microsoft-managed Azure regions.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Contoso retains complete control over security implementation. For clients with strict regulatory requirements, Contoso can deploy E360 within the client's own subscription or on-premises environment, giving the client full control over data residency, network policies, and access controls. Kubernetes Namespace Isolation, Network Policies, and RBAC provide infrastructure-level security boundaries between tenants.
- **Con:** Contoso must implement, test, and certify all security controls themselves. Container image scanning, runtime security monitoring, network segmentation, secret management, and audit logging must all be built or procured separately. The greater number of self-managed components increases the attack surface.

> **Trade-off:** Option 1 provides a pre-certified security foundation suitable for most enterprise requirements. Option 2 provides maximum flexibility for clients with bespoke security needs. **For regulated industries (which many of Contoso's 800 client programmes likely serve), the ability to deploy within a client's own security perimeter (Option 2) could be a decisive differentiator.**

---

#### 4.10 Future-Proofing and Innovation Potential

**Option 1 (Azure-Native):**

- **Pro:** Direct alignment with Microsoft's innovation roadmap, including Fabric's ongoing enhancements for generative AI workloads. Contoso's broader collaboration with Microsoft on Azure OpenAI Service positions E360 to incorporate advanced AI capabilities as they mature. Microsoft Fabric was proposed as the core analytics platform specifically highlighting its integrated ETL, data warehousing, real-time analytics, and Power BI experiences, with the ability to support multi-tenant SaaS deployments on Azure.
- **Con:** E360's technical evolution becomes coupled to Microsoft's product roadmap. Features outside Azure's ecosystem (e.g., new open-source analytics frameworks) become harder to adopt.

**Option 2 (Cloud-Agnostic):**

- **Pro:** Technology independence allows adoption of the best available tools as they emerge. The proposed architecture already includes forward-looking open-source technologies: dbt Core for transformations, Iceberg REST Catalog / Hive Metastore for an open table format with bronze-silver-gold data patterns, and ClickHouse Cloud for analytical acceleration. These are vendor-neutral, community-supported technologies that can evolve independently of any cloud provider.
- **Con:** Contoso's team must independently evaluate, adopt, and integrate new technologies — a resource-intensive process. Without a platform vendor driving innovation, the pace of feature advancement depends entirely on Contoso's engineering investment.

> **Trade-off:** Option 1 provides accelerated innovation through platform leverage at the cost of vendor dependency. Option 2 provides innovation freedom at the cost of engineering effort. **Notably, the proposed future architecture already blends both approaches — using Azure-native services (Data Factory, ADLS Gen2, Key Vault) alongside open-source tools (dbt Core, Iceberg, ClickHouse). This hybrid technology selection suggests Contoso's own architects envision a middle path.**

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

**Honest Assessment:**
- Option 1 (Fabric) does **not** satisfy cloud agnosticism in its current form. Microsoft Fabric is Azure-specific.
- Option 2 (K8s) satisfies this if: (a) container images have no Azure SDK hard dependencies, (b) storage is abstracted behind a provider interface (e.g., using Apache Arrow Flight, Delta Lake), (c) identity uses portable OIDC rather than Entra ID exclusively.
- **Mitigation for Option 1:** Wrap Fabric interactions in an abstraction layer/adapter interface so that a future provider could be swapped, even if not immediately portable.

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

1. **Is cloud agnosticism a contractual requirement, or a preference?** – This determines whether Option 2 is mandatory.
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

| Resource | URL |
|---|---|
| **Azure Landing Zone – Official Overview** | https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/ |
| **CAF Enterprise-Scale Landing Zone** | https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/enterprise-scale/architecture |
| **Multitenant SaaS – Architectural approaches overview** | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview |
| **Multitenant – Compute approaches** | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/compute |
| **Multitenant – Networking approaches** | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/networking |
| **Multitenant – Identity approaches** | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity |
| **Multitenant – Deployment & config** | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/deployment-configuration |
| **Multitenant – Cost management** | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/cost-management-allocation |
| **Multitenant Checklist** | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/checklist |
| **SaaS tenancy models (SQL)** | https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns |
| **Deployment Stamp pattern** | https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp |
| **Hub-Spoke Network Topology** | https://learn.microsoft.com/en-us/azure/architecture/networking/architecture/hub-spoke |
| **AKS Baseline Landing Zone** | https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks/baseline-aks |
| **Azure Policy for Landing Zones** | https://learn.microsoft.com/en-us/azure/governance/policy/overview |
| **Subscription vending / tenant onboarding (bicep)** | https://learn.microsoft.com/en-us/azure/architecture/landing-zones/subscription-vending |

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

### The Narrative

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

| Resource | URL |
|---|---|
| Microsoft Fabric AI Features Overview | https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview |
| Fabric Copilot in Power BI | https://learn.microsoft.com/en-us/power-bi/create-reports/copilot-introduction |
| Azure OpenAI on your data (RAG) | https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/use-your-data |
| Fabric Data Activator | https://learn.microsoft.com/en-us/fabric/data-activator/data-activator-introduction |
| Fabric Lakehouse Overview | https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview |
| AI + Analytics reference architecture (Azure) | https://learn.microsoft.com/en-us/azure/architecture/solution-ideas/articles/advanced-analytics-on-big-data |
| Responsible AI in Azure OpenAI | https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/responsible-ai |
| Azure AI Search (for RAG grounding) | https://learn.microsoft.com/en-us/azure/search/search-what-is-azure-search |
| Implement RAG with Azure OpenAI | https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview |

---

<a name="closest-refs"></a>
## 8. Microsoft Reference Architectures Closest to E360

The E360 architecture is a **multi-tenant, analytics-heavy SaaS platform** with a layered Lakehouse pattern, RBAC, embedded BI, and an API gateway. The following Microsoft reference architectures are the closest matches and should be cited when discussing with Contoso.

### 8.1 Exact-Match Reference Architectures

| Architecture | Why It Matches E360 | Link |
|---|---|---|
| **Architect multitenant solutions on Azure** (full series) | The definitive guide. Covers all isolation models, identity, networking, data, compute, and cost for multi-tenant SaaS. | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview |
| **Multitenant SaaS on Azure SQL – Design Patterns** | E360's per-client database with shared semantic layer maps directly to the "database-per-tenant" or "elastic pool" pattern described here. | https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns |
| **Power BI Embedded – Multi-Tenancy** | E360 uses Power BI for persona dashboards. This guide covers workspace-per-tenant vs. RLS approaches in Power BI Embedded — directly applicable. | https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy |
| **Deployment Stamps pattern** | E360's Option 2 (IaC-automated per-client deployment) is essentially the Deployment Stamp pattern. | https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp |
| **Multitenant – Storage and data approaches** | Covers OneLake folder isolation, shared vs. dedicated databases, sharding — all relevant to E360's Lakehouse layer. | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data |

### 8.2 Layer-by-Layer Mapping to Microsoft References

| E360 Layer | Contoso's Components | Closest Microsoft Reference | Link |
|---|---|---|---|
| **Layer 0: Infrastructure** | Azure PaaS, ADLS Gen2, ClickHouse, VNet | Azure Landing Zone + Hub-Spoke | https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/landing-zone/ |
| **Layer 1: Data & Integration** | ADF, Connectors, Key Vault | Data integration with ADF in multitenant solutions | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data |
| **Layer 2: Lakehouse Storage** | Iceberg, Bronze/Silver/Gold, ClickHouse | Fabric Lakehouse architecture | https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-overview |
| **Layer 3: Semantic Config** | UDAL, Taxonomy, Metadata | Multitenant configuration & deployment | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/deployment-configuration |
| **Layer 4: Compute & Logic** | dbt Core, Scoring Engine, Workflow | Compute approaches for multitenant solutions | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/compute |
| **Layer 5: API & Gateway** | API Gateway, GraphQL/REST, Auth | API Management in multitenant architectures | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/api-management |
| **Layer 6: Presentation** | Power BI Embedded, Dashboards, Config UI | Power BI Embedded multi-tenancy | https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy |
| **Layer 8: Agentic AI** | Azure OpenAI, Semantic Kernel | Azure OpenAI RAG pattern | https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview |
| **Cross-cutting: Identity** | Entra ID, RBAC/RLS | Identity in multitenant solutions | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity |
| **Cross-cutting: Governance** | Purview, dbt Lineage | Governance in multitenant solutions | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/governance-compliance |
| **Cross-cutting: Networking** | VNet, Private Endpoints | Networking in multitenant solutions | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/networking |

### 8.3 Additional High-Value References

| Resource | Relevance | Link |
|---|---|---|
| **Multitenant architecture checklist** | Comprehensive gating checklist before go-live | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/checklist |
| **Measure consumption in multitenant solutions** | How to charge-back / meter per-tenant costs (relevant for E360's SaaS pricing) | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/measure-consumption |
| **Noisy Neighbor antipattern** | E360's shared compute risk — how to mitigate one tenant impacting others | https://learn.microsoft.com/en-us/azure/architecture/antipatterns/noisy-neighbor/noisy-neighbor |
| **AKS multi-tenancy operator guide** | For Option 2: namespace isolation, network policies, RBAC in AKS | https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation |
| **Subscription vending** | Automating per-tenant Azure subscription provisioning via Bicep/Terraform | https://learn.microsoft.com/en-us/azure/architecture/landing-zones/subscription-vending |
| **SaaS Technical Foundations (Training)** | Microsoft Learn module covering SaaS architecture foundations | https://learn.microsoft.com/en-us/training/saas/saas-technical-foundations/ |
| **Fabric Copilot overview** | For the AI story in Layer 8 | https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview |
| **Azure Well-Architected Framework** | Cross-cutting quality pillars (reliability, security, cost, ops, perf) | https://learn.microsoft.com/en-us/azure/well-architected/ |

---

## Quick Reference Card (for the meeting)

| Topic | Key Point | Link |
|---|---|---|
| Multi-tenancy guide | Full series from Microsoft | https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview |
| Fabric workspace isolation | One workspace per client | https://learn.microsoft.com/en-us/fabric/fundamentals/workspaces |
| AKS multi-tenancy | Namespace isolation | https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation |
| Okta + Entra federation | SAML 2.0 / OIDC | https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-saml-single-sign-on |
| Landing zone | CAF Enterprise-Scale | https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/enterprise-scale/architecture |
| Fabric Copilot / AI | Copilot overview | https://learn.microsoft.com/en-us/fabric/fundamentals/copilot-fabric-overview |
| Deployment Stamp pattern | Scale-out model | https://learn.microsoft.com/en-us/azure/architecture/patterns/deployment-stamp |
| Azure Arc (hybrid K8s) | Multi-cloud K8s | https://learn.microsoft.com/en-us/azure/azure-arc/kubernetes/overview |
| Power BI Embedded multi-tenant | Workspace-per-tenant | https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy |
| Multitenant SaaS SQL patterns | DB-per-tenant / elastic pool | https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns |
| Azure Well-Architected Framework | Quality pillars | https://learn.microsoft.com/en-us/azure/well-architected/ |

---

