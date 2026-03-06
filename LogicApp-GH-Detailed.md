# Azure Logic App (Consumption) — Auto-Close ServiceNow Incident on Azure Monitor Alert Resolution


---

## 📌 Table of Contents

1. [Solution Overview](#solution-overview)
2. [Architecture Overview](#architecture-overview)
3. [Prerequisites](#prerequisites)
4. [Complete Logic App Workflow — Block-by-Block (Azure Portal)](#complete-logic-app-workflow)
5. [Workflow Diagram](#workflow-diagram)
6. [Detailed Block Configuration](#detailed-block-configuration)
   - [Block 1 — HTTP Trigger](#block-1--http-trigger)
   - [Block 2 — Parse JSON](#block-2--parse-json)
   - [Block 3 — Top-Level Condition (monitorCondition)](#block-3--top-level-condition)
   - [Block 4A — TRUE Branch: Parse CPU Metric Value](#block-4a--true-branch-parse-cpu-metric-value)
   - [Block 5A — Nested Condition: Determine Severity](#block-5a--nested-condition-determine-severity)
   - [Block 6A — ServiceNow: Create Record (Incident)](#block-6a--servicenow-create-record)
   - [Block 4B — FALSE Branch: Initialize alertId Variable](#block-4b--false-branch-initialize-alertid-variable)
   - [Block 5B — ServiceNow: List Records](#block-5b--servicenow-list-records)
   - [Block 6B — Condition: Was Incident Found?](#block-6b--condition-was-incident-found)
   - [Block 7B — ServiceNow: Update Record (Close Incident)](#block-7b--servicenow-update-record)
7. [Key Expressions Reference](#key-expressions-reference)
8. [ServiceNow Field Mapping](#servicenow-field-mapping)
9. [Approach Comparison: monitorCondition vs Delay](#approach-comparison)
10. [Why monitorCondition Approach Wins](#why-monitorcondition-approach-wins)
11. [Implementation Checklist](#implementation-checklist)

---

## Solution Overview

| Item | Detail |
|------|--------|
| **Azure Service** | Logic App — Consumption (Multi-tenant) |
| **Trigger Source** | Azure Monitor Action Group (Common Alert Schema) |
| **Integration Target** | ServiceNow (Incident Table) |
| **Metric Monitored** | CPU Utilization (%) on Azure VM |
| **Alert Threshold** | CPU > 80% triggers alert |
| **Severity Mapping** | 80–90% → P3 (SEV 2), 90–95% → P2 (SEV 1), >95% → P1 (SEV 0) |
| **Auto-Close Trigger** | `monitorCondition = 'Resolved'` in Azure Monitor payload |
| **Correlation Key** | `alertId` stored in ServiceNow custom field `u_azure_alert_id` |

---

## Architecture Overview

High-level end-to-end flow from Azure Monitor through to ServiceNow:

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Flow diagram" 
     src="https://github.com/user-attachments/assets/14ec9e81-a537-4ea3-aa0a-47f9e9f18d1e"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


> **Key**: The same Logic App handles both `Fired` and `Resolved` events. Azure Monitor sends both payloads automatically — no polling needed.

---

## Prerequisites

### 1. Azure Monitor Action Group
- Navigate to: **Azure Portal → Monitor → Alerts → Action Groups**
- Edit your Action Group → Under **Actions**, select your Logic App
- ✅ **Enable Common Alert Schema = ON**

> This ensures every alert payload (Fired & Resolved) contains:
> - `data.essentials.monitorCondition` → "Fired" or "Resolved"
> - `data.essentials.alertId` → Unique Alert Resource ID
> - `data.essentials.resolvedDateTime` → ISO timestamp of resolution

### 2. ServiceNow Custom Field
- Create a custom field on the **Incident** table:
  - **Field Label**: `Azure Alert ID`
  - **Field Name (Column)**: `u_azure_alert_id`
  - **Type**: `String`
- This is the **correlation key** that links ServiceNow incidents back to Azure Monitor alerts.

### 3. Logic App Connections
- ServiceNow connector authenticated with a ServiceNow service account
- Permissions: `incident` table — **Read**, **Create**, **Write**

---

## Complete Logic App Workflow

### Block-by-Block (Azure Portal Designer View)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  BLOCK 1 ── TRIGGER                                                     │
│       When a HTTP request is received                                   │
│       Method: POST                                                      │
│       (Azure Monitor Action Group posts here)                           │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  BLOCK 2 ── DATA OPERATION                                              │
│      Parse JSON                                                         │
│       Content : triggerBody()                                           │
│       Schema  : Common Alert Schema (Azure Monitor)                     │
└───────────────────────────┬─────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  BLOCK 3 ── CONTROL                                                     │
│      Condition                                                          │
│       Name   : "Check monitorCondition"                                 │
│       Expression:                                                       │
│       @equals(triggerBody()?['data']?['essentials']                     │
│               ?['monitorCondition'], 'Fired')                           │
└────────────┬───────────────────────────────┬────────────────────────────┘
             │ TRUE                          │ FALSE
             │ (Alert Fired)                 │ (Alert Resolved)
             ▼                               ▼
┌────────────────────────┐     ┌─────────────────────────────────────────┐
│ BLOCK 4A               │     │ BLOCK 4B                                │
│     Initialize Variable│     │     Initialize Variable                 │
│  Name : varCPUValue    │     │  Name : varAlertId                      │
│  Type : Float          │     │  Type : String                          │
│  Value:                │     │  Value:                                 │
│  triggerBody()         │     │  triggerBody()?['data']                 │
│  ?['data']             │     │  ?['essentials']?['alertId']            │
│  ?['alertContext']     │     │                                         │
│  ?['condition']        │     └──────────────┬──────────────────────────┘
│  ?['allOf'][0]         │                    │
│  ?['metricValue']      │                    ▼
└────────────┬───────────┘     ┌─────────────────────────────────────────┐
             │                 │ BLOCK 5B                                │
             ▼                 │     ServiceNow — List Records           │
┌────────────────────────┐     │  Table : incident                       │
│ BLOCK 5A               │     │  Query :                                │
│     Condition          │     │  u_azure_alert_id=@{variables           │
│  Name:"Determine       │     │  ('varAlertId')}^state!=7               │
│  Severity"             │     │                                         │
│                        │     │  (Finds open incidents matching         │
│  Branch 1:             │     │   the Azure alertId)                    │
│  varCPUValue >= 95     │     └──────────────┬──────────────────────────┘
│  → P1 (SEV 0)          │                    │
│                        │                    ▼
│  Branch 2:             │     ┌─────────────────────────────────────────┐
│  varCPUValue >= 90     │     │ BLOCK 6B                                │
│  AND < 95              │     │     Condition                           │
│  → P2 (SEV 1)          │     │  Name : "Was Incident Found?"           │
│                        │     │  Expression:                            │
│  Branch 3 (else):      │     │  @greater(length(body('List_Records')   │
│  varCPUValue >= 80     │     │  ?['result']), 0)                       │
│  AND < 90              │     │                                         │
│  → P3 (SEV 2)          │     └──────┬──────────────────────────────────┘
└────────────┬───────────┘            │ TRUE (Incident found)
             │                        ▼
             ▼            ┌─────────────────────────────────────────┐
┌────────────────────────┐│ BLOCK 7B                                │
│ BLOCK 6A               ││     ServiceNow — Update Record          │
│     ServiceNow —       ││  Table  : incident                      │
│     Create Record      ││  Record ID:                             │
│                        ││  body('List_Records')?['result'][0]     │
│  Table: incident       ││  ?['sys_id']                            │
│  Fields:               ││                                         │
│  • short_description   ││  Fields to Update:                      │
│  • urgency             ││  • state        → 6  (Resolved)         │
│  • impact              ││    (or 7 for Closed)                    │
│  • severity            ││  • close_code   → "Resolved by Caller"  │
│  • description         ││  • close_notes  → "Auto-closed: Azure   │
│  • u_azure_alert_id ←  ││    Monitor alert resolved at            │
│    (store alertId here)││    @{triggerBody()?['data']             │
│                        ││    ?['essentials']                      │
└────────────────────────┘│    ?['resolvedDateTime']}'              │
                          └─────────────────────────────────────────┘
```

---

## Workflow Diagram

Detailed Logic App internal flow showing all decision points and actions:

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Flow diagram" 
     src="https://github.com/user-attachments/assets/89dc08aa-dcd2-41eb-9be5-fea0173453a5"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


---

## Detailed Block Configuration

### Block 1 — HTTP Trigger

| Setting | Value |
|---------|-------|
| **Action Type** | `When a HTTP request is received` |
| **Method** | `POST` |
| **Who calls it** | Azure Monitor Action Group |
| **Schema** | Use [Common Alert Schema](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-common-schema-definitions) |

> After saving, Azure generates the **HTTP POST URL** → paste this into your Azure Monitor Action Group.

---

### Block 2 — Parse JSON

| Setting | Value |
|---------|-------|
| **Action Type** | `Data Operations → Parse JSON` |
| **Content** | `triggerBody()` |
| **Schema** | Paste the Common Alert Schema JSON schema |

> **Tip**: Click "Use sample payload to generate schema" and paste a sample Fired alert payload from Azure Monitor.

---

### Block 3 — Top-Level Condition

| Setting | Value |
|---------|-------|
| **Action Type** | `Control → Condition` |
| **Name** | `Check monitorCondition` |
| **Left Value** | `triggerBody()?['data']?['essentials']?['monitorCondition']` |
| **Operator** | `is equal to` |
| **Right Value** | `Fired` |

---

### Block 4A — TRUE Branch: Parse CPU Metric Value

| Setting | Value |
|---------|-------|
| **Action Type** | `Variables → Initialize Variable` |
| **Name** | `varCPUValue` |
| **Type** | `Float` |
| **Value** | `triggerBody()?['data']?['alertContext']?['condition']?['allOf'][0]?['metricValue']` |

---

### Block 5A — Nested Condition: Determine Severity

This uses **Switch** or nested **Conditions** in Azure Portal:

| Branch | Condition | Severity Variable | Urgency | Impact |
|--------|-----------|-------------------|---------|--------|
| Branch 1 | `varCPUValue >= 95` | `P1` | `1` | `1` |
| Branch 2 | `varCPUValue >= 90` AND `varCPUValue < 95` | `P2` | `2` | `2` |
| Branch 3 (else) | `varCPUValue >= 80` AND `varCPUValue < 90` | `P3` | `3` | `3` |

> Use `Set Variable` actions inside each branch to set `varSeverity`, `varUrgency`, `varImpact`.

---

### Block 6A — ServiceNow: Create Record

| Setting | Value |
|---------|-------|
| **Action Type** | `ServiceNow → Create Record` |
| **Connection** | Your ServiceNow connection |
| **Table Name** | `incident` |

**Fields to populate:**

| ServiceNow Field | Value / Expression |
|------------------|--------------------|
| `short_description` | `Azure Monitor Alert: CPU utilization exceeded threshold` |
| `description` | CPU value + resource name | From alert payload |
| `urgency` | `@{variables('varUrgency')}` |
| `impact` | `@{variables('varImpact')}` |
| `category` | `infrastructure` |
| `u_azure_alert_id` | `@{triggerBody()?['data']?['essentials']?['alertId']}` ← **Correlation Key** |

---

### Block 4B — FALSE Branch: Initialize alertId Variable

| Setting | Value |
|---------|-------|
| **Action Type** | `Variables → Initialize Variable` |
| **Name** | `varAlertId` |
| **Type** | `String` |
| **Value** | `triggerBody()?['data']?['essentials']?['alertId']` |

---

### Block 5B — ServiceNow: List Records

| Setting | Value |
|---------|-------|
| **Action Type** | `ServiceNow → List Records` |
| **Connection** | Your ServiceNow connection |
| **Table Name** | `incident` |
| **Query** | `u_azure_alert_id=@{variables('varAlertId')}^state!=7` |
| **Max Records** | `1` |

> This finds the open incident that was created when the alert fired.  
> `state != 7` ensures we don't try to close an already-closed ticket.

---

### Block 6B — Condition: Was Incident Found?

| Setting | Value |
|---------|-------|
| **Action Type** | `Control → Condition` |
| **Name** | `Was Incident Found?` |
| **Left Value** | `@length(body('List_Records')?['result'])` |
| **Operator** | `is greater than` |
| **Right Value** | `0` |

---

### Block 7B — ServiceNow: Update Record (Close Incident)

| Setting | Value |
|---------|-------|
| **Action Type** | `ServiceNow → Update Record` |
| **Connection** | Your ServiceNow connection |
| **Table Name** | `incident` |
| **Record sys_id** | `@{body('List_Records')?['result'][0]?['sys_id']}` |

**Fields to update:**

| ServiceNow Field | Value |
|-----------------|-------|
| `state` | `6` (Resolved) or `7` (Closed) — depends on your SNOW process |
| `close_code` | `Resolved by Caller` |
| `close_notes` | `Auto-closed by Azure Logic App. Azure Monitor alert resolved at: @{triggerBody()?['data']?['essentials']?['resolvedDateTime']}` |

---

## Key Expressions Reference

| Purpose | Expression |
|---------|-----------|
| Get monitorCondition | `triggerBody()?['data']?['essentials']?['monitorCondition']` |
| Get alertId | `triggerBody()?['data']?['essentials']?['alertId']` |
| Get resolvedDateTime | `triggerBody()?['data']?['essentials']?['resolvedDateTime']` |
| Get CPU metric value | `triggerBody()?['data']?['alertContext']?['condition']?['allOf'][0]?['metricValue']` |
| Get resource name | `triggerBody()?['data']?['essentials']?['configurationItems'][0]` |
| Get alert fired time | `triggerBody()?['data']?['essentials']?['firedDateTime']` |
| Check list result count | `@length(body('List_Records')?['result'])` |
| Get first record sys_id | `@{body('List_Records')?['result'][0]?['sys_id']}` |

---

## ServiceNow Field Mapping

### On Incident Creation (Alert Fired)

| SNOW Field | Value | Notes |
|-----------|-------|-------|
| `short_description` | `Azure Monitor Alert: High CPU` | Static or dynamic |
| `description` | CPU value + resource name | From alert payload |
| `urgency` | `1` / `2` / `3` | Mapped from severity |
| `impact` | `1` / `2` / `3` | Mapped from severity |
| `category` | `infrastructure` | Static |
| `u_azure_alert_id` | `alertId` from payload | **Correlation Key** ← Critical |

### On Incident Closure (Alert Resolved)

| SNOW Field | Value | Notes |
|-----------|-------|-------|
| `state` | `6` or `7` | 6=Resolved, 7=Closed |
| `close_code` | `Resolved by Caller` | Standard SNOW close code |
| `close_notes` | Auto-close message + timestamp | Include `resolvedDateTime` |

---

## Approach Comparison

| Criteria | ✅ monitorCondition (Recommended) | ❌ Delay-Based (Alternate) |
|----------|----------------------------------|---------------------------|
| **Architecture** | Event-driven | Polling-based |
| **Cost** | Low — short-lived runs | High — 15-min idle runs |
| **Reliability** | Deterministic, event-triggered | Race conditions possible |
| **Complexity** | Simple branching in Logic App | Extra API calls + auth needed |
| **Idempotency** | Re-fires handled cleanly | Risk of duplicate/incorrect closures |
| **Extra Permissions** | ServiceNow access only | Azure RBAC + Alerts Management API |
| **Scalability** | Scales with alert volume | Throttle risk under high load |
| **Alert State Accuracy** | Always accurate (direct event) | May be stale after 15 min delay |
| **Maintenance** | Low — no polling logic | High — polling + API versioning |

---

## Why monitorCondition Approach Wins

### ✅ Event-Driven Architecture
Azure Monitor sends a **Resolved** payload automatically when CPU drops below the threshold after the configured 15-minute evaluation window. There is no need to poll — the system notifies you.

### ✅ Correlation Key Pattern
By storing `alertId` in `u_azure_alert_id` at incident creation time, the Resolved branch can **always find the right ticket** — even across re-fires, regardless of how many alerts are in-flight simultaneously.

### ✅ Cost Efficiency
Each Logic App run completes in seconds (Fired) or a few seconds (Resolved). There are no long-running instances, no idle wait time, and no unnecessary Azure resource consumption.

### ✅ Handles Edge Cases

| Scenario | Behavior |
|----------|----------|
| Alert resolves, incident already manually closed | `List Records` returns empty → no action (graceful skip) |
| Alert fires multiple times | Each fire creates a new incident with its own unique `alertId` |
| Alert resolves before incident is created | `List Records` returns empty → no action |
| Multiple open incidents for same alert | Query filters by `alertId` — pinpoints the exact record |

---

## Summary — Logic App Blocks (Quick Reference)

```
BLOCK 1  →  Trigger: When HTTP request received (POST)
BLOCK 2  →  Parse JSON (Common Alert Schema)
BLOCK 3  →  Condition: monitorCondition == 'Fired'?
            │
            ├── TRUE (Fired)
            │   BLOCK 4A  →  Initialize Variable: varCPUValue
            │   BLOCK 5A  →  Condition: Determine Severity (P1/P2/P3)
            │                  ├── ≥95%  → Set Variable: P1, urgency=1, impact=1
            │                  ├── 90–95%→ Set Variable: P2, urgency=2, impact=2
            │                  └── 80–90%→ Set Variable: P3, urgency=3, impact=3
            │   BLOCK 6A  →  ServiceNow: Create Record (incident)
            │                  └── Store alertId in u_azure_alert_id
            │
            └── FALSE (Resolved)
                BLOCK 4B  →  Initialize Variable: varAlertId
                BLOCK 5B  →  ServiceNow: List Records
                              └── Query: u_azure_alert_id=varAlertId AND state!=7
                BLOCK 6B  →  Condition: Was Incident Found?
                              └── TRUE:
                                  BLOCK 7B  →  ServiceNow: Update Record
                                                └── state=6, close_notes=auto-close msg
``` 

---


*Logic App: Consumption (Multi-tenant) | Azure Monitor Common Alert Schema | ServiceNow ITSM*
