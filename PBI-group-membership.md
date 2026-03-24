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


## Troubleshooting

Here's code to test both:

## Test Scenario 1: Power BI Authorization Cache

```js
/**
 * Test if Power BI's internal cache has updated
 * by attempting to access the workspace via Power BI API
 */
async function testPowerBICache(workspaceId, userAccessToken) {
    console.log('\n=== TEST 1: Power BI Authorization Cache ===');
    
    try {
        const response = await axios.get(
            `https://api.powerbi.com/v1.0/myorg/groups/${workspaceId}`,
            {
                headers: {
                    'Authorization': `Bearer ${userAccessToken}`,
                    'Content-Type': 'application/json'
                }
            }
        );
        
        console.log('✅ Power BI Cache Test: SUCCESS');
        console.log(`   Status: ${response.status}`);
        console.log(`   Workspace Name: ${response.data.name}`);
        console.log('   → User CAN access workspace via API');
        console.log('   → Power BI cache HAS been updated');
        return true;
        
    } catch (error) {
        if (error.response?.status === 403) {
            console.log('❌ Power BI Cache Test: DENIED (403 Forbidden)');
            console.log('   → User CANNOT access workspace via API');
            console.log('   → Power BI cache NOT updated yet');
            console.log('   → Need to wait longer for cache refresh');
            return false;
        } else if (error.response?.status === 404) {
            console.log('⚠️  Power BI Cache Test: NOT FOUND (404)');
            console.log('   → Workspace might not exist or user has no access');
            return false;
        } else {
            console.log('⚠️  Power BI Cache Test: ERROR');
            console.log(`   Status: ${error.response?.status}`);
            console.log(`   Error: ${error.message}`);
            throw error;
        }
    }
}
```

## Test Scenario 2: Token Claims

```js
/**
 * Test if user's token contains the required group claim
 * by decoding the JWT token
 */
async function testTokenClaims(userAccessToken, expectedGroupId) {
    console.log('\n=== TEST 2: Token Claims ===');
    
    try {
        const tokenParts = userAccessToken.split('.');
        if (tokenParts.length !== 3) {
            throw new Error('Invalid JWT token format');
        }
        
        const payload = JSON.parse(
            Buffer.from(tokenParts[1], 'base64').toString('utf-8')
        );
        
        console.log('Token Payload Info:');
        console.log(`   User: ${payload.name || payload.preferred_username}`);
        console.log(`   UPN: ${payload.upn || payload.preferred_username}`);
        console.log(`   OID: ${payload.oid}`);
        console.log(`   Issued At: ${new Date(payload.iat * 1000).toISOString()}`);
        console.log(`   Expires At: ${new Date(payload.exp * 1000).toISOString()}`);
        
        if (payload.groups && Array.isArray(payload.groups)) {
            console.log(`\n   Groups in token (${payload.groups.length} total):`);
            payload.groups.forEach((groupId, index) => {
                const marker = groupId === expectedGroupId ? '✅ TARGET GROUP' : '';
                console.log(`   ${index + 1}. ${groupId} ${marker}`);
            });
            
            if (payload.groups.includes(expectedGroupId)) {
                console.log('\n✅ Token Claims Test: SUCCESS');
                console.log('   → Token CONTAINS the required group');
                console.log('   → Token has been refreshed with new membership');
                return true;
            } else {
                console.log('\n❌ Token Claims Test: MISSING GROUP');
                console.log('   → Token does NOT contain the required group');
                console.log('   → Token needs to be refreshed');
                return false;
            }
        } else if (payload.wids) {
            console.log('\n⚠️  Token uses role claims (wids) instead of groups');
            console.log('   → Token might be using different claim type');
            console.log(`   → Roles in token: ${payload.wids.join(', ')}`);
            return null;
        } else {
            console.log('\n⚠️  Token Claims Test: NO GROUP CLAIMS');
            console.log('   → Token does not contain "groups" claim');
            console.log('   → Might be using group overage scenario');
            console.log('   → Check "_claim_names" and "_claim_sources" in token');
            
            if (payload._claim_names || payload._claim_sources) {
                console.log('   → Token uses group overage (too many groups)');
                console.log('   → Groups must be retrieved via Graph API instead');
            }
            return null;
        }
        
    } catch (error) {
        console.log('⚠️  Token Claims Test: ERROR');
        console.log(`   Error: ${error.message}`);
        throw error;
    }
}
```

## Combined Test Function

```js
/**
 * Run both tests and provide diagnosis
 */
