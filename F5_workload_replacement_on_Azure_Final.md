# Contoso Load Balancer Refresh – Azure Transition Proposal (2‑Phase Approach)

**Purpose:** Document Contoso’s current state and propose a phased approach to transition internet-facing application delivery from F5 to Azure, followed by a modernization program for internal (LAN) access.

---

## What Contoso Currently Has (Current State Summary)

Based on discovery discussions, Contoso operates a **hybrid application delivery** model with multiple F5 form factors and **two distinct access paths** (LAN and Internet).

### 1) Mixed F5 footprint
- **F5 virtual appliances on‑prem**
- **F5 virtual appliances in Azure**
- **F5 hardware appliances on‑prem**

This indicates both legacy and newer deployments co-exist, likely with different lifecycles, ownership, and operational processes.

### 2) Two access planes / two entry paths
Contoso effectively maintains **two separate access planes**:
- **LAN-facing (Internal) F5** for users inside the corporate network.
- **Internet-facing (External) publishing path** for users outside the corporate network (WFH/public internet).

### 3) Dedicated DNS services for LAN vs Internet users (instead of split-horizon DNS)
Contoso uses **dedicated DNS deployments** for name resolution based on user location/source:
- **LAN users** resolve application FQDNs using **internal (LAN) DNS**, which returns **private IP/VIPs** for internal access.
- **Internet/WFH users** resolve application FQDNs using **public (Internet) DNS**, which returns **public endpoints/VIPs** for external access.

This enables the same application to be accessed either via:
- **private network routing** (LAN) or
- **public network + perimeter controls** (Internet).

### 4) Perimeter security for external users is anchored on Palo Alto
For external access, inbound traffic flows through **Palo Alto Networks** devices as part of the perimeter enforcement chain before reaching application infrastructure.

### 5) Segmentation firewall exists between F5 and web/app servers (for both LAN and Internet paths)
Contoso has a **segmentation firewall between F5/publishing tiers and the web/app server tier** for **both**:
- **LAN user traffic**, and
- **Internet user traffic**.

This firewall enforces segmentation policy before requests reach the backend servers.

