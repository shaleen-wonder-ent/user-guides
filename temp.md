# Azure vWAN Scalable Shared Services Architecture (Route Table Design)

## 🧭 Objective

Design a **scalable and secure Azure Virtual WAN (vWAN)** setup where one shared services VNet (CustC) can be accessed by all customer VNets (CustA–J), but customers cannot access each other.

---

## 🧩 Desired Behavior

| Connection | Should Access | Should NOT Access |
|-------------|----------------|-------------------|
| CustA | CustC | CustB, CustD, CustE, … |
| CustB | CustC | CustA, CustD, CustE, … |
| CustD–J | CustC | Everyone else |
| CustC | All (CustA–J) | — |

**Goal:** CustC acts as a **shared services hub**, reachable by all customers, while all customers remain isolated.

---

## 🔹 Scalable Route Table Design

Instead of creating one route table per customer (which doesn’t scale), use just **two route tables**:

### Route Tables

| Route Table | Purpose |
|--------------|----------|
| `RT-Customers` | For all customer spokes (CustA–J) |
| `RT-SharedServices` | For CustC (the shared VNet) |

---

### Configuration Summary

| Connection | Associated Route Table | Propagates To |
|-------------|------------------------|----------------|
| CustA–J | RT-Customers | RT-SharedServices |
| CustC | RT-SharedServices | RT-Customers |

---

### Routing Behavior

- Each customer spoke (A–J) **sends its routes only to RT-SharedServices**.
- The shared services VNet (CustC) **sends its routes to RT-Customers**.

This means:

✅ CustA ↔ CustC allowed  
✅ CustB ↔ CustC allowed  
❌ CustA ↛ CustB  
❌ CustD ↛ CustE  
✅ CustC ↔ All Customers

---

## 🧱 Diagram (Conceptual)

```
     +------------------------+
     |   vWAN Hub             |
     |                        |
     |  [RT-Customers]        | <---- CustA,B,D,E,F,G,H,I,J (Associate)
     |       ↑                |
     |       | Propagation    |
     |       ↓                |
     |  [RT-SharedServices]   | <---- CustC (Associate)
     +------------------------+

Traffic Flow:
CustA -> CustC ✅
CustB -> CustC ✅
CustA -> CustB ❌
CustD -> CustC ✅
CustC -> CustA,B,D,E,F,G,H,I,J ✅
```

---

## ⚙️ Azure CLI Example

```bash
# Create custom route tables
az network vhub route-table create   --name RT-Customers   --vhub-name MyVHub   --resource-group MyRG

az network vhub route-table create   --name RT-SharedServices   --vhub-name MyVHub   --resource-group MyRG

# Associate customer spokes
for cust in CustA CustB CustD CustE CustF CustG CustH CustI CustJ; do
  az network vhub connection update     --name ${cust}-Conn     --vhub-name MyVHub     --resource-group MyRG     --associated-route-table RT-Customers     --propagated-route-tables RT-SharedServices
done

# Associate shared services VNet
az network vhub connection update   --name CustC-Conn   --vhub-name MyVHub   --resource-group MyRG   --associated-route-table RT-SharedServices   --propagated-route-tables RT-Customers
```

---

## 🛡️ Best Practice Combo

| Layer | Function |
|--------|-----------|
| **vWAN route tables** | Handle network isolation (who can see whom) |
| **Azure Firewall / NVA** | Enforce deeper security policies, logging, or app control |

Example: Allow CustC shared services access (DNS, Bastion, API) while blocking unauthorized lateral traffic.

---

## ✅ Scalability and Manageability

| Aspect | Rating | Notes |
|--------|---------|-------|
| **Route table scalability** | ⭐⭐⭐⭐⭐ | Only 2 route tables needed regardless of customer count |
| **Performance** | ⭐⭐⭐⭐⭐ | Managed by vWAN, no bottleneck |
| **Isolation strength** | ⭐⭐⭐⭐☆ | Full L3 separation; can add Firewall for L7 control |
| **Ease of onboarding new customers** | ⭐⭐⭐⭐⭐ | Add → Associate → Done |

---

## ✅ TL;DR – Final Verdict

This **2-route-table vWAN architecture** is:
- **Highly scalable** — supports 100s of spokes.
- **Simple to manage** — minimal route tables.
- **Secure by design** — no customer-to-customer visibility.
- **Ready for shared services expansion** — CustC acts as the service hub.

---

**References**
- [Azure Virtual WAN Route Tables – Microsoft Docs](https://learn.microsoft.com/en-us/azure/virtual-wan/how-to-manage-route-tables)
- [Azure Firewall Manager with vWAN](https://learn.microsoft.com/en-us/azure/firewall-manager/overview)
