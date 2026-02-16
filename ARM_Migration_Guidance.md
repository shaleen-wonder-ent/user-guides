# Azure Resource Manager (ARM) Migration Guidance

## Executive Summary

-   Azure Service Management (Classic / ASM) is retired and no longer
    supported for production workloads.
-   Any remaining Classic resources introduce operational, security, and
    compliance risk.
-   Migration to Azure Resource Manager (ARM) ensures governance,
    automation, security, and future compatibility.


------------------------------------------------------------------------

## 1. Understanding ARM vs. Classic (ASM)

Azure originally used the **Azure Service Management (ASM)** model,
commonly called **Classic**.\
**Azure Resource Manager (ARM)** is the modern deployment and management
layer for Azure resources.

All new Azure features, tooling, and security capabilities are built
exclusively for ARM.

```
  ----------------------------------------------------------------------------------------
    Feature              Classic (ASM)                      ARM
  -------------------- -------------------------- ----------------------------------------
  Deployment Model     Cloud Services                Resource Groups

  Access Control       Co-admin / Service Admin      Role-Based Access Control (RBAC)                                        

  Templates            Not supported                 ARM Templates / Bicep / Terraform

  Tagging              Limited                       Full support

  Policy & Governance  Not available                 Azure Policy & Blueprints

  Support Status       Retired                       Fully supported
  ----------------------------------------------------------------------------------------

```
------------------------------------------------------------------------

## 2. How to Check if Your Resources Are on ARM or Classic

### 2.1 Virtual Machines
```
  -----------------------------------------------------------------------------------------------------------------
  Method                                                Steps
  ----------------------------------------------------- -----------------------------------------------------------
  Azure Portal                                          Navigate to **Virtual Machines**.
                                                        If listed under "Virtual machines (classic)", it is ASM.

  Azure CLI                                             az vm list --output table -> lists ARM VMs.

  PowerShell                                            Get-AzVM -> lists ARM VMs.
                                                        Get-AzureVM -> lists Classic VMs.

  Azure Resource Graph                                  Use the queries below.
  -----------------------------------------------------------------------------------------------------------------
```
ARM VMs:

``` kusto
Resources
| where type == "microsoft.compute/virtualmachines"
| project name, resourceGroup, location
```

Classic VMs:

``` kusto
Resources
| where type == "microsoft.classiccompute/virtualmachines"
| project name, resourceGroup, location
```

### 2.2 Storage Accounts

ARM Storage Accounts:

``` kusto
Resources
| where type == "microsoft.storage/storageaccounts"
| project name, resourceGroup, location, kind
```

Classic Storage Accounts:

``` kusto
Resources
| where type == "microsoft.classicstorage/storageaccounts"
| project name, resourceGroup, location
```

------------------------------------------------------------------------

## 3. Migration Path --- Classic (ASM) to ARM

Azure provides a platform-supported migration path to move Classic
resources to ARM with minimal downtime.

High-Level Flow:

VALIDATE → PREPARE → COMMIT (or ABORT)

-   Validate: Check for unsupported configurations
-   Prepare: ARM resources are staged
-   Commit: Finalize migration (irreversible)
-   Abort: Rollback before commit

> Use Azure PowerShell or the Azure Portal migration workflow depending on subscription configuration.

------------------------------------------------------------------------

## 4. Prerequisites & Pre-Migration Checklist

-   Inventory all Classic resources (VMs, Storage, VNets, NSGs, Reserved IPs)
-   Review unsupported configurations
-   Back up all VMs and data
-   Remove deprecated extensions
-   Ensure ARM quotas are sufficient
-   Update automation scripts referencing Classic APIs
-   Align migration with organization's Azure Landing Zone standards
-   Schedule approved maintenance window

------------------------------------------------------------------------

## 5. Benefits of ARM Migration

-   Unified Management via Resource Groups and Management Groups
-   Role-Based Access Control (RBAC)
-   Azure Policy & Governance controls
-   Infrastructure as Code (ARM / Bicep / Terraform)
-   Cost Management with Tagging
-   Security integration (Defender for Cloud, Azure Advisor)
-   Modern networking features
-   Continued platform support and feature access

------------------------------------------------------------------------

## 6. Potential Impact & Risks
```
  -------------------------------------------------------------------------------------------------
  Area                           Impact                          Mitigation
  -----------------------       ----------------------------   ------------------------------------
  Downtime                      Brief reboot during commit      Schedule maintenance window                                   

  IP Address Changes            Public IP may change            Reserve static IPs in advance
                                  
  Automation Breakage           Classic scripts will fail       Refactor to Az PowerShell / az CLI                             

  Monitoring                    Alerts do not migrate           Recreate using Azure Monitor

  DNS & Certificates            Cloud Service DNS may change    Plan DNS and certificate updates
                       
  ----------------------------------------------------------------------------------------------------
```
------------------------------------------------------------------------


## Key Takeaway

Azure Service Management (Classic) is retired and no longer supported
for production workloads.\
Any remaining Classic resources must be migrated immediately to avoid
operational, security, and compliance risks.

------------------------------------------------------------------------