### 6) Some on‑prem applications are behind on‑prem WAF policies
A subset of applications (and tools) are still **hosted on‑prem** and protected by **on‑prem WAF policies** (currently on F5 in at least one path).

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                           CURRENT CONTOSO ARCHITECTURE (UPDATED)                         │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────────────┐                                  ┌────────────────────┐│
│  │ DEDICATED LAN DNS (INTERNAL) │                                  │ DEDICATED INTERNET ││
│  │ • Internal resolvers         │                                  │ DNS (PUBLIC)       ││
│  │ • app.contoso.internal       │                                  │ • app.contoso.com  ││
│  │   → Private VIP (10.x.x.x)   │                                  │   → Public VIP/IP  ││
│  └──────────────┬───────────────┘                                  └──────────┬─────────┘│
│                 │                                                             │          │
│                 ▼                                                             ▼          │
│  ┌──────────────────────────────────┐                         ┌─────────────────────────┐│
│  │ INTERNAL PATH (LAN)              │                         │ EXTERNAL PATH (INTERNET)││
│  │                                  │                         │                         ││
│  │ Internal Users (Office Network)  │                         │ External Users (WFH /   ││
│  │              │                   │                         │ Public Internet)        ││
│  │              ▼                   │                         │            │            ││
│  │ LAN‑facing F5                    │                         │            ▼            ││
│  │ • F5 Virtual (On‑Prem)           │                         │ Palo Alto Networks      ││
│  │ • F5 Hardware (On‑Prem)          │                         │ (Perimeter enforcement) ││
│  │ • LTM + (APM/WAF as applicable)  │                         │            │            ││
│  │              │                   │                         │            ▼            ││
│  │              ▼                   │                         │ Internet‑facing F5      ││
│  │ Segmentation Firewall            │                         │ • F5 Virtual (On‑Prem)  ││
│  │ (between F5 and servers)         │                         │ • F5 Virtual (Azure)    ││
│  │              │                   │                         │ • LTM+(APM/WAF as appl) ││
│  │              ▼                   │                         │            │            ││
│  │ On‑Prem Web/App Servers          │                         │            ▼            ││
│  │ (some apps behind on‑prem WAF)   │                         │ Segmentation Firewall   ││
│  │                                  │                         │ (between F5 and servers)││
│  │                                  │                         │            │            ││
│  │                                  │                         │            ▼            ││
│  │                                  │                         │ On‑Prem Web/App Servers ││
│  │                                  │                         │ (some apps behind WAF)  ││
│  └──────────────────────────────────┘                         └─────────────────────────┘│
│                                                                                          │
│ Notes:                                                                                   │
│ • DNS is implemented as dedicated LAN DNS and dedicated Internet/Public DNS.             │
│ • Segmentation firewall exists between the publishing tier and server tier on both paths │
│ • Exact device ordering in the external path can vary by application; validate per-app.  │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```


---

## Recommended Approach Overview (Internet Migration First, LAN Modernization Later)

This approach is designed to:
- deliver **quick, low-risk wins** first (Internet path),
- reduce the externally exposed attack surface rapidly, and
- then address internal/LAN access as a **modernization** initiative rather than a like-for-like “lift-and-shift” migration.

---

# Phase 1 – Internet Facing (External Users)

> **Primary objective:** Replace the **internet-facing F5** as the external entry point by introducing **Azure Front Door Premium + WAF**.  
> **Important:** Phase 1 is intentionally focused only on **external users**. **Internal (LAN) users are not changed in Phase 1.**

Because there are some unknowns today (e.g., whether internet-facing F5 is providing app-critical APM/iRules/persistence and whether Palo Alto can publish apps directly end-to-end), Phase 1 is recommended as **two sub-phases**.

---

## Phase 1a – Introduce Azure Front Door as the Internet Entry Point (Low-risk Cutover)

### What we will do
- Publish external application endpoints via **Azure Front Door Premium** (global entry point).
- Enable **Front Door WAF** policies for:
  - OWASP protections
  - rate limiting (as required)
  - geo filtering (as required)
  - bot protections (as required)
- Update **public DNS** to resolve application FQDNs to **Azure Front Door**.
- Configure **the existing Internet-facing F5 as a temporary origin** behind Front Door.

### Why Phase 1a (most important “WHY”)
Phase 1a delivers the fastest measurable value with minimal disruption:
1. **Security uplift at the edge:** WAF enforcement happens before traffic reaches HCL perimeter and on‑prem infrastructure.
2. **Minimal change behind the scenes:** The origin remains the current internet F5, so application behavior is less likely to change unexpectedly.
3. **Fast rollback:** If needed, rollback is straightforward (DNS/origin configuration).
4. **Enables controlled validation:** Allows testing of WAF tuning, TLS settings, routing rules, and logging/monitoring before making deeper network/path changes.

**Key point:** Phase 1a changes only the **public entry point** (public DNS → Azure Front Door). Everything behind the selected origin continues to operate as it does today.

**Note:** The “temporary origin” can be the existing internet-facing F5 (and its current downstream chain, including Palo Alto) to preserve the current publishing behavior. The exact device order will follow the current HCL ingress design and will be validated during discovery.

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                         PHASE 1a (INTERNET PATH) – LOW-RISK CUTOVER                      │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  External Users (WFH / Internet)                                                         │
│          │                                                                               │
│          ▼                                                                               │
│  Public DNS (e.g., app.Contoso.com)                                                      │
│          │  resolves to                                                                  │
│          ▼                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────────────────┐   │
│  │ Azure Front Door Premium (NEW Internet Entry Point)                               │   │
│  │  • Global L7 Load Balancing                                                       │   │
│  │  • TLS/SSL Termination at the edge                                                │   │
│  │  • WAF Policy (OWASP + Custom Rules as required)                                  │   │
│  │  • Bot / Geo-IP / Rate limiting (as required)                                     │   │
│  │  • Health probes                                                                  │   │
│  └───────────────────────────────────────────────────────────────────────────────────┘   │
│          │                                                                               │
│          │ forwards approved requests (HTTPS)                                            │
│          ▼                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────────────────┐   │
│  │ Internet-facing F5 (TEMPORARY ORIGIN – Phase 1a)                                  │   │
│  │  • Keeps current publishing behavior while Front Door is introduced               │   │
│  │  • Preserves existing iRules / persistence / APM (if in use) during validation    │   │
│  │  • Reduces change risk for applications                                           │   │
│  └───────────────────────────────────────────────────────────────────────────────────┘   │
│          │                                                                               │
│          ▼                                                                               │
│  ┌───────────────────────────────────────────────────────────────────────────────────┐   │
│  │ Palo Alto Networks (Existing Perimeter)                                           │   │
│  │  • Firewall policy enforcement                                                    │   │
│  │  • NAT / routing (as currently implemented)                                       │   │
│  └───────────────────────────────────────────────────────────────────────────────────┘   │
│          │                                                                               │
│          ▼                                                                               │
│  On‑Prem Applications                                                                    │
│                                                                                          │
│  Phase 1a Goal: Azure Front Door becomes the public entry point quickly, while keeping   │
│  the rest of the existing external publishing chain stable during validation.            │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                            NETWORK FLOW (USER VIEW) – INTERNET USERS                        │
│                    This shows how a user request flows in Phase 1a vs Phase 1b              │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│  1) USER ACTION                                                                             │
│     User (WFH / Internet) opens: https://app.Contoso.com                                    │
│                                                                                             │
│  2) DNS RESOLUTION                                                                          │
│     Public DNS resolves app.Contoso.com → Azure Front Door (anycast IP / edge endpoint)     │
│                                                                                             │
│  3) TLS CONNECTION (EDGE)                                                                   │
│     Browser establishes TLS session with Azure Front Door (certificate presented at edge)   │
│                                                                                             │
│  4) SECURITY + ROUTING AT EDGE                                                              │
│     Azure Front Door applies WAF + routing rules and selects a healthy origin based on      │
│     health probes:                                                                          │
│       • WAF (OWASP protections + custom rules)                                              │
│       • Bot / Geo-IP / Rate limiting (as configured)                                        │
│     If allowed, the request is forwarded to the configured origin.                          │
│                                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ PHASE 1a (TRANSITION / LOW-RISK)                                                      │  │
│  │                                                                                       │  │
│  │   User → Public DNS → Azure Front Door (WAF) → Internet-facing F5 (TEMP ORIGIN)       │  │
│  │        → Palo Alto (perimeter) → On‑prem App → Response returns same path             │  │
│  │                                                                                       │  │
│  │   WHY: Keeps existing app publishing behavior intact while validating Front Door.     │  │
│  └───────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                             │
│  ┌───────────────────────────────────────────────────────────────────────────────────────┐  │
│  │ PHASE 1b (STEADY STATE / TARGET END STATE)                                            │  │
│  │                                                                                       │  │
│  │   User → Public DNS → Azure Front Door (WAF) → Palo Alto                              │  │
│  │        (Origin target / Perimeter enforcement point) → On‑prem App → Response returns │  │
│  │        same path                                                                      │  │
│  │                                                                                       │  │
│  │   WHY: Retires Internet-facing F5 once dependencies are confirmed and routing is ready│  │
│  └───────────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                             │
│  5) RESPONSE PATH (BOTH PHASES)                                                             │
│     On‑prem App → (Palo Alto) → (F5 only in Phase 1a) → Azure Front Door → User             │
│                                                                                             │
│  NOTES                                                                                      │
│  • Internal/LAN users are NOT changed in Phase 1 (handled separately in Phase 2).           │
│  • In Phase 1b, Internet-facing F5 is bypassed/retired for external access.                 │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1b – Fully Replace Internet-facing F5 (Complete External Path Transition)

### What we will do
After successful validation of Phase 1a (and confirmation of app dependencies), we will transition the origin away from the internet-facing F5:

- Preferred target origin: **Palo Alto** (existing perimeter), if Palo Alto can publish the apps end-to-end and F5-only dependencies are removed/mitigated.
- Alternative target origin (if required): another agreed design where Front Door routes to the correct backend endpoints without needing internet-facing F5.

Then:
- **Decommission / retire the internet-facing F5** (once cutover criteria and rollback plans are satisfied).

### Why Phase 1b (most important “WHY”)
1. **Meets the end goal:** Eliminates dependence on internet-facing F5 for external access.
2. **Reduces cost and operational overhead:** Removes external F5 management, licensing, and maintenance burden.
3. **Reduces attack surface:** Simplifies the internet ingress chain and centralizes external policy controls in Azure + perimeter security controls.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│               OPTION 1: HYBRID APPROACH (RECOMMENDED) – PHASE 1b                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│                              ┌──────────────────┐                               │
│                              │  SPLIT-HORIZON   │                               │
│                              │      DNS         │                               │
│                              └────────┬─────────┘                               │
│                     ┌─────────────────┼─────────────────┐                       │
│                     │                 │                 │                       │
│                     ▼                 │                 ▼                       │
│  ┌──────────────────────────┐         │  ┌──────────────────────────────────┐   │
│  │   INTERNAL PATH (LAN)    │         │  │   EXTERNAL PATH (INTERNET)       │   │
│  │                          │         │  │                                  │   │
│  │  ┌────────────────────┐  │         │  │  ┌────────────────────────────┐  │   │
│  │  │  Internal Users    │  │         │  │  │    External Users          │  │   │
│  │  │  (Office Network)  │  │         │  │  │    (WFH / Internet)        │  │   │
│  │  └─────────┬──────────┘  │         │  │  └─────────────┬──────────────┘  │   │
│  │            │             │         │  │                │                 │   │
│  │            ▼             │         │  │                ▼                 │   │
│  │  ┌────────────────────┐  │         │  │  ┌────────────────────────────┐  │   │
│  │  │   ON-PREM F5       │  │         │  │  │   AZURE FRONT DOOR PREMIUM │  │   │
│  │  │   (LAN-FACING)     │  │         │  │  │   + WAF (OWASP/Bot/Geo/IP) │  │   │
│  │  │   • LTM / WAF /    │  │         │  │  │   (Internet entry point)   │  │   │
│  │  │     APM (as-is)    │  │         │  │  └─────────────┬──────────────┘  │   │
│  │  │   KEEP AS-IS       │  │         │  │                │                 │   │
│  │  └─────────┬──────────┘  │         │  │                ▼                 │   │
│  │            │             │         │  │  ┌────────────────────────────┐  │   │
│  │            ▼             │         │  │  │   PALO ALTO NETWORKS       │  │   │
│  │  ┌────────────────────┐  │         │  │  │   (Perimeter / Origin tgt) │  │   │
│  │  │  ON-PREM APPS      │  │         │  │  └─────────────┬──────────────┘  │   │
│  │  └────────────────────┘  │         │  │                │                 │   │
│  └──────────────────────────┘         │  │                ▼                 │   │
│                                       │  │  ┌────────────────────────────┐  │   │
│                                       │  │  │      ON-PREM APPS          │  │   │
│                                       │  │  │ (Internet F5 bypassed)     │  │   │
│                                       │  │  └────────────────────────────┘  │   │
│                                       │  └──────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  BENEFITS:                                               CONSIDERATIONS:        │
│   • Lower risk: LAN unchanged                             • Two mgmt planes     │
│   • Immediate security uplift for internet traffic         • LAN F5 remains     │
│   • No ExpressRoute required for Phase 1                   • Licensing continues│
│   • Internet-facing F5 retired for external access         • Phase 2 =          │
│                                                                   modernization │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Routing Clarity (Dependency on On‑Prem Appliances / Hardware / On‑Prem WAF)

### Question: How will routing work when there is dependency on on‑prem appliances/hardware, with on‑prem applications behind on‑prem WAF?
**Answer:** We will maintain the existing **split-horizon DNS** model and implement a **hybrid multi-path routing approach**:

- **Internal (LAN) users:** continue to reach applications through the **existing LAN-facing F5** (no change in Phase 1).  
- **External (Internet/WFH) users:** are transitioned to **Azure Front Door Premium + WAF** as the public entry point (Phase 1a), and then moved to the Phase 1b steady state once validation is complete.

This approach preserves existing internal routing and on‑prem dependencies (including on‑prem WAF policies), while enabling a controlled and reversible migration for the Internet-facing path.

#### Diagram: Scenario A – Internal User (LAN) Accessing On‑Prem Application (No Change in Phase 1)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SCENARIO A: INTERNAL USER (OFFICE/LAN) → ON‑PREM APPLICATION                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Internal User (Office/LAN)                                                  │
│           │                                                                  │
│           ▼                                                                  │
│  Internal DNS (Private / Split-horizon)                                      │
│           │  resolves to Private VIP                                         │
│           ▼                                                                  │
│  LAN-facing F5 (Existing – KEEP AS‑IS in Phase 1)                            │
│   • LTM / WAF / APM (as currently implemented)                               │
│           │                                                                  │
│           ▼                                                                  │
│  On‑Prem Applications (including apps behind on‑prem WAF policies)           │
│                                                                              │
│  Note: No LAN routing changes are required in Phase 1.                       │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### Diagram: Scenario A – Internal User (LAN) Accessing On‑Prem Application (No Change in Phase 1)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SCENARIO A: INTERNAL USER (OFFICE/LAN) → ON‑PREM APPLICATION                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Internal User (Office Network)                                              │
│           │                                                                  │
│           ▼                                                                  │
│  Internal DNS (Split‑horizon / Private)                                      │
│   • app.Contoso.internal → Private IP/VIP (10.x.x.x)                         │
│           │                                                                  │
│           ▼                                                                  │
│  On‑Prem F5 (LAN‑facing) — KEEP AS‑IS IN PHASE 1                             │
│   • LTM (load balancing)                                                     │
│   • WAF (internal app protection, if used)                                   │
│   • APM (SSO/auth policies, if used)                                         │
│           │                                                                  │
│           ▼                                                                  │
│  On‑Prem Applications                                                        │
│   • Includes apps protected by existing on‑prem WAF policies                 │
│                                                                              │
│  Outcome: No routing changes for LAN users during Phase 1.                   │
└──────────────────────────────────────────────────────────────────────────────┘
```
---

