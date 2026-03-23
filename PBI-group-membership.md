# Power BI – Entra ID Group Assignment Propagation Delay: Solutions

## Problem

When a user clicks the Power BI link from the Medicare application, the backend:

1. Creates a **guest user** in Entra ID (Azure AD).
2. Assigns the user to the **required security group**.
3. **Redirects** the user to the Power BI home page.

However, the user sees an error: **"User doesn't have access to group"** because the group membership has not yet propagated through Entra ID and Power BI services. After a few minutes and a page refresh, access works fine.

**Root Cause:** Entra ID group membership changes can take anywhere from a few seconds up to ~60 minutes to propagate across Microsoft services (including Power BI). Power BI also has its own internal cache/sync cycle.

---

## Solution Options

### Option 1: Polling with Microsoft Graph API (Recommended)

Before redirecting the user to Power BI, **poll the Microsoft Graph API** to confirm the group membership has been activated, then redirect only after confirmation.

#### API Endpoint – `checkMemberGroups`

```http
POST https://graph.microsoft.com/v1.0/users/{user-id}/checkMemberGroups
Content-Type: application/json
Authorization: Bearer {token}

{
  "groupIds": [
    "<power-bi-security-group-id>"
  ]
}
```

- **If the response contains the group ID** → membership is active → safe to redirect.
- **If the response is empty** → membership has not propagated yet → wait and retry.

#### Required Permissions

- `GroupMember.Read.All` or `Directory.Read.All`

#### Sample Implementation (C#)

```csharp
public async Task<bool> WaitForGroupMembership(string userId, string groupId, int maxRetries = 20, int delaySeconds = 15)
{
    for (int i = 0; i < maxRetries; i++)
    {
        var requestBody = new CheckMemberGroupsPostRequestBody
        {
            GroupIds = new List<string> { groupId }
        };

        var result = await _graphClient.Users[userId]
            .CheckMemberGroups
            .PostAsCheckMemberGroupsPostResponseAsync(requestBody);

        if (result?.Value?.Contains(groupId) == true)
        {
            return true; // Membership is active – safe to redirect
        }

        await Task.Delay(TimeSpan.FromSeconds(delaySeconds));
    }

    return false; // Timed out – membership did not propagate in time
}
```

#### Sample Implementation (Python)

```python
import time
import requests

def wait_for_group_membership(access_token, user_id, group_id, max_retries=20, delay_seconds=15):
    url = f"https://graph.microsoft.com/v1.0/users/{user_id}/checkMemberGroups"
    headers = {
        "Authorization": f"Bearer {access_token}",
        "Content-Type": "application/json"
    }
    body = {"groupIds": [group_id]}

    for attempt in range(max_retries):
        response = requests.post(url, headers=headers, json=body)
        if response.status_code == 200:
            member_groups = response.json().get("value", [])
            if group_id in member_groups:
                return True  # Membership active – safe to redirect
        time.sleep(delay_seconds)

    return False  # Timed out
```

---

### Option 2: Verify Access via Power BI REST API

In addition to (or instead of) the Graph API check, verify that Power BI itself recognizes the user by calling the **Power BI REST API**.

#### API Endpoint – Get Group Users

```http
GET https://api.powerbi.com/v1.0/myorg/groups/{workspaceId}/users
Authorization: Bearer {service-principal-or-master-user-token}
```

#### Sample Logic

```python
def user_has_workspace_access(access_token, workspace_id, user_email):
    url = f"https://api.powerbi.com/v1.0/myorg/groups/{workspace_id}/users"
    headers = {"Authorization": f"Bearer {access_token}"}
    response = requests.get(url, headers=headers)

    if response.status_code == 200:
        members = response.json().get("value", [])
        for member in members:
            if member.get("emailAddress", "").lower() == user_email.lower():
                return True
    return False
```

> **Note:** This approach requires a **service principal** or **master user** token with admin-level Power BI API permissions.

---

### Option 3: Intermediate "Please Wait" Landing Page

If API polling is not feasible, implement a **landing/splash page** that:

