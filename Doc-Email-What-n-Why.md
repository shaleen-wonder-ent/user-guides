# DocAI & InspectAI — Azure Architecture Explain Story

> **Purpose of this document**
> This is a plain-language companion to the Azure pricing estimate. For every Azure service in the solution, it explains **what it does for us** and **why we chose it** over alternatives. Use it for stakeholder reviews, onboarding new engineers, and defending design decisions in security/architecture reviews.

---

## 🎯 Solution Overview

We are building two AI agents that work together on a shared Azure platform:

| Agent | What it does | Volume |
|---|---|---|
| **DocAI** | Insurance email triage — reads incoming emails (with PDF attachments), classifies them, extracts data, and routes them to the correct downstream action | **10,000 emails/day** (~300K/month). 75% have PDF attachments. 60% handled by ML, 40% escalated to Claude (LLM) for first 3 pages of the attachment. |
| **InspectAI** | Property inspection agent — ingests large inspection reports (100+ pages), summarizes findings, and produces inspection outcomes | **500 inspections/day** (~15K/month). Documents are 100+ pages each. |

Both agents share the same Azure landing zone, security perimeter, observability stack, and CI/CD pipeline — which keeps cost low and operations consistent.

---

## 🏛️ Architectural Principles (Why the Design Looks This Way)

1. **PaaS-first, no IaaS VMs** — fully managed services, no patching, no OS to harden, no autoscale groups to manage.
2. **Network-isolated by default** — every data-handling service has a Private Endpoint. Nothing customer-touching is reachable from the public internet directly.
3. **Event-driven and asynchronous** — workers scale independently of the email/inspection ingest rate; bursts are absorbed by queues.
4. **Defense in depth** — WAF at edge, Private Link in the middle, Defender for runtime, Key Vault for secrets, Managed Identity everywhere.
5. **Multi-region ready** — primary in East US, DR in West US 2. Backups, geo-replication, and paired Premium services are configured for failover.
6. **Cost-aware** — verbose logs go to Basic Logs tier, blob storage uses lifecycle to Cool, ACA scales to zero where possible.

---

## 📦 Service-by-Service: What It Does + Why We Chose It

### 1. Azure Front Door (Premium) + WAF
**What it does**
The single global entry point for all user traffic to the DocAI and InspectAI web UIs and APIs. It terminates TLS, routes traffic to backend Container Apps, caches static assets, and inspects every request with Web Application Firewall rules.

