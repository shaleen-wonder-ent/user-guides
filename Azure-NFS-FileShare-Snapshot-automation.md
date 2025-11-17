# Azure Files NFS Share Snapshot Automation

## Overview

Azure Files doesn't natively support scheduled snapshots for NFS shares (unlike SMB shares which support Azure Backup). This guide provides a robust automation solution using Azure Automation, Azure Functions, or AKS CronJobs to create and manage snapshots for Premium NFS file shares.

## Problem Statement

- **Current Situation**: Using Velero for AKS backups, but it doesn't cover Azure Files NFS shares
- **Azure Backup Limitation**: Recovery Services Vault doesn't support NFS file shares
- **File Share Type**: Premium NFS file shares
- **Requirement**: Automated, scheduled snapshot creation and retention management

---

## Solution Options

### Azure Automation Account with PowerShell (Recommended)


---

## Solution: Azure Automation Account (Recommended)

### Prerequisites

- Azure Storage Account with Premium NFS file shares
- Azure Automation Account
- Managed Identity or Service Principal with appropriate permissions

### Step 1: Create Azure Automation Account

```bash
# Variables
RESOURCE_GROUP="rg-automation"
LOCATION="eastus"
AUTOMATION_ACCOUNT="auto-nfs-snapshots"

# Create resource group (if needed)
az group create --name $RESOURCE_GROUP --location $LOCATION

# Create Automation Account
az automation account create \
  --resource-group $RESOURCE_GROUP \
  --name $AUTOMATION_ACCOUNT \
  --location $LOCATION \
  --sku Basic
```

### Step 2: Enable Managed Identity

#### Using Azure Portal (Easiest)

1. Navigate to your Automation Account in Azure Portal
2. Go to **Account Settings** → **Identity**
3. Under **System assigned** tab, toggle **Status** to **On**
4. Click **Save**
5. Copy the **Object (principal) ID** for the next step

### Step 3: Assign RBAC Permissions

```bash
# Get the Managed Identity Principal ID
PRINCIPAL_ID=$(az automation account show \
  --resource-group $RESOURCE_GROUP \
  --name $AUTOMATION_ACCOUNT \
  --query identity.principalId -o tsv)

# Assign "Storage Account Contributor" role to the storage account
STORAGE_ACCOUNT_ID="/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<STORAGE_RG>/providers/Microsoft.Storage/storageAccounts/<STORAGE_ACCOUNT_NAME>"

az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Account Contributor" \
  --scope $STORAGE_ACCOUNT_ID
```

### Step 4: PowerShell Runbook Script

Create a new runbook in the Automation Account with the following PowerShell script:

