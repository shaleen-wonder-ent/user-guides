# Azure Files NFS Share Snapshot Automation - Complete Guide

## Overview

Azure Files doesn't natively support scheduled snapshots for NFS shares (unlike SMB shares which support Azure Backup). This guide provides a **production-tested** automation solution using Azure Automation Account to create and manage snapshots for Premium NFS file shares.

## Problem Statement

- **Current Situation**: Using Velero for AKS backups, but it doesn't cover Azure Files NFS shares
- **Azure Backup Limitation**: Recovery Services Vault doesn't support NFS file shares
- **File Share Type**: Premium NFS file shares
- **Requirement**: Automated, scheduled snapshot creation and retention management

---

## Prerequisites

- Azure Subscription with appropriate permissions
- Azure Storage Account with Premium NFS file shares (already created)
- Azure CLI 2.x or higher (pre-installed in Automation Accounts)
- PowerShell 7.2+ runtime (available in Automation Account)

---

## Step-by-Step Implementation

### Step 1: Create Azure Automation Account

```bash
#!/bin/bash

# Variables - UPDATE THESE
SUBSCRIPTION_ID="YOUR-SUBSCRIPTION-ID"
RESOURCE_GROUP="rg-automation"
LOCATION="eastus"
AUTOMATION_ACCOUNT="auto-nfs-snapshots"

echo "=========================================="
echo "Creating Azure Automation Account"
echo "=========================================="

# Set subscription
az account set --subscription $SUBSCRIPTION_ID

# Create resource group for automation
echo "Creating resource group..."
az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --output none

# Create Automation Account
echo "Creating Automation Account..."
az automation account create \
  --resource-group $RESOURCE_GROUP \
  --name $AUTOMATION_ACCOUNT \
  --location $LOCATION \
  --sku Basic \
  --output none

echo " Automation Account created: $AUTOMATION_ACCOUNT"
```

---

### Step 2: Enable Managed Identity

**Option A: Using Azure Portal (Recommended)**

1. Navigate to your Automation Account in Azure Portal
2. Go to **Account Settings** → **Identity**
3. Under **System assigned** tab, toggle **Status** to **On**
4. Click **Save**
5. **Copy the Object (principal) ID** - you'll need it for permissions

**Option B: Using Azure REST API via CLI**

```bash
# Variables
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RESOURCE_GROUP="rg-automation"
AUTOMATION_ACCOUNT="auto-nfs-snapshots"

# Enable system-assigned managed identity
echo "Enabling Managed Identity..."
az rest --method PATCH \
  --uri "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Automation/automationAccounts/${AUTOMATION_ACCOUNT}?api-version=2023-11-01" \
  --body '{"identity": {"type": "SystemAssigned"}}' \
  --output none

# Wait for identity propagation
echo "Waiting for identity propagation..."
sleep 15

# Get the Principal ID
PRINCIPAL_ID=$(az rest --method GET \
  --uri "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Automation/automationAccounts/${AUTOMATION_ACCOUNT}?api-version=2023-11-01" \
  --query identity.principalId -o tsv)

echo " Managed Identity enabled"
echo "  Principal ID: $PRINCIPAL_ID"
```

---

### Step 3: Assign Required RBAC Permissions

**IMPORTANT**: The Managed Identity needs TWO roles for full functionality:

#### Required Permissions

| Role | Purpose | Scope |
|------|---------|-------|
| **Storage Account Contributor** | Create/delete snapshots, manage file shares | Storage Account |
| **Storage Account Key Operator Service Role** | Retrieve storage account keys | Storage Account |

#### Assign Permissions Script

```bash
#!/bin/bash

# Variables - UPDATE THESE
SUBSCRIPTION_ID="YOUR-SUBSCRIPTION-ID"
AUTOMATION_RG="rg-automation"
AUTOMATION_ACCOUNT="auto-nfs-snapshots"
STORAGE_RG="rg-storage"
STORAGE_ACCOUNT="mystorageaccount"

echo "=========================================="
echo "Assigning RBAC Permissions"
echo "=========================================="

# Get Principal ID
PRINCIPAL_ID=$(az rest --method GET \
  --uri "https://management.azure.com/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${AUTOMATION_RG}/providers/Microsoft.Automation/automationAccounts/${AUTOMATION_ACCOUNT}?api-version=2023-11-01" \
  --query identity.principalId -o tsv)

echo "Principal ID: $PRINCIPAL_ID"

# Construct storage account resource ID
STORAGE_ACCOUNT_ID="/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${STORAGE_RG}/providers/Microsoft.Storage/storageAccounts/${STORAGE_ACCOUNT}"

# Assign Role 1: Storage Account Contributor
echo "Assigning Storage Account Contributor role..."
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Account Contributor" \
  --scope $STORAGE_ACCOUNT_ID \
  --output none

echo "Storage Account Contributor role assigned"

# Assign Role 2: Storage Account Key Operator Service Role
echo "Assigning Storage Account Key Operator Service Role..."
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Account Key Operator Service Role" \
  --scope $STORAGE_ACCOUNT_ID \
  --output none

echo "Storage Account Key Operator Service Role assigned"

# Verify assignments
echo ""
echo "Verifying role assignments..."
az role assignment list \
  --assignee $PRINCIPAL_ID \
  --scope $STORAGE_ACCOUNT_ID \
  --output table

echo ""
echo "=========================================="
echo " Permissions configured successfully"
echo "=========================================="
```

