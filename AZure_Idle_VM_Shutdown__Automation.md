# Guide: Deploying the Idle VM Shutdown Solution to Production

This guide provides the complete, end-to-end steps to create, configure, and activate an automated solution that shuts down idle VMs to save costs.

---

## Part 1: Setup and Permissions

This part ensures the Automation Account exists and has the correct permissions to manage your VMs.

### Step 1.1: Navigate to Your Existing Automation Account

We will use the same account created for the auto-tagging solution.

1.  In the Azure Portal, search for and navigate to your Automation Account: **`vm-tagger-automation-account`**.

> ### ⚠️ **Step 1.2: Verify and Assign the 'Contributor' Role (Crucial Step)**
> *This is the most important step. The script runs as the Automation Account's own "Managed Identity" (a robot account), not as your user. This identity must have permission to see and stop VMs.*

1.  In the `vm-tagger-automation-account`, go to the **"Identity"** menu (under "Account Settings").
2.  On the "System assigned" tab, you will see an **Object (principal) ID**. This is the unique ID for the automation's identity.
3.  Click the **"Azure role assignments"** button.
4.  Check if the `Contributor` role is already assigned at the `Subscription` scope. If it is, you are all set. If not, proceed with the next step.
5.  Click **"+ Add role assignment (Preview)"**.
6.  Configure the new role assignment:
    *   **Scope**: `Subscription`
    *   **Subscription**: Select your `Pay-As-You-Go` subscription.
    *   **Role**: `Contributor`
7.  Click **"Save"**. The Automation Account now has the necessary permissions.

---

## Part 2: Create and Configure the Runbook

This is where we'll place the PowerShell script that contains the shutdown logic.

### Step 2.1: Create the Runbook

1.  In your Automation Account, navigate to **"Runbooks"** (under "Process Automation").
2.  Click **"+ Create a runbook"**.
3.  Configure the new runbook:
    *   **Name**: **`Stop-Idle-VMs`**
    *   **Runbook type**: `PowerShell`
    *   **Runtime version**: `7.2 (preview)`
    *   **Description**: `Checks for idle VMs based on CPU metrics and shuts them down.`
4.  Click **"Create"**.

### Step 2.2: Add the Production-Ready Script

1.  The editor for the `Stop-Idle-VMs` runbook will open. Delete any placeholder text.
2.  **Copy and paste the entire robust script below** into the editor. This is the final version that correctly handles authentication and subscription context.

