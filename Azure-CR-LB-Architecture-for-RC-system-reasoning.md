
# Why This Architecture Works

This document explains *why* the final design is correct, what engineering constraints it satisfies, and why alternative Azure services were not selected. It provides architectural justification in simple, defensible language you can present to customers, auditors, or internal reviewers.

---

# 1. Why the Architecture Works

The design solves the core RC system requirements:

###  Requirement 1 — “A specific Broker must be reachable via its own hostname”
Each Broker has a distinct name:

- `broker-us-1.company.com`
- `broker-us-2.company.com`
- `broker-in-3.company.com`
- `broker-in-4.company.com`

Each hostname must always map to **exactly one Broker**.

**Solution:**  
- One **CR-LB frontend (Anycast)** per Broker  
- One **Standard Load Balancer** per Broker  
- One **Broker VM/AKS pool** behind that SLB

This produces a guaranteed **1:1 mapping**:

```
DNS → CR-LB Frontend → SLB-Bx → Broker Bx
```

No load balancing across different brokers.  
No cross-traffic.  
No ambiguity.

This matches the product requirement exactly.

---

###  Requirement 2 — “The Controller and Target must reach Azure through the closest entry point”
This eliminates >100 ms of unnecessary public internet latency.

**Solution:** Azure Cross-Region Load Balancer  
- Uses **Anycast global IPs**
- Routes the client to the **closest Azure Edge POP**
- Then uses Azure’s **private backbone**, not the public internet

This ensures:
- Lowest possible latency from any country  
- Stable jitter  
- Lower packet loss  
- Consistent routing regardless of ISP conditions

Traffic Manager cannot do this.  
Front Door cannot do this for *raw TCP*.

Only CR-LB provides **Anycast + TCP pass-through**.

---

###  Requirement 3 — “Support TCP + TLS/mTLS end-to-end”
Your protocol is:
- Custom TCP  
- TLS/mTLS  
- Termination must happen **only on the Broker**

**Solution:**  
Both CR-LB and SLB operate at **Layer 4 pass-through**:  
- They do NOT terminate TLS  
- They do NOT inspect TLS  
- They do NOT break certificate trust

This ensures:
- End-to-end cryptographic integrity  
- No need to manage certs on Azure LBs  
- No TLS complexity like SNI routing  
- Security audits become simpler

Traffic Manager and Front Door both interfere at Layer 7.  
CR-LB + SLB do not.

---

###  Requirement 4 — “Targets globally must discover the nearest Broker by latency”
The Target:
- Probes all Broker FQDNs  
- Measures latency  
- Chooses the lowest-RTT Broker  

This only works if:
- Each Broker has its **own hostname**
- Each hostname resolves to a **unique Anycast IP**
- Each Anycast IP routes to a **specific Azure region**
- LBs do not mix or distribute requests across Brokers

Our design fulfills all four conditions.

---

###  Requirement 5 — “Scale to many Brokers and new regions in the future”
The final design is modular:

To add Broker B5 (Europe):

1. Add **SLB-B5-EU**  
2. Add **CR-LB frontend FE-B5-EU**  
3. Deploy **Broker B5**  
4. Add DNS  
5. Clients probe new Broker automatically  

No refactoring, no downtime, no architectural redesign.

This is exactly how global security vendors (ZScaler, Citrix, Palo Alto GlobalProtect) scale to hundreds of nodes.

---

#  2. Why Azure Cross‑Region Load Balancer (CR‑LB) was chosen

###  It supports:
- **TCP** (your protocol)
- **Anycast global IPs** (key requirement)
- **Layer 4 pass‑through**
- **Deterministic routing to region**
- **One LB with many frontends** (perfect for per-Broker design)

###  CR‑LB provides:
- Global Anycast ingress  
- Fastest regional entry point  
- Azure backbone routing  
- Sub‑100 ms transit between continents  
- Full TCP support  
- No TLS termination  
- No complex configuration needed  

This aligns *perfectly* with your system.

---

#  3. Why Azure Traffic Manager was NOT chosen

Traffic Manager (TM) is:
- **DNS-based**
- **Latency-based routing**
- Works at the **DNS level**, not at the connection level
- Does **not** guarantee fastest end-to-end path  
- Does **not** use Azure’s private backbone  
- Does **not** support sub-100ms predictable RTT  
- Does **not** support TCP pass-through  
- Does **not** protect against bad internet paths  

Traffic Manager is ideal for:
- “Which region should my app run in?”
- “Which website endpoint is available?”

Traffic Manager is NOT suited for:
- Real-time, latency-sensitive TCP tunnels  
- Multi-hop session coordination  
- Per-endpoint specific routing  
- Anycast global ingress  
- Remote desktop/remote control traffic  

Therefore, **Traffic Manager cannot satisfy your latency or routing guarantees**.

---

#  4. Why Azure Front Door was NOT chosen

Front Door:
- Works at **Layer 7**
- Only supports **HTTP/HTTPS/WebSockets**
- Terminates TLS  
- Requires HTTP headers  
- Does not support raw TCP  
- Would break your Broker’s mTLS model  
- Adds undesired TLS complexity

Because your protocol is **custom TCP + mTLS**, Front Door cannot be used.

---

#  5. Why DNS is still required

DNS is essential because:
- Each Broker needs its **own hostname**
- Your Targets & Controllers use the hostname to:
  - Validate TLS certificates  
  - Decide target Broker  
  - Perform latency probes  
- DNS maps each hostname to its **CR-LB frontend**

DNS does *not* do routing.  
DNS only does **mapping**.

CR-LB performs the routing.

This separation is **exactly correct**.

---

#  6. Why Standard Load Balancer (SLB) is required per Broker

SLB:
- Provides regional **public entry point**
- Performs **health probes** on Broker instances
- Ensures only healthy Broker nodes receive traffic
- Allows the Broker VM/Pod to scale if needed
- Provides **regional high availability**

You cannot bypass SLB and connect CR-LB → VM directly.

Azure requires:
**CR-LB → SLB → VM/AKS**

Also:
SLB is *very* lightweight and inexpensive.

The “one SLB per Broker” architecture guarantees:
- Deterministic routing  
- No load balancing across Brokers  
- No cross-traffic or mixing  
- Exactly one backend per hostname  

---

#  7. Why this architecture is correct for your specific RC system

###  Works for Controller → Broker and Target → Broker  
Both enter Azure at the nearest edge, then go to correct region.

###  Ensures brokers are reached deterministically  
Each hostname maps to exactly one Broker.

###  Supports your session flow  
Controller picks B1 → Target picks B3 → Controller reconnects to B3 → session established.

###  Achieves sub‑100 ms latency requirement  
Anycast + Azure backbone ensures best path.

###  Works globally without redesign  
Adding regions or Brokers is trivial.

###  Keeps TLS/mTLS end‑to‑end  
Azure never terminates TLS.

###  Supports your feature roadmap  
Scaling horizontally is easy and incremental.

---

#  8. Final Summary

This architecture was selected because:

- It provides **global low-latency ingress** via Anycast  
- It ensures **deterministic routing per broker hostname**  
- It keeps **TLS/mTLS fully end-to-end**  
- It supports **custom TCP protocols**  
- It isolates each Broker behind **its own SLB**  
- It scales to **any number of Brokers or regions**  
- It avoids the limitations of Traffic Manager & Front Door  
- It matches how large-scale remote access systems are built

This is the most correct, cloud-native, future-proof architecture for your RC platform.

---

**End of Document**