**Why Both Roles Are Needed:**

- **Storage Account Contributor**: Allows the runbook to create snapshots, list shares, and delete old snapshots
- **Storage Account Key Operator**: Allows retrieving storage keys, which are used by Azure CLI commands (older Azure CLI versions in Automation Accounts don't support `--auth-mode login` for file shares)

---

### Step 4: Create PowerShell Runbook

#### Via Azure Portal:

1. Navigate to **Automation Account** → **Process Automation** → **Runbooks**
2. Click **+ Create a runbook**
   - **Name**: `NFS-Snapshot-Runbook`
   - **Runbook type**: `PowerShell`
   - **Runtime version**: `7.2`
   - **Description**: `Automated NFS file share snapshot creation and retention`
3. Click **Create**

#### Runbook Script (Production-Ready)

```powershell
<#
.SYNOPSIS
    Automated snapshot creation for Azure Files NFS shares

.DESCRIPTION
    Creates snapshots using storage account keys with retention management

.NOTES
    Author: Azure Automation
    Version: 3.4
    Last Updated: 2025-11-17
    Tested: Azure CLI 2.x in Automation Account
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
    Disable-AzContextAutosave -Scope Process | Out-Null
    $connection = Connect-AzAccount -Identity
    Write-Output " Successfully connected to Azure"
    Write-Output "  Subscription: $($connection.Context.Subscription.Name)"
}
catch {
    Write-Error "Failed to connect to Azure: $_"
    throw
}

# Get storage account key
try {
    Write-Output "Retrieving storage account key..."
    $storageKeys = Get-AzStorageAccountKey -ResourceGroupName $ResourceGroupName -Name $StorageAccountName
    $storageKey = $storageKeys[0].Value
    Write-Output " Storage account key retrieved"
}
catch {
    Write-Error "Failed to get storage account key: $_"
    Write-Error "Error details: $($_.Exception.Message)"
    throw
}

# Function to create snapshot
function New-FileShareSnapshotCLI {
    param(
        [string]$AccountName,
        [string]$AccountKey,
        [string]$ShareName,
        [string]$Prefix
    )
    
    try {
        $timestamp = Get-Date -Format "yyyyMMddHHmmss"
        
        Write-Output "Creating snapshot for file share: $ShareName"
        
        # Create snapshot using Azure CLI
        # Note: metadata keys must be alphanumeric only (no hyphens or special chars)
        $output = az storage share snapshot `
            --account-name $AccountName `
            --account-key $AccountKey `
            --name $ShareName `
            --metadata createdby=automation timestamp=$timestamp prefix=$Prefix `
            --output json 2>&1
        
        if ($LASTEXITCODE -ne 0) {
            throw "Azure CLI error: $output"
        }
        
        $result = $output | ConvertFrom-Json
        $snapshotTime = $result.snapshot
        
        Write-Output " Successfully created snapshot for $ShareName"
        Write-Output "  Snapshot Time: $snapshotTime"
        
        return $snapshotTime
    }
    catch {
        Write-Error "Failed to create snapshot for ${ShareName}: $_"
        throw
    }
}

