# Azure Virtual WAN: Scalable Shared Services Architecture for Hub & Spoke

<img width="2655" height="1922" alt="Untitled diagram-2025-10-22-114150" src="https://github.com/user-attachments/assets/457875c8-326d-4c0b-8a60-e1106872d09f" />

**Legend:**
- **Contoso OnPrem:** Your on-premises network, connected to Azure via VPN/ExpressRoute.
- **Hub (vWAN Hub):** Central point for connectivity and routing.
- **Customer Spokes (Customer 1–6):** Individual customer VNets. These are isolated from each other (no direct communication).
- **Shared Service Spoke:** Centralized VNet for shared services (e.g., DNS, Bastion, APIs). Accessible to all customer spokes and can communicate with each.
- **Route Table: Customers:** Associated with all customer spokes.
- **Route Table: Shared Service:** Associated with the shared service spoke.
- **Solid arrows:** Show route table association for each spoke.
- **Dotted arrows:** Show route table propagation (allowed communication flows).
- **Bidirectional arrows between Shared Service Spoke and Customers:** Indicate mutual communication is allowed.
- **No arrows between customer spokes:** Indicates no direct communication (isolation).

**Traffic Flow:**
- **Customer Spokes (1–6):**  
  - Can communicate with Hub and Shared Service Spoke.
  - Cannot communicate directly with other customer spokes.
- **Shared Service Spoke:**  
  - Can communicate with all customer spokes.
  - Acts as a central point for shared resources.

**How it works:**  
Route table associations and propagation ensure only the Shared Service Spoke can communicate with all customer spokes, and customer spokes are isolated from one another except via Shared Services.

---

## Overview

This document presents a **scalable, secure networking architecture** for Azure Virtual WAN (vWAN) using the Hub & Spoke model. The solution ensures:

- **Customer spokes (VNets):** Are isolated from each other.
- **Shared Services spoke:** (Master Spoke) is accessible to all customer spokes and can initiate communication with any spoke.

---

## Scenario

Suppose you have **6 spokes** in your vWAN setup, representing different customers, plus one shared services spoke (total 7 spokes). The requirements are:

- **Customer isolation:** Spokes cannot communicate with each other.
- **Shared Services access:** All spokes must access a central "Shared Services" spoke (e.g., for DNS, Bastion, APIs).
- **Shared Services spoke:** Must communicate with any spoke.

**Topology:**  
- **Contoso OnPrem:** Connects to Hub (via VPN/ExpressRoute).
- **Hub:** Has 6 spokes (Customer 1 to Customer 6) and 1 shared services spoke.
- **Shared Services Spoke:** Accessible to all spokes and vice versa.
- **Customer Spokes:** Cannot communicate with each other except via Shared Services.

---

## Solution: Route Table Design

To achieve these requirements **efficiently and at scale**, we use only **two route tables**:

| Route Table         | Description                                   |
|---------------------|-----------------------------------------------|
| RT-Customers        | Associated with all customer spokes (Customer 1–6)     |
| RT-SharedServices   | Associated with the Shared Services spoke     |

### Route Table Propagation

| Spoke/Connection         | Associated Route Table | Propagated To          |
|--------------------------|-----------------------|------------------------|
| Customer Spokes (1–6)    | RT-Customers          | RT-SharedServices      |
| Shared Services Spoke    | RT-SharedServices     | RT-Customers           |

#### **Traffic Flow Results**

| From      | To                   | Allowed? |
|-----------|----------------------|----------|
| Cust1-6     | Shared Service Spoke | Yes       |
| Cust1-6     | Cust6-1                | **No**       |
| Shared Service Spoke | Cust1–6   | Yes       |
| Any Spoke | Hub                  | Yes       |
| Any Spoke | Other Spoke          | **No**       |

---

## Components

- **Azure Virtual WAN Hub:** Centralized networking, routing, and security.
- **Spokes (VNets):** Individual customer networks (Customer 1–6).
- **Shared Services Spoke (Master Spoke):** Hosts common services (e.g., DNS, Bastion, API).
- **vWAN Route Tables:** Provides logical segmentation and controls traffic flow.
- **Azure Firewall / NVA (optional):** For advanced security, traffic filtering, and logging.

---

## Implementation Steps

### 1. Create Route Tables

```bash
az network vhub route-table create \
  --name RT-Customers \
  --vhub-name MyVHub \
  --resource-group MyRG

az network vhub route-table create \
  --name RT-SharedServices \
  --vhub-name MyVHub \
  --resource-group MyRG
```

### 2. Associate Spokes

```bash
for cust in Customer1 Customer2 Customer3 Customer4 Customer5 Customer6; do
  az network vhub connection update \
    --name ${cust}-Conn \
    --vhub-name MyVHub \
    --resource-group MyRG \
    --associated-route-table RT-Customers \
    --propagated-route-tables RT-SharedServices
done
```

### 3. Associate Shared Services

```bash
az network vhub connection update \
  --name SharedService-Conn \
  --vhub-name MyVHub \
  --resource-group MyRG \
  --associated-route-table RT-SharedServices \
  --propagated-route-tables RT-Customers
```

---

## Best Practices

| Layer                 | Function                                                   |
|-----------------------|-----------------------------------------------------------|
| **vWAN route tables** | Ensures network isolation and controlled connectivity      |
| **Azure Firewall/NVA**| Adds L7 security, logging, and granular access control     |

- **Onboard new customers:** Just associate their VNet with RT-Customers. No need to reconfigure all route tables.
- **Scale easily:** Only two route tables required, regardless of customer count.

---

## References

- [Azure Virtual WAN Route Tables – Microsoft Documentation](https://learn.microsoft.com/en-us/azure/virtual-wan/how-to-manage-route-tables)
- [Azure Firewall Manager with vWAN](https://learn.microsoft.com/en-us/azure/firewall-manager/overview)
- [Hub and Spoke network topology in Azure](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/hub-spoke)

---

## Detailed Explanation

### **What Are We Achieving?**

- **Customer Isolation:** Each customer (spoke) can only access shared services and the hub, not other customers.
- **Centralized Services:** All spokes can leverage shared infrastructure (DNS, Bastion, etc.) securely.
- **Operational Simplicity:** Minimal route table management, easy onboarding, and high scalability.

### **How Is It Possible?**

- Using **vWAN route table associations and propagation**, we precisely control which VNets can reach which others:
    - **Customer spokes** can only reach shared services.
    - **Shared services spoke** can reach any customer.
    - **No customer-to-customer traffic** is possible unless explicitly allowed.

### **Essential Components**

1. **vWAN Hub:** Central point for routing and connectivity.
2. **Spoke VNets:** Individual customer networks attached to the hub.
3. **Route Tables:** Logical separation (RT-Customers, RT-SharedServices) to define allowed communication.
4. **Firewall/NVA (optional):** Advanced inspection and policy enforcement.
5. **Azure CLI/Portal:** For provisioning and configuration.

---

## Conclusion

This **2-route-table vWAN architecture** is:

- **Highly scalable:** Supports any number of spokes with just 2 route tables.
- **Secure by design:** No lateral communication between customer spokes.
- **Easy to manage:** Minimal configuration, straightforward customer onboarding.
- **Ready for shared services expansion:** The shared services spoke acts as a robust service hub.

For further details, consult [Azure Virtual WAN Route Tables – Microsoft Docs](https://learn.microsoft.com/en-us/azure/virtual-wan/how-to-manage-route-tables).

---

