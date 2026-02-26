## 7. DNS Resolution Issues with AMPLS

### 7.1 Understanding DNS Resolution with AMPLS

When you implement Azure Monitor Private Link Scope (AMPLS), DNS resolution becomes critical because it determines whether traffic flows through your private endpoint (secure, via private network) or through the public endpoint (over the internet). Unlike standard Azure services where DNS automatically updates when you create a private endpoint, AMPLS requires careful configuration of DNS zones, conditional forwarders, and network connectivity to ensure proper resolution across hybrid environments.

**Why DNS is Critical for AMPLS:**
- **Security**: Incorrect DNS resolution can cause traffic to bypass your private endpoint, defeating the security purpose of AMPLS
- **Connectivity**: On-premises systems need proper DNS forwarding to resolve private endpoint IPs
- **Compliance**: Regulatory requirements may mandate that log data never traverses public networks
- **Troubleshooting**: Most AMPLS connectivity issues stem from DNS misconfiguration rather than network routing

**The DNS Challenge:**
Azure Monitor uses multiple FQDNs for different functions (data ingestion, querying, agent management), and each must resolve to your private endpoint IP rather than public Azure IPs. In hybrid environments with on-premises systems, this requires coordination between Azure Private DNS Zones and on-premises DNS infrastructure.

---

### 7.2 Common DNS Issues

#### Issue #1: DNS Resolves to Public IP Instead of Private IP

**Symptom:**
```bash
nslookup workspace-id.ods.opinsights.azure.com
# Returns: Public IP (13.x.x.x or 20.x.x.x) instead of Private IP (10.x.x.x)
```

**What This Means:**
Your client (VM, on-prem server, or user workstation) is resolving the Log Analytics workspace endpoint to Azure's public IP address. This causes traffic to flow over the internet instead of through your private endpoint, completely bypassing the security and network isolation that AMPLS provides.

**Root Causes:**
1. **Private DNS Zone not linked to VNet**: The Private DNS Zone exists but isn't linked to the VNet where your resources reside
2. **Conditional forwarders not configured**: On-premises DNS servers don't know to forward Azure Monitor queries to Azure DNS
3. **DNS cache contains old records**: Previous public IP resolutions are cached on the client or intermediate DNS servers
4. **VNet DNS settings incorrect**: VNet is configured to use custom DNS servers that don't forward to Azure DNS (168.63.129.16)
5. **Split-brain DNS misconfigured**: Public DNS zone conflicts with private DNS zone

**Step-by-Step Resolution:**

**Step 1: Verify Private DNS Zone Exists and Has Records**
```bash
# Navigate in Azure Portal:
Private DNS Zones → privatelink.ods.opinsights.azure.com

# Check for A records like:
# Name: <workspace-id>
# IP: 10.1.100.50 (your private endpoint IP)
```

**Step 2: Verify VNet Link**
```bash
# In Private DNS Zone:
Virtual network links → Should show your Hub VNet (or spoke VNets)

# Important: Auto-registration should be DISABLED for AMPLS zones
# (A records are created by private endpoint, not VMs)
```

**Step 3: Check VNet DNS Configuration**
```bash
# Navigate to: VNet → DNS servers
# Should be one of:
# - Default (Azure-provided DNS) ← Best for Azure VMs
# - Custom: 10.1.0.4, 10.1.0.5 (your custom DNS VMs that forward to 168.63.129.16)
```

**Step 4: Configure On-Premises Conditional Forwarders**
```bash
# On your on-prem DNS server (192.168.1.10):
# Create conditional forwarders for each zone:

Zone: ods.opinsights.azure.com
Forward to: 10.1.0.4, 10.1.0.5 (Azure DNS VMs in Central India)

Zone: oms.opinsights.azure.com  
Forward to: 10.1.0.4, 10.1.0.5

Zone: agentsvc.azure-automation.net
Forward to: 10.1.0.4, 10.1.0.5

Zone: monitor.azure.com
Forward to: 10.1.0.4, 10.1.0.5
```

**Step 5: Flush DNS Cache**
```bash
# On Windows client:
ipconfig /flushdns
ipconfig /displaydns  # Verify cache is empty

# On Linux client:
sudo systemd-resolve --flush-caches
sudo systemd-resolve --statistics  # Verify cache stats

# On Windows DNS Server:
Clear-DnsServerCache

# On the DNS VM itself (if using custom DNS):
ipconfig /flushdns
Restart-Service DNS  # If running Windows DNS
```

**Step 6: Test Resolution**
```bash
# From on-prem PC:
nslookup workspace-id.ods.opinsights.azure.com 192.168.1.10

# Expected output:
Server:  on-prem-dns.contoso.com
Address: 192.168.1.10

Name:    workspace-id.ods.opinsights.azure.com
Address: 10.1.100.50  # ← Private IP, SUCCESS!

# If still returns public IP, check next steps
```

**Step 7: Verify End-to-End Path**
```bash
# From on-prem PC, trace the resolution path:
nslookup -debug workspace-id.ods.opinsights.azure.com

# From Azure DNS VM (10.1.0.4), test direct resolution:
nslookup workspace-id.ods.opinsights.azure.com

# This should return private IP since the VM can query Azure DNS (168.63.129.16)
```

---

#### Issue #2: Missing or Incomplete Private DNS Zones

**Symptom:**
Some Azure Monitor functions work (e.g., log queries) but others fail (e.g., agent connectivity, diagnostics upload, automation)

**What This Means:**
Azure Monitor uses multiple endpoints for different purposes. If you only created DNS zones for the primary workspace endpoints but not supporting services, partial functionality will occur. For example, logs might be queryable but the Azure Monitor Agent cannot connect to report health status.

**Required DNS Zones for Full LAW + AMPLS Functionality:**

