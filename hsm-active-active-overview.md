# Managed HSM — Dual-Region Active-Active Provisioning
## Pre-requisites & Step-by-Step Overview

> **Purpose.** A one-page mental map of *what gets built and in what order* when provisioning Azure Managed HSM in an **Active-Active dual-region** topology where the org already runs a **Hub-and-Spoke** network, on-prem clients reach Azure via **Conditional DNS Forwarders**, and **all HSM traffic must stay on the private network** (no public internet path).
>
> **Audience.** Infrastructure, security, and platform engineers who need to picture the end-state before touching the CLI.
>
> **How to use it.** Read this once to form the mental model. Then follow ([Deep Technical document](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html)) phase-by-phase for the deep-technical commands, design rationale, risk register, and runbooks.
>


<img width="1536" height="1024" alt="cd1152056c" src="https://github.com/user-attachments/assets/75278981-1516-4e87-b8e8-22c0ae281d22" />


---

## What you are building (in one paragraph)

Two regional **Managed HSM clusters** (one Primary, one DR; FIPS 140-3 Level 3 validated) that share the same keys via **native multi-region replication** and therefore expose the **same key URI** from both regions. Each region's workloads reach **their own local HSM** through a **Private Endpoint** in a dedicated HSM PE spoke that is peered into that region's **existing hub**. On-prem callers reach the same HSM over **ExpressRoute** via the org's **conditional DNS forwarders**. The global public FQDN `<hsm>.managedhsm.azure.net` is fronted by an **internal, Microsoft-managed traffic routing layer** that Azure provisions automatically as part of `az keyvault region add` (DNS TTL 5 s) — there is **no customer-deployed Azure Traffic Manager profile** and **no operator endpoint toggle**. **Public network access on the HSM is OFF** — the data plane is reachable only on the private network.

---

## Pre-requisites

Confirm each item before starting. Anything missing means the steps below will not work cleanly.

### Network (already in place — you only *extend* it)
- [ ] **Hub-and-Spoke per region** in both Primary and DR (hub VNet, ExpressRoute Gateway, Azure Firewall / NVA, Azure Bastion, on-prem-facing **DNS forwarder VMs**).
- [ ] **Hub-to-hub global VNet peering** between Primary and DR hubs.
- [ ] **ExpressRoute** (or site-to-site VPN) from on-prem to both regional hubs.
- [ ] On-prem DNS server able to **conditional-forward** zones to the regional DNS forwarder VMs in Azure.

### IP address allocation
- [ ] **One free /24 per region** for the new HSM PE spoke VNet.

  *Why a whole /24?* The Private Endpoint itself is just one NIC — it consumes a single IP and a /28 subnet is technically enough. We still ask IPAM for a /24 because (a) the org's IPAM allocates VNets in /24 units, and (b) it leaves room **inside the same VNet** for things you will almost certainly want later: a diagnostics / jump VM in the same spoke, a second Private Endpoint (e.g. for the backup storage account), or future HSM-adjacent services — without going back to IPAM for another allocation. Don't shrink below /24 without checking with the networking team.

### Identity & people
- [ ] Microsoft Entra ID tenant + workload subscription with the required Azure **resource providers registered**: `Microsoft.KeyVault`, `Microsoft.Network`, `Microsoft.Compute`, `Microsoft.Sql`, `Microsoft.Storage`, `Microsoft.Insights`.
- [ ] Entra groups created for **HSM admins / operators / auditors**, plus a **break-glass** emergency account.
- [ ] **M-of-N quorum agreed** — pick the people who will jointly hold the keys to the HSM.

  *In plain English:* **N** = the total number of trusted people ("custodians") you nominate to hold the HSM's master key material — recommended **N = 5** (typically one each from Security, Infra, Compliance, App, Audit). **M** = how many of those N people must come together to unlock or rebuild the HSM — recommended **M = 3**. So "3-of-5" means **any 3 of the 5 custodians together** can unseal the HSM; no single person (and no two of them) ever can. Each of the N custodians generates an **offline RSA 3072/4096 key pair** on a hardened workstation and keeps their private key in their own custody (hardware token / smart card / encrypted USB); only the public `.cer` files are handed in for the ceremony in Step 7.
- [ ] Subscription **quotas approved** for Managed HSM (1 per region) in both regions.