1. Displays a user-friendly message: *"Your workspace access is being prepared. Please wait..."*
2. Periodically checks group membership (via Graph API or Power BI API) using AJAX/fetch calls.
3. Automatically redirects to Power BI once access is confirmed.
4. Shows a timeout message with a manual retry button if propagation exceeds the maximum wait time.

#### Sample Frontend (JavaScript)

```javascript
async function pollAndRedirect(userId, groupId, powerBiUrl) {
    const maxAttempts = 20;
    const delayMs = 15000; // 15 seconds

    document.getElementById("status").innerText =
        "Setting up your Power BI access. Please wait...";

    for (let i = 0; i < maxAttempts; i++) {
        const isMember = await checkGroupMembership(userId, groupId);
        if (isMember) {
            window.location.href = powerBiUrl;
            return;
        }
        await new Promise(resolve => setTimeout(resolve, delayMs));
    }

    document.getElementById("status").innerText =
        "Access setup is taking longer than expected. Please try again in a few minutes.";
    document.getElementById("retryBtn").style.display = "block";
}

async function checkGroupMembership(userId, groupId) {
    // Call your backend API that wraps the Microsoft Graph checkMemberGroups call
    const response = await fetch(`/api/check-membership?userId=${userId}&groupId=${groupId}`);
    const data = await response.json();
    return data.isMember;
}
```

---

### Option 4: Direct User Assignment (Bypass Group Propagation)

Instead of relying on group-based access, **assign the user directly** to the Power BI workspace using the Power BI REST API. Direct assignments take effect immediately.

#### API Endpoint – Add User to Workspace

```http
POST https://api.powerbi.com/v1.0/myorg/groups/{workspaceId}/users
Content-Type: application/json
Authorization: Bearer {token}

{
    "emailAddress": "user@example.com",
    "groupUserAccessRight": "Member",
    "principalType": "User"
}
```

> **Trade-off:** This avoids propagation delays entirely but means you must manage individual user assignments rather than group-based access. Consider this for scenarios where immediate access is critical.

---

## Recommended Architecture

```
User clicks Power BI link in Medicare App
        │
        ▼
Backend: Create guest user in Entra ID
        │
        ▼
Backend: Add user to security group
        │
        ▼
Redirect to intermediate "Please Wait" page
        │
        ▼
Frontend/Backend: Poll Graph API (checkMemberGroups)
  ┌─────┴─────┐
  │           │
  ▼           ▼
 Active      Not Yet
  │            │
  ▼            ▼
Redirect    Wait 15s
to Power BI  & Retry
              │
              ▼
         Max retries?
          │        │
          ▼        ▼
        Yes       No
          │        │
          ▼        ▼
     Show error  Loop back
     + retry btn  to poll
```

---

## Summary

| Option | Approach | Propagation Delay Handled? | Complexity |
|--------|----------|---------------------------|------------|
| **1** | Poll Microsoft Graph API before redirect | ✅ Yes | Medium |
| **2** | Verify via Power BI REST API | ✅ Yes | Medium |
| **3** | Intermediate landing page with polling | ✅ Yes | Low–Medium |
| **4** | Direct workspace assignment (skip groups) | ✅ Eliminated entirely | Low |

**Best Practice:** Combine **Option 1 + Option 3** — use a friendly landing page that polls the Microsoft Graph API in the background and redirects the user only after group membership is confirmed.

---

## References

- [Microsoft Graph API: checkMemberGroups](https://learn.microsoft.com/en-us/graph/api/directoryobject-checkmembergroups)
- [Microsoft Graph API: List group members](https://learn.microsoft.com/en-us/graph/api/group-list-members)
- [Power BI REST API: Add Group User](https://learn.microsoft.com/en-us/rest/api/power-bi/groups/add-group-user)
- [Power BI REST API: Get Group Users](https://learn.microsoft.com/en-us/rest/api/power-bi/groups/get-group-users)
- [Entra ID Group Membership Propagation](https://learn.microsoft.com/en-us/entra/fundamentals/groups-view-azure-portal)