```powershell
<#
.SYNOPSIS
    Automated snapshot creation and retention management for Azure Files NFS shares

.DESCRIPTION
    This runbook creates snapshots for specified NFS file shares and manages retention policy

.NOTES
    Author: Azure Automation
    Version: 1.0
#>

param(
    [Parameter(Mandatory=$false)]
    [string]$ResourceGroupName = "rg-storage",
    
    [Parameter(Mandatory=$false)]
    [string]$StorageAccountName = "mystorageaccount",
    
    [Parameter(Mandatory=$false)]
    [string[]]$FileShareNames = @("nfs-share-1", "nfs-share-2"),
    
    [Parameter(Mandatory=$false)]
    [int]$RetentionDays = 30,
    
    [Parameter(Mandatory=$false)]
    [string]$SnapshotPrefix = "auto-snapshot"
)

# Import required modules
Import-Module Az.Accounts
Import-Module Az.Storage

# Connect using Managed Identity
try {
    Write-Output "Connecting to Azure using Managed Identity..."
    Connect-AzAccount -Identity
    Write-Output "Successfully connected to Azure"
}
catch {
    Write-Error "Failed to connect to Azure: $_"
    throw
}

# Get Storage Account Context
try {
    Write-Output "Getting storage account context for: $StorageAccountName"
    $storageAccount = Get-AzStorageAccount -ResourceGroupName $ResourceGroupName -Name $StorageAccountName
    $ctx = $storageAccount.Context
    Write-Output "Successfully retrieved storage account context"
}
catch {
    Write-Error "Failed to get storage account context: $_"
    throw
}

# Function to create snapshot
function New-FileShareSnapshot {
    param(
        [string]$ShareName,
        [Microsoft.Azure.Commands.Common.Authentication.Abstractions.IStorageContext]$Context,
        [string]$Prefix
    )
    
    try {
        $timestamp = Get-Date -Format "yyyyMMdd-HHmmss"
        $snapshotName = "$Prefix-$timestamp"
        
        Write-Output "Creating snapshot for file share: $ShareName"
        
        # Create snapshot
        $share = Get-AzStorageShare -Name $ShareName -Context $Context
        $snapshot = $share.CloudFileShare.Snapshot()
        
        Write-Output "✓ Successfully created snapshot for $ShareName at $($snapshot.SnapshotTime)"
        
        return $snapshot
    }
    catch {
        Write-Error "Failed to create snapshot for ${ShareName}: $_"
        throw
    }
}

# Function to clean up old snapshots
function Remove-OldSnapshots {
    param(
        [string]$ShareName,
        [Microsoft.Azure.Commands.Common.Authentication.Abstractions.IStorageContext]$Context,
        [int]$RetentionDays
    )
    
    try {
        Write-Output "Checking for old snapshots to delete for: $ShareName"
        
        # Get all snapshots for the share
        $snapshots = Get-AzStorageShare -Name $ShareName -Context $Context -SnapshotTime * | 
                     Where-Object { $_.IsSnapshot -eq $true }
        
        $cutoffDate = (Get-Date).AddDays(-$RetentionDays)
        $deletedCount = 0
        
        foreach ($snapshot in $snapshots) {
            if ($snapshot.SnapshotTime -lt $cutoffDate) {
                Write-Output "  Deleting old snapshot from: $($snapshot.SnapshotTime)"
                Remove-AzStorageShare -Share $snapshot.CloudFileShare -Force
                $deletedCount++
            }
        }
        
        Write-Output "✓ Deleted $deletedCount old snapshot(s) for $ShareName"
    }
    catch {
        Write-Error "Failed to clean up old snapshots for ${ShareName}: $_"
        throw
    }
}

# Main execution
Write-Output "=========================================="
Write-Output "NFS File Share Snapshot Automation"
Write-Output "=========================================="
Write-Output "Storage Account: $StorageAccountName"
Write-Output "Resource Group: $ResourceGroupName"
Write-Output "Retention Days: $RetentionDays"
Write-Output "File Shares: $($FileShareNames -join ', ')"
Write-Output "=========================================="

$successCount = 0
$failureCount = 0
$results = @()

foreach ($shareName in $FileShareNames) {
    try {
        Write-Output "`nProcessing file share: $shareName"
        Write-Output "------------------------------------------"
        
        # Verify share exists
        $share = Get-AzStorageShare -Name $shareName -Context $ctx -ErrorAction Stop
        
        # Create snapshot
        $snapshot = New-FileShareSnapshot -ShareName $shareName -Context $ctx -Prefix $SnapshotPrefix
        
        # Clean up old snapshots
        Remove-OldSnapshots -ShareName $shareName -Context $ctx -RetentionDays $RetentionDays
        
        $successCount++
        $results += [PSCustomObject]@{
            ShareName = $shareName
            Status = "Success"
            SnapshotTime = $snapshot.SnapshotTime
            Message = "Snapshot created successfully"
        }
    }
    catch {
        $failureCount++
        $errorMessage = $_.Exception.Message
        Write-Error "Failed to process ${shareName}: $errorMessage"
        
        $results += [PSCustomObject]@{
            ShareName = $shareName
            Status = "Failed"
            SnapshotTime = $null
            Message = $errorMessage
        }
    }
}

# Summary
Write-Output "`n=========================================="
Write-Output "Execution Summary"
Write-Output "=========================================="
Write-Output "Total Shares: $($FileShareNames.Count)"
Write-Output "Successful: $successCount"
Write-Output "Failed: $failureCount"
Write-Output "=========================================="

# Output results
Write-Output "`nDetailed Results:"
$results | Format-Table -AutoSize

if ($failureCount -gt 0) {
    Write-Warning "Some snapshots failed to create. Please review the errors above."
    exit 1
}