| DNS Zone | Purpose | Example FQDN | Without It |
|----------|---------|--------------|------------|
| `privatelink.monitor.azure.com` | Azure Monitor global endpoint | `global.handler.control.monitor.azure.com` | Data collection fails; agent cannot register |
| `privatelink.oms.opinsights.azure.com` | Log ingestion endpoint | `<workspace-id>.oms.opinsights.azure.com` | Cannot ingest new log data |
| `privatelink.ods.opinsights.azure.com` | Data access endpoint | `<workspace-id>.ods.opinsights.azure.com` | Cannot query existing log data |
| `privatelink.agentsvc.azure-automation.net` | Azure Automation/Update Mgmt | `<guid>.agentsvc.azure-automation.net` | Update Management, Change Tracking fail |
| `privatelink.blob.core.windows.net` | Diagnostics storage | `<account>.blob.core.windows.net` | VM diagnostics upload fails |

**How to Identify Missing Zones:**
```bash
# Enable NSG flow logs or use Azure Monitor agent logs to see failed DNS queries
# Look for public IP connections instead of private IP

# Check agent logs on a VM:
# Windows: C:\WindowsAzure\Logs\TransparentInstaller.log
# Linux: /var/opt/microsoft/omsagent/log/omsagent.log

# Look for errors like:
# "Failed to connect to workspace-id.oms.opinsights.azure.com (13.x.x.x)"
# ^ This public IP indicates missing or misconfigured DNS zone
```

**Resolution:**

**Step 1: Create All Required Private DNS Zones**
```bash
# Via Azure Portal:
Create a resource → Private DNS zone

# Create each zone:
1. privatelink.monitor.azure.com
2. privatelink.oms.opinsights.azure.com  
3. privatelink.ods.opinsights.azure.com
4. privatelink.agentsvc.azure-automation.net
5. privatelink.blob.core.windows.net  # Only if using diagnostics

# Via Azure CLI:
az network private-dns zone create \
  --resource-group <rg-name> \
  --name privatelink.oms.opinsights.azure.com

# Repeat for each zone
```

**Step 2: Link Each Zone to Your Hub VNet**
```bash
# For each Private DNS Zone:
Virtual network links → Add

# Settings:
- Link name: link-to-hub-vnet
- Virtual network: <your-hub-vnet>
- Enable auto registration: NO (for AMPLS, leave unchecked)

# Via Azure CLI:
az network private-dns link vnet create \
  --resource-group <rg-name> \
  --zone-name privatelink.oms.opinsights.azure.com \
  --name link-to-hub \
  --virtual-network <hub-vnet-id> \
  --registration-enabled false
```

**Step 3: Verify A Records Exist**
```bash
# When you create a private endpoint, Azure automatically creates A records
# in the appropriate Private DNS Zones (if they're linked)

# Check each zone for records:
Private DNS Zone → Overview → Recordsets

# Example for workspace-id abc123:
# Zone: privatelink.ods.opinsights.azure.com
# Record: abc123  → 10.1.100.50
#
# Zone: privatelink.oms.opinsights.azure.com  
# Record: abc123  → 10.1.100.50

# If records are missing, delete and recreate the private endpoint
```

**Step 4: Update Conditional Forwarders for All Zones**
```bash
# On on-prem DNS (192.168.1.10), add forwarders for each:

ods.opinsights.azure.com         → 10.1.0.4, 10.1.0.5
oms.opinsights.azure.com         → 10.1.0.4, 10.1.0.5
agentsvc.azure-automation.net    → 10.1.0.4, 10.1.0.5
monitor.azure.com                → 10.1.0.4, 10.1.0.5
blob.core.windows.net            → 10.1.0.4, 10.1.0.5  # If needed
```

---

#### Issue #3: AMPLS Access Mode Configuration Problem

**Symptom:**
- DNS sometimes resolves to private IP, sometimes to public IP
- Inconsistent connectivity between different VMs or users
- Connection works from Azure VMs but not from on-premises

**What This Means:**
AMPLS has a network isolation setting that controls whether resources can be accessed from public networks. If set to "Open" (accept public access), clients may randomly choose public or private endpoints based on DNS TTL, caching, or routing. This creates unpredictable behavior and defeats the security purpose of AMPLS.

**Understanding AMPLS Access Modes:**

| Mode | Setting | Behavior | When to Use |
|------|---------|----------|-------------|
| **Open** | Accept public access = Yes | Allows access via both private endpoint AND public endpoint | Migration/testing phase only |
| **Private Only** | Accept public access = No | Forces all access through private endpoint; public access blocked | Production (recommended) |

**Why "Open" Mode Causes DNS Issues:**
- Clients on public networks get public IPs from DNS
- Clients on private networks get private IPs from DNS  
- Creates split behavior and confusion
- Security teams can't enforce private-only connectivity

**Check Current Configuration:**

**Via Azure Portal:**
```bash
Navigate to: AMPLS Resource → Network Isolation

Settings:
1. "Accept access from public networks not connected through a Private Link Scope"
   - YES = Open mode (⚠️ Not recommended for production)
   - NO = Private Only mode (✅ Recommended)

2. "Accept access only from resources within the Private Link Scope"
   - YES = Only LAW resources added to AMPLS are accessible
   - NO = All LAW resources in subscription are accessible via this PE
```

**Via Azure CLI:**
```bash
# Check current settings:
az monitor private-link-scope show \
  --name <ampls-name> \
  --resource-group <rg-name> \
  --query "{publicNetworkAccess:publicNetworkAccess, accessModeSettings:accessModeSettings}"

# Output:
{
  "publicNetworkAccess": "Enabled",  # ← This means "Open" mode
  "accessModeSettings": {
    "ingestionAccessMode": "Open",
    "queryAccessMode": "Open"
  }
}
```

**Resolution - Set to Private Only Mode:**