# Function to clean up old snapshots
function Remove-OldSnapshotsCLI {
    param(
        [string]$AccountName,
        [string]$AccountKey,
        [string]$ShareName,
        [int]$RetentionDays
    )
    
    try {
        Write-Output "Checking for old snapshots to delete for: $ShareName"
        
        # List all shares including snapshots
        # Note: use --include-snapshots (not --include snapshots) for older Azure CLI
        $allSharesJson = az storage share list `
            --account-name $AccountName `
            --account-key $AccountKey `
            --include-snapshots `
            --output json 2>&1
        
        if ($LASTEXITCODE -ne 0) {
            throw "Failed to list shares: $allSharesJson"
        }
        
        $allShares = $allSharesJson | ConvertFrom-Json
        
        # Filter snapshots for this specific share
        $snapshots = $allShares | Where-Object { 
            $_.name -eq $ShareName -and $null -ne $_.snapshot 
        }
        
        if ($null -eq $snapshots -or $snapshots.Count -eq 0) {
            Write-Output "  No existing snapshots found for $ShareName"
            return
        }
        
        Write-Output "  Found $($snapshots.Count) existing snapshot(s)"
        
        $cutoffDate = (Get-Date).AddDays(-$RetentionDays)
        $deletedCount = 0
        $keptCount = 0
        
        foreach ($snapshot in $snapshots) {
            try {
                $lastModified = [DateTime]::Parse($snapshot.properties.lastModified)
                $ageInDays = [Math]::Round(((Get-Date) - $lastModified).TotalDays, 1)
                
                if ($lastModified -lt $cutoffDate) {
                    Write-Output "  Deleting old snapshot: $($snapshot.snapshot) (Age: $ageInDays days)"
                    
                    $deleteOutput = az storage share delete `
                        --account-name $AccountName `
                        --account-key $AccountKey `
                        --name $ShareName `
                        --snapshot $snapshot.snapshot `
                        --output none 2>&1
                    
                    if ($LASTEXITCODE -eq 0) {
                        $deletedCount++
                        Write-Output "     Deleted successfully"
                    }
                    else {
                        Write-Warning "    Failed to delete: $deleteOutput"
                    }
                }
                else {
                    Write-Output "  Keeping snapshot: $($snapshot.snapshot) (Age: $ageInDays days)"
                    $keptCount++
                }
            }
            catch {
                Write-Warning "  Error processing snapshot: $_"
            }
        }
        
        Write-Output " Cleanup complete - Deleted: $deletedCount | Kept: $keptCount"
    }
    catch {
        Write-Warning "Failed to clean up old snapshots for ${ShareName}: $_"
        Write-Warning "Continuing with next share..."
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
Write-Output "Current Time (UTC): $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss')"
Write-Output "=========================================="

$successCount = 0
$failureCount = 0
$results = @()

foreach ($shareName in $FileShareNames) {
    try {
        Write-Output "`nProcessing file share: $shareName"
        Write-Output "------------------------------------------"
        
        # Create snapshot
        $snapshotTime = New-FileShareSnapshotCLI `
            -AccountName $StorageAccountName `
            -AccountKey $storageKey `
            -ShareName $shareName `
            -Prefix $SnapshotPrefix
        
        # Clean up old snapshots
        Remove-OldSnapshotsCLI `
            -AccountName $StorageAccountName `
            -AccountKey $storageKey `
            -ShareName $shareName `
            -RetentionDays $RetentionDays
        
        $successCount++
        $results += [PSCustomObject]@{
            ShareName = $shareName
            Status = "Success"
            SnapshotTime = $snapshotTime
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

Write-Output "`n=========================================="
Write-Output " All snapshots created successfully!"
Write-Output "Completed at: $(Get-Date -Format 'yyyy-MM-dd HH:mm:ss UTC')"
Write-Output "=========================================="
```

#### Save and Publish:

1. Paste the script into the runbook editor
2. Click **Save**
3. Click **Publish**
4. Click **Yes** to confirm

---

### Step 5: Test the Runbook

#### Test in Azure Portal:

1. After publishing, click **Test pane**
2. Enter your parameters:
   ```
   ResourceGroupName: publicstoreRG
   StorageAccountName: premfilestorest
   FileShareNames: aznfsfile,nfs-share-2
   RetentionDays: 30
   SnapshotPrefix: auto-snapshot
   ```
3. Click **Start**
4. Monitor the output pane

---

### Step 6: Verify Snapshots Were Created

```bash
# Verify snapshots exist
az storage share list \
  --account-name premfilestorest \
  --include-snapshots \
  --query "[?name=='aznfsfile' && snapshot!=null].[name,snapshot,properties.lastModified]" \
  --output table
```

---

### Step 7: Create Schedule

#### Using Azure Portal (Recommended):

1. Go to **Automation Account** → **Runbooks** → **NFS-Snapshot-Runbook**
2. Click **Schedules** → **Add a schedule**
3. Click **Link a schedule to your runbook**
4. Click **Create a new schedule**:
   - **Name**: `DailyNFSSnapshot-2AM-UTC`
   - **Description**: `Daily NFS snapshot at 2 AM UTC`
   - **Starts**: Tomorrow's date at 02:00
   - **Timezone**: `(UTC) Coordinated Universal Time`
   - **Recurrence**: `Recurring`
   - **Recur every**: `1 Day`
   - **Set expiration**: `No`
5. Click **Create**
6. **Configure parameters** (leave as defaults if you set them correctly in the script):
   ```
   ResourceGroupName: publicstoreRG
   StorageAccountName: premfilestorest
   FileShareNames: aznfsfile
   RetentionDays: 30
   SnapshotPrefix: auto-snapshot
   ```
7. Click **OK**

#### Using Azure CLI:

```bash
# Create schedule
az automation schedule create \
  --resource-group rg-automation \
  --automation-account-name auto-nfs-snapshots \
  --name "DailyNFSSnapshot-2AM-UTC" \
  --frequency Day \
  --interval 1 \
  --start-time "2025-11-18T02:00:00+00:00" \
  --time-zone "UTC" \
  --description "Daily NFS snapshot at 2 AM UTC"

# Note: Linking runbook to schedule must be done via Portal
# Go to: Automation Account > Runbooks > NFS-Snapshot-Runbook > Schedules > Link to schedule
```

---

## Monitoring and Alerts

### View Job History

**Azure Portal:**
1. **Automation Account** → **Process Automation** → **Jobs**
2. Filter by Runbook name: `NFS-Snapshot-Runbook`
3. Click on any job to view detailed output and logs

**Azure CLI:**
```bash
# List recent jobs
az automation job list \
  --resource-group rg-automation \
  --automation-account-name auto-nfs-snapshots \
  --query "[?runbook.name=='NFS-Snapshot-Runbook'].{Status:status,StartTime:startTime,EndTime:endTime}" \
  --output table
```

### Create Alert for Failed Jobs

```bash
# Create action group for notifications
az monitor action-group create \
  --name "NFS-Snapshot-Alerts" \
  --resource-group rg-automation \
  --short-name "NFSAlert" \
  --email-receiver name=admin email=admin@example.com

# Create alert rule (requires Azure Portal for Automation Account alerts)
# Go to: Automation Account > Monitoring > Alerts > New alert rule
```

### Log Analytics Query

If you have Log Analytics configured:

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

## Snapshot Recovery

### List Available Snapshots

```bash
# List all snapshots for a share
az storage share list \
  --account-name premfilestorest \
  --include-snapshots \
  --query "[?name=='aznfsfile' && snapshot!=null].[name,snapshot,properties.lastModified]" \
  --output table
```

#### Option 2: Copy Files from Snapshot

```bash
# Mount the snapshot via a temporary VM or pod
# Then copy needed files back to the main share

# Example using Azure CLI
az storage file copy start \
  --account-name premfilestorest \
  --source-share aznfsfile \
  --source-path "path/to/file.txt" \
  --destination-share aznfsfile \
  --destination-path "restored/file.txt" \
  --source-snapshot "2025-11-17T08:00:05.0000000Z"
```

---

## Best Practices

### 1. Retention Policy

| Retention Period | Use Case | Recommended For |
|------------------|----------|-----------------|
| **7 days** | Development/Testing | Non-critical data |
| **30 days** | Standard Production | Most production workloads |
| **90 days** | Compliance Requirements | Regulated industries |
| **365 days** | Long-term Archival | Critical business data |

### 2. Snapshot Schedule

| Schedule | Frequency | Best For |
|----------|-----------|----------|
| **Daily 2 AM** | Once per day | Standard workloads |
| **Every 6 hours** | 4 times per day | High-change data |
| **Hourly** | 24 times per day | Mission-critical (high cost) |

### 3. Multiple File Shares

```powershell
# In the runbook parameters, add multiple shares:
FileShareNames: share1,share2,share3
```

### 4. Cost Management

- **Snapshots are incremental**: Only changed blocks are stored
- **Monitor costs**: Use Azure Cost Management
- **Typical cost**: ~$0.05 per GB/month for differential data
- **Example**: 100GB share with 10% daily change
  - Daily snapshot cost: ~$0.50/month (10GB × $0.05)
  - 30 days retention: ~$15/month

## Cost Estimation

### Azure Automation Account

| Component | Cost |
|-----------|------|
| **Basic SKU** | Free for first 500 minutes/month |
| **Job runtime** | $0.002 per minute (after free tier) |
| **Typical job duration** | 1-2 minutes |
| **Monthly cost (daily runs)** | ~$0.12 (30 × 2 min × $0.002) |

### Storage Snapshots

| Scenario | Monthly Cost Estimate |
|----------|----------------------|
| **100GB share, 5% daily change, 30-day retention** | ~$7.50 |
| **500GB share, 10% daily change, 30-day retention** | ~$75 |
| **1TB share, 5% daily change, 90-day retention** | ~$225 |

**Total Solution Cost**: $7.50 - $225/month (depending on data size and retention)

---

## Additional Resources

- [Azure Files NFS Documentation](https://learn.microsoft.com/azure/storage/files/storage-files-how-to-create-nfs-shares)
- [Azure Automation Documentation](https://learn.microsoft.com/azure/automation/)
- [Azure Files Snapshots](https://learn.microsoft.com/azure/storage/files/storage-snapshots-files)
- [Azure RBAC Built-in Roles](https://learn.microsoft.com/azure/role-based-access-control/built-in-roles)
- [Azure Storage Pricing](https://azure.microsoft.com/pricing/details/storage/files/)

---