```powershell
<#
.SYNOPSIS
This runbook finds and shuts down (deallocates) Azure VMs that have been idle
for a specified number of days.
#>

# ----------------- NORMAL OPERATING CONFIGURATION -----------------
# This section contains the user-configurable variables that control the script's behavior.

# Define the number of days to look back for CPU metrics. A VM is checked against this period.
$IdleDays = 5

# Set the CPU percentage threshold. If a VM's MAXIMUM CPU has not exceeded this
# value in the last $IdleDays, it is considered idle.
$CpuThresholdPercentage = 5

# A safety switch. Set to $true to allow the script to shut down VMs.
# Set to $false to run in "report-only" mode, which only identifies idle VMs without stopping them.
$EnableShutdown = $true

# Define the name of the tag used to protect critical VMs from being shut down by this script.
# Any VM with this tag will be skipped.
$ExclusionTagName = "AutoShutdown-Exclude"
# ------------------------------------------------------------------


# Section 1: Connect to Azure using the Automation Account's Managed Identity.
# A Managed Identity provides an identity for applications to use when connecting to resources that support Azure AD authentication.
# This is a secure way to authenticate because you don't need to manage any credentials in code.
try {
    # Write a status message to the output stream. This is useful for logging and debugging.
    Write-Output "Connecting to Azure with Managed Identity..."
    # The 'Connect-AzAccount -Identity' command authenticates using the system-assigned managed identity of the Automation Account.
    # We assign the output to $null to suppress the default output of the command, keeping the logs clean.
    $null = Connect-AzAccount -Identity
    # Confirm successful connection.
    Write-Output "Successfully connected to Azure."
}
catch {
    # If the 'try' block fails, this 'catch' block will execute.
    # Write an error message indicating the failure. This is critical for troubleshooting permission issues.
    Write-Error "Failed to connect to Azure using Managed Identity. Please check permissions."
    # 'throw' stops the script execution and propagates the original error. This prevents the script from continuing in a failed state.
    throw
}

# Section 2: Discover and set the subscription context.
# In environments with multiple subscriptions, it's crucial to ensure the script operates on the correct one.
try {
    # Announce the start of the subscription discovery process.
    Write-Output "Discovering subscription context..."
    # 'Get-AzSubscription' retrieves all subscriptions the Managed Identity has access to.
    $accessibleSubscriptions = Get-AzSubscription
    # Check if any subscriptions were returned. If not, the identity may lack permissions.
    if ($null -eq $accessibleSubscriptions) {
        # Throw a specific, actionable error message if no subscriptions are found.
        throw "Get-AzSubscription returned no subscriptions. Please verify the Managed Identity has a 'Contributor' role assignment on the target subscription."
    }
    # For simplicity, this script targets the first accessible subscription found.
    # In a multi-subscription environment, you might add logic here to select a specific one by name or ID.
    $subscriptionId = $accessibleSubscriptions[0].Id
    # Log which subscription is being targeted.
    Write-Output "Setting context to subscription ID: $subscriptionId"
    # 'Set-AzContext' directs all subsequent Azure commands in this session to operate against the specified subscription.
    # '| Out-Null' suppresses the command's output for cleaner logs.
    Set-AzContext -Subscription $subscriptionId | Out-Null
    # Confirm which subscription context has been successfully set.
    Write-Output "Context successfully set to subscription: $($accessibleSubscriptions[0].Name)"
}
catch {
    # Catch any errors during the subscription context setup.
    # '$_' is an automatic variable in PowerShell that contains the current error record.
    Write-Error "A critical error occurred while setting the subscription context. Error: $_"
    # Stop the script to prevent it from running against the wrong subscription or failing on subsequent commands.
    throw
}

# Section 3: Define the time frame for the metric query.
# This sets the start and end times for fetching CPU data.
$EndTime = Get-Date # Set the end time to the current date and time.
$StartTime = $EndTime.AddDays(-$IdleDays) # Calculate the start time by subtracting the configured number of idle days.
# Log the calculated time window for transparency.
Write-Output "Metric Query Window: From $StartTime to $EndTime"

# Section 4: Get all running VMs in the subscription.
# This is more efficient than getting all VMs and then checking their status.
Write-Output "Fetching all running VMs in the subscription..."
# 'Get-AzVM -Status' retrieves a list of all VMs along with their power state.
# 'Where-Object { $_.PowerState -eq 'VM running' }' filters this list to include only the VMs that are currently running.
$vms = Get-AzVM -Status | Where-Object { $_.PowerState -eq 'VM running' }

# Check if the $vms variable is empty or null.
if (!$vms) {
    # If no running VMs were found, there's nothing to do.
    Write-Output "No running VMs found. Exiting."
    # 'return' safely exits the script.
    return
}

# Log how many running VMs were found and will be analyzed.
Write-Output "Found $($vms.Count) running VMs. Analyzing each for idle status..."

# Section 5: Loop through each running VM to analyze its CPU usage.
# The 'foreach' loop iterates over the collection of VM objects stored in the $vms variable.
foreach ($vm in $vms) {
    # Print a separator for clear logging for each VM.
    Write-Output "-----------------------------------------------------"
    # Log which VM is currently being analyzed.
    Write-Output "Analyzing VM: $($vm.Name) in Resource Group: $($vm.ResourceGroupName)"

    # Check if the VM has the exclusion tag. This is the safety mechanism to prevent shutting down critical systems.
    # '$vm.Tags' is a hashtable of tags on the VM. 'ContainsKey' checks if a specific key (tag name) exists.
    if ($vm.Tags.ContainsKey($ExclusionTagName)) {
        # If the tag exists, log that the VM is being skipped and move to the next one.
        Write-Output "VM has the exclusion tag '$ExclusionTagName'. Skipping."
        # 'continue' immediately starts the next iteration of the 'foreach' loop.
        continue
    }

    # Section 6: Query Azure Monitor for the maximum CPU percentage for the current VM.
    $metricName = "Percentage CPU" # The official name of the CPU metric for VMs.
    try {
        # 'Get-AzMetric' retrieves metric data from Azure Monitor.
        # -ResourceId: Specifies the unique ID of the VM to query.
        # -MetricName: The specific metric to retrieve.
        # -StartTime / -EndTime: The time window for the query.
        # -AggregationType: Specifies how to summarize the metric data. 'Maximum' finds the highest peak value in the time series.
        $metric = Get-AzMetric -ResourceId $vm.Id -MetricName $metricName -StartTime $StartTime -EndTime $EndTime -AggregationType Maximum
    }
    catch {
        # If 'Get-AzMetric' fails (e.g., due to transient API issues), log a warning and skip this VM.
        Write-Warning "Could not retrieve CPU metrics for VM '$($vm.Name)'. Skipping."
        continue
    }
    
    # It's possible for the metric query to succeed but return no data (e.g., for a newly created VM).
    if (!$metric.Data) {
        # Log a warning and skip the VM if no data was returned.
        Write-Warning "No CPU metric data was returned for VM '$($vm.Name)'. Skipping."
        continue
    }
    
    # From the returned metric data, calculate the single highest value.
    # '$metric.Data' contains the time series data. We pipe this to 'Measure-Object' to calculate the maximum value of the 'Maximum' property.
    $maxCpu = ($metric.Data | Measure-Object -Property Maximum -Maximum).Maximum
    
    # Check if a maximum CPU value could be determined. It might be null if the data format is unexpected.
    if ($null -eq $maxCpu) {
        # Log a warning and skip the VM if the max CPU value couldn't be calculated.
        Write-Warning "Could not determine maximum CPU for VM '$($vm.Name)'. Skipping."
        continue
    }

    # Log the determined maximum CPU utilization for the period.
    Write-Output "Maximum CPU utilization in the last $IdleDays days: $($maxCpu)%"

    # Section 7: Check if the VM is idle and perform the appropriate action.
    # Compare the VM's max CPU usage against the configured threshold.
    if ($maxCpu -lt $CpuThresholdPercentage) {
        # If the max CPU is below the threshold, the VM is considered idle.
        Write-Output "VM is IDLE (Max CPU is below $($CpuThresholdPercentage)%)."

        # Double-check the safety switch before performing the shutdown.
        if ($EnableShutdown) {
            # Log the shutdown action.
            Write-Output "SHUTTING DOWN VM: $($vm.Name)..."
            try {
                # 'Stop-AzVM' initiates the shutdown (and deallocation) of the VM.
                # -Force: Suppresses the confirmation prompt.
                # -AsJob: Runs the operation as a background job. This is useful in runbooks to avoid timeouts on long-running operations.
                # 'Wait-Job': Pauses the script and waits for the background job to complete.
                Stop-AzVM -Name $vm.Name -ResourceGroupName $vm.ResourceGroupName -Force -AsJob | Wait-Job
                # Confirm that the shutdown command completed successfully.
                Write-Output "Successfully shut down (deallocated) VM: $($vm.Name)."
            }
            catch {
                # If 'Stop-AzVM' fails, log the error.
                Write-Error "Failed to shut down VM '$($vm.Name)'. Error: $_"
            }
        } else {
            # If the shutdown safety switch is off, log that the action was skipped.
            Write-Output "Report-only mode is enabled. Shutdown action was skipped."
        }
    } else {
        # If the max CPU is at or above the threshold, the VM is considered active.
        Write-Output "VM is ACTIVE (Max CPU has been at or above $($CpuThresholdPercentage)%). No action taken."
    }
}

# Final confirmation message when the script has processed all VMs.
Write-Output "-----------------------------------------------------"
Write-Output "Idle VM check complete."
```