**Via Azure Portal:**
```bash
AMPLS → Network Isolation

Change settings to:
☑ Accept access from public networks not connected through a Private Link Scope: NO
☑ Accept access only from resources within the scope: YES (recommended)

Click Save
```

**Via Azure CLI:**
```bash
# Set to private only:
az monitor private-link-scope update \
  --name <ampls-name> \
  --resource-group <rg-name> \
  --query-access-mode PrivateOnly \
  --ingestion-access-mode PrivateOnly

# Verify:
az monitor private-link-scope show \
  --name <ampls-name> \
  --resource-group <rg-name> \
  --query "accessModeSettings"
```

**Important Considerations:**
- After switching to Private Only, public internet clients CANNOT access the workspace
- Ensure all VMs, on-prem systems, and users have connectivity via private endpoint BEFORE switching
- Test thoroughly in non-production first
- Plan a maintenance window for production cutover

**Testing Checklist Before Switching to Private Only:**
- [ ] All Azure VMs can resolve workspace FQDN to private IP
- [ ] On-premises systems can resolve workspace FQDN to private IP  
- [ ] Test from Azure VM: `Test-NetConnection workspace-id.ods.opinsights.azure.com -Port 443`
- [ ] Test from on-prem: `Test-NetConnection workspace-id.ods.opinsights.azure.com -Port 443`
- [ ] Azure Monitor Agent shows "Connected" status
- [ ] Log queries execute successfully from all locations
- [ ] Have rollback plan (can switch back to Open if issues occur)

---

#### Issue #4: Conditional Forwarder Not Working

**Symptom:**
```bash
# From on-prem PC:
nslookup workspace-id.ods.opinsights.azure.com
# Returns public IP

# But from Azure DNS VM directly:
nslookup workspace-id.ods.opinsights.azure.com
# Returns private IP  ✓
```

**What This Means:**
The Azure DNS VMs (10.1.0.4, 10.1.0.5) can correctly resolve the private endpoint, but on-premises clients aren't reaching those DNS VMs. This indicates a problem with the conditional forwarder configuration or network connectivity between on-premises and Azure DNS infrastructure.

**Common Causes:**
1. Conditional forwarder doesn't exist or is configured incorrectly
2. On-premises DNS cannot reach Azure DNS VMs (firewall/NSG blocking)
3. Conditional forwarder uses wrong IP addresses
4. DNS query doesn't match the forwarder zone (specificity issue)
5. On-premises firewall blocks UDP 53 to Azure

**Diagnostic Steps:**

**Step 1: Verify Conditional Forwarder Exists**
```powershell
# On on-prem DNS server (Windows):
Get-DnsServerZone | Where-Object {$_.ZoneType -eq "Forwarder"}

# Look for:
# ZoneName: ods.opinsights.azure.com
# MasterServers: {10.1.0.4, 10.1.0.5}

# Or via DNS Manager GUI:
# DNS Manager → Conditional Forwarders → Check for "ods.opinsights.azure.com"
```

**Step 2: Test Forwarder Directly**
```bash
# From on-prem DNS server, query Azure DNS VM directly:
nslookup workspace-id.ods.opinsights.azure.com 10.1.0.4

# If this works, forwarder routing is the issue
# If this fails, network connectivity is the issue
```

**Step 3: Check Network Connectivity**
```bash
# From on-prem DNS server to Azure DNS VM:
Test-NetConnection -ComputerName 10.1.0.4 -Port 53
Test-NetConnection -ComputerName 10.1.0.5 -Port 53

# Expected output:
# TcpTestSucceeded : True (for TCP)
# PingSucceeded : True (if ICMP allowed)

# If fails: Check ExpressRoute/VPN connectivity, NSG rules, Azure Firewall
```

**Step 4: Check Forwarder IP Addresses**
```bash
# Ensure forwarder points to correct IPs
# Common mistake: Using private endpoint IP instead of DNS VM IP

# WRONG:
Forward to: 10.1.100.50  # ← This is the PE IP, not DNS!

# CORRECT:  
Forward to: 10.1.0.4, 10.1.0.5  # ← DNS VMs that can reach Azure DNS
```

**Step 5: Check Zone Specificity**
```bash
# DNS uses most specific match
# If you have both:
#   opinsights.azure.com → Old forwarder
#   ods.opinsights.azure.com → New forwarder
# The more specific one wins (ods.opinsights.azure.com)

# Verify with:
nslookup -debug workspace-id.ods.opinsights.azure.com

# Look for "Using domain suffix search list"
# and "Sending request to server" to see which forwarder is used
```

**Resolution:**

**Create/Fix Conditional Forwarder (Windows DNS):**
```powershell
# Remove old forwarder if exists:
Remove-DnsServerZone -Name "ods.opinsights.azure.com" -Force

# Create correct forwarder:
Add-DnsServerConditionalForwarderZone `
  -Name "ods.opinsights.azure.com" `
  -MasterServers 10.1.0.4,10.1.0.5 `
  -ForwarderTimeout 5

# Repeat for all required zones:
Add-DnsServerConditionalForwarderZone `
  -Name "oms.opinsights.azure.com" `
  -MasterServers 10.1.0.4,10.1.0.5 `
  -ForwarderTimeout 5

Add-DnsServerConditionalForwarderZone `
  -Name "monitor.azure.com" `
  -MasterServers 10.1.0.4,10.1.0.5 `
  -ForwarderTimeout 5

Add-DnsServerConditionalForwarderZone `
  -Name "agentsvc.azure-automation.net" `
  -MasterServers 10.1.0.4,10.1.0.5 `
  -ForwarderTimeout 5

# Verify:
Get-DnsServerZone | Where-Object {$_.ZoneType -eq "Forwarder"} | Format-Table
```

