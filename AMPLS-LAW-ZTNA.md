# Secure Entra ID Audit Logs to LAW via ZTNA with AMPLS – Step-by-Step Guide

## Scenario
- **Audit logs from Entra ID** must be ingested into a Log Analytics Workspace (LAW).
- **Access to LAW logs must be restricted** to ZTNA-enabled devices only (NO public access).
- **Custom DNS** is in use.
- **Zscaler/Prisma/Azure Private Access/other ZTNA** is present.

---

## Step 1: Confirm/Create Log Analytics Workspace (LAW)
- I believe this is already done; still, if not, create via Azure Portal:
  - Azure Portal ==> Log Analytics workspaces ==> + Create
  - Fill in Subscription, Resource Group, Name, Region ==> Review + create

---

## Step 2: Create Azure Monitor Private Link Scope (AMPLS)

1. **Go to Azure Portal**
2. **Search “Azure Monitor Private Link Scope”** and click it
3. **Click “+ Create”**
4. **Fill:**
   - Subscription, Resource Group (same as LAW recommended)
   - Name (e.g., ampls-entra-law)
   - Region (any; best to match LAW’s region)
5. **Review + Create ==> Create**
6. **Wait for deployment, then click “Go to resource”**

---

## Step 3: Add LAW to AMPLS

1. In your AMPLS resource, go to **“Azure Monitor Resources”** in the left menu.
2. Click **“+ Add”**
3. Find/select your LAW, click **Add**
4. LAW should show as “Succeeded” in the resource list

---

## Step 4.0: Hub & Spoke Architecture – Where to Place the AMPLS Private Endpoint

### **Best Practice for AMPLS (Azure Monitor Private Link Scope):**
- **Create the Private Endpoint for AMPLS in the Hub VNet**  
  - **Why?**  
    - Centralizes monitoring network traffic.
    - Easier to manage DNS and firewall rules.
    - Spokes can access the monitoring services via VNet peering to the hub.
    - Keeps application workloads (spokes) isolated from monitoring endpoints.

#### **Steps:**

1. **In Azure Portal, go to AMPLS ==> Networking ==> Private Endpoint connections ==> Add.**
2. **Choose Hub VNet**  
   - Select the Hub VNet (not the Spoke VNet).
3. **Create a dedicated subnet for Private Endpoints in the Hub**  
   - Example: `Hub-PE-Subnet` with CIDR `/27` (e.g., `10.1.100.0/27`)
   - **Why dedicated?**  
     - Avoids IP exhaustion
     - Keeps NSG rules simple and secure
     - Only PEs live here; no VMs
4. **Enable “Integrate with Private DNS zones”**  
   - Unless you have complex custom DNS (see below Step 4.1 for custom DNS)
   - This ensures all VNets peered to the hub can resolve monitoring endpoints correctly

5. **Ensure “Network Policy for Private Endpoints” is enabled**
   - This allows NSG/UDR rules to apply to the subnet for additional control

---

### **If LAW is in a Spoke VNet:**
- **Still create the AMPLS PE in the Hub VNet**  
  - All spokes peered to the hub can access monitoring endpoints via the hub.
  - You do NOT need a PE in every spoke unless you have specific isolation requirements.

- **LAW in Spoke**  
  - AMPLS manages the connection; the PE in the hub enables secure access.
  
- **AMPLS Private Endpoint (PE) should be created in the Hub VNet.**
 
- **LAW itself can be deployed in a Spoke VNet.**
	- The actual resource (LAW) is in the Spoke, but its private access (via AMPLS Private Endpoint) is exposed in the Hub.

---

### **What VNet and Subnet to Choose?**

- **VNet:** Always choose the Hub VNet for the AMPLS PE.
- **Subnet:** Create a new subnet in the Hub VNet, dedicated for private endpoints.
  - Example: `10.1.100.0/27` (Hub-PE-Subnet)
  - **Do NOT reuse app or shared subnets!**

---

### **How Spoke VNets Access LAW via AMPLS PE in Hub:**

- **VNet Peering:**  
  - Ensure spoke VNets are peered to the hub VNet (allow traffic from spoke to hub).
- **DNS:**  
  - Peered VNets must be linked to the Private DNS zones (or custom DNS must forward queries).
