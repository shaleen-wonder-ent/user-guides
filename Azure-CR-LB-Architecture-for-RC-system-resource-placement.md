
# Step-by-step Build Order (Azure CR‑LB → per‑Broker SLB → Brokers)

> This is a **procedural checklist** (no code) for the finalized design:
> - **One Cross‑Region Load Balancer (CR‑LB)** with **multiple frontends** (one per Broker).
> - **One Standard Load Balancer (SLB) per Broker** (no shared backend pool).
> - **TLS/mTLS terminates only on Brokers** (L4 pass‑through at LBs).

---

## 0) Inputs & Naming (Agree up-front)
- **Regions:** e.g., `USA (East US or West US)`, `India (Central India or South India)`, future: `EU`, `APAC`.
- **Broker list:** `B1 (US)`, `B2 (US)`, `B3 (IN)`, `B4 (IN)` (extendable).
- **DNS names:** `broker-us-1.company.com`, `broker-us-2.company.com`, `broker-in-3.company.com`, `broker-in-4.company.com`.
- **CR‑LB frontends:** `fe-b1-us`, `fe-b2-us`, `fe-b3-in`, `fe-b4-in`.
- **Regional SLBs:** `slb-b1-us`, `slb-b2-us`, `slb-b3-in`, `slb-b4-in`.
- **VNets/Subnets:** `vnet-rc-<region>`, subnets: `broker-snet`, `mgmt-snet` (optional: `firewall-snet`).
- **Ports:** Broker listener port(s), health probe port.
- **Key Vault names:** `kv-rc-<region>`.
- **Resource groups:** `rg-rc-global`, `rg-rc-us`, `rg-rc-in`, etc.

---

## 1) Global/Shared Foundations
1. **Create Resource Groups**
   - `rg-rc-global` (for CR‑LB, Azure DNS if used)
   - `rg-rc-us` (US region resources)
   - `rg-rc-in` (India region resources)
   - (Future regions get their own RGs)

2. **Create (or confirm) Public DNS Zone**
   - `company.com` (Azure DNS or your existing external DNS)

---

## 2) Per Region (repeat for each region: US, India, etc.)
3. **Create VNet**
   - Name: `vnet-rc-<region>`
   - Address space sized for Brokers + infra

4. **Create Subnets**
   - `broker-snet` (Brokers/Nodes)
   - `mgmt-snet` (bastion/jumpbox/monitoring) — optional
   - `firewall-snet` — optional

5. **Create Network Security Groups**
   - NSG for `broker-snet`: allow **Broker listener port(s)** inbound from **Internet** (or CR‑LB path), allow **health probe** from SLB, restrict all else.
   - Associate NSG to `broker-snet`.

6. **(Optional) Azure Firewall / NAT Gateway**
   - If you need egress allow‑listing or stable outbound IPs for Brokers.

7. **Create Azure Key Vault (per region)**
   - `kv-rc-<region>`
   - Upload **Broker server certs** (CN/SAN = Broker FQDN)
   - Upload **CA chain** for mTLS validation

8. **Prepare Compute for Brokers (per Broker)**
   - Decide **VMSS** or **AKS**.
   - Assign **Managed Identity** with **Get/List** secrets on Key Vault.
   - Configure Broker to fetch certs from Key Vault at startup/rotation.

---

## 3) Per Broker (one SLB per Broker)
> Repeat the following steps **for each Broker** B1, B2, B3, B4…

9. **Create Standard Public Load Balancer (per Broker)**
   - Name: `slb-bX-<region>`
   - Frontend: Public IP (regional)
   - Health Probe: TCP on dedicated probe port (e.g., 9000 or your choice)
   - Rule/NAT: Forward **Broker listener port(s)** to the **Broker Bx** instance(s)

10. **Associate Backend**
   - If **single instance**: use **inbound NAT** to Bx.
   - If **pair/HA**: create **backend pool** with Bx‑1/Bx‑2; add LB rule + probe.

11. **Validate Broker reachability**
   - From outside, ensure SLB forwards to Bx and health probe is green.

---

## 4) Cross‑Region Load Balancer (one global, multiple frontends)
12. **Create Cross‑Region Load Balancer**
   - Resource group: `rg-rc-global`

13. **Create Frontend IP (Anycast) per Broker**
   - `fe-b1-us` → Backend = `slb-b1-us`
   - `fe-b2-us` → Backend = `slb-b2-us`
   - `fe-b3-in` → Backend = `slb-b3-in`
   - `fe-b4-in` → Backend = `slb-b4-in`
   - (Future: add more `fe-bX-<region>` frontends)

14. **(Optional) Configure secondary region fallback**
   - If required, configure FE failover mapping to an alternate SLB when primary is unhealthy.

---

## 5) DNS (per Broker hostname)
15. **Create DNS Records (A or CNAME)**
   - `broker-us-1.company.com` → CR‑LB frontend IP `fe-b1-us`
   - `broker-us-2.company.com` → CR‑LB frontend IP `fe-b2-us`
   - `broker-in-3.company.com` → CR‑LB frontend IP `fe-b3-in`
   - `broker-in-4.company.com` → CR‑LB frontend IP `fe-b4-in`

16. **Set sensible TTLs**
   - Consider lower TTL (e.g., 60–300s) during early rollout for agility.

---

## 6) Security & Observability
17. **Lock down NSGs**
   - Only Broker listener + SLB health probe inbound
   - Limit management access (Bastion/Jumpbox) if needed

18. **Monitor & Alerts**
   - Enable **Azure Monitor + Log Analytics** on LBs & Brokers
   - Alerts: SLB probe failures, Broker connection spikes, cert expiry, CPU/mem

19. **Key Vault Rotation Plan**
   - Define rotation cadence
   - Ensure Broker restart/reload process for new certs

20. **(Optional) DDoS Protection Standard**
   - Recommended for multiple public IPs and critical production workloads.

---

## 7) Client App (Controller/Target) Checklist
21. **Broker discovery**
   - Maintain list of all Broker FQDNs
   - Parallel probe for RTT and pick lowest latency

22. **mTLS**
   - Present client certificates (Controller/Target) to Broker
   - Validate Broker server certificate (CN/SAN must match FQDN)

23. **Reconnect logic**
   - Support “switch broker” when instructed (e.g., B1 → B3)

---

## 8) Future Expansion (repeatable pattern)
24. **Add a new Broker**
   - Deploy Broker compute → Create **SLB‑Bx** → Add **FE‑Bx** on CR‑LB → Add **DNS for broker‑Bx** → Add to client list

25. **Add a new Region**
   - Create VNet/Subnets/NSGs/Key Vault → Deploy Brokers → Create SLBs per Broker → Add CR‑LB frontends → Add DNS records

---

###  End of Checklist