**Create Conditional Forwarder (Linux BIND DNS):**
```bash
# Edit named.conf:
sudo nano /etc/bind/named.conf.local

# Add forwarder zones:
zone "ods.opinsights.azure.com" {
    type forward;
    forward only;
    forwarders { 10.1.0.4; 10.1.0.5; };
};

zone "oms.opinsights.azure.com" {
    type forward;
    forward only;
    forwarders { 10.1.0.4; 10.1.0.5; };
};

zone "monitor.azure.com" {
    type forward;
    forward only;
    forwarders { 10.1.0.4; 10.1.0.5; };
};

zone "agentsvc.azure-automation.net" {
    type forward;
    forward only;
    forwarders { 10.1.0.4; 10.1.0.5; };
};

# Restart DNS:
sudo systemctl restart bind9

# Verify:
dig @localhost workspace-id.ods.opinsights.azure.com
```

**Fix Network Connectivity (if Test-NetConnection failed):**
```bash
# Check ExpressRoute/VPN connectivity:
# Azure Portal → VPN Gateway → Connections → Status = Connected

# Check NSG on Azure DNS VM subnet:
# Allow inbound UDP 53 from on-prem IP ranges

# Example NSG rule:
Priority: 100
Name: Allow-OnPrem-DNS
Source: 192.168.0.0/16  # Your on-prem network
Source ports: *
Destination: VirtualNetwork
Destination ports: 53
Protocol: UDP
Action: Allow

# Check on-prem firewall:
# Allow outbound UDP 53 to 10.1.0.4, 10.1.0.5
```

---

#### Issue #5: Azure DNS VM Cannot Resolve Private DNS Zone

**Symptom:**
```bash
# From Azure DNS VM (10.1.0.4):
nslookup workspace-id.ods.opinsights.azure.com
# Returns public IP or NXDOMAIN (not found)
```

**What This Means:**
Your Azure DNS VMs (the ones receiving forwarded queries from on-premises) cannot query Azure's Private DNS Zones. This usually means the VMs aren't configured to use Azure-provided DNS (168.63.129.16) or the Private DNS Zones aren't linked to the VNet where these DNS VMs reside.

**Root Causes:**
1. **DNS VMs use custom DNS instead of Azure DNS**: VMs are configured to use themselves or other DNS servers
2. **Private DNS Zone not linked to DNS VM VNet**: Zone linked to hub but DNS VMs are in different VNet
3. **Recursive DNS loop**: DNS VM forwards to itself or another DNS VM in a loop
4. **VNet peering DNS settings**: Spoke VNet not using hub DNS settings

**Diagnostic Steps:**

**Step 1: Check DNS VM's DNS Configuration**
```bash
# On Windows DNS VM:
Get-DnsClientServerAddress -AddressFamily IPv4

# Expected output:
InterfaceAlias : Ethernet
ServerAddresses : {168.63.129.16}  # ← Azure DNS

# Or check in portal:
VM → Networking → Network Interface → DNS servers
Should show: "Inherit from virtual network" or "168.63.129.16"
```

**Step 2: Check VNet DNS Settings**
```bash
# Navigate to: VNet (where DNS VMs reside) → DNS servers

# Should be:
- Default (Azure-provided DNS)  ← Recommended for DNS VMs
# OR custom but NOT self-referencing:
- 168.63.129.16  # Explicitly set to Azure DNS
```

**Step 3: Check Private DNS Zone VNet Links**
```bash
# For each Private DNS Zone:
Private DNS zone → Virtual network links

# Must include ALL VNets where resources need to resolve:
- Hub VNet (where private endpoint is)
- DNS VM VNet (if different from hub)
- Spoke VNets (if VMs directly query workspace)
```

**Step 4: Test Resolution from DNS VM**
```bash
# SSH/RDP to DNS VM (10.1.0.4)

# Test if it can reach Azure DNS:
nslookup azure.microsoft.com 168.63.129.16
# Should resolve successfully

# Test if it can resolve private endpoint:
nslookup workspace-id.ods.opinsights.azure.com 168.63.129.16
# Should return private IP (10.1.100.50)

# Test default resolution:
nslookup workspace-id.ods.opinsights.azure.com
# Should also return private IP (using VM's configured DNS)
```

**Resolution:**

**Fix DNS VM Configuration (Windows):**
```powershell
# On the DNS VM:
# Set DNS server to use Azure DNS for external queries:

# Option 1: Configure forwarder in DNS server settings
Set-DnsServerForwarder -IPAddress 168.63.129.16 -PassThru

# Option 2: Inherit from VNet (recommended)
# In Azure Portal:
VM → Networking → Network Interface → DNS servers
Select: "Inherit from virtual network"

# Then in VNet:
VNet → DNS servers → Default (Azure-provided DNS)

# Restart DNS service:
Restart-Service DNS

# Restart VM network (or reboot):
Restart-Computer
```

**Fix DNS VM Configuration (Linux):**
```bash
# Edit resolv.conf (method 1 - temporary):
sudo nano /etc/resolv.conf
# Add:
nameserver 168.63.129.16

# For permanent configuration (method 2 - systemd-resolved):
sudo nano /etc/systemd/resolved.conf
# Add under [Resolve]:
DNS=168.63.129.16
FallbackDNS=168.63.129.16

sudo systemctl restart systemd-resolved

# For permanent configuration (method 3 - NetworkManager):
sudo nano /etc/NetworkManager/conf.d/azure-dns.conf
# Add:
[global-dns-domain-*]
servers=168.63.129.16

sudo systemctl restart NetworkManager
```