- **Security:**  
  - NSG rules in the Hub-PE-Subnet control access; you can restrict which spokes can reach PEs.
- **Monitoring:**  
  - All monitoring traffic (LAW queries, agent traffic, etc.) flows through the hub’s PE for central oversight.

---

### **Final Guidance**
- **Always use Hub VNet for AMPLS PE unless you have a regulatory reason to isolate monitoring per spoke.**
- **Create a dedicated subnet for PEs in the hub.**
- **Peered spokes will access LAW securely via hub PE.**

---

## Example Diagram

```
+-----------------------+
|       Hub VNet        |
| +-------------------+ |
| | Hub-PE-Subnet     | |      <--- AMPLS Private Endpoint lives here
| +-------------------+ |
+-----------------------+
         |
   VNet Peering
         |
+-----------------------+      +-----------------------+
|     Spoke VNet 1      |      |     Spoke VNet 2      |
|     (Apps/LAW)        |      |    (Apps/LAW)         |
+-----------------------+      +-----------------------+

All monitoring traffic (LAW queries) from spokes routes to LAW via Hub PE.
```

---

## So consider this:

- **We are placing the AMPLS Private Endpoint in the Hub VNet for centralized monitoring and security.**
- **All LAW queries and monitoring traffic from any spoke will route securely and privately through the hub.**
- **This enables simple, scalable management and aligns with Azure best practices for large environments.**
  
## Step 4.1: Create a Private Endpoint for AMPLS

### Default (Azure DNS) Configuration

- When you create the Private Endpoint, select “Integrate with private DNS zones: Yes.”
- Azure auto-creates and links the following zones to your VNet:
  - privatelink.oms.opinsights.azure.com
  - privatelink.ods.opinsights.azure.com
  - privatelink.monitor.azure.com
  - privatelink.agentsvc.azure-automation.net
- **No further DNS configuration needed** for Azure VMs using Azure DNS (168.63.129.16).

### Custom DNS Configuration (On-Prem/Org-wide DNS)

```
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
# High level info
- During Private Endpoint creation for AMPLS, set “Integrate with private DNS zone” to No.
- Afterwards:
  - Provide the required Azure Private Link zone names to your DNS team:
  - privatelink.oms.opinsights.azure.com
  - privatelink.ods.opinsights.azure.com
  - privatelink.monitor.azure.com
  - privatelink.agentsvc.azure-automation.net
# DNS team will set up conditional forwarders to Azure DNS (168.63.129.16) or the IP of your Azure DNS Private Resolver inbound endpoint.

OPTIONAL:
If you use Storage Account exports or Application Insights, consider adding:
 - privatelink.blob.core.windows.net
 - privatelink.applicationinsights.azure.com
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

```

- If your devices (including ZTNA clients) use custom DNS, you **must manually configure conditional forwarders** for each Azure Private DNS zone.
- **Configure your DNS servers (e.g., Windows DNS):**
  - Add conditional forwarders for:
    - privatelink.oms.opinsights.azure.com
    - privatelink.ods.opinsights.azure.com
    - privatelink.monitor.azure.com
    - privatelink.agentsvc.azure-automation.net
  - Forward to Azure DNS (`168.63.129.16`) or to your Azure DNS Private Resolver inbound endpoint IP (if you have one).
- **Sample Windows DNS Manager Steps:**
  1. Open DNS Manager
  2. Right-click “Conditional Forwarders” > “New Conditional Forwarder…”
  3. Add the zone name (e.g., privatelink.oms.opinsights.azure.com)
  4. Add the Azure DNS IP (`168.63.129.16`), or your Azure DNS Private Resolver IP
  5. Repeat for each zone


### What to Choose: Default or Custom DNS?

| Scenario                                    | Recommendation     | Why                                                                |
|----------------------------------------------|--------------------|--------------------------------------------------------------------|
| Azure VMs only, no on-prem, no ZTNA         | Default (Azure DNS)| Simpler, automatic, no extra config                                |
| Org-wide, hybrid, ZTNA, custom DNS in use   | Custom DNS         | Required so devices using your DNS resolve LAW endpoints privately  |
| ZTNA clients use org DNS                    | Custom DNS         | Ensures ZTNA devices resolve to Private Endpoint, not public IP     |
| On-prem access to Azure via Private Link     | Custom DNS         | Same reason as above                                               |

