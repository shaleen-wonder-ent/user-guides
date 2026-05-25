# AUM-Centric Patch Management — A Practical Guide

> **Scope:** Replace existing WSUS + multi-tool patch workflow with an Azure Update Manager (AUM)–centric single pane of glass, anchored on Azure Arc for the on-prem estate.

---

## 1. Executive Summary

If your current patch management estate is built around **WSUS for distribution**, **multiple secondary tools for verification**, and a **manual reconciliation step for false positives**, your team owns the toolchain, the reports, and the gaps between them.

This guide proposes a **single-console model**: onboard every Windows/Linux server and SQL host to **Azure Arc** (free), and let **Azure Update Manager (AUM)** become the **only place** you assess, schedule, execute, audit, and report on OS-level patching. WSUS is retired (or retained only for true air-gapped pockets); the secondary verification tools become redundant; false-positive reconciliation collapses because AUM's data source is the OS's own update agent (WU / package manager), not a third-party scanner.

To preserve **historical continuity**, ingest each server's pre-Arc patch history (`Get-HotFix`, `dpkg/yum/dnf history`, SQL CU registry) **once at onboarding** into a Log Analytics custom table. From Day 1, your Workbook UNIONs that one-time history with AUM's ongoing data — a single timeline per server, per business unit, per estate.

**Outcome promise:** *one console, one report, one schedule engine, one audit trail — across on-prem, Azure, AWS, GCP, Windows, Linux, and SQL.*

---

## 2. Current State — A Common Pattern

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Flow diagram" 
     src="https://github.com/user-attachments/assets/0b058a8f-c8c5-445b-b443-6b0039a5684d"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


### Common pain points

| # | Pain | Operational cost |
|---|---|---|
| 1 | Multiple tools, each with its own UI and dataset | High context-switching; team specialism per tool |
| 2 | False positives between scanners and WSUS | Hours per cycle on manual reconciliation |
| 3 | Reports stitched manually from CSV exports | Slow, error-prone, no real-time view |
| 4 | WSUS is Windows-only | Linux + SQL CUs covered by separate workflows |
| 5 | No single compliance % across the estate | Risk reporting to leadership is approximate |
| 6 | WSUS is deprecated by Microsoft | Strategic dead-end |

---

## 3. Target State — AUM-Centric

**WSUS vs. Azure Arc + Azure Update Manager — comparative architectures**
<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Flow diagram" 
     src="https://github.com/user-attachments/assets/e98921e2-5264-48fb-82d9-06b593a39947"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


> **Figure 1 — Comparative architectures.** *Left:* WSUS requires an on-prem server to sync from Microsoft Update and push approved updates to Windows clients. *Right:* Azure Update Manager uses the Azure Arc agent on each server to orchestrate patching from a cloud control plane, while each machine downloads update content directly from Microsoft Update (or a Connected Cache at branch sites).

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Flow diagram" 
     src="https://github.com/user-attachments/assets/086d1332-cd0c-4006-b785-dc691f65e899"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


### What changes

