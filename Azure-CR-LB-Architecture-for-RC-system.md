
# Azure CR‑LB Architecture for RC System (Final) — **One SLB per Broker**

**Context:** Controller/Target choose a **specific Broker hostname**. Requirement: **one Standard LB per Broker** (no shared pool).  
**Goal:** Lowest‑latency global ingress via **Anycast**, L4 pass‑through, TLS termination on Brokers, and clean per‑broker isolation.

---

## High‑Level Architecture


```mermaid
flowchart TB

%% Clients
C1["Controller (USA)"]
T1["Target (India)"]

%% DNS (per-broker FQDNs)
subgraph DNS["DNS (A/CNAME per Broker)"]
  DNS_B1["broker-us-1.company.com"]
  DNS_B2["broker-us-2.company.com"]
  DNS_B3["broker-in-3.company.com"]
  DNS_B4["broker-in-4.company.com"]
end

C1 -->|"Resolve chosen broker"| DNS_B1
T1 -->|"Resolve all brokers"| DNS_B1
T1 --> DNS_B2
T1 --> DNS_B3
T1 --> DNS_B4

%% Cross-Region Load Balancer (one resource, many frontends)
subgraph CRLB["Azure Cross-Region Load Balancer (Anycast Frontends)"]
  FE_B1["Frontend fe-b1-us (Anycast)"]
  FE_B2["Frontend fe-b2-us (Anycast)"]
  FE_B3["Frontend fe-b3-in (Anycast)"]
  FE_B4["Frontend fe-b4-in (Anycast)"]
end

DNS_B1 --> FE_B1
DNS_B2 --> FE_B2
DNS_B3 --> FE_B3
DNS_B4 --> FE_B4

%% Regions & One SLB per Broker
subgraph US["USA Region"]
  SLB_B1["Standard LB: slb-b1-us"]
  SLB_B2["Standard LB: slb-b2-us"]
  B1["Broker B1 (TLS/mTLS terminates)"]
  B2["Broker B2 (TLS/mTLS terminates)"]
  SLB_B1 --> B1
  SLB_B2 --> B2
end

subgraph IN["India Region"]
  SLB_B3["Standard LB: slb-b3-in"]
  SLB_B4["Standard LB: slb-b4-in"]
  B3["Broker B3 (TLS/mTLS terminates)"]
  B4["Broker B4 (TLS/mTLS terminates)"]
  SLB_B3 --> B3
  SLB_B4 --> B4
end

%% CR-LB routes frontends to regional SLBs
FE_B1 -->|"Route to US"| SLB_B1
FE_B2 -->|"Route to US"| SLB_B2
FE_B3 -->|"Route to India"| SLB_B3
FE_B4 -->|"Route to India"| SLB_B4
```

**What changed vs earlier draft:**  
- **One Standard LB per Broker** (`slb-b1-us`, `slb-b2-us`, `slb-b3-in`, `slb-b4-in`).  
- Each Broker keeps a **unique FQDN** that maps to its **own CR‑LB Frontend** (unique Anycast IP).  
- No regional pooling; each SLB forwards only to **its** Broker instance(s).

---

## Network Flow (Controller starts on B1, Target selects B3, Controller switches to B3)


```mermaid
sequenceDiagram
  autonumber
  participant C1 as Controller (USA)
  participant T1 as Target (India)
  participant DNS as DNS
  participant FE_B1 as CR-LB FE (B1 Anycast)
  participant FE_B3 as CR-LB FE (B3 Anycast)
  participant SLB_B1 as SLB B1 (US)
  participant SLB_B3 as SLB B3 (India)
  participant B1 as Broker B1 (US)
  participant B3 as Broker B3 (India)

  Note over C1: Need session code
  C1->>DNS: Resolve broker-us-1.company.com
  DNS-->>C1: Anycast IP (FE_B1)
  C1->>FE_B1: TCP+TLS (nearest Azure edge)
  FE_B1->>SLB_B1: Route to US region
  SLB_B1->>B1: L4 forward
  B1-->>C1: mTLS established (code)

  Note over T1: Choose nearest broker by RTT
  T1->>DNS: Resolve all 4 brokers
  DNS-->>T1: Anycast IPs (FE_B1..FE_B4)
  T1->>FE_B3: TCP+TLS (nearest edge)
  FE_B3->>SLB_B3: Route to India region
  SLB_B3->>B3: L4 forward
  B3-->>T1: mTLS established

  Note over B1: Inter-broker signal
  B1-->>C1: Reconnect to B3

  C1->>DNS: Resolve broker-in-3.company.com
  DNS-->>C1: Anycast IP (FE_B3)
  C1->>FE_B3: TCP+TLS (nearest edge)
  FE_B3->>SLB_B3: Route to India region
  SLB_B3->>B3: L4 forward
  B3-->>C1: mTLS established

  Note over C1,B3: Session runs on two TCP tunnels
```


## Why this satisfies requirements

- **Per‑broker isolation:** one SLB per broker hostname meets “reach specific broker” need.  
- **Best latency globally:** Anycast ingress at nearest edge + Azure backbone.  
- **TLS/mTLS stays end‑to‑end:** no termination at Azure LBs.  
- **Simple scale‑out:** add broker → add CR‑LB frontend + SLB + DNS.  
- **Health & resiliency:** SLB health probes; client can re‑probe list to fail over.

---

##  Build Outline

1. **Region (US/India) per Broker**
   - Create **Standard LB** `slb-bX-<region>` (public).  
   - Health probe (TCP) + rule/NAT to **Broker Bx**.  
   - NSG: allow listener + probe only.  
   - Broker VM/AKS with **Managed Identity** to **Key Vault** for certs.

2. **Global**
   - Create **one Cross‑Region LB**.  
   - Create **one Frontend IP per Broker** (`fe-bX-<region>`).  
   - Backend mapping: each FE → its regional **SLB Bx**.  
   - DNS A/CNAME: `broker-<region>-<n>.company.com` → that FE IP.

3. **Ops**
   - Monitor SLB probes, Broker connection counts, cert expiry.  
   - Alerts in **Azure Monitor**/**Log Analytics**.

---