### Tooling
- [ ] Azure CLI ≥ `2.61` with the `keyvault` extension on a hardened admin workstation. The Security Domain ceremony (Step 7) runs from this workstation over the **private path** — typically a jump VM in the existing Primary app spoke, reached via the existing hub Bastion. Public access on the HSM stays **Disabled** throughout. (Temporarily opening the HSM public endpoint with a tight IP allow-list is a discouraged fallback used only when the private path cannot be ready in time — see [§4.3](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#43-network-access-during-activation).)

---

## The steps — what happens, in order

Each step is intentionally one or two lines. The deep-technical commands and rationale live in the matching phase of the [Deep Technical document](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html) shown in the right-hand column.

| # | Step | Why it exists | Detail in Deep Technical document |
|---|---|---|---|
| **1** | **Capture** the existing hub, gateway, firewall, Bastion, app-spoke VNets, and **DNS forwarder VM IPs** in both regions. | You are extending the platform, not duplicating it — you need its IDs. | Phase 1 §1.0 / Phase 2 §2.0 |
| **2** | **Create one new HSM PE spoke VNet per region** (`vnet-org-hsm-pri` / `vnet-org-hsm-dr`) with a `/28` `snet-pe-hsm` subnet (private-endpoint network policies = Disabled), and **peer each spoke into its existing regional hub**. | This is the only new VNet plumbing in the whole deployment. Small blast radius. | Phase 1 §1.2–1.4 / Phase 2 §2.2 |
| **3** | **Create a regional Private DNS zone** `privatelink.managedhsm.azure.net` in the **central connectivity subscription** — **one zone per region, never shared**. Link each zone **only** to its own region's hub + app spokes + HSM PE spoke (tag each zone `region=primary` / `region=dr` to make accidental cross-linking visually obvious). | Split-horizon DNS makes Active-Active hands-free in Azure: each region resolves the HSM FQDN to its **local** PE IP, so failover needs no DNS edit on the private path. **Linking a VNet to the wrong zone silently mis-routes HSM traffic across regions and breaks the low-latency Active-Active design — see Pitfalls (R2) below.** | Phase 1 §1.5 / Phase 2 §2.3 |
| **4** | **Add a Conditional Forwarder rule** on the on-prem DNS servers (and on the Azure DNS forwarder VMs if applicable) for `privatelink.managedhsm.azure.net` → forwarder VM IPs in each region. | Lets on-prem callers resolve the HSM to the regional PE without inventing a new DNS topology. | Phase 3 §3.1 |
| **5** | **Provision the Primary HSM** with public access already **Disabled** (`az keyvault create --hsm-name ... --public-network-access Disabled`). 20–30 min wait. Comes up in **Pending** state. Skip ahead and create the Primary Private Endpoint + DNS zone group now (Step 8 for Primary) so a jump VM in the Primary app spoke can reach the HSM for activation. | This is the HSM itself. Activation in Step 6 runs over the private path. | Phase 4 §4.2–4.3 |
| **6** | **Security Domain ceremony — M-of-N**, run from a jump VM in the existing Primary app spoke (reached via the hub Bastion). Gather the M+ custodians, run `az keyvault security-domain download`, store the resulting `SD.json` in **3 offline locations**, custodians keep their private keys. HSM is now **Active**. | The single most critical step. Lose the SD file plus M custodian keys and **every key in the HSM is permanently unrecoverable**. | Phase 5 §5.1–5.5, Appendix B |
| **7** | **Add the DR region** to the same HSM (`az keyvault region add --region <dr>`). Azure provisions the DR cluster, replicates the Security Domain (no second ceremony), and wires the new pool into Microsoft's internal, platform-managed routing layer under the canonical `<hsm>.managedhsm.azure.net` FQDN (DNS TTL 5 s). | This is what makes Active-Active real: same FQDN, same key URIs, RPO ≈ 0, fully automatic global routing — no customer Traffic Manager profile to deploy or operate. | Phase 6 §6.1 + Phase 3 §3.2 |
| **8** | **Create one Private Endpoint per region** into each region's `snet-pe-hsm`, register the A record into that region's Private DNS zone via the PE's DNS zone group, and confirm public network access is **Disabled** on both pools (`--public-network-access Disabled`). | After this step the only path to the HSM is the private network. | Phase 7 §7.1–7.5 |
| **9** | **Assign HSM data-plane RBAC** to the Entra groups (Administrator at `/`, Crypto Officer at `/keys`) and **create the per-workload keys** (`cmk-vmdisk-prod`, `cmk-sqlmi-prod`, `cmk-app-prod`). Grant each workload's managed identity least-privilege role at `/keys/<name>` only. | HSM uses its own **local RBAC**, separate from Azure RBAC. Per-key scoping limits blast radius. | Phase 8 §8.1–8.4, Appendix A |
| **10** | **Wire the workloads to their keys**: Disk Encryption Set → VM disks, TDE Protector → SQL Managed Instance, SDK envelope encryption → application tier. Add retry / circuit-breaker for the ~6-minute replication lag. Always call the HSM at the canonical `https://<hsm>.managedhsm.azure.net/...` — never a regional sub-FQDN or PE IP. | This is where encryption actually starts using your HSM keys. | Phase 9 §9.1–9.6 |
| **11** | **Enable backups, diagnostics, and alerts**: daily + event-triggered HSM backup to immutable storage in a third region; `AuditEvent` to Log Analytics; alerts on SD operations, role changes, key delete/purge, auth-failure bursts, replication lag. | Backups protect against accidental key deletion / ransomware on the storage tier. Logs are mandatory for audit. | Phase 10 §10.1–10.5 |
| **12** | **DR drill + Go-Live**: validate platform-managed global routing convergence (resolve the canonical FQDN from out-of-region while the drill HSM's Primary PE is intentionally made unreachable, expect DR endpoint within one TTL ~5 s), do the annual SD restore into a throwaway test HSM, tick the Go-Live checklist (public access OFF, soft-delete + purge protection ON, **no customer Traffic Manager profile present**, backups verified, alerts firing on tests, runbooks handed over). | A DR plan you don't rehearse is not a DR plan. The Go-Live checklist is the final gate before production traffic. | Phase 11 §11.2–11.5, Phase 12 §12.1 |

---

## Critical pitfalls to avoid (read once, remember always)

The architecture is sound by design. The two failure modes below are **operational, not architectural** — both have caused real-world incidents in similar deployments. They are covered in depth in [§1.6 of the Deep Technical document](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#16-risk-register--operational-risks) (Risk Register R1, R2); the summary here exists so a first-time reader cannot miss them.

- **R1 — Applications must call the HSM at the canonical FQDN.** Failover of the global `<hsm>.managedhsm.azure.net` FQDN is fully automatic and handled by Microsoft's internal, platform-managed routing layer (DNS TTL 5 s) — *but only for clients that resolve this exact name*. An application or operator script that hard-codes a **regional sub-FQDN** (`<hsm>.<region>.managedhsm.azure.net`) or a **PE IP** (`10.12.1.4` / `10.22.1.4`) bypasses the platform router and will keep targeting the dead region until redeployed. In-Azure workloads using the canonical FQDN are unaffected during a real outage: their region-local Private DNS continues to resolve to the surviving region's PE automatically. *Mitigation:* the canonical-FQDN requirement is enforced in [Phase 9 §9.3](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#93-application-tier-envelope-encryption), audited quarterly by [§10.5 check B](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#105-periodic-configuration-audits-quarterly), and validated in the DR drill ([§11.2](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#112-quarterly-drill-no-production-impact) step 6) by confirming the platform router converges on the DR endpoint within one TTL after a simulated Primary outage. Go-Live ([§12.1](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#121-pre-go-live)) explicitly blocks on the absence of any `*.trafficmanager.net` hostname in app config (a leftover from older designs).

- **R2 — Private DNS zone mis-linking.** The two regional `privatelink.managedhsm.azure.net` zones look identical and live in the **same** central connectivity subscription. Linking a VNet to the **wrong** zone (e.g. a DR app spoke accidentally linked to the Primary-region zone) silently mis-routes HSM traffic across regions, breaks the low-latency Active-Active design, and surfaces only during failover — when the workload tries to reach a PE IP that does not exist in its own region. *Mitigation:* enforce **strictly one zone per region**, each linked **only** to its own region's hub + app spokes + HSM PE spoke (Step 3); tag zones `region=primary` / `region=dr` so cross-links are visually obvious; run the quarterly automated audit in [§10.5](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#105-periodic-configuration-audits-quarterly) (`az network private-dns link vnet list`) that fails the compliance scan if any link's location doesn't match its zone's region tag; and during testing **always validate DNS responses** from a workload in each region (`nslookup org-hsm-prod.privatelink.managedhsm.azure.net` must return `10.12.1.4` in Primary and `10.22.1.4` in DR).

---

## Mental model — how traffic flows after Step 12

Read these four lines and you have the whole picture:

1. **Primary-region workload** → resolves `org-hsm-prod.privatelink.managedhsm.azure.net` to **Primary PE IP** (via the Primary regional Private DNS zone) → traffic over the spoke-to-hub-to-PE-spoke path → **Primary HSM**.
2. **DR-region workload** → resolves the same FQDN to **DR PE IP** (via the DR regional Private DNS zone) → **DR HSM**. No cross-region hop in steady state.
3. **On-prem client** → on-prem DNS conditional-forwards `privatelink.managedhsm.azure.net` to the regional **DNS forwarder VMs** in Azure → those VMs answer with the **regional PE IP** → traffic over **ExpressRoute** to the regional PE → **HSM**.
4. **Public-DNS / hybrid caller** (anything not on the private path above) → resolves the global canonical FQDN `<hsm>.managedhsm.azure.net` through Microsoft's **internal, platform-managed routing layer** (DNS TTL 5 s, provisioned automatically by `az keyvault region add`) → the currently-healthy regional endpoint. **No operator action** is required during a regional outage — the router stops returning the unhealthy region's endpoint within one TTL.

---

## What this doc deliberately does **not** cover

- Exact CLI commands, variable values, and IP plan — see ([Deep Technical document §3](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#3-naming-ip-plan-and-variables)).
- Design decisions and the **full** risk register (the two highlights above are summarised; the complete write-up with mitigations enforced in each phase lives in ([Deep Technical document §1.6](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#16-risk-register--operational-risks))).
- The full Security Domain ceremony script — see ([Deep Technical document Appendix B](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#appendix-b--security-domain-ceremony-detailed)).
- Troubleshooting matrix — see ([Deep Technical document Appendix C](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#appendix-c--troubleshooting)).

---

**Next step.** Open ([Deep Technical document §3.3](https://shaleen-wonder-ent.github.io/user-guides/HSM_Implementation_Steps.html#33-cli-variables-paste-into-your-shell)), set the CLI variables, and start at Phase 0.