**In summary:**  
- Use **default/Azure DNS** if all clients are Azure VMs using Azure DNS.  
- Use **custom DNS/conditional forwarders** if any client (on-prem, ZTNA, hybrid) uses org DNS.

**Why:**  
Custom DNS is essential to ensure all relevant endpoints resolve to the **private IP** in Azure, not the public one. Without this, ZTNA or on-prem clients will attempt to use the public endpoint, which is blocked if LAW is set to private-only!


---

## Step 5: Configure LAW Network Isolation (Private Only)

1. Go to your **LAW** in Azure Portal
2. Left menu ==> **Network isolation**
3. Ensure settings are:
   - **Query access:** Private only (disable public access)
   - **Ingestion access:**  
     - If you want Entra ID logs to work, **leave ingestion as “enabled” for public** (Entra ID diagnostic settings send logs via public endpoints).
     - If you want ingestion private-only and accept Entra logs may fail, set both to private only.
   - Save changes

---

## Step 6: Enable Entra ID Audit Log Ingestion

1. In Azure Portal, go to **Microsoft Entra ID** ==> **Monitoring** ==> **Diagnostic settings**
2. **+ Add diagnostic setting**
3. Set a name, **select “AuditLogs”** (and any others you need)
4. **Destination:**
   - Check “Send to Log Analytics workspace”
   - Choose your LAW (the one you added to AMPLS)
5. Save

---

## Step 7: Configure Private DNS for ZTNA Access

**For ZTNA clients to resolve LAW endpoints to the private IP, ensure:**

- The following FQDNs resolve to the private endpoint IP in your VNet:
  - `<workspace-id>.oms.opinsights.azure.com`
  - `<workspace-id>.ods.opinsights.azure.com`
  - `monitor.azure.com`
  - (and possibly others if you use more Azure Monitor features)
- If ZTNA clients use custom DNS, set up conditional forwarding for these domains to a DNS server in Azure that knows about your Private DNS Zones (Azure DNS Private Resolver or your own DNS VM).
- **ZTNA DNS Policy Example:**  
  - `*.oms.opinsights.azure.com` ==> forward to Azure DNS resolver
  - `*.ods.opinsights.azure.com` ==> forward to Azure DNS resolver
  - `monitor.azure.com` ==> forward to Azure DNS resolver

**If using Zscaler/Prisma/etc., document for ZTNA team:**
- List all required domains (above)
- Specify that these must resolve to the Azure VNet Private Endpoint IP (not public IP)
- DNS server IP(s) in Azure to which ZTNA should forward

---

## Step 8: ZTNA Access Policy for LAW

**To access/query LAW through ZTNA, provide ZTNA team:**

- **FQDNs:** As above (LAW workspace-specific and general Azure Monitor)
- **Private IP(s):** The actual IP(s) of your AMPLS Private Endpoint (find in Azure Portal ==> Private Endpoint resource ==> IP addresses)
- **TCP Ports:** 443 (HTTPS)
- **Policy:** Allow ZTNA users/devices to access these FQDNs via the ZTNA service, resolving to the private IPs, over TCP 443
- **Test from ZTNA device:**  
  - `nslookup <workspace-id>.oms.opinsights.azure.com` ==> private IP
  - Use Azure CLI/REST API/PowerShell from ZTNA device to query LAW (should succeed only when on ZTNA)

---

## Step 9: Test Access

1. **Try to query LAW from:**
   - A device NOT on ZTNA (should fail)
   - A device ON ZTNA (should succeed)
2. Use CLI, REST API, PowerShell, or Azure Portal (if allowed by ZTNA browser policies)

**Example CLI:**
```bash
az monitor log-analytics query \
  --workspace "<workspace-id>" \
  --analytics-query "AuditLogs | take 10" \
  --timespan PT1H
```

**Expected:**
- Success ONLY if device is using ZTNA and DNS is resolving to private IP

---


## Final Checklist for ZTNA Team

Give your ZTNA/network team this info:
- List of FQDNs to allow (see Step 7)
- Private endpoint IP(s) to allow
- TCP port: 443
- DNS policy: Forward Azure Monitor Private Link FQDNs to Azure DNS resolver
- Test procedure: nslookup and query from ZTNA device

---

---
