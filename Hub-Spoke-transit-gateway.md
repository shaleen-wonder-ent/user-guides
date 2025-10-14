# Hub-Spoke Architecture: Contoso On-Premises Connectivity via VPN Gateway

## Overview

This document explains how **Contoso On-Premises** infrastructure connects to the **Azure Hub-Spoke network** architecture using a **VPN Gateway**.

---

## Network Topology

- **Hub VNet** hosts the **VPN Gateway**.
- **Spoke VNets** (Spoke 1, Spoke 2, etc.) connect to the Hub using **VNet Peering**.
- **Contoso On-Premises** connects to the **Hub** via a **Site-to-Site (S2S) VPN**.

### Objective

Provide network connectivity between **Contoso On-Premises** and **Cust1 Azure Subscription** hosted in **Spoke 1**.

---

## Connectivity Flow

1. **Site-to-Site VPN (Hub <-> Contoso On-Premises)**  
   - A route-based VPN Gateway is deployed in the **Hub VNet**.
   - The VPN connection terminates at the on-premises VPN device.
   - (Optional) Use **BGP** for automatic route propagation.

2. **VNet Peering (Hub <-> Spoke 1)**  
   - Spoke 1 is peered with the Hub VNet.
   - Configure peering as follows:
     - **Hub → Spoke**: Allow gateway transit = ✅, Allow forwarded traffic = ✅
     - **Spoke → Hub**: Use remote gateways = ✅, Allow virtual network access = ✅

3. **Route Propagation**
   - If BGP is enabled, on-premises routes are automatically propagated to Spoke 1.
   - If not using BGP:
     - Define **User Defined Routes (UDRs)** in Spoke 1 that direct on-premises traffic to the Hub Gateway or NVA.
   - Verify that the Hub route table includes on-premises prefixes.

4. **Network Security Configuration**
   - Ensure **NSGs** allow inbound and outbound traffic between Spoke 1 and on-premises address ranges.
   - If an **Azure Firewall** or **NVA** is in the Hub, configure routes and policies to permit the traffic.
   - Enable **Allow forwarded traffic** on all relevant peerings.

---

## Azure CLI Example

```bash
# Hub to Spoke Peering
az network vnet peering create -g RG -n HubToSpoke --vnet-name HubVnet --remote-vnet SpokeVnet   --allow-vnet-access true --allow-gateway-transit true --allow-forwarded-traffic true

# Spoke to Hub Peering
az network vnet peering create -g RG -n SpokeToHub --vnet-name SpokeVnet --remote-vnet HubVnet   --allow-vnet-access true --use-remote-gateways true --allow-forwarded-traffic true
```

---

## Notes & Best Practices

- Only one VNet in a peering group can use a remote gateway.
- For multiple customers or large-scale environments, use **Azure Virtual WAN / Virtual Hub** for better isolation and simplified routing.
- Validate traffic using `Network Watcher → Connection Troubleshoot`.

---

## Summary

✅ **Contoso On-Premises** connects to the **Hub VNet** via VPN Gateway.  
✅ **Spoke 1 (Cust1 Azure Subscription)** connects to the Hub using **VNet Peering** with **Use Remote Gateway** enabled.  
✅ Traffic flows between **Cust1 Azure workloads** in Spoke 1 and **Contoso On-Premises** through the **Hub VPN Gateway**.

---

## Another way of looking at it

✅ **Solution Overview**

- Hub VNet hosts a single VPN Gateway.
- This gateway establishes multiple Site-to-Site VPN connections:
   - One to your corporate on-prem (Contoso).
   - One to each customer’s on-prem environment (Cust1, Cust2, Cust3, etc.).
- Spoke VNets (customer subscriptions) connect to the hub via VNet Peering.
- Spokes use Use Remote Gateway so they can route through the hub’s VPN Gateway.
- Hub peering enables Allow Gateway Transit to act as a transit network.
- Routing is controlled via BGP or UDRs to ensure isolation.


✅ Architecture Diagram


<img width="738" height="307" alt="image" src="https://github.com/user-attachments/assets/98431e86-7130-435b-b285-cd905dbdfcf3" />



✅ Step-by-Step Implementation
**Step 1: Create Hub VNet and VPN Gateway**
- Create a Hub VNet with a dedicated GatewaySubnet.
- Deploy an Azure VPN Gateway in the Hub VNet.
- Configure the first Site-to-Site VPN connection to your corporate on-prem (HCL).

**Step 2: Add Multiple Site-to-Site Connections**
- For each customer on-prem:
  - Add a new Local Network Gateway (representing customer on-prem IP and address space).
  - Create a Site-to-Site VPN connection from the hub VPN Gateway to that Local Network Gateway.
- Ensure IP ranges do not overlap across customers.

**Step 3: Configure VNet Peering** 
- For each spoke VNet:
  - Peer with the hub VNet.
  - Enable Use Remote Gateway.
- On the hub VNet:
  - Enable Allow Gateway Transit for each peering.

**Step 4: Routing**
- If using BGP, advertise routes for each on-prem network.
- If using UDRs, create route tables in spokes:
  - Default route for on-prem traffic → Hub VNet.
- Apply NSGs or route filters to prevent cross-spoke traffic.

**Step 5: Isolation Controls**
- Use NSGs to block traffic between spokes.
- Optionally deploy an NVA or Azure Firewall in the hub for advanced segmentation.

**Pros & Cons**

- ✅ Single gateway, simpler management.
- ✅ Lower cost compared to multiple gateways.
- ❌ Requires careful routing and security to prevent cross-spoke access.

