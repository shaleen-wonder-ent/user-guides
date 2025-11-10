
# Azure CR‑LB Architecture for RC System — **Component Reference**

**Scope:** Finalized design where **each Broker has its own Regional Standard Load Balancer** (no shared pool).  
**Protocol:** Custom TCP with TLS/mTLS (terminated on Brokers).  
**Internet-facing:** Yes (public).

---

## Components (What & Why)

### 1) **DNS (Public)**
- **Purpose:** Resolve each per‑broker hostname to its **own CR‑LB Frontend (Anycast IP)**.
- **Records:** A/CNAME per broker, e.g.  
  - `broker-us-1.company.com` → `fe-b1-us (Anycast IP)`  
  - `broker-us-2.company.com` → `fe-b2-us (Anycast IP)`  
  - `broker-in-3.company.com` → `fe-b3-in (Anycast IP)`  
  - `broker-in-4.company.com` → `fe-b4-in (Anycast IP)`

### 2) **Azure Cross‑Region Load Balancer (CR‑LB) – Single resource**
- **Why:** Global **Anycast ingress** so clients enter the **nearest Azure edge POP** anywhere in the world.
- **How used:** **Multiple Frontend IPs** — one per Broker hostname.
- **Behavior:** L4 pass‑through (TCP/UDP). **No TLS termination**. Frontend → correct **regional LB** based on the Broker’s home region.

### 3) **Regional Standard Load Balancer — One per Broker**
- **Why:** You must “reach a specific Broker by hostname.” No shared pool; each Broker is isolated behind its **own SLB**.
- **How used:** **SLB‑B1** fronts **Broker B1**, **SLB‑B2** fronts **Broker B2**, etc.
- **Behavior:** L4 forwarding (NAT or single‑node backend). Health probe removes the broker if unhealthy.

### 4) **Broker Compute (VMSS or AKS) — Per Broker**
- **Why:** Runs the Broker app; **terminates TLS/mTLS**; maintains inter‑broker control channel.
- **Notes:** Use **Managed Identity** + **Key Vault** to fetch server cert & CA chain; autoscale or HA pair if desired.

### 5) **Azure Key Vault (per region)**
- **Why:** Secure storage/rotation of Broker certs; accessed by Brokers via **Managed Identity**.

### 6) **Network Security Groups (NSGs) + (Optional) Azure Firewall**
- **Why:** Restrict inbound to broker listener + health probe; control egress with allow‑lists; log & monitor.

### 7) **Observability (Azure Monitor + Log Analytics)**
- **Why:** Track SLB probe health, connection counts, latency, TLS failures; alert on outages or anomalies.

---

## Naming & Mapping (Example)

| Broker | DNS → CR‑LB Frontend (Anycast IP) | Regional LB (per broker) | Region | Broker Target |
|---|---|---|---|---|
| B1 | `broker-us-1.company.com` → `fe-b1-us` | `slb-b1-us` | USA | `broker-b1` |
| B2 | `broker-us-2.company.com` → `fe-b2-us` | `slb-b2-us` | USA | `broker-b2` |
| B3 | `broker-in-3.company.com` → `fe-b3-in` | `slb-b3-in` | India | `broker-b3` |
| B4 | `broker-in-4.company.com` → `fe-b4-in` | `slb-b4-in` | India | `broker-b4` |

> Add more brokers by adding: **DNS → CR‑LB Frontend → Regional SLB → Broker**.

---

## Security Posture (at a glance)

- **mTLS end‑to‑end** (terminated **only** on Brokers).  
- **No TLS termination** at CR‑LB or SLB (both L4 pass‑through).  
- **Certificates** in **Key Vault**; Brokers use **Managed Identity**.  
- **NSGs** enforce minimal inbound + probe ports; **Firewall** optional for egress policy.  
- **DDoS Protection Standard** recommended on public IP‑heavy deployments.

---

## Availability, Latency & Scale

- **Anycast ingress** → closest Azure edge, then **Azure backbone** to the Broker’s region.  
- **Per‑broker SLB** → isolates health/failure and aligns with “reach specific broker by hostname.”  
- **Scale** by adding more Brokers (new frontend + new SLB + new DNS record).  
- **Failover**: Client can re‑probe list and pick next‑best broker; SLB health probes protect against instance‑level failures.

  ---
  