## Scenario B – External User (WFH/Internet) Accessing On‑Prem Application (Phase 1b Target Flow)

In Phase 1b steady state, external users access on‑prem apps through:
- **Public DNS** resolving to **Azure Front Door Premium**
- **Front Door WAF** enforcing security at the edge (OWASP + custom rules as required)
- Traffic forwarded to **Palo Alto** as the **origin target / perimeter enforcement boundary**
- Palo Alto routes/NATs traffic to **on‑prem applications**
- **Internet-facing F5 is no longer required** for external access (post-validation)

#### Diagram: Scenario B – External User (WFH/Internet) → Azure Front Door (WAF) → Palo Alto → On‑Prem Apps

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  SCENARIO B: EXTERNAL USER (WFH/INTERNET) → ON‑PREM APPLICATION              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  External User (WFH / Internet)                                              │
│           │                                                                  │
│           ▼                                                                  │
│  Public DNS (Split‑horizon / Public)                                         │
│   • app.Contoso.com → Azure Front Door endpoint                              │
│           │                                                                  │
│           ▼                                                                  │
│  Azure Front Door Premium (Internet Entry Point) + WAF                       │
│   • Global L7 load balancing                                                 │
│   • TLS/SSL termination at the edge                                          │
│   • WAF policy enforcement (OWASP, bot, geo, rate-limit as required)         │
│           │                                                                  │
│           ▼                                                                  │
│  Palo Alto Networks (Perimeter / Origin target)                              │
│   • Firewall policy enforcement                                              │
│   • NAT / routing to on‑prem networks                                        │
│           │                                                                  │
│           ▼                                                                  │
│  On‑Prem Applications                                                        │
│   • Reached directly (internet-facing F5 bypassed/retired in Phase 1b)       │
│                                                                              │
│  Outcome: External access uses Front Door + WAF, with Palo Alto enforcing    │
│  perimeter controls before traffic reaches on‑prem applications.             │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1 – Key Validation Items (to confirm Phase 1b readiness)