**Link Private DNS Zones to DNS VM VNet:**
```bash
# For each Private DNS Zone:
# Azure Portal:
Private DNS zone → Virtual network links → Add

Settings:
- Link name: link-to-dns-vm-vnet
- Virtual network: <vnet-where-dns-vms-reside>
- Enable auto registration: NO

# Via Azure CLI:
az network private-dns link vnet create \
  --resource-group <rg-name> \
  --zone-name privatelink.ods.opinsights.azure.com \
  --name link-to-dns-vnet \
  --virtual-network <dns-vm-vnet-id> \
  --registration-enabled false

# Repeat for all Private DNS zones
```

**Fix VNet DNS Settings:**
```bash
# Navigate to: VNet (where DNS VMs reside) → DNS servers

# Change to:
Default (Azure-provided DNS)

# Click Save

# Important: Restart all VMs in this VNet for settings to take effect
# Or restart network service:
az vm restart --ids $(az vm list -g <rg-name> --query "[].id" -o tsv)
```

---

#### Issue #6: NSG Blocking DNS or HTTPS Traffic

**Symptom:**
- DNS queries timeout
- `Test-NetConnection` to port 443 fails
- Agent logs show connection timeouts

**What This Means:**
Network Security Groups (NSGs) on your subnets are blocking the traffic required for DNS resolution or LAW connectivity. Even if DNS resolves correctly to the private IP, if HTTPS (443) traffic to the private endpoint is blocked, log ingestion and queries will fail.

**Required Traffic Flows:**

```
┌────────────────────────────────────────────────────────────┐
│ On-Premises Network (192.168.0.0/16)                       │
│   │                                                        │
│   │ UDP 53 (DNS queries)                                   │
│   └────────────────────────────────────────────────────────┐
│                                                            │
│         ┌──────────────────────────────────────────────────┘
│         │                                                  │
│         ▼                                                  │
│  ┌──────────────────────┐                                  │
│  │  Azure DNS VMs       │                                  │
│  │  10.1.0.4, 10.1.0.5  │                                  │
│  │  Subnet: 10.1.0.0/24 │                                  │
│  └──────────────────────┘                                  │
│         │                                                  │
│         │ UDP 53 (to Azure DNS 168.63.129.16)              │
│         │ TCP 443 (to Private Endpoint)                    │
│         ▼                                                  │
│  ┌────────────────────────┐                                │
│  │  Private Endpoint      │                                │
│  │  10.1.100.50           │                                │
│  │  Subnet: 10.1.100.0/24 │                                │
│  └────────────────────────┘                                │
│         │                                                  │
│         │ HTTPS (to Azure Monitor backend)                 │
│         ▼                                                  │
│    [Azure Monitor / LAW]                                   │
└────────────────────────────────────────────────────────────┘
```

**Required NSG Rules:**

**NSG on DNS VM Subnet (10.1.0.0/24):**

| Priority | Name | Source | Source Ports | Destination | Dest Ports | Protocol | Action |
|----------|------|--------|--------------|-------------|------------|----------|--------|
| 100 | Allow-OnPrem-DNS-Inbound | 192.168.0.0/16 | * | 10.1.0.0/24 | 53 | UDP | Allow |
| 110 | Allow-Azure-DNS-Outbound | 10.1.0.0/24 | * | 168.63.129.16/32 | 53 | UDP | Allow |
| 120 | Allow-PE-HTTPS-Outbound | 10.1.0.0/24 | * | 10.1.100.0/24 | 443 | TCP | Allow |

**NSG on Private Endpoint Subnet (10.1.100.0/24):**

| Priority | Name | Source | Source Ports | Destination | Dest Ports | Protocol | Action |
|----------|------|--------|--------------|-------------|------------|----------|--------|
| 100 | Allow-DNS-VMs-Inbound | 10.1.0.0/24 | * | 10.1.100.0/24 | 443 | TCP | Allow |
| 110 | Allow-VMs-Inbound | 10.0.0.0/8 | * | 10.1.100.0/24 | 443 | TCP | Allow |
| 120 | Allow-OnPrem-Inbound | 192.168.0.0/16 | * | 10.1.100.0/24 | 443 | TCP | Allow |

**NSG on VM Subnets (where agents run):**

| Priority | Name | Source | Source Ports | Destination | Dest Ports | Protocol | Action |
|----------|------|--------|--------------|-------------|------------|----------|--------|
| 100 | Allow-PE-HTTPS-Outbound | * | * | 10.1.100.0/24 | 443 | TCP | Allow |

**Diagnostic Steps:**

**Step 1: Test DNS Connectivity**
```bash
# From on-prem DNS server:
Test-NetConnection -ComputerName 10.1.0.4 -Port 53
Test-NetConnection -ComputerName 10.1.0.5 -Port 53

# If fails: NSG on DNS VM subnet is blocking inbound UDP 53
```

**Step 2: Test Private Endpoint Connectivity**
```bash
# From DNS VM:
Test-NetConnection -ComputerName 10.1.100.50 -Port 443

# From Azure VM with agent:
Test-NetConnection -ComputerName 10.1.100.50 -Port 443

# From on-prem server:
Test-NetConnection -ComputerName 10.1.100.50 -Port 443

# If any fail: NSG on private endpoint subnet is blocking inbound TCP 443
```

**Step 3: Check NSG Effective Rules**
```bash
# In Azure Portal:
VM → Networking → Network Interface → Effective security rules

# Or via Azure CLI:
az network nic list-effective-nsg \
  --resource-group <rg-name> \
  --name <nic-name>

# Look for deny rules that might block:
# - Source: Your on-prem IPs or Azure subnets
# - Destination: DNS VMs or Private Endpoint IPs
# - Ports: 53 (DNS) or 443 (HTTPS)
```

**Step 4: Check NSG Flow Logs (if enabled)**
```bash
# NSG Flow Logs show actual traffic being allowed/denied
# In Azure Portal:
NSG → NSG flow logs → View in Log Analytics

# Query for denied traffic:
AzureNetworkAnalytics_CL
| where FlowStatus_s == "D"  // D = Denied
| where DestPort_d in (53, 443)
| project TimeGenerated, SrcIP_s, DestIP_s, DestPort_d, NSGRule_s
```