**Why we chose it**
- **Global anycast edge** — fast for users no matter where they are.
- **WAF is bundled** — OWASP managed rules, bot protection, rate limiting, custom rules — no separate WAF resource needed.
- **L7 DDoS protection included** — handles application-layer floods without us paying for DDoS Standard ($2,944/mo).
- **Private Link to origins** — Premium tier lets Front Door reach Container Apps over a private connection, so ACA never needs a public ingress.
- **Alternatives considered:** Application Gateway (regional only, no global edge), API Management (great for APIs but doesn't do CDN), Azure CDN (no WAF).

---

### 2. Azure Container Apps (Consumption)
**What it does**
Runs all six microservices that make up DocAI and InspectAI:
- DocAI API (UI/backend)
- Inspect API (UI/backend)
- Email Ingestion Worker
- DocAI Worker (ML + Claude path)
- InspectAI Worker (100+ page document processor)
- Action Agent Worker (downstream actions)

Each app is a container that autoscales based on HTTP requests or queue depth (KEDA).

**Why we chose it**
- **Scale-to-zero** for workers — pay only when there's work in the queue.
- **KEDA built-in** — workers scale automatically with Service Bus queue depth.
- **No Kubernetes to manage** — we get the benefits of containers without the operational overhead of AKS.
- **Dapr-ready** — built-in service-to-service auth, secret management, pub/sub.
- **Alternatives considered:** AKS (too much operational overhead for our team size), App Service (worse fit for queue-driven workers), Azure Functions (cold-start and timeout limits would hurt InspectAI's long-running document processing).

---

### 3. Azure Functions (Consumption)
**What it does**
A lightweight relay between external email systems (Microsoft Graph, Gmail) and our internal pipeline. When a new email arrives, the Graph/Gmail webhook hits a Function, which validates the payload and publishes an event to Event Grid.

**Why we chose it**
- **Free tier covers our volume** — 1M executions/month free; we use ~400K.
- **Built-in webhook bindings** — minimal code to wire up.
- **Separation of concerns** — keeps the "ingestion adapter" decoupled from the main worker logic in Container Apps.
- **Alternatives considered:** Logic Apps (more expensive at our volume), running the webhook inside Container Apps (couples scaling to webhook traffic, which we don't want).

---

### 4. Azure Event Grid (Basic)
**What it does**
A pub/sub backbone for system events — `email.received`, `inspection.uploaded`, `document.classified`, etc. Multiple subscribers (workers, audit logger, monitoring) can react to the same event without coupling.

**Why we chose it**
- **Loose coupling** — adding a new consumer (e.g., audit, ML retraining) doesn't require changes to producers.
- **Basic tier is cheap** — pennies per month at our volume.
- **Native filtering** — subscribers only get the events they care about.
- **Alternatives considered:** Putting everything on Service Bus topics (more expensive, less pub/sub-native), polling SQL for changes (anti-pattern).

---

### 5. Azure Service Bus (Premium)
**What it does**
The reliable message bus between every stage of the DocAI and InspectAI pipelines — Email Ingestion → DocAI Worker → Claude → Action Agent, and Inspection Ingestion → InspectAI Worker → Action Agent. Provides FIFO ordering, dead-letter queues, retries, and sessions.

**Why we chose Premium (not Standard)**
- **Private Endpoint support** — Standard tier only supports public endpoints; Premium lets us match the network-isolation pattern of Blob, SQL, and Key Vault.
- **Geo-DR ready** — Premium supports active-passive geo-disaster recovery for messaging continuity.
- **Predictable throughput** — dedicated messaging units, no noisy-neighbor throttling.
- **Larger messages (100 MB vs 256 KB)** — useful if we ever need to pass document references with metadata.
- **Compliance defensibility** — insurance data must not traverse public networks; Premium + Private Endpoint gives us a clean audit answer.
- **Alternatives considered:** Storage Queues (no FIFO, no sessions, no DLQ), Event Hubs (event streaming, not workflow messaging), Kafka on AKS (operational burden).

---

### 6. Azure Blob Storage (RA-GZRS, Hot + Cool lifecycle)
**What it does**
Stores every raw email (MIME), every PDF attachment, and every inspection document (100+ page PDFs). A lifecycle policy automatically moves blobs from Hot → Cool after 90 days of inactivity, drastically reducing storage cost for older data.

**Why we chose it**
- **Cheapest durable object storage in Azure** — pennies per GB.
- **RA-GZRS** — data is replicated to a paired region (West US 2) and readable from there, supporting DR.
- **Lifecycle policies** — automatic tiering keeps cost predictable as data grows (we're ingesting ~780 GB/month).
- **Blob Soft Delete + Versioning** — built-in ransomware/accidental-delete protection without paying for Azure Backup.
- **Defender for Storage** — scans uploaded attachments for malware (critical for untrusted email attachments).
- **Alternatives considered:** Azure Files (more expensive, designed for SMB/NFS workloads), Data Lake Gen2 (overkill — we don't run analytics on raw blobs).

---

### 7. Azure SQL Database (vCore, General Purpose) + DR Geo-Replica
**What it does**
The system-of-record for metadata: email records, inspection cases, workflow status, audit trails, user assignments, and case routing decisions. A geo-replica in West US 2 enables failover.

**Why we chose it**
- **Strong consistency + ACID** — workflow state and audit trails need transactional guarantees.
- **Fully managed** — automatic patching, backups (35-day PITR built in), threat detection.
- **vCore model** — predictable cost, easy to scale up.
- **Active Geo-Replication** — async replica in DR region with manual or automatic failover.
- **Defender for SQL** — vulnerability assessment + threat detection included.
- **Alternatives considered:** Cosmos DB (eventual consistency doesn't fit workflow status), PostgreSQL Flexible Server (would work, but team has more SQL Server experience), SQL MI (overkill for our scale, more expensive).

---

### 8. Azure Container Registry (Premium)
**What it does**
Stores the Docker images for all six container apps. CI/CD pipeline (GitHub Actions) builds and pushes images here; Container Apps pulls them.

**Why we chose Premium (not Standard)**
- **Private Endpoint support** — keeps image pulls off the public internet.
- **Geo-replication** — image available in DR region for faster cold-start failover.
- **Content trust / signing** — supports image signing for supply-chain security.
- **Repository-scoped tokens** — finer-grained access control than Standard.
- **Alternatives considered:** Docker Hub (no enterprise SLA, no Private Endpoint), GitHub Container Registry (works, but adds cross-cloud egress and complicates RBAC).

---

### 9. Azure Key Vault
**What it does**
Single source of truth for all secrets: SQL connection strings, Claude/Anthropic API keys, Graph/Gmail OAuth client secrets, signing keys, and TLS certificates used by Front Door. All apps use Managed Identity + RBAC to access secrets — **no secrets in app config or environment variables**.

**Why we chose it**
- **Managed Identity integration** — apps don't store any credentials themselves.
- **Soft-delete + purge protection** — accidental deletion is recoverable.
- **CMK (Customer-Managed Keys)** — Blob and SQL encryption keys live here; we control the key lifecycle.
- **Defender for Key Vault** — anomalous access patterns trigger alerts.
- **Alternatives considered:** Azure App Configuration (great for non-secret config, not a secret store), HashiCorp Vault (operational overhead, no native MI integration).

---

### 10. Azure AI Foundry / Anthropic Claude
**What it does**
The LLM that powers the "smart" parts of both agents:
- **DocAI** — reads the first 3 pages of email attachments (40% of emails) to classify intent, extract entities, and recommend actions.
- **InspectAI** — summarizes inspection reports and extracts structured findings from 100+ page documents.

**Why we chose Claude on Azure AI Foundry**
- **Enterprise-grade hosting** — Anthropic Claude served through Azure means we get Azure billing, Private Endpoints, and regional residency.
- **Strong long-context performance** — handles long documents and structured-extraction tasks well.
- **Single LLM, two use cases** — same provider/model for both agents simplifies prompt engineering, evals, and cost tracking.
- **Defender for AI** — monitors prompt usage for abuse and prompt injection signals.
- **Alternatives considered:** Azure OpenAI GPT-4 (good alternative, team chose Claude for long-context quality), self-hosted OSS LLMs on AKS (operational + GPU cost prohibitive at our volume).

---

### 11. Microsoft Entra ID (P1)
**What it does**
Identity provider for end users (50 reviewers/agents using the DocAI and InspectAI UIs), for service-to-service auth (Managed Identities), and for Conditional Access policies (MFA, device compliance).

**Why we chose P1 (not Free)**
- **Conditional Access** — enforce MFA and device compliance for human users.
- **Group-based RBAC** — easier governance than per-user assignments.
- **Audit logs retained** — required for compliance.
- **Alternatives considered:** Free tier (no Conditional Access, deal-breaker), P2 (we don't currently need PIM / Identity Protection).

---

### 12. Azure Private Link / Private Endpoints + Private DNS Zones
**What it does**
Creates a **private IP inside our VNet** for every PaaS service (Blob, SQL, Key Vault, Service Bus, Container Registry, etc.) and resolves their public DNS names to those private IPs.

**Why we chose it**
- **No public exposure** — even if a SAS key or API key leaks, the attacker can't reach the resource from the internet.
- **Data exfiltration prevention** — outbound traffic to `*.blob.core.windows.net` etc. can be locked to our PEs only.
- **Compliance** — PCI/SOC 2/insurance regulators expect "no public network access" on data-handling resources.
- **Consistent posture** — every data-touching service follows the same pattern; no "weakest link" public service.
- **Alternatives considered:** Service Endpoints (cheaper, but only restricts inbound — doesn't give you a private IP), public endpoints + IP allow-listing (operationally brittle, doesn't scale across PaaS that don't expose stable IPs).

---

### 13. Azure Monitor — Log Analytics + Application Insights
**What it does**
The single observability plane:
- **Log Analytics** — central store for diagnostics from every service.
- **Application Insights** — distributed tracing across the API → worker → Claude → action-agent chain.
- **Alerts** — wired to email/Teams for SLO violations.
- **Workbooks/Dashboards** — operational visibility for engineers and product owners.

**Why we chose this configuration (Analytics + Basic Logs split)**
- **Basic Logs tier (~$0.50/GB)** for high-volume verbose logs (ACA stdout, Front Door access) — cuts cost vs. Analytics tier (~$2.76/GB) by ~5×.
- **Analytics tier** reserved for logs we actually query/alert on (App Insights traces, security events, audit).
- **Single workspace** — easier KQL across services than multiple workspaces.
- **Alternatives considered:** Splunk/Datadog (multi-thousand-dollar contracts), self-hosted ELK (operational burden).

---

### 14. Microsoft Defender for Cloud (multiple plans)
**What it does**
Runtime threat detection and security posture for everything in the architecture:
- **Defender for Containers** — vulnerability scans on ACR images, runtime threat detection on ACA.
- **Defender for Storage** — malware scanning on every attachment uploaded to Blob.
- **Defender for SQL** — anomalous query patterns, SQL injection attempts.
- **Defender for Key Vault** — unusual secret-access alerts.
- **Defender for Resource Manager** — suspicious ARM operations.
- **Defender for DNS** — DNS-based attack patterns.
- **Defender for AI** — prompt-abuse and prompt-injection monitoring on Claude calls.
- **Defender CSPM (Foundational, free)** — secure score, recommendations, compliance baselines.

**Why we chose it**
- **Native integration** — no agents to deploy, alerts flow into Defender + Log Analytics automatically.
- **Per-resource pricing** — only pay for what's enabled, where it's enabled.
- **Malware scanning on email attachments is non-negotiable** — emails are an untrusted channel.
- **Alternatives considered:** Third-party CSPM (Wiz, Prisma) — better dashboards but ~10× more expensive; we can always layer one on later.

---

### 15. Bandwidth (Internet Egress + Inter-Region)
**What it does**
Covers outbound data charges:
- **Internet egress** — responses to end users, outbound calls to Graph/Gmail and Claude.
- **Inter-region** — SQL geo-replication traffic, monitoring traffic East US → West US 2.

**Why it's a separate line**
- Many estimates miss this and get surprised on the first bill.
- First 100 GB/month outbound is free; we model 200 GB internet egress + 100 GB inter-region.

---

## 🔁 End-to-End Flow (How a Single Email Travels Through the System)

1. **Microsoft Graph webhook** fires for a new email.
2. **Azure Function** receives the webhook, validates it, publishes an `email.received` event to **Event Grid**.
3. **Event Grid** fans the event out to subscribers, including the **Email Ingestion Worker** (on Container Apps).
4. **Email Ingestion Worker** downloads the email + attachments via Graph, stores them in **Blob Storage**, writes a case record in **Azure SQL**, and enqueues a message on **Service Bus Premium**.
5. **DocAI Worker** picks up the message:
   - 60% of cases → handled by the local ML model, decision written to SQL.
   - 40% of cases → first 3 pages of the PDF are sent to **Claude on Azure AI Foundry**.
6. **DocAI Worker** writes the classification + extracted entities back to SQL and enqueues an `action.required` message.
7. **Action Agent Worker** consumes the action message and triggers downstream behavior (assign to reviewer, auto-reply, route to insurer, etc.).
8. End-to-end traces flow into **Application Insights**; security signals flow into **Defender**.

The InspectAI flow is identical, but with larger documents and a longer per-message processing time inside the InspectAI Worker.

---

## 🛡️ Security Posture Summary

| Control | Where it lives |
|---|---|
| **Edge protection (WAF, L7 DDoS, bot)** | Front Door Premium |
| **Network isolation** | Private Endpoints on every data service |
| **Identity** | Entra ID + Managed Identity everywhere (no static creds) |
| **Secrets** | Key Vault + CMK for Blob/SQL |
| **Malware scanning** | Defender for Storage on Blob |
| **Runtime threat detection** | Defender for Cloud (Containers, SQL, KV, RM, DNS, AI) |
| **Audit** | SQL Audit + Log Analytics + Defender alerts |
| **Backup/recovery** | SQL PITR (35 days), Blob soft-delete + versioning, RA-GZRS |
| **DR** | SQL geo-replica + Service Bus Premium geo-DR (optional) + Blob RA-GZRS, all in West US 2 |

---

## 💰 Cost Summary (Indicative)

| Category | Monthly |
|---|---|
| Compute (Container Apps, Functions) | ~$200 |
| Data (Blob, SQL Primary + DR) | ~$1,300 |
| Messaging (Service Bus Premium, Event Grid) | ~$680 |
| Networking (Front Door, Private Link, DNS, bandwidth) | ~$470 |
| Identity & Secrets (Entra P1, Key Vault, ACR Premium) | ~$480 |
| Observability (Azure Monitor) | ~$950 |
| Security (Defender for Cloud) | ~$165 |
| **Subtotal (platform)** | **~$4,250** |
| AI / Claude (modeled separately) | $5,500–7,500 |
| **Grand total** | **~$9,750–11,750/month** |

---

## ❓ Open Decisions Tracked Here

| Decision | Status | Owner |
|---|---|---|
| Service Bus Premium with Geo-DR replica in West US 2? | Open — adds ~$680/mo | Security/Architecture |
| Container Registry Standard vs Premium? | Open — Premium adds $145/mo for Private Endpoint | Security |
| Long-term retention for inspection documents (7 vs 25 years)? | Open — drives Blob Archive cost | Compliance/Legal |
| DDoS Network Protection Standard? | Closed — **not required** (Front Door Premium covers L7) | Architecture |
| Defender CSPM Premium tier? | Closed for now — Foundational (free) is sufficient | Security |

---

## 📚 Glossary (For Non-Engineers Reading This)

- **PaaS** — Platform as a Service. Azure manages the OS, patching, scaling. We just deploy code/config.
- **Private Endpoint (PE)** — a private IP inside our network that points to a PaaS service, so traffic never touches the public internet.
- **Managed Identity** — an Azure-managed credential that lets one service authenticate to another without storing passwords.
- **Geo-DR** — Geographic Disaster Recovery; a paired region we can fail over to if the primary region has an outage.
- **CMK** — Customer-Managed Key; we own the encryption key (in Key Vault), Azure uses it to encrypt our data.
- **KEDA** — Kubernetes Event-Driven Autoscaling; the engine that scales Container Apps based on queue depth.
- **WAF** — Web Application Firewall; inspects HTTP traffic for attacks (SQLi, XSS, bots).
- **RA-GZRS** — Read-Access Geo-Zone-Redundant Storage; the most durable Azure Blob redundancy tier.
- **PITR** — Point-in-Time Restore; for SQL, lets us restore the DB to any point in the last 35 days.
