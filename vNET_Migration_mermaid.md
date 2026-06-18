***Diagram 1 — Path A: Cisco 8000V (SD-WAN) + Cloud NGFW (SaaS) in the hub***

```mermaid
graph LR
    subgraph BRANCH["Branches / Sites"]
        SITE["Remote Sites"]
    end

    subgraph FABRIC["Cisco SD-WAN Fabric"]
        OVERLAY["SD-WAN Overlay"]
    end

    subgraph AZURE["Azure Virtual WAN"]
        subgraph HUB["Virtual WAN Hub (Microsoft-Managed)"]
            C8KV["Catalyst 8000V x2<br/>(SD-WAN ONLY)<br/>= the 1 NVA"]
            ROUTER["Hub Router<br/>BGP + Routing Intent"]
            CNGFW["Palo Alto Cloud NGFW<br/>(SaaS - NOT an NVA)<br/>does NOT use NVA slot"]
        end
        SPOKES["Spoke VNets<br/>(workloads)<br/>NO UDRs needed"]
    end

    SITE --> OVERLAY
    OVERLAY --> C8KV
    C8KV <-->|eBGP| ROUTER
    ROUTER -->|1. Routing Intent:<br/>detour to firewall| CNGFW
    CNGFW -->|2. inspected, allowed| ROUTER
    ROUTER -->|3. deliver| SPOKES

    style HUB fill:#0078D4,stroke:#fff,color:#fff
    style C8KV fill:#1BA0D7,stroke:#fff,color:#fff
    style ROUTER fill:#FFB900,stroke:#333,color:#000
    style CNGFW fill:#D13438,stroke:#fff,color:#fff
    style BRANCH fill:#107C10,stroke:#fff,color:#fff
    style FABRIC fill:#5C2D91,stroke:#fff,color:#fff
    style SPOKES fill:#5C2D91,stroke:#fff,color:#fff
```

>Why it works: Cloud NGFW is SaaS, not an NVA — so it doesn't consume the hub's single NVA slot, leaving it free for the 8000V. Routing Intent steers traffic to Cloud NGFW inside the hub → no spoke UDRs.

---

***Diagram 2 — Path A: Cisco 8000V with its OWN firewall capability (dual-role)***

```mermaid
graph LR
    subgraph BRANCH["Branches / Sites"]
        SITE["Remote Sites"]
    end

    subgraph FABRIC["Cisco SD-WAN Fabric"]
        OVERLAY["SD-WAN Overlay"]
    end

    subgraph AZURE["Azure Virtual WAN"]
        subgraph HUB["Virtual WAN Hub (Microsoft-Managed)"]
            C8KV["Catalyst 8000V x2<br/>**DUAL-ROLE**<br/>SD-WAN + NGFW firewall<br/>in the SAME box<br/>= the 1 NVA, two jobs"]
            ROUTER["Hub Router<br/>BGP + Routing Intent"]
        end
        SPOKES["Spoke VNets<br/>(workloads)<br/>NO UDRs needed"]
    end

    SITE --> OVERLAY
    OVERLAY --> C8KV
    C8KV <-->|eBGP| ROUTER
    ROUTER -->|1. Routing Intent:<br/>detour to 8000V firewall| C8KV
    C8KV -->|2. inspected by its own<br/>NGFW, allowed| ROUTER
    ROUTER -->|3. deliver| SPOKES

    style HUB fill:#0078D4,stroke:#fff,color:#fff
    style C8KV fill:#D13438,stroke:#fff,color:#fff
    style ROUTER fill:#FFB900,stroke:#333,color:#000
    style BRANCH fill:#107C10,stroke:#fff,color:#fff
    style FABRIC fill:#5C2D91,stroke:#fff,color:#fff
    style SPOKES fill:#5C2D91,stroke:#fff,color:#fff
```
>Why it works: The same 8000V does both SD-WAN and firewalling (its own NGFW capability) — one NVA, two roles — so it satisfies the one-NVA-per-hub rule. Routing Intent steers traffic to the 8000V's firewall role inside the hub → no spoke UDRs. (No Palo Alto here — Cisco does the firewalling.)

---
***Diagram 3 — Path B: Cisco 8000V (SD-WAN) in hub + Palo Alto VM-Series in its own spoke***

```mermaid
graph LR
    subgraph BRANCH["Branches / Sites"]
        SITE["Remote Sites"]
    end

    subgraph FABRIC["Cisco SD-WAN Fabric"]
        OVERLAY["SD-WAN Overlay"]
    end

    subgraph AZURE["Azure Virtual WAN"]
        subgraph HUB["Virtual WAN Hub (Microsoft-Managed)"]
            C8KV["Catalyst 8000V x2<br/>(SD-WAN ONLY)<br/>= the 1 NVA"]
            ROUTER["Hub Router<br/>(NO Routing Intent<br/>for firewall)"]
        end
        subgraph SECSPOKE["Security Spoke (yours)"]
            VMS["Palo Alto<br/>VM-Series"]
        end
        subgraph WSPOKE["Workload Spoke (yours)"]
            W["Workloads"]
            UDR["UDR: 0.0.0.0/0<br/>-> VM-Series<br/>(on EVERY spoke,<br/>now + future)"]
        end
    end

    SITE --> OVERLAY
    OVERLAY --> C8KV
    C8KV <-->|eBGP| ROUTER
    ROUTER -->|1. forward toward spoke| WSPOKE
    W -.->|2. UDR redirects| VMS
    VMS -.->|3. inspected, allowed,<br/>sent to destination| W

    style HUB fill:#0078D4,stroke:#fff,color:#fff
    style C8KV fill:#1BA0D7,stroke:#fff,color:#fff
    style ROUTER fill:#FFB900,stroke:#333,color:#000
    style BRANCH fill:#107C10,stroke:#fff,color:#fff
    style FABRIC fill:#5C2D91,stroke:#fff,color:#fff
    style SECSPOKE fill:#8E562E,stroke:#fff,color:#fff
    style VMS fill:#D13438,stroke:#fff,color:#fff
    style WSPOKE fill:#5C2D91,stroke:#fff,color:#fff
    style UDR fill:#107C10,stroke:#fff,color:#fff
```
>Why it works: The 8000V (SD-WAN) takes the hub's single NVA slot; the VM-Series lives in a separate spoke, so there's no second-NVA conflict. But Routing Intent can't reach a spoke firewall, so traffic is steered via UDRs on every spoke → the "1 or 50" maintenance burden.
---

# Cisco SD-WAN (Catalyst 8000V) + Firewall — Quick Comparison of the 3 Options

| | **Diagram 1** | **Diagram 2** | **Diagram 3** |
|---|---|---|---|
| **Design** | Path A | Path A | Path B |
| **The 1 NVA in hub** | 8000V (SD-WAN only) | 8000V (**dual-role**) | 8000V (SD-WAN only) |
| **Firewall** | Cloud NGFW (SaaS, in hub) | 8000V's own NGFW (in hub) | VM-Series (in a spoke) |
| **Steering** | Routing Intent | Routing Intent | UDRs on spokes |
| **Spoke UDRs?** | ❌ None | ❌ None | ✅ Every spoke |
| **Keeps Palo Alto?** | ✅ (Cloud NGFW) | ❌ (Cisco firewalling) | ✅ (exact VM-Series) |
| **Why no NVA conflict** | SaaS ≠ NVA slot | One NVA, two roles | Firewall is in a spoke |

---