**Resolution - Create/Update NSG Rules:**

**Via Azure Portal:**
```bash
# Navigate to: NSG → Inbound/Outbound security rules → Add

# Example: Allow on-prem DNS to Azure DNS VMs
Source: IP Addresses
Source IP: 192.168.0.0/16
Source port ranges: *
Destination: IP Addresses
Destination IP: 10.1.0.0/24
Service: Custom
Destination port ranges: 53
Protocol: UDP
Action: Allow
Priority: 100
Name: Allow-OnPrem-DNS
```

**Via Azure CLI:**
```bash
# Allow on-prem to DNS VMs (UDP 53):
az network nsg rule create \
  --resource-group <rg-name> \
  --nsg-name <dns-vm-nsg-name> \
  --name Allow-OnPrem-DNS \
  --priority 100 \
  --source-address-prefixes 192.168.0.0/16 \
  --source-port-ranges '*' \
  --destination-address-prefixes 10.1.0.0/24 \
  --destination-port-ranges 53 \
  --access Allow \
  --protocol Udp \
  --direction Inbound

# Allow DNS VMs to Private Endpoint (TCP 443):
az network nsg rule create \
  --resource-group <rg-name> \
  --nsg-name <dns-vm-nsg-name> \
  --name Allow-PE-HTTPS \
  --priority 110 \
  --source-address-prefixes 10.1.0.0/24 \
  --source-port-ranges '*' \
  --destination-address-prefixes 10.1.100.0/24 \
  --destination-port-ranges 443 \
  --access Allow \
  --protocol Tcp \
  --direction Outbound

# Allow VMs to Private Endpoint (TCP 443):
az network nsg rule create \
  --resource-group <rg-name> \
  --nsg-name <pe-subnet-nsg-name> \
  --name Allow-VMs-LAW \
  --priority 100 \
  --source-address-prefixes 10.0.0.0/8 \
  --source-port-ranges '*' \
  --destination-address-prefixes 10.1.100.0/24 \
  --destination-port-ranges 443 \
  --access Allow \
  --protocol Tcp \
  --direction Inbound
```

**Important Notes:**
- NSG rules are stateful - you only need to allow the initial connection direction
- Allow rules take precedence based on priority (lower number = higher priority)
- Default rules (65000+) allow VNet-to-VNet traffic, so explicit rules may not be needed for intra-VNet traffic
- Always test after adding rules: `Test-NetConnection` or `telnet`

---

### 7.3 Complete DNS Troubleshooting Workflow

```
┌─────────────────────────────────────────────────────────────┐
│ START: DNS Issue Reported                                   │
│ (Agent not connecting, queries failing, etc.)               │
└────────────────┬────────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Test DNS Resolution                                 │
│                                                             │
│ From problem system:                                        │
│   nslookup workspace-id.ods.opinsights.azure.com            │
│                                                             │
│ Expected: Private IP (10.x.x.x)                             │
│ Actual: ?                                                   │
└────────────────┬────────────────────────────────────────────┘
                 │
        ┌────────┼────────┐
        │                 │
  Returns Public IP   Returns Private IP
        │                 │
        ▼                 ▼
┌──────────────────┐  ┌───────────────────────────┐
│ DNS Config Issue │  │ Step 2: Test Connectivity │
└────────┬─────────┘  └────────┬──────────────────┘
         │                     │
         │                     ▼
         │            Test-NetConnection -Port 443
         │                     │
         │            ┌────────┼────────┐
         │            │                 │
         │       Succeeds            Fails
         │            │                 │
         │            ▼                 ▼
         │     ┌──────────────┐  ┌─────────────┐
         │     │ Check App    │  │ NSG/Firewall│
         │     │ /Agent Logs  │  │ Issue       │
         │     └──────────────┘  └──────┬──────┘
         │                              │
         ▼                              │
┌─────────────────────────────────────────┐
│ Step 3: Check Private DNS Zones         │
│ - Do zones exist?                       │
│ - Are they linked to VNet?              │
│ - Do A records exist?                   │
└────────┬────────────────────────────────┘
         │
   ┌─────┼─────┐
   │           │
Missing    Exists
   │           │
   ▼           ▼
[Fix:     ┌────────────────────────────┐
 Create   │ Step 4: Check VNet DNS     │
 Zones]   │ Settings                   │
          │ - Default or Custom?       │
          │ - Points to 168.63.129.16? │
          └────────┬───────────────────┘
                   │
            ┌──────┼──────┐
            │             │
         Wrong         Correct
            │             │
            ▼             ▼
        [Fix:      ┌───────────────────────┐
         Set to    │ Step 5: Check         │
         Default]  │ Conditional Forwarders│
                   │ (on-prem DNS)         │
                   └────────┬──────────────┘
                            │
                     ┌──────┼──────┐
                     │             │
                 Missing       Exists
                     │             │
                     ▼             ▼
                 [Fix:       ┌──────────────┐
                  Create     │ Step 6: Check│
                  Fwders]    │ AMPLS Mode   │
                             └────────┬─────┘
                                      │
                               ┌──────┼──────┐
                               │             │
                            Open      Private Only
                               │             │
                               ▼             ▼
                           [Fix:         ┌──────────┐
                            Set to       │ Step 7:  │
                            Private]     │ Check NSG│
                                         └────┬─────┘
                                              │
                                        ┌─────┼─────┐
                                        │           │
                                    Blocking    Allowing
                                        │           │
                                        ▼           ▼
                                    [Fix:     ┌─────────┐
                                     Add      │ SUCCESS │
                                     Rules]   └─────────┘
```

---

### 7.4 DNS Verification Commands Reference