async function diagnoseAccessIssue(workspaceId, groupId, userAccessToken) {
    console.log('╔════════════════════════════════════════════════╗');
    console.log('║  Power BI Access Issue Diagnostic Test        ║');
    console.log('╚════════════════════════════════════════════════╝');
    console.log(`\nWorkspace ID: ${workspaceId}`);
    console.log(`Expected Group ID: ${groupId}`);
    console.log(`Token Length: ${userAccessToken.length} chars`);
    
    const cacheReady = await testPowerBICache(workspaceId, userAccessToken);
    const tokenHasGroup = await testTokenClaims(userAccessToken, groupId);
    
    console.log('\n╔════════════════════════════════════════════════╗');
    console.log('║  DIAGNOSIS                                     ║');
    console.log('╚════════════════════════════════════════════════╝');
    
    if (cacheReady === true && tokenHasGroup === true) {
        console.log('✅ BOTH READY: User should have access');
        console.log('   → Power BI cache is updated ✅');
        console.log('   → Token contains group ✅');
        console.log('   → Access should work now');
        return 'READY';
    } else if (cacheReady === false && tokenHasGroup === true) {
        console.log('⚠️  ISSUE: Power BI Authorization Cache');
        console.log('   → Token is correct ✅');
        console.log('   → Power BI cache NOT updated yet ❌');
        console.log('   → ROOT CAUSE: Power BI cache delay');
        console.log('   → SOLUTION: Wait 3-5 minutes for cache refresh');
        return 'CACHE_ISSUE';
    } else if (cacheReady === false && tokenHasGroup === false) {
        console.log('⚠️  ISSUE: BOTH not ready');
        console.log('   → Token does NOT have group ❌');
        console.log('   → Power BI cache NOT updated ❌');
        console.log('   → ROOT CAUSE: Both token and cache need refresh');
        console.log('   → SOLUTION: Force token refresh AND wait for cache');
        return 'BOTH_ISSUE';
    } else if (cacheReady === true && tokenHasGroup === false) {
        console.log('⚠️  ISSUE: Token Claims (Unusual)');
        console.log('   → Power BI cache is ready ✅');
        console.log('   → Token does NOT have group ❌');
        console.log('   → ROOT CAUSE: Token needs refresh');
        console.log('   → SOLUTION: Force token refresh');
        return 'TOKEN_ISSUE';
    } else {
        console.log('⚠️  INCONCLUSIVE');
        console.log('   → Unable to determine root cause');
        console.log('   → Check logs above for details');
        return 'UNKNOWN';
    }
}
```

## Helper: Get User's Power BI Token

```js
/**
 * Get access token for user to call Power BI API
 */
async function getUserPowerBIToken(userId) {
    const tokenResponse = await axios.post(
        `https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token`,
        new URLSearchParams({
            client_id: CLIENT_ID,
            client_secret: CLIENT_SECRET,
            grant_type: 'urn:ietf:params:oauth:grant-type:jwt-bearer',
            assertion: userToken,
            requested_token_use: 'on_behalf_of',
            scope: 'https://analysis.windows.net/powerbi/api/.default'
        })
    );
    
    return tokenResponse.data.access_token;
}
```

## Usage Example

```js
async function runDiagnostics() {
    const workspaceId = 'YOUR-WORKSPACE-ID';
    const groupId = 'YOUR-GROUP-ID';
    const userEmail = 'user@domain.com';
    
    try {
        console.log('Step 1: Checking Entra membership...');
        const user = await getUserByEmail(userEmail);
        const isMemberInEntra = await checkUserInGroup(user.id, groupId);
        console.log(`Entra Membership: ${isMemberInEntra ? '✅ Confirmed' : '❌ Not found'}`);
        
        if (!isMemberInEntra) {
            console.log('ERROR: User not in Entra group yet. Wait for propagation.');
            return;
        }
        
        console.log('\nStep 2: Getting user access token...');
        const userToken = await getUserPowerBIToken(user.id);
        console.log('Token acquired ✅');
        
        console.log('\nStep 3: Running diagnostic tests...\n');
        const diagnosis = await diagnoseAccessIssue(workspaceId, groupId, userToken);
        
        console.log('\n╔════════════════════════════════════════════════╗');
        console.log('║  RESULT                                        ║');
        console.log('╚════════════════════════════════════════════════╝');
        console.log(`Diagnosis: ${diagnosis}`);
        
    } catch (error) {
        console.error('Diagnostic failed:', error.message);
        console.error(error);
    }
}

