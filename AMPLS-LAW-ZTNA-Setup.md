# Contoso Azure LAW, AMPLS, Private Endpoint & ZTNA Integration Setup

---

## Architecture Diagram

<img style="width:110%; max-width:100%; height:auto;" alt="Architecture Diagram" src="https://github.com/user-attachments/assets/0390f9c5-c08b-4ba4-8502-64413256edb7" />


---

## 1. Resource Structure

### LAW (Log Analytics Workspace)
- **Resource Group:** `centralLAWContosoRG`
  - **LAW Resource:** `centralLAWContoso`

### AMPLS (Azure Monitor Private Link Scope)
- **Resource Group:** `centralLAWContosoRG`
  - **AMPLS Resource:** `centralLAWContoso-AMPLS`

### VNets
- **Central Region**
  - **Resource Group:** `HUB-VNET-Central-RG`
    - **VNet:** `HUB-VNET-Central`
    - **Subnet:** `HUB-VNET-SUBNET-Central`
- **South Region**
  - **Resource Group:** `HUB-VNET-South-RG`
    - **VNet:** `HUB-VNET-South`
    - **Subnet:** `HUB-VNET-SUBNETSouth`

### Private Endpoints (PEs)
- **AMPLS Private Endpoint Connections:**
  - `ampls-PE-Central-VNet` in `HUB-VNET-Central` (Central)
  - `ampls-PE-South-VNet` in `HUB-VNET-South` (South)

---

## 2. Conditional Forwarding Setup

### Steps

1. **Identify Azure Private DNS Zones Needed for AMPLS/LAW:**
   - `privatelink.oms.opinsights.azure.com`
   - `privatelink.ods.opinsights.azure.com`
   - `privatelink.monitor.azure.com`
   - `privatelink.agentsvc.azure-automation.net`

2. **Find Azure DNS Private Resolver Inbound Endpoint IPs:**
   - Central VNet: Deploy (if not present) Azure DNS Private Resolver with inbound endpoint (e.g., `10.10.10.4`)
   - South VNet: Deploy Azure DNS Private Resolver with inbound endpoint (e.g., `10.20.20.4`)

3. **Configure Conditional Forwarders on Custom DNS Server:**
   - On your organization’s DNS servers, create conditional forwarders for each zone above:
     - Example:
       - Zone: `privatelink.oms.opinsights.azure.com`
       - Forward to: Central inbound endpoint IP (`10.10.10.4`)
     - Repeat for each required zone and for South inbound endpoint IP (`10.20.20.4`).

4. **For High Availability (HA):**
   - For each Private Link zone, add both Central and South inbound endpoint IPs as forwarders.
   - This ensures DNS queries resolve even if one region is down.

---

## 3. Technical Notes & Actions

### Network & Security

- **DNS Traffic:**  
  DNS traffic will not work unless firewalls allow it!  
  Explicitly open **UDP/53** and **TCP/53** from DNS server IPs to the inbound endpoint IP on all security devices in the path (NSG, Fortigate, Palo Alto).

- **Azure DNS Private Resolver Inbound Endpoint:**  
  - Not exposed to the internet.
  - Responds only to DNS queries sent over private IP within Azure or via ExpressRoute.

- **Action Checklist:**
  - Allow DNS traffic (UDP/53, TCP/53) from your DNS server IPs/subnets to inbound endpoint IPs in NSG and Fortigate firewall policies.
  - If DNS queries come from on-prem or through DMZ, Palo Alto must allow DNS traffic to Azure inbound endpoint.
  - Ensure ExpressRoute/SD-WAN routes DNS traffic properly.

### Traffic Path (for LAW Query)

- **From Internal:**  
  `Client (on-prem, Azure VM, ZTNA) --> Internal Firewall (Fortigate HUB) --> Private Endpoint (AMPLS PE in HUB) --> LAW`
- **From External:**  
  `Palo Alto (DMZ) --> Fortigate (HUB) --> Private Endpoint (AMPLS PE) --> LAW`

---

## 4. ZTNA Access for LAW Queries

### Steps

1. **Identify FQDNs to Secure:**
   - List LAW endpoints/FQDNs to publish as ZTNA resources:
     - `<workspace-id>.oms.opinsights.azure.com`
     - `<workspace-id>.ods.opinsights.azure.com`
     - `monitor.azure.com`

2. **ZTNA Resource Publishing:**
   - Assign these resources to ZTNA connector/gateway(s) with network access to Azure Private Endpoint.

3. **ZTNA Connector DNS Configuration:**
   - Use custom DNS servers (with conditional forwarders to Azure DNS Private Resolver).

4. **ZTNA Connector Routing:**
   - Ensure ZTNA connector can route to LAW Private Endpoint via HUB firewall.

5. **Recommended Approach: DNS for ZTNA Connector FQDN Resolution**
   - ZTNA Connector uses corporate DNS servers for name resolution.
   - Corporate DNS configured with conditional forwarders:
     - For all Azure Private Link FQDNs
     - Forward queries to Azure DNS Private Resolver inbound endpoints.
   - Azure DNS Private Resolver inbound endpoint:
     - Resolves FQDNs to correct Private Endpoint IP (registered in Azure Private DNS Zone).
     - Always returns current, correct private IP, even if endpoint changes.

6. **Key Points:**
   - ZTNA connector/gateway must use enterprise DNS (with conditional forwarders).
   - When ZTNA connector/gateway queries the FQDN, the DNS response is the private IP registered in the Azure Private DNS Zone—not the public IP.
   - Azure Private Endpoints register private IPs for their FQDNs in the Private DNS Zone.
   - Conditional forwarders ensure FQDN queries are answered by the Azure Private DNS Zone, always returning the private IP.
   - If ZTNA gateway uses this DNS setup, it will never resolve the public IP (unless DNS setup is wrong/missing).

---

## 5. Technical Clarifications

- **Conditional Forwarders for HA:**  
  You can enter both Central and South inbound endpoint IPs as conditional forwarders in your DNS server for each zone. DNS will round-robin or automatically failover if one endpoint is unavailable.

- **Firewall/NSG Ports:**  
  Explicitly ensure both UDP/53 and TCP/53 are permitted for DNS traffic between your DNS server IPs/subnets and the Azure DNS Private Resolver inbound endpoint IPs.

- **ZTNA Connector Routing:**  
  NSG and firewall rules must allow traffic from the ZTNA connector subnet to the AMPLS PE subnet (**TCP/443**). All other access should be denied for Zero Trust enforcement.

- **Private DNS Zone Linking:**  
  Both HUB VNets (Central and South) must be linked to the same Private DNS Zone in Azure, so both Private Endpoints register their IPs and are discoverable by DNS.

- **DR/Failover Commentary:**  
  If one HUB or endpoint is down, DNS will respond with the other HUB’s Private Endpoint IP (if both are registered). Ensure routing and firewall policies support this scenario for business continuity.

---

## 6. Validation & Best Practice

- **Test DNS Resolution:**  
  Run `nslookup <workspace-id>.oms.opinsights.azure.com` from ZTNA connector/gateway. It should resolve to the Private Endpoint IP.

- **Test Connectivity:**  
  Attempt LAW query via ZTNA. Confirm access succeeds only when routed via ZTNA.

- **Monitor & Audit:**  
  Regularly review firewall/NSG and DNS logs to ensure Zero Trust posture is enforced.

---