| Capability | Before | After |
|---|---|---|
| **Distribution** | WSUS server fleet | Microsoft Update + Delivery Optimization (or Connected Cache for branch sites) |
| **Assessment** | WSUS + 3rd-party scanner | AUM (built on the OS's own WU / package manager — single source of truth) |
| **Scheduling** | GPO + WSUS computer groups | AUM maintenance configurations, tag-scoped |
| **Reporting** | Manual CSV stitching | One Workbook unioning Day-0 history + AUM ARG data, refreshed live |
| **False positives** | Cross-tool reconciliation | Eliminated — the assessment source IS the install source (see assessment-cadence note below) |
| **Linux** | Separate workflow | Same console, same scheduler, same reports |
| **SQL CUs (Azure VM)** | Manual | Installed by AUM via the SQL VM integration (GA Apr 2024) |
| **SQL CUs (Arc-enabled SQL on-prem)** | Manual | AUM applies only SQL/Windows updates marked **Important** or **Critical**; CUs/SPs not so marked still require manual install |
| **Audit** | Multiple exports | Single ARG query → CSV / Workbook / Power BI |

> **Cadence caveat — read this once, remember forever.** The compliance % you see in AUM is the result of the most recent **Periodic Assessment** (runs every 24h by default), not a real-time scan. “The assessment source IS the install source” eliminates *tool-disagreement* false positives — it does not make assessment instantaneous. Trigger an on-demand assessment after any deployment if you need an immediate refresh.

---

## 4. Common Questions — Mapped to Solution

### 4.1 "Can I get a report of security patches installed **BEFORE** I onboarded the server?"

**Native AUM answer:** No — AUM begins recording from the moment its extension reports.

**The solution — one-time Day-0 backfill into Log Analytics:**

1. At onboarding (Arc agent install), automatically deliver a small script to each machine that extracts the OS's full update history. **Recommended delivery: Custom Script Extension via `az connectedmachine extension create`** (GA); Arc Run Command is an alternative but is still preview as of May 2026. See Appendix A.6 for both patterns.
2. The script writes structured JSON to a Data Collection Rule → custom Log Analytics table `LegacyPatchHistory_CL`.
3. From that point on, **the Workbook unions** `LegacyPatchHistory_CL` with `patchinstallationresources` (AUM ARG) to give an unbroken timeline per machine.

**Scripts (provided in Appendix A):**
- Windows: `Get-HotFix` + `Get-WmiObject Win32_QuickFixEngineering` + `WindowsUpdate.log` parse
- Linux: `dpkg -l` / `rpm -qa` + `/var/log/dpkg.log` + `dnf history list`
- SQL Server: `sys.dm_server_registry` + CU registry keys (per instance)

**What you see:** Workbook with a "Patch History" tab per machine, showing two stacked colors — *Pre-Arc (imported once)* + *Post-Arc (AUM live)*. No gap in the timeline.

> **Pilot latency expectation:** plan for **~30 minutes** between Arc onboarding and the first AUM data appearing in the portal / ARG (extension install + first periodic assessment cycle). This is documented and normal — don't panic-debug if a freshly onboarded machine looks empty in the first 20 minutes.

> **Caveat — be honest:** the pre-Arc data is only as complete as what the OS retained. If WSUS purged client install logs older than X days, that's what you get. Most Windows hosts have `Get-HotFix` data going back to OS install, so coverage is usually excellent.

---

### 4.2 "Can I get a comprehensive dashboard for ALL servers, before + after?"

**Yes — three layers, pick the depth you need:**

#### Layer 1 — Built-in (zero customization, available Day 1 of onboarding)
- **Update Manager portal** → Overview, Machines, History, Schedules
- Cross-subscription, cross-cloud, cross-tenant
- Filterable by RG, tag, OS, compliance state
- Drill-down per KB / per machine / per deployment run
- **Cost:** included with AUM

#### Layer 2 — Azure Workbook (custom, ~1 day to build)
- Tiles: estate compliance %, top 10 unpatched machines, critical/security backlog by BU (tag), patch velocity (last 30/60/90 days), pre-vs-post-Arc patch timeline per machine
- Unions `LegacyPatchHistory_CL` with `patchassessmentresources` + `patchinstallationresources`
- Pinned to a shared Azure Dashboard
- Exportable to PDF
- **Cost:** Log Analytics ingestion (~$2.76 / GB) + Workbook runtime free. *Note: at very large estates, Workbook queries consume Log Analytics query units and (for ARG-backed tiles) ARG capacity — usually trivial, but worth measuring if you have 10K+ machines.*

#### Layer 3 — Power BI executive view (optional, ~3 days to build)
- Connect Power BI to ARG via Azure Resource Graph connector
- Page 1: Estate health (KPIs, gauges, "machines outside SLA")
- Page 2: Trending — compliance % over time, MTTR for critical patches
- Page 3: Compliance by business unit / cost center (driven by Azure tags)
- Page 4: Audit — every deployment run, every KB, every machine, exportable
- Scheduled refresh hourly or daily
- **Cost:** Power BI Pro licenses for report authors (~$14 / user / mo at list price; varies with Microsoft 365 plan bundling)

**Sample queries** in Appendix B.
**Workbook skeleton** in Appendix C.

> **Single source of truth:** every number on every tile traces back to either `LegacyPatchHistory_CL` (one-time import) or AUM's ARG resources. No third-party scanner means no reconciliation step.

---

### 4.3 "Can AUM replace WSUS — one place to check, act, report?"

**Yes for the vast majority of enterprise estates.** A WSUS + multi-scanner setup with mostly-connected servers is exactly where AUM shines — a clean replacement is the norm.

#### Replacement scorecard

| Capability you need today | WSUS does it | AUM does it | Notes |
|---|:-:|:-:|---|
| Schedule patches by ring (dev → test → prod) | ✅ (computer groups + GPO) | ✅ (maintenance configs + tag-based dynamic scope) | AUM is more flexible — tag-driven |
| Approve / defer specific KBs | ✅ | ✅ (KB exclude/include lists) | Different UX; functional parity |
| Reboot orchestration | ⚠️ (via GPO) | ✅ Native (`IfRequired`, `Never`, `Always`) | AUM wins |
| Pre/post scripts | ❌ | ✅ Native (Event Grid → webhook → Azure Functions / Logic Apps / Automation runbooks) | AUM wins |
| Linux patching | ❌ | ✅ Native | AUM wins |
| Compliance reporting across estate | ⚠️ (manual) | ✅ ARG + Workbooks | AUM wins |
| Cross-cloud (AWS/GCP) patching | ❌ | ✅ (Arc-onboard them) | AUM wins |
| Hotpatching (Windows Server 2025) | ❌ | ✅ Hotpatch enabled by Azure Arc — **free for Arc-enabled WS2025 as of May 2026**; fewer reboots, faster compliance | AUM wins |
| Future-proof / actively invested | ❌ (deprecated 2024) | ✅ SQL CU support (Apr 2024), Hotpatch for Arc-WS2025 (Jul 2025; free May 2026), Arc Gateway maturing through 2025–2026 | AUM wins |
| Local-cache for low-bandwidth branch | ✅ | ⚠️ Pair with **Microsoft Connected Cache** | WSUS wins as standalone |
| True air-gapped operation | ✅ | ❌ Needs Azure connectivity (direct/proxy/Private Link) | WSUS wins for air-gap only |
| **Third-party application patching** (Adobe, Chrome, Java, Zoom, …) | ⚠️ via custom packages / ConfigMgr | ❌ OS + Microsoft updates only | WSUS/ConfigMgr wins — see §4.3.1 |

> **About third-party app patching:** AUM patches what the OS update agent patches — Windows OS, Microsoft products surfaced via Microsoft Update, and Linux distro packages. It does **not** ship third-party app updates (Adobe Reader, Chrome, 7-Zip, etc.). For those, the standard companions are **Intune** (with the Enterprise App Catalog / Win32 apps), **Configuration Manager**, or a dedicated 3rd-party patch product (Patch My PC, Ivanti, etc.). This is out of scope for this guide but should be acknowledged when sizing WSUS retirement.

#### 4.3.1 When you might still want WSUS (or a hybrid mode)

Even in mostly-connected estates, three scenarios justify keeping WSUS — fully or partially:

1. **Strictly air-gapped / isolated networks** — no path to Azure endpoints, even via proxy or Private Link. AUM cannot operate here. Keep WSUS for that segment only.
2. **Severe Internet-bandwidth constraints at branch sites** — AUM-managed servers download patches individually from Microsoft Update. **Microsoft Connected Cache** is the recommended modern fix, but if you have an existing, healthy WSUS already acting as a local content source, you can keep it for that role.
   > **MCC pilot caveat:** Microsoft Connected Cache for Enterprise & Education supports caching Windows Server updates, but its primary deployment pattern (Delivery Optimization + `DOCacheHost` policy via Intune / MDM / registry) is more mature for Windows clients. Validate server-fleet caching in a pilot before standardizing on it for the WS estate.
3. **Extensive third-party application patching today via WSUS/ConfigMgr** — until that workstream is replaced (Intune / ConfigMgr / 3rd-party tool), WSUS or ConfigMgr stays in play for those packages. AUM handles the OS side in parallel.

**Hybrid mode — WSUS as content source, AUM as control plane.** Some organizations keep WSUS purely for local update *content distribution* (bandwidth optimization) while using AUM for *scheduling, reporting, and audit*. This is supported — but with one important caveat:

> ⚠️ **In hybrid mode, AUM respects WSUS approvals.** If your Windows clients are configured (via GPO/registry) to use a WSUS server as their update source, AUM's deployment will only install updates that have been **approved on that WSUS**. Unapproved updates are skipped, even if AUM has selected them. Microsoft Learn states this plainly: *"Ensure that updates are approved in WSUS, to prevent the failure of Update Manager deployments."* Plan WSUS approval workflows accordingly, or — preferred — point clients at Microsoft Update directly and use Connected Cache for bandwidth, which avoids the dual-approval trap entirely.

#### Recommendation by estate segment

| Estate segment | Recommended tool |
|---|---|
| Corporate Windows + Linux + SQL hosts with normal Azure connectivity | **AUM (replaces WSUS)** |
| Branch offices with bandwidth concerns | **AUM + Microsoft Connected Cache** (local content cache, central control) — preferred over WSUS-as-content-source |
| Branch offices with existing healthy WSUS, near-term cutover not feasible | **Hybrid: WSUS for content + AUM for scheduling/reporting** (mind the approval caveat above) |
| True air-gapped (if any) | **Keep WSUS** for that pocket only |
| Third-party app patching (Adobe, Chrome, Java, …) | **Intune / ConfigMgr / 3rd-party tool** (separate workstream) |
| Workstations / laptops (out of scope of this guide) | **Intune** (separate workstream) |

#### Azure VM ↔ Arc-enabled server — capability parity (read before writing Bicep)

AUM is *not* perfectly symmetric across hosting models. The differences below are the ones that bite during scheduled-patching rollouts:

| Capability | Azure VM | Arc-enabled server |
|---|---|---|
| Periodic Assessment | ✅ | ✅ |
| One-time patch run | ✅ | ✅ |
| Scheduled patching | ✅ — **requires `patchMode = AutomaticByPlatform` with Customer-Managed Schedules** (see Appendix E); schedules silently no-op without it | ✅ — no `patchMode` prerequisite |
| Automatic VM Guest Patching | ✅ | ❌ (not applicable) |
| Hotpatching | ✅ (WS Datacenter Azure Edition; also Standard/Datacenter via Arc) | ✅ for WS2025 — **free as of May 2026** |
| SQL Server CU install | ✅ via SQL VM integration (GA Apr 2024) | ⚠️ Only updates marked **Important** or **Critical**; other CUs/SPs manual |
| Pricing | Free | $5 / server / mo (or free under Defender P2 / ESU enabled by Arc / WS SA or PAYG) |
| RBAC role for write/manage | VM Contributor / Owner | Azure Connected Machine Resource Administrator (+ Onboarding role for first connect) |

#### What "one place" looks like operationally

| Action | Where to go |
|---|---|
| Check what's pending on `srvprod-sql-01` | Portal → Update Manager → Machines → `srvprod-sql-01` |
| Schedule the monthly Patch Tuesday rollout | Portal → Update Manager → Maintenance configurations → create/edit |
| See compliance % for "BU=Finance" | Dashboard tile (Workbook) — auto-filtered by tag |
| Export audit evidence for ISO 27001 | Resource Graph Explorer → run shared query → CSV export |
| Verify a specific KB is installed across the estate | ARG query against `patchinstallationresources/softwarepatches` (see Appendix B.5) |
| Investigate a deployment failure | Update Manager → History → drill into run → see exit code + agent log |

> **One console. One data source. One report. One audit trail.**

---

## 5. Reference Architecture


<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Flow diagram" 
     src="https://github.com/user-attachments/assets/02ed086b-85e2-440f-94ea-0c586c47ba6c"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


### Networking — what the firewall team actually needs

All traffic is **outbound TCP 443** from each Arc machine. 

| Allow-list category | Items |
|---|---|
| **Required service tags** | `AzureActiveDirectory`, `AzureTrafficManager`, `AzureResourceManager`, `AzureArcInfrastructure`, `Storage`, `AzureFrontDoor.Frontend` *(required from April 2026)*, `WindowsAdminCenter` *(if WAC used)* |
| **Always-required URLs** | `*.his.arc.azure.com`, `*.guestconfiguration.azure.com`, `guestnotificationservice.azure.com`, `*.servicebus.windows.net`, `login.microsoftonline.com`, `pas.windows.net` |
| **Install / agent-update URLs** | `download.microsoft.com` (Windows), `packages.microsoft.com` (Linux) |
| **Patch-content URLs (separate from AUM control plane)** | Microsoft Update FQDNs (`*.windowsupdate.com`, `*.update.microsoft.com`, `*.delivery.mp.microsoft.com`) for Windows; distro mirrors for Linux — may instead point to **WSUS / Microsoft Connected Cache / local repo** on the customer network |

#### Options for restrictive networks

- **Azure Arc Private Link Scope (PLS)** — per-region private endpoint covering `*.his.arc.azure.com` + `*.guestconfiguration.azure.com`.
  - ⚠️ **Some endpoints stay public even with PLS** — `login.microsoftonline.com`, `download.microsoft.com`, `guestnotificationservice.azure.com`, `*.servicebus.windows.net` do **not** flow through Private Link. Hand this fact to the firewall team explicitly; it is the single most common PLS planning miss.
- **Azure Arc Gateway** (modern alternative, broader rollout through 2025–2026) — reduces the outbound surface to ~7 FQDNs; 1 gateway supports up to ~2,000 Arc servers per region, with a per-subscription cap on gateways. Preferred over PLS for large or strict-egress estates.

#### Two separate paths from each server — don't conflate them

- **AUM control plane** — outbound 443 to the endpoints above. Small, infrequent.
- **Patch content** — separate path: Microsoft Update (default), Connected Cache (branch), WSUS (hybrid), or distro repo / RHUI / SUSE / `packages.microsoft.com` (Linux). AUM never carries the binary update payload.

---

## 6. Cost Model

### Per-server monthly meters (list price, USD, May 2026 — for planning purposes only)

| Meter | Rate | Required for AUM solution? |
|---|---|---|
| Arc connection | **$0** | ✅ Yes (free) |
| Azure Update Manager (Arc machine) | ~**$5 / server / mo** *(see free-scenarios inset below)* | ✅ Yes |
| Log Analytics ingestion | ~**$2.76 / GB** | ✅ Yes — minimal volume for AUM telemetry (~50–200 MB/server/mo typical) |
| Azure Monitor Agent (AMA) | **$0** (agent) | ✅ Yes (free agent; pays via Log Analytics) |
| Defender for Servers Plan 2 | ~$15 / server / mo *(list; varies by region/currency; covers vuln assessment, MDE integration, FIM, etc.)* | ❌ Optional — only if you want vuln assessment + threat detection |
| Defender for SQL | ~$15 / vCore / mo | ❌ Optional — evaluate alongside Defender for Servers |
| Microsoft Sentinel | ~$2–4 / GB | ❌ Optional |

> **AUM is free for Arc-enabled servers in four scenarios** (per the official AUM pricing page — verify before publishing as Microsoft may extend the list):
> 1. **Azure Local** clusters (Azure benefits enabled).
> 2. Servers with **Extended Security Updates (ESU) enabled by Azure Arc**.
> 3. Subscriptions with **Microsoft Defender for Servers Plan 2** enabled.
> 4. Servers under **Windows Server Management enabled by Azure Arc** (active WS Software Assurance or WS PAYG).
>
> If any of the above applies, the $5/server/month line collapses to **$0** for those servers. In particular, **if Defender for Servers Plan 2 is already deployed for security reasons, AUM becomes a no-cost bonus** — net spend is the security spend, and the $5 + $15 = $20 line from the worked example becomes $15.

### Worked example — 500-server estate (typical mid-sized enterprise)

| Line item | Math | Monthly cost |
|---|---|---|
| Arc connections | 500 × $0 | $0 |
| AUM (500 servers) | 500 × $5 | $2,500 |
| Log Analytics ingestion | 500 × 100 MB = 50 GB × $2.76 | $138 |
| **Total AUM-required spend** | | **~$2,640 / mo** |
| Defender for Servers P2 (if added) | 500 × $15 | $7,500 |
| **Total with security add-on** | | **~$10,140 / mo** |

### Cost offsets (what you **stop** paying)

| Saving | Typical annual value (500 servers) |
|---|---|
| WSUS infrastructure (2–4 HA servers + SQL + storage + ops) | ~$30K–60K/yr |
| 3rd-party patch verification tool licenses | ~$25K–100K/yr |
| 3rd-party compliance reporting tool | ~$15K–50K/yr |
| FTE time on manual reconciliation (0.3–0.5 FTE) | ~$30K–60K/yr |
| **Conservative annual offset** | **~$100K–270K/yr** |

> **Net effect:** AUM-required spend (~$32K/yr) is typically **3–8× less than the tools and effort it replaces**, even before counting risk reduction from faster, more accurate patching.

---

## 7. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|:-:|:-:|---|
| Branch sites overwhelm WAN with Windows Update traffic | Med | Med | Deploy **Microsoft Connected Cache** or **BranchCache** before cutover |
| Air-gapped systems can't use AUM | High in some industries | High | **Retain WSUS** for those segments only; document scope |
| Pilot Workbook numbers don't reconcile to WSUS | Med | Med | Investigation phase in pilot; usually a missing extension or DCR issue |
| Tag taxonomy not enforced → maintenance configs miss machines | Med | High | Use **Azure Policy `Append` effect** to enforce tags at create-time |
| Operator team unfamiliar with AUM UI | High initially | Low | 2–3 hour enablement workshop; documented runbooks |
| Defender for Servers auto-provisioning surprises finance | Med | Med | Explicitly **scope** Defender enablement; don't enable tenant-wide by default |
| Pre-Arc patch history is incomplete | Low–Med | Low | Honest reporting — show `Source` column in Workbook (Pre-Arc vs AUM); document gap |
| SPN secrets used for onboarding leak | Low | High | **7-day max lifetime** secrets, never write to disk on operator laptop, rotate after rollout |
| AUM extension upgrade fails (file lock — known issue) | Low | Low | Documented remediation: delete + re-trigger assessment, or AV exclusion for `C:\Packages\Plugins\` |
| Hybrid WSUS-content-source mode silently drops updates AUM picked | Med (if hybrid retained) | High | Decide early: prefer **Connected Cache** over hybrid. If hybrid is unavoidable, automate **WSUS auto-approval** for the classifications AUM is allowed to install, and reconcile in the Workbook |
| Third-party app patches assumed "covered by AUM" | Med | Med | Make scope explicit in §4.3 / §10; route 3rd-party apps to **Intune / ConfigMgr** as a separate workstream |

---

## Appendix A — Day-0 History Backfill Scripts

### A.1 Windows — `Get-HotFix` + WU log

> See **A.6** for how to deliver this script to an Arc machine. The recommended GA path is **Custom Script Extension** — the Arc `run-command` CLI is still preview as of May 2026 and uses a different shape than Azure-VM run-command.

```powershell
# backfill-windows.ps1 — emits JSON for ingestion via DCR → LegacyPatchHistory_CL

$ErrorActionPreference = 'Stop'
$machineName = $env:COMPUTERNAME
$now = (Get-Date).ToUniversalTime()

# Layer 1: WMI QFE — fast, structured, complete back to OS install
$hotfixes = Get-WmiObject -Class Win32_QuickFixEngineering |
    Select-Object @{N='MachineName';E={$machineName}},
                  @{N='KB';E={$_.HotFixID}},
                  @{N='Description';E={$_.Description}},
                  @{N='InstalledOn';E={$_.InstalledOn}},
                  @{N='InstalledBy';E={$_.InstalledBy}},
                  @{N='Source';E={'Pre-Arc-QFE'}},
                  @{N='ExtractedAt';E={$now}}

# Layer 2: WU agent COM (richer metadata — title, severity, category)
$session = New-Object -ComObject Microsoft.Update.Session
$searcher = $session.CreateUpdateSearcher()
$history = $searcher.QueryHistory(0, $searcher.GetTotalHistoryCount())

$wuHistory = $history | Where-Object { $_.ResultCode -eq 2 } |
    Select-Object @{N='MachineName';E={$machineName}},
                  @{N='KB';E={
                      if ($_.Title -match 'KB\d+') { $matches[0] } else { $null }
                  }},
                  @{N='Title';E={$_.Title}},
                  @{N='Date';E={$_.Date}},
                  @{N='Operation';E={
                      switch ($_.Operation) { 1{'Install'} 2{'Uninstall'} default{'Other'} }
                  }},
                  @{N='Source';E={'Pre-Arc-WU'}},
                  @{N='ExtractedAt';E={$now}}

# Emit JSON — DCR will route to LegacyPatchHistory_CL
@($hotfixes + $wuHistory) | ConvertTo-Json -Depth 4
```

### A.2 Linux (Debian/Ubuntu) — `dpkg` + apt history
```bash
#!/usr/bin/env bash
# Delivered via Custom Script Extension or Arc run-command — see A.6
set -euo pipefail
machine=$(hostname)
now=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Currently installed packages (security relevance via reverse-lookup later)
dpkg-query -W -f='${binary:Package}|${Version}|${Status}\n' |
  awk -F'|' -v m="$machine" -v t="$now" '
    {print "{\"MachineName\":\""m"\",\"Package\":\""$1"\",\"Version\":\""$2"\",\"Status\":\""$3"\",\"Source\":\"Pre-Arc-dpkg\",\"ExtractedAt\":\""t"\"}"}'

# Historical install/upgrade events
zgrep -h "" /var/log/dpkg.log* 2>/dev/null |
  awk -v m="$machine" -v t="$now" '
    /\sinstall\s|\supgrade\s/ {
      printf "{\"MachineName\":\"%s\",\"Time\":\"%sT%sZ\",\"Action\":\"%s\",\"Package\":\"%s\",\"Version\":\"%s\",\"Source\":\"Pre-Arc-dpkg-log\",\"ExtractedAt\":\"%s\"}\n",
        m, $1, $2, $3, $4, $5, t
    }'
```

### A.3 Linux (RHEL/Rocky/Alma) — `dnf` / `rpm`
```bash
#!/usr/bin/env bash
set -euo pipefail
machine=$(hostname)
now=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

# Installed packages
rpm -qa --qf '{"MachineName":"'"$machine"'","Package":"%{NAME}","Version":"%{VERSION}-%{RELEASE}","InstallTime":"%{INSTALLTIME:date}","Source":"Pre-Arc-rpm","ExtractedAt":"'"$now"'"}\n'

# DNF history (full transactions)
dnf history list --reverse 2>/dev/null | tail -n +3 | while read -r line; do
  id=$(echo "$line" | awk '{print $1}')
  [[ "$id" =~ ^[0-9]+$ ]] || continue
  date=$(echo "$line" | awk '{print $4, $5}')
  action=$(echo "$line" | awk '{print $6}')
  echo "{\"MachineName\":\"$machine\",\"TxId\":$id,\"Time\":\"$date\",\"Action\":\"$action\",\"Source\":\"Pre-Arc-dnf-history\"}"
done
```

### A.4 SQL Server — CU/GDR registry per instance
```sql
-- Run via SQL Arc extension or sqlcmd; emit to JSON for DCR ingest
SET NOCOUNT ON;
DECLARE @machine sysname = CAST(SERVERPROPERTY('MachineName') AS sysname);
DECLARE @instance sysname = CAST(SERVERPROPERTY('InstanceName') AS sysname);
DECLARE @ver sysname = CAST(SERVERPROPERTY('ProductVersion') AS sysname);
DECLARE @level sysname = CAST(SERVERPROPERTY('ProductLevel') AS sysname);
DECLARE @update sysname = CAST(SERVERPROPERTY('ProductUpdateLevel') AS sysname);
DECLARE @ref sysname = CAST(SERVERPROPERTY('ProductUpdateReference') AS sysname);

SELECT
  @machine                  AS MachineName,
  ISNULL(@instance,'DEFAULT') AS InstanceName,
  @ver                      AS Version,
  @level                    AS Level,
  @update                   AS UpdateLevel,
  @ref                      AS KB,
  SYSUTCDATETIME()          AS ExtractedAt,
  'Pre-Arc-SQL'             AS Source
FOR JSON AUTO;
```

### A.5 DCR + custom table for ingestion

Create a custom table named `LegacyPatchHistory_CL` in your Log Analytics workspace and a Data Collection Rule (DCR) that routes the JSON emitted by the backfill scripts in §A.3 / §A.4 into it. Ship the DCR + table as Bicep alongside the rest of your AUM infrastructure (e.g. `infra/aum/dcr-legacy.bicep`).

Table schema:

```jsonc
{
  "TimeGenerated": "datetime",
  "MachineName":   "string",
  "Source":        "string",  // Pre-Arc-QFE | Pre-Arc-WU | Pre-Arc-dpkg | Pre-Arc-rpm | Pre-Arc-SQL
  "KB":            "string",
  "Package":       "string",
  "Version":       "string",
  "Action":        "string",
  "Description":   "string",
  "Title":         "string",
  "InstalledOn":   "datetime",
  "InstalledBy":   "string",
  "RawPayload":    "dynamic"
}
```

### A.6 Execution patterns — how to invoke these scripts on Arc machines

Arc machines use `Microsoft.HybridCompute/machines/runCommands` (PUT-based, **preview** as of May 2026), **not** the Azure-VM `Microsoft.Compute/virtualMachines.invoke(commandId='RunPowerShellScript')` pattern. The two CLIs look similar but the parameters and GA state differ — don't copy Azure-VM examples verbatim.

**Pattern A — Recommended (GA): Custom Script Extension on the Arc machine.** Idempotent, runnable as part of an onboarding pipeline.

> The CLI flag is `--type` (not `--extension-type`). `--extension-type` is an Azure-VM-only flag and will fail with *"unrecognized arguments"* on `az connectedmachine extension create`.

```powershell
# Windows
az connectedmachine extension create `
  --resource-group $rg `
  --machine-name $name `
  --location $location `
  --name "DayZeroBackfillWindows" `
  --publisher Microsoft.Compute `
  --type CustomScriptExtension `
  --type-handler-version 1.10 `
  --settings '{ "fileUris": ["https://<blob>/backfill-windows.ps1"], "commandToExecute": "powershell -ExecutionPolicy Unrestricted -File backfill-windows.ps1" }'

# Linux
az connectedmachine extension create `
  --resource-group $rg `
  --machine-name $name `
  --location $location `
  --name "DayZeroBackfillLinux" `
  --publisher Microsoft.Azure.Extensions `
  --type CustomScript `
  --settings '{ "fileUris": ["https://<blob>/backfill-deb.sh"], "commandToExecute": "bash backfill-deb.sh" }'
```

**Pattern B — Arc Run Command (preview).** Use only if your operator workflow specifically requires it, and mark it preview to the customer.

```powershell
# PowerShell (preview)
New-AzConnectedMachineRunCommand `
  -ResourceGroupName $rg `
  -MachineName $name `
  -Location $location `
  -RunCommandName "DayZeroBackfillWindows" `
  -SourceScriptUri "<SAS URI to backfill-windows.ps1>"

# az CLI equivalent (preview) — note: `create` not `invoke`; `--script` not `--scripts`; no `--command-id`
az connectedmachine run-command create `
  --resource-group $rg `
  --machine-name $name `
  --name "DayZeroBackfillWindows" `
  --location $location `
  --script "@backfill-windows.ps1"
```

---

## Appendix B — Sample Queries

### B.1 Compliance % across the estate (ARG)
```kql
patchassessmentresources
| where type has "patchassessmentresults"
| extend critical  = toint(properties.availablePatchCountByClassification.critical)
| extend security  = toint(properties.availablePatchCountByClassification.security)
| extend other     = toint(properties.availablePatchCountByClassification.other)
| extend total     = critical + security + other
| extend machineName = tostring(split(id,"/")[8])
| summarize
    Machines       = count(),
    Compliant      = countif(total == 0),
    NonCompliant   = countif(total > 0)
| extend CompliancePct = round(100.0 * Compliant / Machines, 1)
```

### B.2 Top 20 machines with most pending security patches
```kql
patchassessmentresources
| where type has "patchassessmentresults"
| extend machineName = tostring(split(id,"/")[8])
| extend security    = toint(properties.availablePatchCountByClassification.security)
| extend critical    = toint(properties.availablePatchCountByClassification.critical)
| project machineName, critical, security, total = critical + security
| top 20 by total desc
```

### B.3 Pre + Post unified history for a single machine (Log Analytics + ARG join)
```kql
// Run in the Log Analytics workspace (uses arg() cross-service query)
let post =
    arg("").patchinstallationresources
    | where type =~ "microsoft.compute/virtualmachines/patchinstallationresults/softwarepatches"
        or type =~ "microsoft.hybridcompute/machines/patchinstallationresults/softwarepatches"
    | extend machineName = tostring(split(id, "/")[8])
    | project
        TimeGenerated = todatetime(properties.lastModifiedDateTime),
        MachineName   = machineName,
        KB            = tostring(properties.kbId),
        Source        = "AUM";
let pre =
    LegacyPatchHistory_CL
    | where Source startswith "Pre-Arc"
    | project TimeGenerated, MachineName, KB, Source;
union pre, post
| where MachineName == "srvprod-sql-01"
| order by TimeGenerated desc
```

### B.4 Deployment-run success rate over the last 30 days
```kql
patchinstallationresources
| where type =~ "microsoft.compute/virtualmachines/patchinstallationresults"
    or type =~ "microsoft.hybridcompute/machines/patchinstallationresults"
| where todatetime(properties.lastModifiedDateTime) > ago(30d)
| extend status = tostring(properties.status)
| summarize Runs = count() by status
| extend TotalRuns = toscalar(
    patchinstallationresources
    | where type =~ "microsoft.compute/virtualmachines/patchinstallationresults"
        or type =~ "microsoft.hybridcompute/machines/patchinstallationresults"
    | where todatetime(properties.lastModifiedDateTime) > ago(30d)
    | count)
| extend Pct = round(100.0 * Runs / TotalRuns, 1)
```

### B.5 Has a specific KB landed across the estate?
```kql
patchinstallationresources
| where type =~ "microsoft.compute/virtualmachines/patchinstallationresults/softwarepatches"
    or type =~ "microsoft.hybridcompute/machines/patchinstallationresults/softwarepatches"
| extend machineName       = tostring(split(id, "/")[8])
| extend kbId              = tostring(properties.kbId)
| extend installationState = tostring(properties.installationState)
| where kbId == "KB5005565"
| summarize Machines = make_set(machineName) by installationState
```

---

## Appendix C — Workbook Structure

```
Workbook: "Patch Estate — AUM-Centric"
├── Tab 1: Estate overview
│   ├── KPI: Compliance %
│   ├── KPI: Critical patches pending
│   ├── KPI: Machines onboarded
│   └── Stacked bar: pending by BU (tag)
├── Tab 2: Machines
│   ├── Grid: machine | OS | last assessed | critical | security | other | last deploy
│   └── Drill-down → Tab 4 for that machine
├── Tab 3: Deployment history
│   ├── Timeline: deployment runs (last 90d)
│   └── Grid: run | maintenance config | machines | success% | duration
├── Tab 4: Per-machine timeline (drill-down)
│   ├── Banner: machine identity, OS, last assessment, compliance state
│   ├── Stacked timeline: Pre-Arc (orange) + AUM (blue)
│   └── Grid: every KB ever installed, sortable by date
└── Tab 5: Audit export
    ├── Parameters: date range, BU, OS
    └── Grid (CSV-exportable): full deployment + install history
```

Author the Workbook in **Portal → Monitor → Workbooks → New**, then export it via **Advanced editor → JSON** and check the result into source control (e.g. `infra/aum/workbook-patch-estate.json`) so the same template can be re-imported across environments.

> **ARG retention — plan for the export.** Azure Resource Graph keeps `patchassessmentresources` for **7 days** and `patchinstallationresources` (incl. `/softwarepatches`) for **30 days**. Tabs **3, 4, and 5** above will start dropping rows beyond those windows. For longer history, the Workbook reads from `LegacyPatchHistory_CL` (Day-0 backfill) **plus** a continuous export of `patchinstallationresources` to a `PatchInstall_CL` custom table — a scheduled Azure Function or Logic App writing through a DCR is the lightest pattern. Build this *before* the first patch cycle if your audit team needs >30-day history.

---

## Appendix D — Bulk Arc Onboarding Pattern

### D.1 Active Directory GPO (Windows estate)
1. Create SPN `sp-arc-onboard` with role `Azure Connected Machine Onboarding` scoped to the target subscription/RG.
2. Issue a short-lived credential — Microsoft recommends **certificate-based authentication** or short-lived client secrets for Arc onboarding SPNs; the 7-day client-secret lifetime used in this guide is a policy choice, not a Microsoft hard rule. Store the credential in a secure ops vault (not in GPO).
3. Generate onboarding script from the Azure Arc portal: **Servers → Add → Add at scale → Group Policy**.
4. Place `OnboardingScript.ps1` + the encrypted credentials JSON on a SYSVOL UNC path.
5. Link GPO to OUs containing target servers — scheduled task runs once, then disables.
6. Rotate SPN secret after rollout.

### D.2 Configuration Manager (SCCM)
- Use the Configuration Manager pre-built script package from the Arc portal (Add at scale → Configuration Manager).
- Same SPN pattern.

### D.3 Linux — Ansible / shell
```bash
# /etc/ansible/playbooks/arc-onboard.yml
- hosts: linux_servers
  become: yes
  tasks:
    - name: Install azcmagent
      shell: |
        curl -L -o /tmp/install.sh https://aka.ms/azcmagent-linux
        bash /tmp/install.sh
      args:
        creates: /usr/bin/azcmagent
    - name: Connect to Arc
      shell: |
        azcmagent connect \
          --service-principal-id "{{ arc_spn_id }}" \
          --service-principal-secret "{{ arc_spn_secret }}" \
          --tenant-id "{{ tenant_id }}" \
          --subscription-id "{{ subscription_id }}" \
          --resource-group "{{ arc_rg }}" \
          --location "{{ arc_region }}" \
          --tags "env={{ env }},bu={{ bu }},patchring={{ patchring }}"
      no_log: true
```

### D.4 Post-onboarding — auto-enable AUM via Azure Policy
Use built-in policy initiative **"Configure periodic checking for missing system updates on azure virtual machines and Azure Arc-enabled servers"** scoped to the Arc resources subscription / management group. Effect: `DeployIfNotExists`. New machines automatically get the patch extension + a daily assessment schedule.

---

## Appendix E — Maintenance Configurations

> **Prerequisite for Azure VMs (not Arc):** for the Azure VM to honour a maintenance configuration of scope `InGuestPatch`, the VM must be set to `patchMode = AutomaticByPlatform` with platform-safety-checks bypassed on user schedule. Without this, the schedule silently no-ops. Arc-enabled machines have **no equivalent prerequisite**. Set this once per VM — example at the end of this appendix.

> **Dynamic-scope limits (Public Cloud, May 2026):** per dynamic scope — **1,000** resources, **50** tag filters, **50** RG filters. Per region/subscription — **250** schedules, **200** dynamic scopes per schedule. Size your rings accordingly.

Example Bicep for the four-ring rollout pattern (one per ring). Apply scope dynamically via `tagFilter`.

```bicep
param location string = resourceGroup().location

resource ring0 'Microsoft.Maintenance/maintenanceConfigurations@2023-04-01' = {
  name: 'mc-ring0-canary'
  location: location
  properties: {
    maintenanceScope: 'InGuestPatch'
    extensionProperties: {
      InGuestPatchMode: 'User'
    }
    maintenanceWindow: {
      startDateTime: '2026-06-09 02:00'   // Tue 9 Jun 2026 = 2nd Tuesday — keep startDateTime aligned with recurEvery
      duration: '03:55'
      timeZone: 'India Standard Time'
      recurEvery: '1Month Second Tuesday'
    }
    installPatches: {
      rebootSetting: 'IfRequired'
      windowsParameters: {
        classificationsToInclude: ['Critical', 'Security']
      }
      linuxParameters: {
        classificationsToInclude: ['Critical', 'Security']
      }
    }
  }
}

// Dynamic scope assignment (tag-based)
resource ring0Assign 'Microsoft.Maintenance/configurationAssignments@2023-04-01' = {
  name: 'assign-ring0-canary'
  properties: {
    maintenanceConfigurationId: ring0.id
    filter: {
      resourceTypes: [
        'Microsoft.HybridCompute/machines'
        'Microsoft.Compute/virtualMachines'
      ]
      tagSettings: {
        filterOperator: 'All'
        tags: {
          patchring: ['ring0']
        }
      }
    }
  }
}
```

Replicate for Ring 1 (next day), Ring 2 (T+7), Ring 3 (T+14). One file, four resources.

### Cross-RG dynamic scope — deploy the `configurationAssignments` at subscription scope

If the tag scope spans more than one resource group (the common case for ring-based rollout), the `configurationAssignments` resource must be deployed at **subscription scope**, not at the RG that holds the maintenance configuration:

```bicep
// main.bicep — module call
targetScope = 'subscription'

module ring0Assign './ring0-assignment.bicep' = {
  name: 'assign-ring0-canary'
  scope: subscription()      // required for cross-RG dynamic scope
  params: {
    configId: ring0.outputs.configId
  }
}
```

If the tag scope is contained inside a single RG, deploying alongside the maintenance configuration at RG scope also works — but the subscription-scope pattern is the safer default.

### Azure-VM-only: set `patchMode = AutomaticByPlatform` (Customer-Managed Schedules)

For an *existing* Azure VM, the canonical day-to-day pattern is an in-place CLI update — it's idempotent and avoids the partial-`Microsoft.Compute/virtualMachines` re-declaration that Bicep would otherwise require (a full VM template demands `hardwareProfile`, `storageProfile`, full `osProfile`, etc.).

```bash
# Windows VM
az vm update -g $rg -n $vmName \
  --set osProfile.windowsConfiguration.patchSettings.patchMode=AutomaticByPlatform \
        osProfile.windowsConfiguration.patchSettings.automaticByPlatformSettings.bypassPlatformSafetyChecksOnUserSchedule=true

# Linux VM
az vm update -g $rg -n $vmName \
  --set osProfile.linuxConfiguration.patchSettings.patchMode=AutomaticByPlatform \
        osProfile.linuxConfiguration.patchSettings.automaticByPlatformSettings.bypassPlatformSafetyChecksOnUserSchedule=true
```

For **new** VMs being created in Bicep / ARM, set `patchMode` and `automaticByPlatformSettings.bypassPlatformSafetyChecksOnUserSchedule` inside the VM's `osProfile.windowsConfiguration.patchSettings` (or `linuxConfiguration.patchSettings`) at creation time — the same shape, but inside a full VM resource declaration.

**Arc-enabled servers require no `patchMode` setting — omit this step entirely for Arc.**

---

## Appendix F — WSUS Decommission Checklist

Per WSUS server / scope:

- [ ] All managed Windows machines are now Arc-onboarded and report to AUM
- [ ] AUM has shown ≥ 1 successful assessment cycle for all those machines
- [ ] AUM compliance % reconciles to WSUS last report within ±2%
- [ ] At least 1 full patch cycle (Patch Tuesday + deployment) executed via AUM
- [ ] Maintenance configurations cover all rings/BUs of in-scope machines
- [ ] GPO that pointed WU clients to WSUS is unlinked
- [ ] Verify clients are now pulling from MU / Connected Cache (event log: `Windows Update Client`, source `Microsoft-Windows-WindowsUpdateClient`)
- [ ] Audit/compliance team has signed off on AUM Workbook as primary report source
- [ ] WSUS server documented (config, approval rules, computer groups) — for rollback
- [ ] WSUS server placed in **read-only / quiesced** state for 30 days as safety buffer
- [ ] After 30 days uneventful — decommission WSUS server, reclaim SQL + storage

---