To safely move from Phase 1a to Phase 1b, validate:

### A) Internet-facing F5 dependency assessment
- Does the internet-facing F5 provide **APM/SSO** (auth workflows)?
- Are there **iRules** used for routing/header/cookie manipulation?
- Any special **persistence** methods beyond standard cookie affinity?
- Any special **TLS** requirements (mTLS/client certificates, unusual cipher rules)?

### B) Palo Alto publishing feasibility
- Can Palo Alto perform the required **DNAT/SNAT** and routing to application targets?
- Are applications reachable without relying on F5-only VIP constructs?
- Does the security team allow this publishing model and monitoring requirements?

---

# Phase 2 – Internal (LAN) Facing: Modernization Program (Not a Like-for-Like Migration)

> **Primary objective:** Improve the internal access model and reduce long-term complexity through **modernization**, not simply replacing the LAN-facing F5 with another appliance/service.

Phase 2 is intentionally positioned as modernization because internal environments often include:
- legacy authentication patterns (LDAP/Kerberos/forms-based SSO),
- complex iRules and long-lived exceptions,
- tight coupling to internal routing/VLAN/DMZ segmentation, and
- stricter latency/performance expectations for office users.

---

## What we will do (high-level)

After Phase 1 is complete and stable:

### Option A (Preferred): Modernize internal access patterns
- Reassess whether every internal app must remain behind a “LAN VIP” pattern.
- Modernize identity and access:
  - move toward **Microsoft Entra ID** (identity-based access and Zero Trust controls)
  - reduce reliance on network location as the primary security boundary
