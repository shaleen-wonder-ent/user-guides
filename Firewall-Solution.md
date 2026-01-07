## Architecture

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Architecture" 
     src="https://github.com/user-attachments/assets/916d3e8c-4684-48f2-b33b-e53588c1c47b"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


# Firewall-based solution (Azure Firewall in vWAN hubs)

## Goal
- **Block subscription-to-subscription (spoke-to-spoke) traffic by default**
  - same hub and cross-hub
- **Preserve connectivity to DC/on-prem**
  - In this lab, laptop uses **P2S** primarily for client→Azure access and testing.

## Important note about P2S (User VPN)
P2S is typically **client-initiated**. It is good for laptop→Azure reachability testing, but it is not a perfect proxy for “on-prem DC routing” (S2S/ER).  
The firewall design still applies the same way when the “DC” is an S2S/ER connection.

## High-level design
Per hub:
- Deploy **Azure Firewall** (or approved NVA) and manage using **Azure Firewall Manager**.
- Ensure spoke traffic that could be inter-spoke is **steered through the firewall**.
- Enforce security rules:
  - Deny inter-spoke by default
  - Allow spoke↔DC/on-prem as required
  - Allow explicit exceptions (subnet/app/port/protocol)

## Key principle
A firewall only helps if the traffic **traverses it**.  
So you need *routing/steering configuration* to force inter-spoke flows (and possibly on-prem flows) through the firewall.

In vWAN this can be achieved using:
- **Routing Intent and Routing Policies** (preferred if available in the tenant/region), and/or
- vHub route tables + UDRs where applicable.

## Implementation steps (conceptual)

### 1) Deploy Azure Firewall in each hub
- Hub1 (East US): Azure Firewall (and optionally a Firewall Policy)
- Hub2 (Central India): Azure Firewall

Manage policies centrally with Firewall Manager:
- Global base policy (deny spoke-to-spoke)
- Child policies per hub/region for local exceptions

### 2) Configure routing/steering so spoke traffic traverses firewall
Depending on what is available in the environment:

#### Preferred: Routing Intent (vWAN)
- Configure routing intent to send private traffic categories through Azure Firewall.
- Validate effective routes so that inter-spoke flows are sent to firewall as next hop.

#### Alternative: Route-table steering
- Use vHub route tables to ensure spokes learn routes via firewall/NVA next hop (pattern varies by deployment).

### 3) Define firewall policy (baseline)
Create rules (examples; adjust for your environment):

#### Deny all inter-spoke (default)
- Source: all spoke address spaces (or spoke IP groups)
- Destination: all spoke address spaces
- Protocol/Ports: Any
- Action: Deny

#### Allow spoke → on-prem/DC
- Source: spoke IP groups
- Destination: on-prem/DC prefixes (or IP group)
- Ports: required ports (DNS/AD/HTTP/HTTPS/etc.)
- Action: Allow

#### Allow exceptions on request (explicit)
- Source: Sub1 subnet(s)
- Destination: Sub2 subnet(s)
- Ports: only what is required (e.g., 443)
- Action: Allow
- Documented with change control

### 4) Logging and monitoring
Enable:
- Firewall logs to Log Analytics / Sentinel
- Alerts on denied inter-spoke attempts (optional)

## Validation
### Expected outcomes
- Sub1 ↔ Sub2: blocked by firewall (even if routing accidentally becomes permissive)
- Sub3 ↔ Sub4: blocked by firewall
- Sub1/2 ↔ Sub3/4: blocked by firewall (depending on cross-hub routing)
- Spokes ↔ DC/on-prem: allowed as per policy
- Laptop (P2S) → spokes: allowed as needed for admin/testing (subject to NSGs/OS firewall)

### How to test
- From each spoke VM, attempt to reach another spoke VM private IP on a test port (e.g., 3389/443).
  - Should be denied (and visible in firewall logs).
- Validate DC/on-prem connectivity.
- Validate explicit exception once created.

## Why this approach is generally preferred at scale
- **Hard enforcement** (routing mistakes don’t automatically open connectivity).
- **Auditable** (single place to see allowed/blocked flows).
- **Granular exceptions** (port/protocol/app-level controls).
- Better alignment with enterprise security/compliance requirements.

## Considerations
- Additional cost (Firewall + data processing).
- Requires correct routing/steering setup to ensure traffic traverses the firewall.
- Still recommended to keep route distribution tight (defense-in-depth), but firewall is the primary control.