runDiagnostics();
```

## Simpler Test Scripts

### Quick Test 1: Check Power BI Cache

```js
const axios = require('axios');

async function quickTestPowerBICache() {
    const workspaceId = 'YOUR-WORKSPACE-ID';
    const userAccessToken = 'USER-ACCESS-TOKEN';
    
    try {
        const response = await axios.get(
            `https://api.powerbi.com/v1.0/myorg/groups/${workspaceId}`,
            { headers: { 'Authorization': `Bearer ${userAccessToken}` } }
        );
        
        console.log('✅ SUCCESS: Power BI cache is ready');
        console.log('User CAN access the workspace');
        return true;
    } catch (error) {
        if (error.response?.status === 403) {
            console.log('❌ DENIED: Power BI cache NOT ready yet');
            console.log('ROOT CAUSE: Power BI authorization cache delay');
            console.log('SOLUTION: Wait 3-5 more minutes');
        } else {
            console.log(`ERROR: ${error.message}`);
        }
        return false;
    }
}

quickTestPowerBICache();
```

### Quick Test 2: Check Token Claims

```js
function quickTestTokenClaims() {
    const userAccessToken = 'USER-ACCESS-TOKEN';
    const expectedGroupId = 'YOUR-GROUP-ID';
    
    const payload = JSON.parse(
        Buffer.from(userAccessToken.split('.')[1], 'base64').toString()
    );
    
    console.log('Token Info:');
    console.log(`  User: ${payload.preferred_username}`);
    console.log(`  Groups in token: ${payload.groups?.length || 0}`);
    
    if (payload.groups?.includes(expectedGroupId)) {
        console.log('✅ SUCCESS: Token contains the group');
        console.log('ROOT CAUSE: Power BI cache delay (not token)');
        return true;
    } else {
        console.log('❌ MISSING: Token does NOT contain the group');
        console.log('ROOT CAUSE: Token needs refresh');
        return false;
    }
}

quickTestTokenClaims();
```

## Expected Test Results

### Scenario A: Power BI Cache Issue (Most Likely)

| Test | Result |
|------|--------|
| ✅ Entra Membership | Confirmed |
| ✅ Token Claims | Group present |
| ❌ Power BI Cache | Access denied (403) |

> → **ROOT CAUSE:** Power BI authorization cache
> → **SOLUTION:** Wait 3-5 minutes

### Scenario B: Token Issue

| Test | Result |
|------|--------|
| ✅ Entra Membership | Confirmed |
| ❌ Token Claims | Group missing |
| ❌ Power BI Cache | Access denied (403) |

> → **ROOT CAUSE:** Token needs refresh
> → **SOLUTION:** Force new token with fresh claims

## How to Get User Access Token for Testing

```js
// If you're using Express session-based auth
const userToken = req.session.accessToken;

// If you're using MSAL
const tokenResponse = await msalClient.acquireTokenSilent({
    scopes: ['https://analysis.windows.net/powerbi/api/.default'],
    account: userAccount
});
const userToken = tokenResponse.accessToken;

// If using Keycloak
const userToken = req.kauth.grant.access_token.token;
```