- Standardize internal ingress patterns:
  - platform-native gateways for Azure-hosted workloads
  - internal reverse proxies/gateways for on‑prem workloads as appropriate
- Improve operational consistency:
  - Infrastructure as Code (IaC)
  - centralized policy management
  - unified logging/monitoring patterns

### Option B (If LAN VIP model must be preserved): Design an internal gateway model with private connectivity
- If internal access must be served by Azure-hosted gateways, it typically requires:
  - **site-to-site VPN** or **ExpressRoute**
  - internal DNS changes
  - careful latency and routing assessment (avoid hairpinning where possible)

> The final Phase 2 approach should be chosen based on internal user experience requirements, application dependencies, and the results/telemetry from Phase 1.

> **[PLACEHOLDER – : WILL PUT THE PHASE 2 (LAN MODERNIZATION) DIAGRAM HERE, LATER]**  

---

## Why Phase 2 is modernization (most important “WHY”)

1. **Avoid re-creating legacy complexity** in a new platform.  
2. **Align with Zero Trust**: move from “LAN = trusted” to identity and conditional access controls.  
3. **Reduce operational overhead**: standardize patterns, reduce custom iRules-like logic, improve repeatability.  
4. **Make connectivity a deliberate decision**: if private routing to Azure gateways is needed, design it intentionally (VPN vs ExpressRoute) with SLA/latency/operational considerations.

---

## Summary / Outcome

### Phase 1 (Internet facing) – two sub-phases
- **Phase 1a:** Azure Front Door becomes the public entry point, with **internet-facing F5 as temporary origin** for low-risk adoption.
- **Phase 1b:** Transition origin away from internet-facing F5 (preferably to **Palo Alto**), then **retire internet-facing F5** for external access.

### Phase 2 (Internal LAN facing)
- Execute as a **modernization program** (not a like-for-like migration), using Phase 1 learnings and validated app requirements.

---

## Next Steps (Suggested)
1. Confirm list of **internet-facing apps** and current F5/WAF/APM dependencies per app.
2. Define Phase 1a success criteria (security, performance, rollback).
3. Pilot 1–2 applications through Front Door with tuned WAF policies.
4. Complete Phase 1a validation checklist to determine readiness for Phase 1b.
5. Begin Phase 2 discovery planning: iRules/APM inventory, internal traffic patterns, and target modernization outcomes.