Write-Output "`n✓ All snapshots created successfully!"
```

### Step 5: Create Schedule

```bash
# Create a daily schedule at 2 AM UTC
az automation schedule create \
  --resource-group $RESOURCE_GROUP \
  --automation-account-name $AUTOMATION_ACCOUNT \
  --name "DailyNFSSnapshot" \
  --frequency "Day" \
  --interval 1 \
  --start-time "2025-11-18T02:00:00+00:00" \
  --time-zone "UTC" \
  --description "Daily NFS snapshot at 2 AM UTC"
```

### Step 6: Link Runbook to Schedule

```powershell
# In Azure Portal or via PowerShell
# Link the runbook to the schedule with parameters

$params = @{
    ResourceGroupName = "rg-storage"
    StorageAccountName = "mystorageaccount"
    FileShareNames = @("nfs-share-1", "nfs-share-2")
    RetentionDays = 30
    SnapshotPrefix = "auto-snapshot"
}

# Link via Portal: Automation Account > Runbooks > [Your Runbook] > Link to schedule
```
---


## Monitoring and Alerting

### Azure Monitor Alert for Failed Snapshots

```bash
# Create action group for notifications
az monitor action-group create \
  --name "NFS-Snapshot-Alerts" \
  --resource-group $RESOURCE_GROUP \
  --short-name "NFSAlert" \
  --email-receiver name=admin email=admin@example.com

# For Automation Account
az monitor metrics alert create \
  --name "NFS-Snapshot-Failure" \
  --resource-group $RESOURCE_GROUP \
  --scopes "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.Automation/automationAccounts/$AUTOMATION_ACCOUNT" \
  --condition "count JobStreams where Status == 'Failed' > 0" \
  --description "Alert when NFS snapshot job fails" \
  --action "NFS-Snapshot-Alerts"
```

### Log Analytics Query

```kusto
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.AUTOMATION"
| where Category == "JobStreams"
| where RunbookName_s == "NFS-Snapshot-Runbook"
| where StreamType_s == "Error"
| project TimeGenerated, ResultDescription_s
| order by TimeGenerated desc
```

---

## Snapshot Recovery Process

### Restore Files from Snapshot (CLI)

```bash
# List available snapshots
az storage share list \
  --account-name mystorageaccount \
  --include snapshots \
  --query "[?name=='nfs-share-1' && snapshot!=null].[name,snapshot]" \
  --output table

# Mount snapshot in AKS (ReadOnly)
# Add snapshot to PV definition
```

---

## Best Practices

1. **Retention Policy**: Set appropriate retention based on RPO/RTO requirements
2. **Testing**: Regularly test snapshot restoration procedures
3. **Monitoring**: Set up alerts for failed snapshot operations
4. **Documentation**: Keep track of snapshot schedules and retention policies
5. **Cost Management**: Monitor storage costs as snapshots accumulate
6. **Security**: Use Managed Identity instead of connection strings
7. **Tagging**: Tag snapshots with metadata for easy identification

---

## Cost Considerations

- **Snapshot Storage**: Incremental, only changed blocks are stored
- **Typical Cost**: ~$0.05 per GB/month for snapshot differential
- **Retention**: Balance retention period with storage costs
- **Monitoring**: Use Azure Cost Management to track snapshot costs

---

## Troubleshooting

### Common Issues

**Issue**: "Insufficient permissions"
```bash
# Solution: Verify RBAC assignment
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --scope $STORAGE_ACCOUNT_ID
```

**Issue**: "Storage account not found"
```bash
# Solution: Verify storage account exists and name is correct
az storage account show \
  --name mystorageaccount \
  --resource-group rg-storage
```

**Issue**: "Runbook fails to import Az modules"
```powershell
# Solution: Update modules in Automation Account
# Portal: Automation Account > Modules > Update Azure Modules
```

---


## Additional Resources

- [Azure Files NFS Documentation](https://learn.microsoft.com/azure/storage/files/storage-files-how-to-create-nfs-shares)
- [Azure Automation Documentation](https://learn.microsoft.com/azure/automation/)
- [Azure Files Snapshots](https://learn.microsoft.com/azure/storage/files/storage-snapshots-files)
- [AKS Workload Identity](https://learn.microsoft.com/azure/aks/workload-identity-overview)

---

## Support and Maintenance

- Review snapshot logs weekly
- Test restoration procedures monthly
- Update retention policies quarterly
- Audit permissions and access annually

---
