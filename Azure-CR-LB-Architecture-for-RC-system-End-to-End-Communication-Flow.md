
# End‑to‑End Communication Flow — (Final, per‑Broker SLB)

This explains how a session is established when **each Broker has its own Regional Standard Load Balancer** and **Azure CR‑LB provides Anycast ingress**.

---

<img width="986" height="437" alt="image" src="https://github.com/user-attachments/assets/e0e1763c-4d36-4db8-9f42-5f0e9ed7c17c" />


---

## 1) Controller starts with any Broker
- The Controller (C1) has a list of Broker hostnames.  
- It picks one (e.g., `broker-us-1.company.com`).  
- DNS returns an **Anycast IP** for that Broker’s **CR‑LB Frontend**.  
- C1 connects to the **nearest Azure edge**, then Azure routes to the **Broker’s own SLB** in its home region, and the SLB forwards to the **Broker**.  
- The Broker completes **mTLS** and issues a **Connection Code**.
- Controller is now logically “attached” to Broker B1

## 2) Target chooses the nearest Broker
- The Target (T1) has the same list of Broker hostnames.  
- It quickly probes all of them, measures latency, and picks the **lowest‑RTT** Broker (say `broker-in-3.company.com`).  
- DNS again returns the **Anycast IP** for that Broker’s **CR‑LB Frontend**.  
- T1 connects to the **closest Azure edge**, Azure routes to **SLB‑B3 (India)**, which forwards to **Broker B3**.  
- **mTLS** completes between T1 and B3.

## 3) Controller reconnects to the Target’s Broker
- Brokers maintain a lightweight inter‑broker channel.  
- The first Broker (B1) informs C1: “The Target is on **B3**; please reconnect.”  
- C1 resolves `broker-in-3.company.com`, gets the **Anycast IP** for **FE‑B3**, connects at its **nearest edge**, and is forwarded to **SLB‑B3 → B3**.  
- Now both **C1 and T1 terminate TLS on B3**.

## 4) Session runs
- Two optimized TCP tunnels are active: **C1↔B3** and **T1↔B3**.  
- **RC Server** is *not* in the data path (only used for policy/metadata outside the session).  
- When either side hangs up, B3 closes tunnels and updates session state.

---

## Why this is optimal
- **Anycast** guarantees shortest entry into Azure anywhere in the world.  
- **One SLB per Broker** enforces “reach this exact Broker by hostname.”  
- **TLS/mTLS** is end‑to‑end; Azure does not terminate.  
- Scaling is simple: **add a Broker** → **add a Frontend + SLB + DNS**.

---
