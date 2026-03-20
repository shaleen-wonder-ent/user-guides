# Azure Service Health Alerts — Configuration Guide

## Table of Contents

1. [Overview](#overview)
2. [Scope & Limitations](#scope--limitations)
3. [Option 1: Configure at Subscription Level (Portal)](#option-1-configure-service-health-alerts-portal--single-subscription)
4. [Option 2: Automate via Azure Policy (DeployIfNotExists)](#option-2-automate-via-azure-policy-recommended)
5. [Option 3: Automate via PowerShell / CLI](#option-3-powershell--cli-automation)
6. [Option 4: Centralize with Log Analytics](#option-4-log-analytics-centralization)
7. [Directory-Level / Multi-Team Considerations](#directory--multi-team-strategy)
8. [Azure Status Page — Subscription Reality](#azure-status-page--reality)
9. [Impact Assessment Summary](#impact-assessment-summary)
10. [Operational Best Practices](#operational-best-practices)

---

## Overview

Azure Service Health provides alerts for:

| Event Type          | Description                 |
| ------------------- | --------------------------- |
| Service Issues      | Active outages              |
| Planned Maintenance | Upcoming maintenance        |
| Health Advisories   | EOL, deprecations, upgrades |
| Security Advisories | Security-related notices    |

**Goal:**
Ensure all subscriptions (current + future) generate alerts and notify relevant Distribution Lists (DLs).

---

## Scope & Limitations

| Scope Level      | Supported        | Notes                                 |
| ---------------- | ---------------- | ------------------------------------- |
| Resource Group   | ✅ (storage only) | Alert is stored here, NOT scoped here |
| Subscription     | ✅                | Actual alert scope                    |
| Management Group | ❌                | Not supported natively                |
| Azure AD Tenant  | ❌                | Not supported                         |

> ⚠️ **Important:** Activity Log Alerts are always **subscription-scoped**, even if stored in a resource group. There is no native way to create a single alert spanning all subscriptions in a management group or directory. The workaround is to **automate deployment** of the same alert configuration to every subscription.

---

## Option 1: Configure Service Health Alerts (Portal – Single Subscription)

This is the quickest way to set up alerts for **a single subscription** and is recommended as a first step / proof of concept.

### Step 1: Create Action Group

1. Azure Portal → Monitor → Alerts → Action Groups → **+ Create**
2. Configure:
   - **Subscription**: Select the target subscription
   - **Resource Group**: Select or create (e.g., `rg-monitoring-alerts`)
   - **Action Group Name**: e.g., `ag-servicehealth-global`
   - **Display Name**: e.g., `SH-Notify`
3. Under **Notifications** tab:
   - **Type**: Email/SMS message/Push/Voice
   - **Name**: e.g., `DL-Ops-Team`
   - **Email**: Enter the Distribution List email (e.g., `ops-team@company.com`)
   - Add additional DLs as needed for different teams
4. Click **Review + create → Create**

> ⚠️ **Impact:** No workload impact. Notification-only resource.

### Step 2: Create Service Health Alert

1. Azure Portal → Service Health → Health alerts → **+ Create service health alert**
2. Configure:
   - **Scope**: Subscription
   - **Services**: All
   - **Regions**: All
   - **Event Types**:
     - ✅ Service Issues
     - ✅ Planned Maintenance
     - ✅ Health Advisories
     - ✅ Security Advisories
3. Under **Actions**: Attach the Action Group created in Step 1
4. Under **Details**:
   - **Alert rule name**: e.g., `alert-servicehealth-global`
   - **Resource group**: Same RG as the Action Group
   - **Enable upon creation**: ✅ Yes (Under Advanced options)
     <img width="661" height="545" alt="image" src="https://github.com/user-attachments/assets/9dcc320a-93c4-495b-a0ae-83d5c303ade2" />
5. Click **Create alert rule**

> ⚠️ **Important:** The portal also offers a **"Quick Alert"** flow where you can enter email addresses inline without creating an Action Group. While this works for testing, it does **not scale** and does not support Distribution Lists well. Always use the **Action Group approach** for production deployments.

> ⚠️ **Impact:** No workload impact. Read-only monitoring rule. It will start sending email notifications to the configured DLs when health events occur.

---

## Option 2: Automate via Azure Policy (Recommended)

This is the **recommended approach** for ensuring all subscriptions (current and future) have Service Health alerts configured. Assign the policy at the **Root Management Group** to cover everything.

### Key Benefits

- Covers all subscriptions automatically
- Auto-deploys alerts for new subscriptions
- Enforces standardization across the organization

### Action Group Strategy

| Approach         | Recommendation                      |
| ---------------- | ----------------------------------- |
| Per-subscription | ✅ Best (simple & reliable)          |
| Centralized      | ⚠️ Requires RBAC + cross-sub access |

> ⚠️ Cross-subscription Action Groups may fail if permissions are not correctly configured. Per-subscription Action Groups are recommended for reliability.

### Step 1: Create the Action Group

Create an Action Group in the target subscription(s) as described in [Option 1, Step 1](#step-1-create-action-group). This Action Group will be referenced by the policy.

### Step 2: Create the Custom Policy Definition

1. Navigate to **Azure Portal → Policy → Definitions → + Policy definition**
2. Fill in the **Basics** tab:

   | Field | Value | Notes |
   |---|---|---|
   | **Definition location** | Click the `...` button → In the **Scope** panel, select the **Management Group** where the policy should be available. Select **Tenant Root Group** to make it available org-wide, or select a specific Management Group (e.g., `Shaleen-MG`) to limit visibility to that group and its children. | ⚠️ The definition location determines **where the policy can be assigned**. If you define it at a child MG, you cannot assign it at a higher scope. For org-wide use, always select **Tenant Root Group**. |
   | **Name** | `Deploy Service Health Alert on Subscriptions` | Required. Use a descriptive, standardized name. |
   | **Description** | `Ensures every subscription has a Service Health alert configured for service issues, planned maintenance, health advisories, and security advisories.` | Optional but recommended for documentation. |
   | **Category** | Select **Create new** → Enter `Monitoring` (or select **Use existing** → pick `Monitoring` if it already exists) | Categories help organize policies in the portal. Use a consistent category across related policies. |

3. Under **Policy rule**:
   - You can either **paste the JSON directly** into the editor, or click **"Import sample policy definition from GitHub"** to start from a template and modify it.
   - Paste the following JSON into the policy rule editor:

```json
{
  "properties": {
    "displayName": "Deploy Service Health Alert on Subscriptions",
    "policyType": "Custom",
    "mode": "All",
    "description": "Ensures Service Health alerts exist for all subscriptions covering service issues, planned maintenance, health advisories, and security advisories.",
    "parameters": {
      "actionGroupResourceId": {
        "type": "String",
        "metadata": {
          "displayName": "Action Group Resource ID",
          "description": "Full resource ID of the Action Group to use for notifications."
        }
      },
      "alertName": {
        "type": "String",
        "defaultValue": "ServiceHealthAlert-Policy",
        "metadata": {
          "displayName": "Alert Rule Name",
          "description": "Name of the Service Health alert rule."
        }
      }
    },
    "policyRule": {
      "if": {
        "field": "type",
        "equals": "Microsoft.Resources/subscriptions"
      },
      "then": {
        "effect": "deployIfNotExists",
        "details": {
          "type": "Microsoft.Insights/activityLogAlerts",
          "existenceScope": "subscription",
          "existenceCondition": {
            "allOf": [
              {
                "field": "Microsoft.Insights/activityLogAlerts/enabled",
                "equals": "true"
              },
              {
                "count": {
                  "field": "Microsoft.Insights/activityLogAlerts/condition.allOf[*]",
                  "where": {
                    "anyOf": [
                      {
                        "allOf": [
                          {
                            "field": "Microsoft.Insights/activityLogAlerts/condition.allOf[*].field",
                            "equals": "category"
                          },
                          {
                            "field": "Microsoft.Insights/activityLogAlerts/condition.allOf[*].equals",
                            "equals": "ServiceHealth"
                          }
                        ]
                      }
                    ]
                  }
                },
                "greaterOrEquals": 1
              }
            ]
          },
          "roleDefinitionIds": [
            "/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"
          ],
          "deployment": {
            "properties": {
              "mode": "incremental",
              "template": {
                "$schema": "https://schema.management.azure.com/schemas/2018-05-01/subscriptionDeploymentTemplate.json#",
                "contentVersion": "1.0.0.0",
                "parameters": {
                  "actionGroupResourceId": { "type": "string" },
                  "alertName": { "type": "string" }
                },
                "resources": [
                  {
                    "type": "Microsoft.Insights/activityLogAlerts",
                    "apiVersion": "2020-10-01",
                    "name": "[parameters('alertName')]",
                    "location": "Global",
                    "properties": {
                      "enabled": true,
                      "scopes": [
                        "[subscription().id]"
                      ],
                      "condition": {
                        "allOf": [
                          {
                            "field": "category",
                            "equals": "ServiceHealth"
                          },
                          {
                            "field": "properties.incidentType",
                            "in": [
                              "Incident",
                              "Maintenance",
                              "Advisory",
                              "Security"
                            ]
                          }
                        ]
                      },
                      "actions": {
                        "actionGroups": [
                          {
                            "actionGroupId": "[parameters('actionGroupResourceId')]"
                          }
                        ]
                      },
                      "description": "Service Health alert deployed via Azure Policy for planned maintenance, service issues, health advisories, and security advisories."
                    }
                  }
                ]
              },
              "parameters": {
                "actionGroupResourceId": {
                  "value": "[parameters('actionGroupResourceId')]"
                },
                "alertName": {
                  "value": "[parameters('alertName')]"
                }
              }
            }
          }
        }
      }
    }
  }
}
```

4. Click **Save**

> ⚠️ **Impact Notes on Definition Location:**
> - Choosing **Tenant Root Group** as the definition location makes the policy available for assignment at **any scope** in your organization. This is the recommended choice for org-wide standards.
> - Choosing a **child Management Group** (e.g., `Shaleen-MG`) restricts the policy to that group and its children only. This is useful if different teams manage their own policies independently.
> - Creating the policy definition itself has **no impact** — it is just a template. Nothing happens until it is **assigned**.

<img width="1181" height="394" alt="image" src="https://github.com/user-attachments/assets/536e42c0-d558-4289-ba8d-fd1dcd33ca9e" />


### Step 3: Assign the Policy

1. Navigate to **Azure Portal → Policy → Assignments → + Assign policy**
2. Fill in the **Basics** tab:

   | Field | Value | Notes |
   |---|---|---|
   | **Scope** | Click the `...` button → Select the **Management Group** or **Subscription** where the policy should apply. For org-wide coverage, select **Tenant Root Group**. | ⚠️ The assignment scope must be at or below the definition location. If you defined the policy at `Shaleen-MG`, you can only assign it at `Shaleen-MG` or its children. |
   | **Policy definition** | Search for and select `Deploy Service Health Alert on Subscriptions` | This is the custom policy created in Step 2. |
   | **Assignment name** | `Assign-ServiceHealthAlert` | Auto-populated from the policy name; can be customized. |
   | **Description** | `Deploys Service Health alerts to all subscriptions under this scope.` | Optional but recommended. |
   | **Policy enforcement** | `Enabled` | Set to `Enabled` for production. Use `Disabled` for dry-run testing. |

3. Fill in the **Parameters** tab:

   | Parameter | Value | Notes |
   |---|---|---|
   | **Action Group Resource ID** | `/subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.Insights/actionGroups/<ag-name>` | Full ARM resource ID of the Action Group created in Step 1. |
   | **Alert Rule Name** | `ServiceHealthAlert-Policy` (default) | Change if you want a custom name. |

4. Fill in the **Remediation** tab:

   | Field | Value | Notes |
   |---|---|---|
   | **Create a remediation task** | ✅ Checked | Required to backfill existing subscriptions that don't have the alert. |
   | **Managed Identity** | System-assigned (auto-created) | The identity needs `Contributor` role to deploy resources. |
   | **Managed Identity location** | Select a region (e.g., `East US`) | Location for the managed identity resource. |

5. Click **Review + create → Create**

> ⚠️ **Impact Notes on Assignment:**
> - **New subscriptions** added under the assigned scope will automatically get the alert on next policy evaluation cycle (typically **within 30 minutes**).
> - **Existing subscriptions** will only get the alert after the **remediation task completes**. Monitor progress under **Policy → Remediation**.
> - The `deployIfNotExists` effect **only creates resources if they don't exist** — it will **NOT modify or delete** existing alert rules or any other resources.
> - A **Managed Identity** with `Contributor` role will be created. This is a **security consideration** — review with your security team.
> - **No impact on existing workloads.** The policy only creates monitoring/alerting resources.

---

## Option 3: PowerShell / CLI Automation

For teams that prefer scripting over Azure Policy. Good for initial rollout but **not suitable for automatic coverage of future subscriptions**.

```powershell
# Variables
$managementGroupName = "<Your-Management-Group-Name>"
$actionGroupId = "/subscriptions/<sub-id>/resourceGroups/<rg-name>/providers/Microsoft.Insights/actionGroups/<ag-name>"
$alertRuleName = "alert-servicehealth-global"
$resourceGroupName = "rg-monitoring-alerts"

# Get all subscriptions under the management group
$subscriptions = Get-AzManagementGroupSubscription -GroupName $managementGroupName

foreach ($sub in $subscriptions) {
    $subId = $sub.Id -replace '/providers/Microsoft.Management/managementGroups/.*/subscriptions/', ''

    Write-Host "Processing subscription: $subId"
    Set-AzContext -SubscriptionId $subId

    # Ensure the resource group exists
    $rg = Get-AzResourceGroup -Name $resourceGroupName -ErrorAction SilentlyContinue
    if (-not $rg) {
        New-AzResourceGroup -Name $resourceGroupName -Location "eastus"
    }

    # Check if alert already exists
    $existingAlert = Get-AzActivityLogAlert -ResourceGroupName $resourceGroupName -Name $alertRuleName -ErrorAction SilentlyContinue

    if (-not $existingAlert) {
        # Create the Service Health alert
        $condition1 = New-AzActivityLogAlertAlertRuleAnyOfOrLeafConditionObject -Field "category" -Equal "ServiceHealth"

        New-AzActivityLogAlert `
            -Name $alertRuleName `
            -ResourceGroupName $resourceGroupName `
            -Location "Global" `
            -Scope "/subscriptions/$subId" `
            -Condition @($condition1) `
            -Action @(@{ actionGroupId = $actionGroupId }) `
            -Enabled $true `
            -Description "Service Health alert for planned maintenance, service issues, health advisories, and security advisories."

        Write-Host "  ✅ Alert created for subscription $subId"
    } else {
        Write-Host "  ⏭️ Alert already exists for subscription $subId"
    }
}
```

> ⚠️ **Impact Notes:**
> - This script **only creates resources** — it does not modify or delete anything existing.
> - Requires `Contributor` or `Monitoring Contributor` role on each subscription.
> - Creates a new Resource Group (`rg-monitoring-alerts`) in each subscription if it doesn't exist.
> - This is a **one-time run** — new subscriptions added later will NOT be covered unless the script is re-run or combined with Azure Policy.
> - **No impact on existing workloads.**

---

## Option 4: Log Analytics Centralization

For large organizations that want a **single pane of glass** for visibility across all subscriptions.

### Setup

1. Create a **central Log Analytics workspace**
2. Forward **Activity Logs** from all subscriptions to this workspace via **Diagnostic Settings**
3. Create **log-based alerts** in Azure Monitor that query Service Health events across all subscriptions

### Sample KQL Query

```kusto
AzureActivity
| where CategoryValue == "ServiceHealth"
| where OperationNameValue has "Microsoft.ServiceHealth"
| project TimeGenerated, SubscriptionId, Properties_d, Caller
| order by TimeGenerated desc
```
*Read more about* [AzureActivity](https://learn.microsoft.com/en-us/azure/azure-monitor/reference/tables/azureactivity) *click*

> ⚠️ **Impact Notes:**
> - Enabling Diagnostic Settings on subscriptions **adds log forwarding** — this has a minor cost impact based on data ingestion volume.
> - Ingestion delays may occur; this should be treated as a **visibility/reporting layer**, not a replacement for direct Service Health alerts.
> - Does **not affect existing workloads** or resource behavior.
> - Requires **Log Analytics workspace pricing** (pay-per-GB).

---

## Directory / Multi-Team Strategy

Since Contoso has **different teams under different directories managing different products/projects**, here is the recommended approach:

| Scenario         | Approach                                       |
| ---------------- | ---------------------------------------------- |
| Multiple teams   | Assign policy per Management Group             |
| Different DLs    | Use parameterized Action Groups per team       |
| Multiple tenants | Configure per tenant; use Lighthouse if needed |

### Recommended Architecture

```
Tenant Root Group
├── MG-Team-A
│   ├── Sub-A1 → Action Group → DL: team-a@company.com
│   └── Sub-A2 → Action Group → DL: team-a@company.com
├── MG-Team-B
│   ├── Sub-B1 → Action Group → DL: team-b@company.com
│   └── Sub-B2 → Action Group → DL: team-b@company.com
└── MG-SharedServices
    └── Sub-Shared → Action Group → DL: ops@company.com
```

- Assign the **same policy definition** at each Management Group level with **different Action Group parameters** pointing to the appropriate team DLs.
- This ensures each team gets alerts only for their subscriptions while maintaining a consistent policy framework.

> ⚠️ **Impact Note:** Assigning policy at multiple scopes with different parameters is fully supported and has **no conflict or impact** on existing workloads.

---

## Azure Status Page — Reality

### The Ask

Subscribe to [https://azure.status.microsoft.com/en-us/status](https://azure.status.microsoft.com/en-us/status) to receive notifications about wider/global Azure outages.

### The Answer

**No, there is no built-in subscription or notification mechanism for the Azure public status page.**

| Feature                         | Available? | Notes                                              |
| ------------------------------- | ---------- | -------------------------------------------------- |
| Email subscription              | ❌ No       | Not offered by Microsoft                           |
| RSS Feed                        | ❌ No       | Microsoft retired public RSS feeds (early 2023)    |
| Direct API                      | ❌ No       | No public API for the status page                  |
| Azure Service Health (portal)   | ✅ Yes      | Covers your subscriptions — **recommended**        |
| Azure Updates RSS               | ✅ Yes      | New features/announcements only, **not** outages   |

### Why?

The Azure public status page is designed for **broad, global outage visibility** and is typically only updated for **widespread, multi-region incidents**. Microsoft's recommended approach for actionable notifications is **Azure Service Health** within the portal, which provides:

- Personalized impact analysis (only your resources/regions)
- Planned maintenance windows
- Health advisories and EOL notices
- Root cause analysis (RCA) documents post-incident

### Workarounds (Unofficial)

1. **Third-party page monitoring tools**: Services like [Distill.io](https://distill.io), [Visualping](https://visualping.io), or [ChangeTower](https://changetower.com) can monitor the status page for changes and send email/Slack/Teams notifications.
2. **Azure Updates RSS Feed**: Subscribe to `https://azure.microsoft.com/en-us/updates/feed/` for feature announcements and deprecations (not real-time outages).

> ⚠️ **Impact Note:** Third-party monitoring tools are **not maintained or supported by Microsoft** and may have false positives. They should be treated as supplementary, not primary.

### Recommendation

- Use **Service Health alerts** as the primary notification mechanism.
- Treat the status page as a **manual reference only** during major incidents.

---

## Impact Assessment Summary

| Action                                    | Impact on Existing Workloads             | Risk Level             | Notes                                                  |
| ----------------------------------------- | ---------------------------------------- | ---------------------- | ------------------------------------------------------ |
| Create Action Group                       | ✅ None                                   | 🟢 Low                 | Notification resource only                             |
| Create Service Health Alert Rule          | ✅ None                                   | 🟢 Low                 | Read-only monitoring rule                              |
| Assign Azure Policy (audit)              | ✅ None                                   | 🟢 Low                 | Only reports compliance — no changes                   |
| Assign Azure Policy (deployIfNotExists)  | ✅ None on workloads; creates new alerts  | 🟡 Medium              | Creates Managed Identity with Contributor role         |
| Run PowerShell script                     | ✅ None on workloads; creates RG + alert  | 🟢 Low                 | Creates new resources only                             |
| Enable Diagnostic Settings (Log Analytics)| ⚠️ Minor cost impact                     | 🟡 Medium              | Adds log ingestion cost; no workload impact            |
| Use third-party status page monitors      | ✅ None                                   | 🟢 Low                 | External tool; not supported by Microsoft              |

---

## Operational Best Practices

### Naming Convention

| Resource       | Example                      |
| -------------- | ---------------------------- |
| Action Group   | `ag-servicehealth-global`    |
| Alert Rule     | `alert-servicehealth-global` |
| Resource Group | `rg-monitoring-alerts`       |

### Prevent Duplicate Alerts

- Standardize alert names across all subscriptions
- Avoid mixing manual portal deployments with policy-driven deployments
- Use the policy existence condition to detect existing alerts before deploying

### RBAC Requirements

| Component         | Minimum Role Required                   |
| ----------------- | --------------------------------------- |
| Policy Assignment | Contributor (at Management Group scope) |
| Alert Creation    | Monitoring Contributor                  |
| Action Group      | Contributor                             |

### Alert Latency

> ⚠️ Service Health alerts are **not real-time**. Typical delay: **5–15 minutes** from the time Azure detects an issue to when the notification is delivered.

---

## Quick Start: Configure for One Subscription

For a quick proof of concept on a single subscription:

1. **Create an Action Group** → Add team DL emails
2. **Create a Service Health Alert** → Scope to subscription, all event types
3. **Verify** → Go to **Service Health → Health alerts** and confirm the rule is active
4. **Test** → Action Groups have a "Test" feature under **Monitor → Alerts → Action groups → Select group → Test**

**Estimated time:** ~10 minutes per subscription (portal) or ~2 minutes per subscription (scripted).

---

## Conclusion

- Native Management Group–level alerts are **not supported**
- Best practice = **Azure Policy–driven deployment at scale**
- This approach ensures:
  - ✅ Coverage for all current and future subscriptions
  - ✅ Consistency across the organization
  - ✅ Future-proofing with automatic deployment
- **No impact to running workloads** — all actions create monitoring resources only

---

## References

- [Azure Service Health Overview](https://learn.microsoft.com/en-us/azure/service-health/overview)
- [Create Service Health Alerts](https://learn.microsoft.com/en-us/azure/service-health/alerts-activity-log-service-notifications-portal)
- [ARM Template for Service Health Alerts](https://learn.microsoft.com/en-us/azure/service-health/alerts-activity-log-service-notifications-arm)
- [Azure Policy DeployIfNotExists](https://learn.microsoft.com/en-us/azure/governance/policy/concepts/effects#deployifnotexists)
- [Azure Monitor Activity Log Alerts](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/activity-log-alerts)
- [Azure Status Page](https://azure.status.microsoft/en-us/status)
