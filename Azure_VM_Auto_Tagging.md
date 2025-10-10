# Definitive Guide: Automatically Tag Azure VMs with Creator ID

This guide details how to build a robust, event-driven solution using an Azure Automation Account to automatically tag newly created Virtual Machines with the creator's identity (`CreatorId`) and the [...]

**Solution Components:**

1.  **Automation Account**: A service to host and run our PowerShell script (Runbook). It will have a managed identity to securely interact with other Azure resources.
2.  **PowerShell Runbook**: The script that contains the logic to receive event data and apply tags to the VM.
3.  **Webhook**: A unique, secure URL that acts as a trigger for our Runbook.
4.  **Event Grid Subscription**: A service that watches for specific events (like VM creation) in your Azure subscription and sends a notification to the webhook.

---

## Part 1: Create and Configure the Automation Account

### Step 1.1: Create the Automation Account

1.  Log in to the [Azure Portal](https://portal.azure.com).
2.  In the top search bar, type **"Automation Accounts"** and select it from the services list.
3.  Click the **"+ Create"** button.
4.  Fill in the **Basics** tab with the following details:
    *   **Subscription**: Select your target Azure subscription.
    *   **Resource group**: Click "Create new" and name it **`vm-tagger-automation-rg`**.
    *   **Name**: **`vm-tagger-automation-account`**
    *   **Region**: Select a region, for example, **`East US`**.
5.  Click the **"Advanced"** tab.
6.  Under **Managed identities**, ensure that **System assigned** is set to **On**. This is critical for authentication.
7.  Click **"Review + create"**, and after validation passes, click **"Create"**.

### Step 1.2: Assign Permissions to the Automation Account

The Automation Account needs permission to modify resources (i.e., add tags) across your subscription.

1.  Wait for the deployment to complete, then go to the `vm-tagger-automation-account` resource.
2.  In the left-hand menu, under "Account Settings", click on **"Identity"**.
3.  On the "System assigned" tab, click the **"Azure role assignments"** button.
4.  Click **"+ Add role assignment (Preview)"**.
5.  Configure the role assignment:
    *   **Scope**: `Subscription`
    *   **Subscription**: Select your target subscription.
    *   **Role**: `Contributor`
6.  Click **"Save"**.

### Step 1.3: Create the PowerShell Runbook

This is where our tagging script will live.

1.  In your `vm-tagger-automation-account`, navigate to **"Runbooks"** (under "Process Automation" in the left menu).
2.  Click **"+ Create a runbook"**.
3.  Configure the new runbook:
    *   **Name**: **`Tag-New-VM`**
    *   **Runbook type**: `PowerShell`
    *   **Runtime version**: `5.1`
    *   **Description**: `Tags a newly created VM with CreatorId and CreationDate.`
4.  Click **"Create"**.

### Step 1.4: Add and Publish the Runbook Code

1.  The editor for the `Tag-New-VM` runbook will open. **Delete any existing placeholder text**.
2.  **Copy and paste the entire script below** into the editor pane.

```powershell
<#
.SYNOPSIS
This runbook is triggered by an Event Grid event when a new VM is created.
It tags the VM with the creator's identity and the creation timestamp.
#>

# Define the input parameter that will receive data from the webhook.
param (
    # The WebhookData object is automatically passed by Azure Automation when a webhook is invoked.
    [Parameter(Mandatory = $true)]
    [object] $WebhookData
)

# 1. Process the incoming webhook data from Event Grid
# The webhook data contains a JSON string in its RequestBody. We need to parse it.
Write-Output "Webhook received. Processing data..."
# Convert the JSON string from the webhook's body into a PowerShell object for easy access.
$event = $WebhookData.RequestBody | ConvertFrom-Json
# The actual event details are nested within the 'data' property of the main event object.
$data = $event.data

# 2. Log details for debugging purposes. This output will appear in the Automation Job logs.
Write-Output "Event Type: $($event.eventType)"
Write-Output "Resource URI: $($data.resourceUri)"

# 3. Filter to ensure this is the correct event for a VM creation.
# We only want to act on successful resource creation events for virtual machines.
if ($event.eventType -eq "Microsoft.Resources.ResourceWriteSuccess" -and $data.resourceUri -like "*Microsoft.Compute/virtualMachines*") {

    Write-Output "VM creation event detected. Proceeding with tagging."

    # 4. Extract creator's identity from the event claims.
    # The 'upn' (User Principal Name) is the most reliable identifier (e.g., user@domain.com).
    $creator = $data.claims.upn
    # If the UPN is not available, fall back to the 'name' claim.
    if ([string]::IsNullOrWhiteSpace($creator)) {
        $creator = $data.claims.name
    }
    # If no identity can be found, set a default value.
    if ([string]::IsNullOrWhiteSpace($creator)) {
        $creator = "unknown"
    }

    # Get the exact time the event occurred.
    $creationDate = $event.eventTime

    # Log the extracted information for verification.
    Write-Output "Creator identified as: $creator"
    Write-Output "Creation timestamp: $creationDate"

    # 5. Connect to Azure using the Automation Account's System-Assigned Managed Identity.
    # This is a secure way to authenticate without storing credentials in the script.
    try {
        Write-Output "Connecting to Azure with Managed Identity..."
        
        # What this line does:
        #   'Connect-AzAccount' is the command to log into Azure.
        #   The '-Identity' switch tells PowerShell to use the Azure Managed Identity of the
        #   environment where the script is running. In this case, it's the System-Assigned
        #   Managed Identity of our Automation Account, which we enabled in Step 1.1.
        #
        # Why it's important:
        #   This is the most secure method for authentication in Azure Automation. It avoids
        #   storing any passwords, secrets, or certificates directly in the script. Azure
        #   handles the authentication token lifecycle automatically in the background. The
        #   permissions for this identity were granted in Step 1.2.
        #
        # '$null = ...'
        #   The 'Connect-AzAccount' command outputs a summary of the logged-in context
        #   (subscription, tenant, etc.). By assigning the output to '$null', we are
        #   suppressing this information from appearing in the job logs, keeping them clean.
        $null = Connect-AzAccount -Identity
        
        Write-Output "Successfully connected to Azure."
    }
    catch {
        # If the connection fails, write an error and stop the script.
        Write-Error "Failed to connect to Azure using Managed Identity. Please check permissions."
        throw
    }

    # 6. Get the full Azure resource object for the VM that was just created.
    # The resource URI is provided in the event data.
    $vm = Get-AzResource -ResourceId $data.resourceUri

    # 7. Prepare the new tags to be applied.
    # First, retrieve any existing tags on the VM to avoid overwriting them.
    $tags = $vm.Tags
    # If the VM has no existing tags, initialize an empty hashtable.
    if ($null -eq $tags) {
        $tags = @{}
    }
    # Add or update the 'CreatorId' tag.
    $tags["CreatorId"] = $creator
    # Add or update the 'CreationDate' tag.
    $tags["CreationDate"] = $creationDate

    # 8. Apply the new and updated tags to the VM.
    try {
        Write-Output "Applying tags to VM..."
        # Use Set-AzResource to update the tags on the specified VM.
        # The -Force parameter suppresses confirmation prompts.
        $vm | Set-AzResource -Tag $tags -Force
        Write-Output "Successfully tagged VM: $($vm.Name)"
    }
    catch {
        # If tagging fails, write an error and stop the script.
        Write-Error "Failed to apply tags to the VM. Error: $_"
        throw
    }
}
else {
    # If the event is not for a VM creation, log it and do nothing.
    Write-Output "Event is not a VM creation event. Ignoring."
}
```

3.  Click **"Save"** in the top menu.
4.  After it saves, click **"Publish"** and confirm by clicking "Yes". The runbook must be published to be used by the webhook.

---

## Part 2: Connect Event Grid to the Runbook

### Step 2.1: Create the Runbook Webhook

1.  While viewing the `Tag-New-VM` runbook, click on **"Webhooks"** in the left menu.
2.  Click **"+ Add Webhook"**.
3.  Click **"Create new webhook"**.
4.  On the **"Create a new webhook"** blade, configure:
    *   **Name**: **`VM-Creation-Webhook`**
    *   **Enabled**: `Yes`
    *   **Expires**: Set a future date, for example, one year from today.
    *   **URL**: **CRITICAL STEP** - Click the **copy icon** next to the URL to copy it to your clipboard. Save this URL in a temporary text file.
5.  Click **"Next: Parameters >"**.
6.  On the **"Parameters"** tab, you will see a mandatory parameter named `WebhookData`. In the text box, enter a simple placeholder JSON object:
    ```json
    {}
    ```
7.  Click **"OK"**.
8.  Back on the "Add Webhook" screen, click **"Create"**.

### Step 2.2: Create the Event Grid Subscription

This step connects the subscription-wide events to your webhook.

1.  In the Azure Portal search bar, type **"Subscriptions"** and click on it.
2.  Click on your target subscription.
3.  In the left menu for your subscription, scroll down and click on **"Events"**.
4.  Click the **"+ Event Subscription"** button.
5.  On the **"Create Event Subscription"** page, fill out the **Basics** tab:
    *   **TOPIC DETAILS**
        *   **Topic Type**: `Subscriptions`
        *   **Source Resource**: Your target subscription should be pre-selected.
        *   **Resource group**: Select **`vm-tagger-automation-rg`**.
        *   **System Topic Name**: Enter **`Subscription-VM-Events-Topic`**.
    *   **EVENT SUBSCRIPTION DETAILS**
        *   **Event Subscription Name**: Enter **`vm-creation-trigger-for-automation`**.
    *   **EVENT TYPES**
        *   From the "Filter to Event Types" dropdown, select **`Resource Write Success`**.
    *   **ENDPOINT DETAILS**
        *   **Endpoint Type**: Select **`Webhook`**.
        *   **Endpoint**: Click **"Select an endpoint"**. In the new pane, **paste the webhook URL** you copied earlier. Click **"Confirm Selection"**.
6.  Click the **"Filters"** tab at the top of the page.
7.  Under **"Advanced filters"**, click **"+ Add new filter"**:
    *   **Key**: Select `data.operationName`.
    *   **Operator**: Select `String is exactly`.
    *   **Value**: Type `Microsoft.Compute/virtualMachines/write`.
8.  Click the **"Create"** button at the bottom.

---

## Part 3: Test the Solution

Your automation is now live.

1.  Navigate to "Virtual machines" in the Azure Portal and click **"+ Create"** -> **"Azure virtual machine"**.
2.  Create a new VM with any basic configuration.
3.  Wait for the VM deployment to fully complete. Then, wait an additional 1-2 minutes for the event to process.
4.  Navigate to the resource page for the new VM you just created.
5.  Click on the **"Tags"** menu item.
6.  **Verification**: You should see the following two tags applied:
    *   `CreatorId`: The Azure login of the user who created the VM (e.g., `shaleen-wonder`).
    *   `CreationDate`: The UTC timestamp of the creation event.

---

## Part 4: (Optional) Tag Existing VMs

This solution only tags *new* VMs. To tag your existing VMs, run this one-time script in **Azure Cloud Shell (PowerShell)**.

1.  Open Azure Cloud Shell and select the **PowerShell** environment.
2.  Create a file named `Tag-Existing-VMs.ps1` by running `code Tag-Existing-VMs.ps1`.
3.  Paste the following script into the editor and save it.

```powershell
# --- Configuration ---
# Define a static value for the creator, since we can't know who originally created these VMs.
$BackfillCreatorId = "backfilled-by-script"
# Define a static creation date, representing when the backfill script was run.
$BackfillCreationDate = (Get-Date).ToUniversalTime().ToString("yyyy-MM-dd HH:mm:ss")

# --- Script Logic ---
Write-Host "Connecting to Azure..."
try {
    # Authenticate using the identity of the logged-in user in Cloud Shell.
    $null = Connect-AzAccount -Identity
} catch {
    # If connection fails, show an error and exit.
    Write-Error "Failed to connect. Please ensure you are running this in Cloud Shell or are logged in."
    return
}

Write-Host "Finding all VMs in the current subscription..."
# Get a list of all virtual machines in the subscription context.
$vmsToTag = Get-AzVM

# Check if any VMs were found.
if (!$vmsToTag) {
    Write-Host "No VMs found."
    return
}

Write-Host "Found $($vmsToTag.Count) total VMs."
# Loop through each VM in the collection.
foreach ($vm in $vmsToTag) {
    # Check if the 'CreatorId' tag already exists to avoid overwriting it.
    if ($vm.Tags.ContainsKey("CreatorId")) {
        Write-Host "VM '$($vm.Name)' in RG '$($vm.ResourceGroupName)' already has CreatorId tag. Skipping."
    }
    else {
        Write-Host "Applying backfill tags to VM '$($vm.Name)'..."
        # Get the existing tags from the VM.
        $newTags = $vm.Tags
        # Add the backfill creator ID to the tags hashtable.
        $newTags["CreatorId"] = $BackfillCreatorId
        # Add the backfill creation date to the tags hashtable.
        $newTags["CreationDate"] = $BackfillCreationDate
        try {
            # Apply the updated tags to the VM.
            # -AsJob runs the operation in the background, which is good practice for long-running tasks.
            # Wait-Job pauses the script until the background job is complete.
            $vm | Set-AzVM -Tags $newTags -AsJob | Wait-Job | Out-Null
            Write-Host "Successfully tagged VM '$($vm.Name)'."
        }
        catch {
            # If an error occurs during tagging, report it.
            Write-Error "Failed to tag VM '$($vm.Name)'. Error: $_"
        }
    }
}
Write-Host "Backfill script complete."
```

4.  Run the script from the Cloud Shell prompt:

```bash
./Tag-Existing-VMs.ps1
```
