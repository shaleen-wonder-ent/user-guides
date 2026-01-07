
## Architecture

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Architecture" 
     src="https://github.com/user-attachments/assets/500c6fc8-7574-41f9-bdbb-3151a66e63db"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


# Route-table based solution (vWAN vHub Route Tables)

## Goal
- **Block subscription-to-subscription (spoke-to-spoke) connectivity**:
  - within the same hub (Sub1↔Sub2, Sub3↔Sub4)
  - across hubs (Sub1/2↔Sub3/4)
- **Preserve connectivity to DC/on-prem** (in this lab, represented by **Laptop via P2S**).

## Important note about P2S (User VPN)
P2S is primarily for **client → Azure** access. It is not a perfect substitute for a routed on‑prem/DC network (S2S/ER).  
This route-table model focuses on preventing spoke↔spoke routing in the vHub while allowing the user VPN client to access spokes as required.

## Concepts (simple)
- **Association**: which route table a connection *reads* to forward traffic.
- **Propagation**: which route tables a connection *writes* its prefixes into.

## High-level design
Per hub, create:
- A per-spoke route table for each subscription-spoke:
  - Hub1: `rt-sub1`, `rt-sub2`
  - Hub2: `rt-sub3`, `rt-sub4`
- A “user-vpn” table:
  - Hub1: `rt-p2s`
  - Hub2: `rt-p2s`

### Why per-spoke tables?
If Sub1 and Sub2 share the same route table, they will learn each other’s routes and be able to communicate.  
Per-spoke tables ensure each spoke only learns routes you explicitly allow.

## Implementation steps

### 1) Create route tables (per hub)
In **Hub1 (East US)** create:
- `rt-sub1`
- `rt-sub2`
- `rt-p2s`

In **Hub2 (Central India)** create:
- `rt-sub3`
- `rt-sub4`
- `rt-p2s`

### 2) Configure VNet connections (spokes)
For each spoke connection, set:
- **Associate Route Table** = its per-spoke table
- **Propagate to none** = `Yes` (spoke does not advertise itself to other tables by default)
- **Propagate Default Route** = `Disable`

Example:
- `conn-sub1`: Associate `rt-sub1`, Propagate to none = Yes
- `conn-sub2`: Associate `rt-sub2`, Propagate to none = Yes
- `conn-sub3`: Associate `rt-sub3`, Propagate to none = Yes
- `conn-sub4`: Associate `rt-sub4`, Propagate to none = Yes

This ensures:
- Spokes do not publish their prefixes into any shared table.
- Spokes do not automatically learn other spokes’ routes.

### 3) Configure P2S user VPN (per hub)
Enable **P2S User VPN** on Hub1 and Hub2.

Then ensure the P2S connection (User VPN) is configured such that:
- It can **learn routes to all spokes you want the laptop to reach**.
- Spokes do not learn routes to each other.

Practical approach:
- Make P2S connection associated to `rt-p2s`.
- In `rt-p2s`, add static routes (or ensure appropriate propagation, depending on portal capabilities for user VPN routes) for:
  - Sub1 VNet prefix
  - Sub2 VNet prefix
  - Sub3 VNet prefix
  - Sub4 VNet prefix

> Note: The exact configuration options for P2S route propagation/association differ based on vWAN/user VPN configuration in the portal and client settings. Some setups require configuring address pools and advertised routes explicitly in the User VPN configuration.

### 4) (Optional) Allow “exception” spoke-to-spoke connectivity
If HCL wants to allow Sub1 ↔ Sub2 in the future:
- Create a dedicated table `rt-exception-sub1-sub2`
- Associate both conn-sub1 and conn-sub2 to that table, and allow propagation accordingly
- Keep other spokes separate

This keeps exceptions explicit and isolated.

## Validation
### Expected outcomes
- Sub1 ↔ Sub2: **blocked**
- Sub3 ↔ Sub4: **blocked**
- Sub1/2 ↔ Sub3/4: **blocked**
- Laptop (P2S) → each spoke VM: **allowed** (if P2S routes are configured)

### How to test
From a VM in Sub1:
- Attempt to reach VM private IP in Sub2/Sub3/Sub4 → should fail (no route / unreachable).

From Laptop (User VPN connected):
- Attempt to reach VM private IP in Sub1/Sub2/Sub3/Sub4 → should succeed (assuming NSGs/OS firewall allow).

## Risks / limitations
- This is **routing-based isolation**, not a security control. A misconfiguration (route table association/propagation/labels/default tables) can unintentionally restore reachability.
- Operational overhead increases with number of subscriptions/spokes (more tables to manage).
- P2S is not a perfect representation of on‑prem/DC connectivity; for a true “DC reachable from spokes and vice versa”, use **S2S VPN or ExpressRoute** in the lab.