3.  Click **"Save"**, and then click **"Publish"**.

---

## Part 3: Schedule the Runbook for Production

This will create the daily "polling" mechanism.

1.  While viewing the `Stop-Idle-VMs` runbook, click on **"Schedules"** in the left menu.
2.  Click **"+ Add a schedule"**.
3.  Click **"Link a schedule to your runbook"**.
4.  Click **"+ Add a schedule"**.
5.  Configure the new schedule:
    *   **Name**: **`Daily-Midnight-Scan`**
    *   **Description**: `Runs the idle VM check every day at midnight.`
    *   **Starts**: Leave as today's date. Set the time to a low-traffic hour, like `12:00:00 AM`.
    *   **Time zone**: Select your preferred time zone (e.g., UTC).
    *   **Recurrence**: `Recurring`
    *   **Recur every**: `1` `Day`
6.  Click **"Create"**.
7.  You will be returned to the "Schedule Runbook" page. Click **"OK"** to link the schedule to the runbook.

---

## Part 4: Final Production Activation Steps

This is the final safety check before the solution goes fully live.

### Step 4.1: Tag Critical VMs for Exclusion

This is the most important safety step to prevent accidental shutdown of production systems.

1.  For each critical VM (Domain Controllers, production app servers, etc.), navigate to its resource page.
2.  Go to the **"Tags"** menu.
3.  Add the following tag:
    *   **Name**: `AutoShutdown-Exclude`
    *   **Value**: `true`
4.  Click **"Save"**. Repeat for all critical VMs.

### Step 4.2: Final Verification

1.  Navigate back to the **`Stop-Idle-VMs`** runbook.
2.  Click on **"Schedules"** and confirm that `Daily-Midnight-Scan` is listed and **Enabled**.
3.  Click on **"Edit"** one last time to confirm the variables in the script are set for production (`$IdleDays = 5`, `$CpuThresholdPercentage = 5`, `$EnableShutdown = $true`).
4.  Ensure the runbook status is **"Published"**.

Your "Idle VM Shutdown" solution is now fully deployed and active in production. It will automatically run every day at the scheduled time, check for idle VMs, and shut them down while respecting the [...]
