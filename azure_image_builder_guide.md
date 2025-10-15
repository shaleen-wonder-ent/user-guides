# Azure Image Builder: Hardened VM Images Guide

This guide provides a step-by-step approach to create hardened VM images in Azure using Azure Image Builder. The process allows you to download marketplace images, apply security hardening measures, a[...]

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Environment Setup](#environment-setup)
3. [Role Configuration](#role-configuration)
4. [Gallery & Storage Setup](#gallery--storage-setup)
5. [Hardening Script Creation](#hardening-script-creation)
6. [Template & Deployment](#template--deployment)
7. [Monitoring & Next Steps](#monitoring--next-steps)
8. [Restrict Access to Marketplace Images](#restrict-access-to-marketplace-images)
9. [Automating the Process](#automating-the-process)
10. [Best Practices](#best-practices)
11. [Sharing Hardened Images Across Tenants](#Sharing-Hardened-Images-Across-Tenants)
    
 

## Prerequisites

- Azure subscription with owner/contributor access
- Azure Cloud Shell (Bash) or local environment with Azure CLI installed
- Sufficient permissions to create resources and assign roles

## Environment Setup

This step establishes the environment for Azure Image Builder by defining variables, registering required providers, creating a resource group, and setting up a managed identity.

```bash
#!/bin/bash

# ------------------------------------------------------
# Azure Image Builder - Environment Setup
# ------------------------------------------------------

# --- Define variables ---
# These variables are used throughout the scripts to ensure consistency and make modifications easier.

# Get the current Azure subscription ID and store it in a variable.
subscriptionID=$(az account show --query id -o tsv)
# Define the name for the resource group that will hold all our resources.
resourceGroupName="ImageBuilderRG"
# Specify the Azure region where the resources will be created.
location="eastus"
# Name for the Shared Image Gallery where hardened images will be stored.
imageGalleryName="HardenedImagesGallery"
# Name for the User-Assigned Managed Identity that Azure Image Builder will use.
identityName="AIBIdentity"
# Name for the image definition within the gallery.
imageDefName="Win2019-Hardened"
# Name for the Azure Image Builder template resource.
imageTemplateName="Win2019HardenedTemplate"
# A friendly name for the custom role we will create.
roleDefinitionName="Azure Image Builder Service Role"

# --- Initial Setup ---
echo "Current date: $(date -u +"%Y-%m-%d %H:%M:%S")"
echo "Setting up Azure Image Builder..."

# Register required resource providers. This is a one-time operation per subscription.
# It ensures that your subscription is enabled to use these Azure services.
echo "Registering resource providers..."
# Provider for Azure Image Builder itself.
az provider register --namespace Microsoft.VirtualMachineImages
# Provider for Azure Storage, used for storing scripts.
az provider register --namespace Microsoft.Storage
# Provider for Azure Compute, used for VMs, galleries, and images.
az provider register --namespace Microsoft.Compute
# Provider for Azure Key Vault, often used for storing secrets securely.
az provider register --namespace Microsoft.KeyVault

# Create a resource group if it doesn't already exist.
# This provides a logical container for all the resources related to this image build process.
if [ $(az group exists --name $resourceGroupName) = false ]; then
    echo "Creating resource group $resourceGroupName..."
    az group create --name $resourceGroupName --location $location
fi

# Create a user-assigned managed identity.
# AIB uses this identity to interact with other Azure resources on your behalf (e.g., write an image to the gallery).
echo "Creating managed identity $identityName..."
az identity create --resource-group $resourceGroupName --name $identityName

# Get the full resource ID of the managed identity.
# This is needed for the AIB template to reference the identity.
identityNameResourceId=$(az identity show --resource-group $resourceGroupName --name $identityName --query id -o tsv)
# Get the principal ID (a unique identifier) of the managed identity.
# This is needed to assign permissions (roles) to the identity.
identityNamePrincipalId=$(az identity show --resource-group $resourceGroupName --name $identityName --query principalId -o tsv)

# Output the retrieved IDs for verification.
echo "Identity Resource ID: $identityNameResourceId"
echo "Identity Principal ID: $identityNamePrincipalId"
```

## Role Configuration

This step creates a custom role with necessary permissions and assigns it to the managed identity, enabling Azure Image Builder to access required resources.

```bash
#!/bin/bash

# ------------------------------------------------------
# Azure Image Builder - Role Configuration
# ------------------------------------------------------

# Create a JSON file that defines a custom role.
# This role will contain the minimum permissions required for the Image Builder service to function.
echo "Creating custom role definition..."
cat > aib-role.json <<EOF
{
    "Name": "$roleDefinitionName",
    "IsCustom": true,
    "Description": "Image Builder access to create resources for the image build",
    "Actions": [
        "Microsoft.Compute/galleries/read",
        "Microsoft.Compute/galleries/images/read",
        "Microsoft.Compute/galleries/images/versions/read",
        "Microsoft.Compute/galleries/images/versions/write",
        "Microsoft.Compute/images/write",
        "Microsoft.Compute/images/read",
        "Microsoft.Compute/images/delete"
    ],
    "NotActions": [],
    "AssignableScopes": [
        "/subscriptions/$subscriptionID/resourceGroups/$resourceGroupName"
    ]
}
EOF

# Create the custom role in Azure Active Directory using the JSON file.
# The command outputs the unique name (GUID) of the role.
roleId=$(az role definition create --role-definition aib-role.json --query name -o tsv)
echo "Role created with ID: $roleId"

# Assign the newly created custom role to the managed identity.
# This grants the identity the permissions defined in the role, scoped to our resource group.
echo "Assigning role to identity..."
az role assignment create \
    --assignee $identityNamePrincipalId \
    --role "$roleDefinitionName" \
    --scope "/subscriptions/$subscriptionID/resourceGroups/$resourceGroupName"
```

## Gallery & Storage Setup

This step creates a Shared Image Gallery to store your hardened images and sets up a storage account to host hardening scripts.

```bash
#!/bin/bash

# ------------------------------------------------------
# Azure Image Builder - Gallery & Storage Setup
# ------------------------------------------------------

# Create a Shared Image Gallery (SIG).
# A SIG is a repository for managing and sharing VM images across your organization.
echo "Creating Shared Image Gallery..."
az sig create \
    --resource-group $resourceGroupName \
    --gallery-name $imageGalleryName \
    --location $location

# Create an Image Definition within the gallery.
# This acts as a blueprint for image versions, defining properties like OS, publisher, and SKU.
echo "Creating Image Definition..."
az sig image-definition create \
    --resource-group $resourceGroupName \
    --gallery-name $imageGalleryName \
    --gallery-image-definition $imageDefName \
    --publisher "YourCompany" \
    --offer "WindowsServer" \
    --sku "2019-Hardened" \
    --os-type Windows \
    --os-state generalized \
    --location $location

# Define a unique name for the storage account using the RANDOM variable.
storageAccountName="hardeningscripts$RANDOM"
# Define the name of the container within the storage account.
container="scripts"

# Create a storage account.
# This account will be used to store the hardening scripts that AIB will download and execute.
echo "Creating storage account for hardening scripts..."
az storage account create \
    --name $storageAccountName \
    --resource-group $resourceGroupName \
    --location $location \
    --sku Standard_LRS

# Retrieve one of the keys for the newly created storage account.
# The key is required to authorize operations like creating a container or uploading a blob.
storageAccountKey=$(az storage account keys list \
    --account-name $storageAccountName \
    --resource-group $resourceGroupName \
    --query "[0].value" -o tsv)

# Create a blob container within the storage account.
# This container will hold our hardening script. Public access is set to 'blob' so AIB can access it.
az storage container create \
    --name $container \
    --account-name $storageAccountName \
    --account-key $storageAccountKey \
    --public-access blob
```

## Hardening Script Creation

This step creates a PowerShell script that implements CIS benchmarks and security hardening measures for Windows Server.

```bash
#!/bin/bash

# ------------------------------------------------------
# Azure Image Builder - Hardening Script Creation
# ------------------------------------------------------

# Create a PowerShell script file named 'apply-cis-benchmarks.ps1'.
# This script contains commands to harden the Windows Server image.
echo "Creating CIS benchmark hardening script..."
cat > apply-cis-benchmarks.ps1 <<EOF
# --- CIS Benchmark Hardening Example ---
# This script applies various security settings to the image.

# Enforce strong password policies using the local security policy tool.
secedit /export /cfg C:\secpol.cfg
(gc C:\secpol.cfg).replace("PasswordComplexity = 0", "PasswordComplexity = 1") | Out-File C:\secpol.cfg
(gc C:\secpol.cfg).replace("MinimumPasswordLength = 0", "MinimumPasswordLength = 14") | Out-File C:\secpol.cfg
secedit /configure /db C:\Windows\security\local.sdb /cfg C:\secpol.cfg /areas SECURITYPOLICY

# Ensure the Windows Firewall is enabled for all network profiles.
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True

# Disable the insecure SMBv1 protocol.
Disable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol -NoRestart

# Configure advanced audit policies to log important security events.
auditpol /set /subcategory:"Security System Extension" /success:enable /failure:enable
auditpol /set /subcategory:"System Integrity" /success:enable /failure:enable
auditpol /set /subcategory:"Logon" /success:enable /failure:enable

# --- Registry Hardening ---
# These changes modify the Windows Registry to apply additional security controls.

# Prevent anonymous users from enumerating shares.
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Lsa" /v RestrictAnonymous /t REG_DWORD /d 1 /f
# Disable remote UAC for local accounts to prevent elevation of privilege attacks.
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v LocalAccountTokenFilterPolicy /t REG_DWORD /d 0 /f
EOF

echo "Hardening script created: apply-cis-benchmarks.ps1"
```

## Template & Deployment

This step uploads the hardening script to storage, creates an Image Builder template, and initiates the build process.

```bash
#!/bin/bash

# ------------------------------------------------------
# Azure Image Builder - Template & Deployment
# ------------------------------------------------------

# Upload the PowerShell hardening script to the Azure Storage blob container.
echo "Uploading hardening script to storage..."
az storage blob upload \
    --container-name $container \
    --file apply-cis-benchmarks.ps1 \
    --name apply-cis-benchmarks.ps1 \
    --account-name $storageAccountName \
    --account-key $storageAccountKey

# Construct the public URL to the uploaded script.
# AIB will use this URL to download the script during the build process.
scriptUrl="https://$storageAccountName.blob.core.windows.net/$container/apply-cis-benchmarks.ps1"
echo "Script URL: $scriptUrl"

# Create the Azure Image Builder template file (template.json).
# This ARM template defines the entire image build process from source to distribution.
echo "Creating Image Builder template..."
cat > template.json <<EOF
{
    "\$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {},
    "variables": {
        "identityName": "$identityName",
        "identityId": "$identityNameResourceId"
    },
    "resources": [
        {
            "name": "$imageTemplateName",
            "type": "Microsoft.VirtualMachineImages/imageTemplates",
            "apiVersion": "2020-02-14",
            "location": "$location",
            "dependsOn": [],
            "identity": {
                "type": "UserAssigned",
                "userAssignedIdentities": {
                    "[variables('identityId')]": {}
                }
            },
            "properties": {
                "buildTimeoutInMinutes": 120,
                "vmProfile": {
                    "vmSize": "Standard_D2_v3",
                    "osDiskSizeGB": 127
                },
                "source": {
                    "type": "PlatformImage",
                    "publisher": "MicrosoftWindowsServer",
                    "offer": "WindowsServer",
                    "sku": "2019-Datacenter",
                    "version": "latest"
                },
                "customize": [
                    {
                        "type": "PowerShell",
                        "name": "ApplyCISBenchmarks",
                        "runElevated": true,
                        "scriptUri": "$scriptUrl"
                    },
                    {
                        "type": "WindowsUpdate",
                        "searchCriteria": "IsInstalled=0",
                        "filters": [
                            "exclude:\$_.Title -like '*Preview*'",
                            "include:\$true"
                        ],
                        "updateLimit": 40
                    },
                    {
                        "type": "PowerShell",
                        "name": "DisableServices",
                        "runElevated": true,
                        "inline": [
                            "Set-Service -Name XblAuthManager -StartupType Disabled",
                            "Set-Service -Name XblGameSave -StartupType Disabled",
                            "Set-Service -Name XboxGipSvc -StartupType Disabled",
                            "Set-Service -Name XboxNetApiSvc -StartupType Disabled"
                        ]
                    }
                ],
                "distribute": [
                    {
                        "type": "SharedImage",
                        "galleryImageId": "/subscriptions/$subscriptionID/resourceGroups/$resourceGroupName/providers/Microsoft.Compute/galleries/$imageGalleryName/images/$imageDefName",
                        "runOutputName": "win2019Hardened",
                        "replicationRegions": [
                            "$location"
                        ]
                    }
                ]
            }
        }
    ]
}
EOF

# Deploy the Image Builder template using the JSON file.
# This creates the Image Template resource in Azure.
echo "Deploying Image Builder template..."
az deployment group create \
    --resource-group $resourceGroupName \
    --template-file template.json

# Start the image build process by invoking the 'Run' action on the template resource.
# Azure Image Builder will now provision resources, run customizations, and distribute the image.
echo "Starting image build process..."
az resource invoke-action \
    --resource-group $resourceGroupName \
    --resource-type Microsoft.VirtualMachineImages/imageTemplates \
    --name $imageTemplateName \
    --action Run

# Provide the command to check the status of the build.
echo "Image build process initiated. Check status with:"
echo "az image builder show --name $imageTemplateName --resource-group $resourceGroupName --query lastRunStatus"
```

## Monitoring & Next Steps

### Monitoring the Build Process

Check the status of your image build:

```bash
# This command queries the AIB template resource for the status of the last run.
# The status will change from 'Running' to 'Succeeded' or 'Failed'.
az image builder show --name $imageTemplateName --resource-group $resourceGroupName --query lastRunStatus -o tsv
```

### Enforcing Usage of Hardened Images

After creating hardened images, you can enforce their usage through Azure Policy:

1. Create an Azure Policy that denies creation of VMs from public marketplace images
2. Allow only specific image sources (your hardened images)
3. Assign the policy at the appropriate management group or subscription level

## Restrict Access to Marketplace Images

Implement these controls to prevent use of non-hardened images:

### Azure Policy

Create policies to:
- Deny creation of VMs from public marketplace images
- Allow only specific image sources (your hardened images)

#### Example Policy Definition (JSON)

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Compute/virtualMachines"
        },
        {
          "field": "Microsoft.Compute/imageId",
          "exists": "true"
        },
        {
          "field": "Microsoft.Compute/imageId",
          "notContains": "/resourceGroups/ImageBuilderRG/providers/Microsoft.Compute/galleries/HardenedImagesGallery"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

#### Creating the Policy with Azure CLI

```bash
# Create a policy definition in Azure from a JSON rule file.
# This makes the policy available for assignment within your tenant.
az policy definition create \
    --name "require-hardened-vm-images" \
    --display-name "Require use of hardened VM images" \
    --description "This policy ensures VMs are only created from approved hardened images" \
    --rules "policy-rules.json" \
    --mode All

# Assign the policy to a specific scope, in this case, the entire subscription.
# Any attempt to create a VM that violates the policy rules will now be denied.
az policy assignment create \
    --name "require-hardened-vm-images" \
    --display-name "Require use of hardened VM images" \
    --policy "require-hardened-vm-images" \
    --scope "/subscriptions/$subscriptionID"
```

### Azure Marketplace Private Store

Configure a private Azure Marketplace that only includes your hardened images:

1. Navigate to Azure Portal > Marketplace > Private marketplace
2. Configure private marketplace settings
3. Add only your hardened images to the allowed list
4. Enable the private marketplace for your organization

This ensures users can only deploy VMs from your pre-approved, hardened image catalog.


### Regular Maintenance

Set up a recurring process to:

1. Update your hardening scripts with the latest security best practices
2. Rebuild images on a regular schedule (monthly/quarterly)
3. Incorporate new security patches and updates

## Automating the Process

For regular updates and continuous compliance:

1. **Store hardening scripts in a source control repository**
   - Use Azure DevOps Repos or GitHub to maintain scripts
   - Implement branch protection and pull request workflows for changes
   - Track changes with commit history

2. **Use Azure DevOps pipelines or GitHub Actions to:**
   - Update hardening scripts as needed
   - Trigger new image builds on a schedule or when scripts change
   - Validate images with testing
   - Distribute to Shared Image Gallery

### Sample Azure DevOps Pipeline YAML

```yaml
# Trigger this pipeline on commits to the main branch affecting scripts or templates.
trigger:
  branches:
    include:
    - main
  paths:
    include:
    - 'hardening-scripts/*'
    - 'image-templates/*'

# Schedule the pipeline to run automatically on a cron schedule (e.g., monthly).
schedules:
- cron: "0 0 1 * *"  # Run at midnight on the 1st of every month.
  displayName: Monthly Image Update
  branches:
    include:
    - main
  always: true # Ensures the pipeline runs even if there are no new code changes.

pool:
  vmImage: ubuntu-latest # Use a Microsoft-hosted agent.

steps:
- task: AzureCLI@2 # Use the Azure CLI task.
  inputs:
    azureSubscription: 'Your-Azure-Connection' # Your Azure DevOps service connection name.
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # This script will run in the pipeline's context.
      # Make the main build script executable.
      chmod +x ./scripts/build-hardened-image.sh
      # Execute the script to build the hardened image.
      ./scripts/build-hardened-image.sh
```

## Best Practices

1. **Version control your templates and scripts**
   - Maintain history of hardening changes
   - Use semantic versioning for image versions
   - Document changes in commit messages

2. **Test images thoroughly**
   - Deploy test VMs from your images to verify functionality
   - Implement automated tests for key functionality
   - Validate security settings with compliance scanning tools

3. **Document hardening measures**
   - Keep an inventory of security controls applied
   - Map controls to compliance frameworks (CIS, NIST, etc.)
   - Maintain documentation of intentional deviations

4. **Set up a regular rebuild schedule**
   - Ensure images incorporate latest security updates
   - Align with patch management cycle
   - Implement automated testing post-build

5. **Use parameterized templates**
   - Make templates reusable across different OS types
   - Enable customization for different environments
   - Simplify maintenance with DRY (Don't Repeat Yourself) principles

---

## Azure Hardened Images

# Azure Hardened Images – Storage, Sharing, and Multi-Tenant Flow

## Azure Compute Gallery — Image Storage & Versioning Limits

Azure Compute Gallery (ACG) is where your hardened images are stored, versioned, and shared. It allows you to maintain consistent, compliant images across environments.

| Resource Type | Default Limit | Can Be Increased | Notes |
|----------------|---------------|------------------|--------|
| Image definitions per gallery | 1,000 | ✅ Yes | Each represents a unique OS or configuration type |
| Versions per image definition | 1,000 | ✅ Yes | Allows image versioning for patch cycles |
| Target regions per version | 100 | ✅ Yes | Replicate globally for performance and resilience |
| Replicas per region | 50 | ✅ Yes | Increases availability and deployment speed |
| Total galleries per subscription | 1,000 | ✅ Yes | Supports large-scale multi-image architectures |



---

## Sharing Hardened Images Across Tenants

Azure Compute Gallery supports **cross-tenant image sharing**, allowing centralized management of hardened base images while securely sharing them across multiple Azure AD tenants.

### ✅ You Can Share:
- Entire **Gallery**
- Individual **Image Definitions**
- Individual **Image Versions**

### 🔐 Sharing Mechanisms

You can share images using **Azure CLI** or **Azure Portal**.

#### Example (CLI):
```bash
az sig share update   --gallery-name MyHardenedGallery   --resource-group RG-Hardening   --permissions groups   --target-tenants 11111111-aaaa-bbbb-cccc-222222222222 33333333-aaaa-bbbb-cccc-444444444444
```

Or through the Portal:
> **Azure Compute Gallery → Sharing → Direct sharing → Add tenant IDs**

### 🧭 Types of Sharing

| Mode | Description | Recommended For |
|-------|--------------|-----------------|
| **Private (Default)** | Accessible only within your tenant using RBAC | Internal organizational use |
| **Direct Sharing** | Explicitly share with target tenant IDs | ✅ Best for multi-tenant MSP/SaaS use cases |
| **Community Gallery** | Publicly available to all Azure users | ❌ Not for private hardened images |

### 💡 Access Notes
- Receiving tenants **don’t need to copy** the image — they can directly use it for VM creation.
- Access can be **revoked at any time**.
- For better performance, replicate your images to regions near each tenant.

---

## Example: Multi-Tenant Sharing Flow

### Architecture Overview

```
Your Tenant (A)
│
├── Azure Image Builder → Hardened Images (10)
│
├── Azure Compute Gallery (MyHardenedGallery)
│       ├── Image Def: HardenedUbuntu2204
│       ├── Image Def: HardenedWin2022
│       └── ...
│
└── Shared with:
    ├── Tenant B (Customer1)
    ├── Tenant C (Customer2)
    ├── Tenant D (Customer3)
    └── ... up to 25 tenants
```

### Example: Tenant Deployment Command
Each tenant can create a VM using your shared hardened image:

```bash
az vm create   --resource-group CustomerRG   --name MySecureVM   --image "/subscriptions/<your-sub>/resourceGroups/RG-Hardening/providers/Microsoft.Compute/galleries/MyHardenedGallery/images/HardenedUbuntu2204/versions/1.0.0"   --admin-username azureuser
```

### Key Benefits
- Centralized image lifecycle and version control  
- Consistent security baseline across tenants  
- Reduced risk of unapproved marketplace image usage  
- Easy revocation or version update management  

---

## Conclusion

This automated approach allows you to:

1. Take marketplace images and apply security hardening
2. Store them in a controlled location (Shared Image Gallery)
3. Ensure your organization uses only properly hardened VM images
4. Maintain security compliance across your Azure environment

By using Azure Image Builder, you've established a systematic, repeatable process for creating and maintaining hardened VM images.