**From On-Premises PC:**
```bash
# Basic DNS test:
nslookup workspace-id.ods.opinsights.azure.com

# Test through specific DNS server:
nslookup workspace-id.ods.opinsights.azure.com 192.168.1.10

# Detailed DNS query:
nslookup -debug workspace-id.ods.opinsights.azure.com

# Test connectivity:
Test-NetConnection -ComputerName workspace-id.ods.opinsights.azure.com -Port 443

# Trace route (to see network path):
tracert workspace-id.ods.opinsights.azure.com
# Should show private IPs if using AMPLS correctly
```

**From On-Premises DNS Server:**
```bash
# Test conditional forwarder:
nslookup workspace-id.ods.opinsights.azure.com 10.1.0.4

# List all conditional forwarders (Windows):
Get-DnsServerZone | Where-Object {$_.ZoneType -eq "Forwarder"} | Format-Table ZoneName, MasterServers

# Test connectivity to Azure DNS VMs:
Test-NetConnection -ComputerName 10.1.0.4 -Port 53
Test-NetConnection -ComputerName 10.1.0.5 -Port 53

# Clear DNS cache:
Clear-DnsServerCache
```

**From Azure DNS VM (10.1.0.4):**
```bash
# Test resolution using Azure DNS directly:
nslookup workspace-id.ods.opinsights.azure.com 168.63.129.16

# Test default resolution:
nslookup workspace-id.ods.opinsights.azure.com

# Check DNS server configuration:
Get-DnsClientServerAddress -AddressFamily IPv4

# Test connectivity to private endpoint:
Test-NetConnection -ComputerName 10.1.100.50 -Port 443

# Test connectivity to Azure DNS:
Test-NetConnection -ComputerName 168.63.129.16 -Port 53
```

**From Azure VM with Agent:**
```bash
# Test DNS resolution:
nslookup workspace-id.ods.opinsights.azure.com

# Test connectivity to workspace:
Test-NetConnection -ComputerName workspace-id.ods.opinsights.azure.com -Port 443

# Check agent status (Windows):
Get-Service -Name HealthService
Get-Content "C:\Program Files\Microsoft Monitoring Agent\Agent\Logs\OpsMgrTrace.log"

# Check agent status (Linux):
sudo systemctl status omsagent
sudo tail -f /var/opt/microsoft/omsagent/log/omsagent.log

# Test agent connectivity:
# Windows:
Test-SCOMConnection -WorkspaceId workspace-id

# Linux:
sudo /opt/microsoft/omsagent/bin/omsadmin.sh -l
```

**From Azure Portal / Cloud Shell:**
```bash
# Check Private DNS Zone records:
az network private-dns record-set a list \
  --resource-group <rg-name> \
  --zone-name privatelink.ods.opinsights.azure.com

# Check VNet links:
az network private-dns link vnet list \
  --resource-group <rg-name> \
  --zone-name privatelink.ods.opinsights.azure.com

# Check AMPLS configuration:
az monitor private-link-scope show \
  --name <ampls-name> \
  --resource-group <rg-name>

# Check private endpoint:
az network private-endpoint show \
  --name <pe-name> \
  --resource-group <rg-name> \
  --query "{name:name, privateLinkServiceConnections:privateLinkServiceConnections, networkInterfaces:networkInterfaces}"

# Check NSG effective rules:
az network nic list-effective-nsg \
  --resource-group <rg-name> \
  --name <nic-name>
```

---

### 7.5 DNS Configuration Checklist

Use this checklist to verify all DNS components are configured correctly:

#### Private DNS Zones
- [ ] `privatelink.monitor.azure.com` zone exists
- [ ] `privatelink.oms.opinsights.azure.com` zone exists
- [ ] `privatelink.ods.opinsights.azure.com` zone exists
- [ ] `privatelink.agentsvc.azure-automation.net` zone exists
- [ ] `privatelink.blob.core.windows.net` zone exists (if using diagnostics)
- [ ] All zones linked to Hub VNet
- [ ] All zones linked to Spoke VNets (if applicable)
- [ ] All zones linked to DNS VM VNet (if separate)
- [ ] Auto-registration disabled on all AMPLS-related zones
- [ ] A records exist for workspace endpoints in each zone
- [ ] A record IPs match private endpoint IPs

#### Azure DNS VMs Configuration
- [ ] DNS VMs deployed in Hub VNet (or connected VNet)
- [ ] DNS VMs use Azure-provided DNS (168.63.129.16)
- [ ] VNet DNS setting is "Default (Azure-provided DNS)"
- [ ] DNS VMs can resolve private endpoint (test with nslookup)
- [ ] DNS VMs can reach Azure DNS (test 168.63.129.16 port 53)
- [ ] DNS service running on VMs (if using custom DNS software)
- [ ] Forwarder configured on DNS VMs to 168.63.129.16

#### On-Premises DNS Configuration
- [ ] Conditional forwarder for `ods.opinsights.azure.com` exists
- [ ] Conditional forwarder for `oms.opinsights.azure.com` exists
- [ ] Conditional forwarder for `monitor.azure.com` exists
- [ ] Conditional forwarder for `agentsvc.azure-automation.net` exists
- [ ] Forwarders point to Azure DNS VMs (10.1.0.4, 10.1.0.5)
- [ ] NOT pointing to private endpoint IPs
- [ ] Forwarder timeout set appropriately (5-10 seconds)
- [ ] DNS cache cleared after configuration changes

#### AMPLS Configuration
- [ ] AMPLS resource created
- [ ] Private endpoint created and associated with AMPLS
- [ ] LAW resource added to AMPLS scope
- [ ] AMPLS Network Isolation set to "Private Only"
- [ ] "Accept access from public networks" = NO
- [ ] Private endpoint approved (connection state = Approved)
- [ ] Private endpoint has IP assigned

