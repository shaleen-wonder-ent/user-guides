# Ep360 → Ep360⁺ˡᵘˢ: Modernization Advisory

## Table of Contents

- [1. Key Discovery Questions](#1-key-discovery-questions-to-ask-the-customer)
  - [A. AWS → Azure Migration + Multi-Tenancy](#section-a-aws--azure-migration--multi-tenancy)
  - [B. Hyperscaler-Agnostic Architecture](#section-b-hyperscaler-agnostic-architecture)
- [2. High-Level Target Architecture Discussion](#2-high-level-target-architecture-discussion)
  - [What the Proposed Architecture Gets Right](#what-their-proposed-architecture-gets-right-)
  - [Gaps & Recommendations](#gaps--recommendations-to-discuss)
    - [A. Multi-Tenancy Architecture Overlay](#a-multi-tenancy-architecture-overlay)
    - [B. Hyperscaler-Agnostic Abstraction](#b-hyperscaler-agnostic-abstraction-architecture)
  - [Summary Discussion Framework](#summary-what-to-take-into-the-discussion)
- [3. Microsoft Reference Links](#3-microsoft-reference-links-for-multi-tenancy-architecture)

---

## 1. Key Discovery Questions to Ask the Customer

### Section A: AWS → Azure Migration + Multi-Tenancy

| Area | Questions to Ask |
|---|---|
| **Tenant Model** | How many tenants (customers/business units) will this serve? Is it B2B (each client = tenant) or internal BU-based multi-tenancy? |
| **Data Isolation** | What level of data isolation is required per tenant — shared database with row-level security, database-per-tenant, or schema-per-tenant? (Regulatory/compliance drivers?) |
| **Identity & Access** | Currently on **Okta** — will tenants bring their own IdP (BYOI), or will everyone federate through a single **Microsoft Entra ID** instance? How will RBAC roles (Ops Roles, Functional, Account SPOC, Default) map to Entra RBAC/RLS? |
| **Integration Layer** | Current stack uses **MuleSoft** for SOAP/REST/3rd-party integrations — is there appetite to replace MuleSoft with **Azure API Management (APIM)**, or will MuleSoft remain as middleware? |
| **ServiceNow / SMF** | The **ServiceNow SMF** module (OpsRituals, SLC/COC, Alerts) is deep — will this remain a SaaS dependency, or are parts being internalized? How does tenant-level config work for SMF? |
| **Data Migration** | What databases are in the current AWS Data Layer (DynamoDB, S3, Redshift, Kinesis)? What's the data volume per tenant and total? What's the acceptable migration downtime window? |
| **Gen AI** | Current Gen AI module (Playground, Ops Designer, CXO Insights) uses ChatGPT — will this shift to **Azure OpenAI Service**, and will each tenant get isolated model deployments or shared with tenant-scoped prompts? |
| **Ops Hub** | Ops Hub is currently AWS-native — what are the dependencies (Lambda, SNS, SQS, EventBridge)? What's the effort to re-platform to Azure equivalents? |
| **Compliance** | Any data residency requirements per tenant (geo-fencing)? SOC2, ISO 27001, GDPR applicability? |
| **Cutover Strategy** | Big-bang vs. phased migration? Can they run hybrid (AWS + Azure) during transition? |
| **SLA / Performance** | Per-tenant SLA requirements? Noisy-neighbor concerns — do some tenants require dedicated compute? |

### Section B: Hyperscaler-Agnostic Architecture

| Area | Questions to Ask |
|---|---|
| **Motivation** | Why hyperscaler-agnostic — is it client demand, risk mitigation, cost optimization, or regulatory? |
| **Scope of Agnosticism** | Full stack (compute, data, AI, identity) or only specific layers (e.g., data & compute, but keep identity on Entra)? |
| **Kubernetes** | Is the team experienced with Kubernetes / container orchestration? Are they prepared for **AKS / EKS / GKE** portability? |
| **Data Layer Portability** | The proposed architecture already shows **Apache Iceberg + ClickHouse** — great choices. But is the intent to run these on any cloud, or is cloud-managed preferred (e.g., Azure-managed Iceberg via Fabric, Databricks)? |
| **IaC & GitOps** | What's the current Infrastructure-as-Code maturity? Terraform/Pulumi/CloudFormation? Are they ready for a Terraform-first multi-cloud approach? |
| **CI/CD** | What's the current pipeline tooling? Is **GitHub Actions** or another CI/CD platform acceptable as the cloud-neutral orchestrator? |
| **Vendor Lock-in Tolerance** | Which managed services are acceptable to keep cloud-specific (e.g., Key Vault, monitoring) vs. must be abstracted (e.g., secrets → HashiCorp Vault)? |
| **Abstraction Layer** | Layer 3 shows **UDAL (Unified Data Abstraction Layer)** and **Abstraction** — how far along is this? Is it just for data, or also compute and API routing? |
| **Cost Model** | Have they modeled the cost premium of cloud-agnostic vs. cloud-native? (Typically 15–30% overhead) |
| **Operational Maturity** | Do they have platform engineering / SRE teams to manage multi-cloud infra, or do they need managed services? |

---

## 2. High-Level Target Architecture Discussion

### What Their Proposed Architecture Gets Right ✅

The Ep360⁺ˡᵘˢ proposed diagram makes several strong architectural choices:

- **Layer 0 (Infra):** Azure PaaS + ADLS Gen2 + VNet — solid Azure-native foundation
- **Layer 1 (Integration):** Azure Data Factory + Key Vault — good managed service choices
- **Layer 2 (Storage):** **Iceberg + ClickHouse + Medallion (Bronze/Silver/Gold)** — *excellent* for hyperscaler portability. Iceberg's open table format is the right bet.
- **Layer 3 (Semantic):** UDAL + Taxonomy Designer + Metadata — strong abstraction intent
- **Layer 4 (Compute):** dbt Core for transformations — cloud-agnostic, great choice
- **Layer 5 (API):** API Gateway + GraphQL/REST — standard and portable
- **Layer 6 (Presentation):** Power BI Premium/Embedded — solid for Azure-first, but not cloud-agnostic
- **Layer 8 (AI):** Agentic Intelligence (de-scoped for MVP) — smart to defer
- **Cross-cutting:** Entra ID (RBAC/RLS), Azure Monitor, dbt Lineage — good governance thinking

---

### Gaps & Recommendations to Discuss

### A. Multi-Tenancy Architecture Overlay

The proposed diagram **does not explicitly address multi-tenancy**. This is the biggest gap. The following layer should be introduced:

```
┌─────────────────────────────────────────────────────────────┐
│                   TENANT RESOLUTION LAYER                    │
│                                                              │
│  ┌──────────┐   ┌──────────────┐   ┌─────────────────────┐  │
│  │ Tenant   │──▶│ Tenant       │──▶│ Tenant Config       │  │
│  │ Router   │   │ Context      │   │ Store               │  │
│  │ (APIM /  │   │ Propagation  │   │ (feature flags,     │  │
│  │ Gateway) │   │ (JWT claims, │   │  branding, limits,  │  │
│  │          │   │  headers)    │   │  data partitions)   │  │
│  └──────────┘   └──────────────┘   └─────────────────────┘  │
│                                                              │
│  Isolation Strategy per Layer:                               │
│  ├─ Compute:  Shared clusters, namespace-per-tenant (K8s)   │
│  ├─ Data:     Shared Iceberg catalog, schema-per-tenant     │
│  │            OR Row-Level Security via tenant_id column     │
│  ├─ AI/LLM:  Shared Azure OpenAI, tenant-scoped prompts    │
│  │            with data isolation in vector stores           │
│  ├─ API:     Tenant-aware rate limiting & throttling (APIM) │
│  └─ UI:      Tenant-branded Power BI Embedded workspaces    │
└─────────────────────────────────────────────────────────────┘
```

**Key multi-tenancy decisions by layer:**

| Layer | Recommendation |
|---|---|
| **Layer 0 (Infra)** | Single Azure subscription with resource groups per environment, NOT per tenant. Tenants separated logically, not physically (unless premium tier). |
| **Layer 1 (Data Integration)** | ADF pipelines parameterized by `tenant_id`. Separate Key Vault secrets per tenant. |
| **Layer 2 (Lakehouse)** | Iceberg namespace-per-tenant in the catalog. Bronze/Silver/Gold medallion partitioned by `tenant_id`. ClickHouse with tenant-aware partitioning. |
| **Layer 3 (Semantic)** | UDAL must be tenant-context-aware — taxonomy and metadata per tenant. |
| **Layer 4 (Compute)** | Tenant-scoped dbt projects or dbt variables for tenant isolation. Composite Scoring Engine needs tenant-aware models. |
| **Layer 5 (API)** | Azure APIM with tenant-based subscription keys, rate policies, and routing. GraphQL resolvers must enforce tenant context. |
| **Layer 6 (UI)** | Power BI Embedded with RLS per tenant. Persona Dashboards filtered by tenant context. |
| **Identity** | Entra ID with **multi-tenant app registrations**. Each tenant's IdP federated into Entra. Tenant claim in JWT propagated across all layers. |

---

### B. Hyperscaler-Agnostic Abstraction Architecture

```
┌───────────────────────────────────────────────────────────────┐
│              CLOUD ABSTRACTION STRATEGY                        │
│                                                                │
│  Tier 1: FULLY PORTABLE (no cloud dependency)                 │
│  ├─ Apache Iceberg (storage format)        ✅ Already planned │
│  ├─ ClickHouse (OLAP engine)               ✅ Already planned │
│  ├─ dbt Core (transformation)              ✅ Already planned │
│  ├─ Kubernetes (compute orchestration)     ⚠️  Not in diagram │
│  ├─ Terraform / OpenTofu (IaC)             ⚠️  Not in diagram │
│  ├─ GraphQL/REST APIs                      ✅ Already planned │
│  └─ OpenTelemetry (observability)          ⚠️  Not in diagram │
│                                                                │
│  Tier 2: ABSTRACTED (cloud-managed, swappable)                │
│  ├─ Object Storage: ADLS Gen2 / S3 / GCS   (behind Iceberg)  │
│  ├─ Secrets: Key Vault / Secrets Mgr / HashiCorp Vault        │
│  ├─ API Gateway: APIM / AWS API GW / Kong                     │
│  ├─ Identity: Entra / Okta / Auth0  (use OIDC standard)      │
│  └─ Monitoring: Azure Monitor / CloudWatch / Grafana+Prom     │
│                                                                │
│  Tier 3: CLOUD-SPECIFIC (accept lock-in, high value)          │
│  ├─ Power BI Embedded (presentation)       — Azure only       │
│  ├─ Azure Data Factory (orchestration)     — swap w/ Airflow  │
│  └─ Azure OpenAI (AI layer)               — swap w/ OpenAI   │
│                                                                │
│  ABSTRACTION APPROACH:                                         │
│  ├─ Provider Interface Pattern for infra services              │
│  ├─ Iceberg REST Catalog as the universal data contract        │
│  ├─ Container-first compute (AKS/EKS/GKE portable)            │
│  └─ Config-driven cloud selection (not code changes)           │
└───────────────────────────────────────────────────────────────┘
```

**What's Missing for True Hyperscaler Agnosticism:**

| Gap | Recommendation |
|---|---|
| **Compute Orchestration** | Add **Kubernetes** (AKS today, EKS/GKE tomorrow) as the compute substrate. Layer 4 services (Transformation Engine, Scoring Engine, Workflow Workbench) should be containerized microservices on K8s. |
| **Infrastructure as Code** | **Terraform with cloud-specific modules** — one codebase, multiple providers. Not shown anywhere in the proposed diagram. |
| **Observability** | Replace Azure Monitor dependency with **OpenTelemetry → Grafana stack** for portability. Keep Azure Monitor as one backend, but instrument with OTel. |
| **CI/CD** | Use **GitHub Actions** as the cloud-neutral CI/CD platform. |
| **API Gateway** | For true portability, consider **Kong** or **Envoy** on K8s instead of Azure APIM. Or use APIM as the Azure deployment with an abstraction layer to swap. |
| **Power BI Lock-in** | Power BI Embedded is Azure-only. For cloud-agnostic, an alternative path via **Apache Superset / Metabase / custom React dashboards** is needed. Recommendation: keep Power BI as primary but design the semantic layer (Layer 3 UDAL) so any BI tool can plug in. |
| **AI Layer** | The Agentic Intelligence layer (de-scoped) should plan for **LiteLLM or a similar proxy** to abstract Azure OpenAI / OpenAI / Anthropic / Bedrock behind a uniform interface. |

---

## Summary: What to Take Into the Discussion

```
┌─────────────────────────────────────────────────────────────┐
│         MODERNIZATION DISCUSSION FRAMEWORK                   │
│                                                              │
│  1. MULTI-TENANCY                                            │
│     → Tenant isolation model (shared vs. siloed)             │
│     → Identity federation (Okta → Entra multi-tenant)        │
│     → Data partitioning strategy (Iceberg + tenant_id)       │
│     → Noisy-neighbor protection & per-tenant SLAs            │
│     → Tenant onboarding automation                           │
│                                                              │
│  2. CLOUD PORTABILITY                                        │
│     → What's already portable: Iceberg, ClickHouse, dbt ✅  │
│     → What needs abstraction: Identity, Gateway, Monitoring  │
│     → What to accept as lock-in: Power BI, ADF (with plan)  │
│     → Missing pieces: K8s, Terraform, OTel, LLM proxy       │
│     → Cost/complexity trade-off of full agnosticism          │
│                                                              │
│  3. MIGRATION SEQUENCING (Recommended Phases)                │
│     → Phase 1: Data layer (Iceberg lakehouse on Azure)       │
│     → Phase 2: Compute & API (containerize, deploy on AKS)  │
│     → Phase 3: Presentation (Power BI Embedded + RLS)        │
│     → Phase 4: AI layer (Azure OpenAI + tenant isolation)    │
│     → Phase 5: Cloud abstraction hardening                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Microsoft Reference Links for Multi-Tenancy Architecture

The following Microsoft Learn links are curated and mapped to the Ep360 architecture layers. These are the closest matches to the customer's current and proposed stack.

### 🏗️ Overall Multi-Tenant Architecture (Start Here)

| Resource | Relevance to Ep360 |
|---|---|
| [Architect Multitenant Solutions on Azure](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview) | **Master guide** — covers all layers end-to-end. Start here for the holistic view. |
| [SaaS and Multitenant Solution Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/saas-multitenant-solution-architecture/) | Covers SaaS delivery model patterns; directly applicable if E360 is offered as-a-service to multiple clients. |
| [Tenancy Models for a Multitenant Solution](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/tenancy-models) | Helps decide between shared, dedicated, and hybrid tenant models — the first decision to make. |
| [Architectural Approaches for a Multitenant Solution](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview) | Deep-dive into compute, storage, networking, and messaging approaches per tenancy model. |

### 🔐 Identity & Access (Okta → Entra ID Migration, RBAC/RLS)

Maps to: **Current: Okta + RBAC layer → Proposed: Microsoft Entra ID (RBAC/RLS)**

| Resource | Relevance to Ep360 |
|---|---|
| [Architectural Approaches for Identity in Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity) | How to design authentication, authorization, SSO, and external IdP federation in multi-tenant apps. Directly relevant to Okta → Entra migration. |
| [Architectural Considerations for Identity in a Multitenant Solution](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/identity) | Requirements and trade-offs for identity per tenant — OAuth2, OIDC, tenant isolation in identity. |
| [Single and Multitenant Apps in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/single-and-multi-tenant-apps) | How to register E360 as a multi-tenant app in Entra so each client's users can sign in via their own tenant. |
| [Multitenant Organization Capabilities in Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/multi-tenant-organizations/overview) | Cross-tenant scenarios — useful if E360 clients are large enterprises with multiple Entra tenants. |
| [Resource Isolation with Multiple Tenants in Entra ID](https://learn.microsoft.com/en-us/entra/architecture/secure-multiple-tenants) | Security isolation patterns when using multiple Entra tenants. |
| [Common Considerations for Multitenant User Management](https://learn.microsoft.com/en-us/entra/architecture/multi-tenant-common-considerations) | User lifecycle, B2B collaboration, cross-tenant sync. |

### 🗄️ Data Layer & Storage (Iceberg Lakehouse, ADLS Gen2, ClickHouse)

Maps to: **Layer 2: Lakehouse-Centric Storage Layer (Iceberg & ClickHouse)**

| Resource | Relevance to Ep360 |
|---|---|
| [Architectural Approaches for Storage and Data in Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data) | **Critical read** — covers shared vs. isolated databases, partitioning strategies, storage account isolation. Directly applies to Iceberg namespace-per-tenant and ADLS folder isolation decisions. |
| [Multitenant SaaS Database Tenancy Patterns (Azure SQL)](https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns?view=azuresql) | While SQL-focused, the tenancy patterns (database-per-tenant, shared schema, elastic pools) inform the Iceberg catalog/schema isolation strategy. |
| [Governance and Compliance in Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/governance-compliance) | Data residency, encryption per tenant, audit trails — essential for E360's Data Governance and Security modules. |

### 🌐 API Gateway & Integration (MuleSoft → APIM, GraphQL/REST)

Maps to: **Layer 5: API & Gateway + Integration Layer**

| Resource | Relevance to Ep360 |
|---|---|
| [Use Azure API Management in a Multitenant Solution](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/api-management) | **Directly applicable** — covers shared vs. dedicated APIM instances, tenant routing, subscription keys, rate limiting per tenant. Maps exactly to E360's API Gateway layer. |
| [Map Requests to Tenants in a Multitenant Solution](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/map-requests) | Tenant resolution strategies (host header, path, JWT claims, API key). Critical for the Tenant Router component. |
| [Protect APIs with Application Gateway and API Management](https://learn.microsoft.com/en-us/azure/architecture/web-apps/api-management/architectures/protect-apis) | WAF + APIM layered security architecture. Relevant for E360's Traffic Mgmt & Auth component. |

### ☸️ Compute & Application Logic (AKS Multi-Tenancy)

Maps to: **Layer 4: Compute & Application Logic**

| Resource | Relevance to Ep360 |
|---|---|
| [Use Azure Kubernetes Service (AKS) in a Multitenant Solution](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/aks) | **Essential** — namespace-per-tenant, RBAC, resource quotas, network policies. Directly applicable if E360 containerizes Transformation Engine, Scoring Engine, Workflow Workbench on AKS. |
| [Cluster Isolation Best Practices for AKS](https://learn.microsoft.com/en-us/azure/aks/operator-best-practices-cluster-isolation) | Logical vs. physical isolation, scheduling controls (taints, tolerations, node affinity), pod security. |
| [Best Practices for Azure Kubernetes Service (AKS)](https://learn.microsoft.com/en-us/azure/aks/best-practices) | Comprehensive AKS best practices covering multi-tenancy, security, networking, storage. |
| [AKS + AGIC Multi-Tenant Sample](https://learn.microsoft.com/en-us/samples/azure-samples/aks-multi-tenant-agic/aks-multi-tenant-agic/) | Code sample — per-tenant namespaces with Application Gateway Ingress Controller, RBAC, network policies. |

### 📊 Presentation Layer (Power BI Embedded Multi-Tenancy + RLS)

Maps to: **Layer 6: Presentation & User Experience (Power BI Premium/Embedded)**

| Resource | Relevance to Ep360 |
|---|---|
| [Service Principal Profiles for Multi-Tenancy in Power BI Embedded](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy) | **Directly applicable** — workspace-per-tenant isolation using service principal profiles. The recommended approach for E360's Persona Dashboards per tenant. |
| [Develop Scalable Multitenancy Apps with Power BI Embedding](https://learn.microsoft.com/en-us/power-bi/guidance/develop-scalable-multitenancy-apps-with-powerbi-embedding) | End-to-end guide for building scalable multi-tenant embedded analytics. Covers workspace strategy, deployment automation, capacity management. |
| [Row-Level Security (RLS) with Power BI](https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security) | Implementing RLS with DAX filters — critical for E360's DQI and Portfolio dashboards where data must be filtered per tenant. |
| [Cloud-Based RLS with Embedded Content](https://learn.microsoft.com/en-us/power-bi/developer/embedded/cloud-rls) | Static and dynamic RLS in embedded scenarios. Maps to E360's Entra RBAC/RLS cross-cutting concern. |
| [Security in Power BI Embedded Analytics](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embedded-row-level-security) | Embed token generation with effective identity for RLS enforcement. |

### 🤖 AI & Agentic Intelligence Layer (Azure OpenAI Multi-Tenancy)

Maps to: **Layer 8: Agentic Intelligence Layer + Gen AI module**

| Resource | Relevance to Ep360 |
|---|---|
| [Multitenancy and Azure OpenAI](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/openai) | **Critical read** — dedicated instance per tenant vs. shared instance with tenant-scoped models vs. fully shared. Directly relevant to E360's Gen AI (Playground, Ops Designer, CXO Insights). |
| [Architectural Approaches for AI/ML in Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/ai-machine-learning) | Covers model isolation, training data isolation, inference isolation. Applies to E360's Composite Scoring Engine and Analytics & Correlation. |
| [Design a Secure Multitenant RAG Inferencing Solution](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/secure-multitenant-rag) | **Highly relevant** — if E360's CXO Insights or Ops Designer use RAG (retrieval-augmented generation), this covers tenant-scoped document isolation in vector stores and prompt context filtering. |

### 📋 Quick Reference: Layer-to-Link Mapping

| E360 Layer | Primary Microsoft Reference |
|---|---|
| **Layer 0: Infrastructure** | [Architect Multitenant Solutions on Azure](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview) |
| **Layer 1: Data & Integration** | [Storage and Data in Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data) |
| **Layer 2: Lakehouse Storage** | [Multitenant SaaS Database Tenancy Patterns](https://learn.microsoft.com/en-us/azure/azure-sql/database/saas-tenancy-app-design-patterns?view=azuresql) |
| **Layer 3: Semantic Config** | [Governance and Compliance in Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/governance-compliance) |
| **Layer 4: Compute & App Logic** | [AKS in a Multitenant Solution](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/aks) |
| **Layer 5: API & Gateway** | [APIM in a Multitenant Solution](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/api-management) |
| **Layer 6: Presentation (Power BI)** | [Power BI Embedded Multi-Tenancy](https://learn.microsoft.com/en-us/power-bi/developer/embedded/embed-multi-tenancy) |
| **Layer 8: Agentic AI** | [Multitenancy and Azure OpenAI](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/service/openai) |
| **Cross-cutting: Identity** | [Identity in Multitenant Solutions](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/identity) |
| **Cross-cutting: Tenant Routing** | [Map Requests to Tenants](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/map-requests) |

---

## Final Assessment

The proposed Ep360⁺ˡᵘˢ architecture is a **strong starting point**. The bets on **Iceberg, ClickHouse, and dbt** are sound and future-proof. The two main gaps are:

1. **Multi-tenancy is not a first-class architectural concern** — tenant isolation, routing, and configuration must be explicitly designed into every layer.
2. **The cloud-agnostic story is incomplete** — Kubernetes, Terraform, OpenTelemetry, and BI portability need to be addressed to achieve true hyperscaler independence.

The conversation with the customer should center on **defining the tenant isolation model first** (it affects every layer) and then **deciding where on the lock-in/portability spectrum** they want to land for each architectural layer.

---

*Document generated: 2026-03-31*
*Advisory Status: Pre-engagement Discovery Phase*