#### Network Connectivity
- [ ] ExpressRoute or VPN Gateway connection status = Connected
- [ ] NSG allows UDP 53 from on-prem to Azure DNS VMs
- [ ] NSG allows TCP 443 from Azure DNS VMs to private endpoint
- [ ] NSG allows TCP 443 from VM subnets to private endpoint
- [ ] NSG allows TCP 443 from on-prem to private endpoint (if direct)
- [ ] Azure Firewall (if present) allows required traffic
- [ ] On-prem firewall allows outbound DNS to Azure
- [ ] On-prem firewall allows outbound HTTPS to Azure (via VPN/ER)

#### Testing & Validation
- [ ] nslookup from on-prem returns private IP
- [ ] nslookup from Azure VM returns private IP
- [ ] Test-NetConnection port 443 succeeds from on-prem
- [ ] Test-NetConnection port 443 succeeds from Azure VM
- [ ] Azure Monitor Agent shows "Connected" status
- [ ] Log queries execute successfully
- [ ] Log ingestion working (check Heartbeat table)
- [ ] DNS cache flushed on all systems
- [ ] Agent logs show no connection errors

---

### 7.6 Common Error Messages and Solutions

| Error Message | Location | Cause | Solution |
|---------------|----------|-------|----------|
| `Failed to resolve table or column expression named 'source'` | Log Analytics query | Running DCR transformation query in LAW | Use actual table names; 'source' only works in DCR definitions |
| `Connection timeout to workspace-id.ods.opinsights.azure.com` | Agent logs | DNS resolving to public IP or NSG blocking | Check DNS resolution; verify NSG rules allow 443 |
| `Certificate validation failed` | Agent logs | Time sync issue or firewall HTTPS inspection | Sync time; disable HTTPS inspection for Azure Monitor |
| `NXDOMAIN` | nslookup | DNS zone or A record missing | Create Private DNS Zone and verify A records exist |
| `Server failed` | nslookup | DNS server cannot reach zone or forwarder | Check conditional forwarders; verify VNet links |
| `Query timed out` | nslookup | Network connectivity issue | Check NSG, firewall, ExpressRoute/VPN status |
| `Access forbidden (403)` | Log queries | AMPLS scope or RBAC issue | Add workspace to AMPLS; verify user has access |
| `Public network access disabled` | Queries from internet | AMPLS set to private only | Expected behavior; use private connection |
| `The requested resource does not support http method 'GET'` | Agent connection | Agent trying public endpoint when private required | Check DNS resolution; may be cached |

---

### 7.7 Advanced Troubleshooting Tools

**Enable NSG Flow Logs:**
```bash
# Create Storage Account for flow logs:
az storage account create \
  --name nsgflowlogs<unique> \
  --resource-group <rg-name> \
  --location <location>

# Enable flow logs on NSG:
az network watcher flow-log create \
  --name flowlog-dns-nsg \
  --nsg <dns-vm-nsg-id> \
  --storage-account nsgflowlogs<unique> \
  --enabled true \
  --retention 30

# View flow logs in Log Analytics:
AzureNetworkAnalytics_CL
| where SubType_s == "FlowLog"
| where DestIP_s in ("10.1.0.4", "10.1.0.5", "10.1.100.50")
| project TimeGenerated, SrcIP_s, DestIP_s, DestPort_d, FlowStatus_s
```

**Use Azure Network Watcher:**
```bash
# Test connectivity from VM to private endpoint:
az network watcher test-connectivity \
  --source-resource <vm-id> \
  --dest-address 10.1.100.50 \
  --dest-port 443

# View effective routes:
az network nic show-effective-route-table \
  --resource-group <rg-name> \
  --name <nic-name>

# Next hop:
az network watcher show-next-hop \
  --resource-group <rg-name> \
  --vm <vm-name> \
  --dest-ip 10.1.100.50 \
  --source-ip 10.1.0.4
```

**Packet Capture:**
```bash
# Start packet capture on DNS VM:
# Windows:
netsh trace start capture=yes tracefile=C:\dns-trace.etl

# Linux:
sudo tcpdump -i eth0 -w /tmp/dns-trace.pcap 'port 53 or port 443'

# Reproduce issue, then stop:
# Windows:
netsh trace stop

# Analyze with Wireshark or Network Monitor
```

**DNS Debug Logging (Windows DNS Server):**
```powershell
# Enable DNS debug logging:
Set-DnsServerDiagnostics -All $true

# Log location:
# C:\Windows\System32\dns\dns.log

# Disable after troubleshooting (generates large logs):
Set-DnsServerDiagnostics -All $false
```

---

### 7.8 Emergency Rollback Procedure

If AMPLS DNS configuration causes widespread connectivity issues:

**Immediate Mitigation:**
```bash
# Step 1: Set AMPLS to "Open" mode (allows public access temporarily)
AMPLS → Network Isolation → "Accept access from public networks" = YES
Click Save

# Step 2: Remove conditional forwarders from on-prem DNS (temporarily)
Remove-DnsServerZone -Name "ods.opinsights.azure.com" -Force
Remove-DnsServerZone -Name "oms.opinsights.azure.com" -Force

# Step 3: Clear DNS cache everywhere
# On-prem DNS: Clear-DnsServerCache
# Clients: ipconfig /flushdns
# Azure VMs: ipconfig /flushdns or systemd-resolve --flush-caches

# This restores public endpoint connectivity while you fix the issue
```

**Root Cause Analysis:**
```bash
# Review what was changed before the issue
# Check Azure Activity Log:
az monitor activity-log list \
  --resource-group <rg-name> \
  --start-time 2026-02-26T00:00:00Z \
  --query "[?contains(operationName.value, 'DNS') || contains(operationName.value, 'PrivateEndpoint')]"

# Document findings and fix before re-enabling private-only mode
```

---

