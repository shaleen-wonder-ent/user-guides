# Power BI Cache Timing Test — Polling Logic

## 1. The Problem

When a user is added to an Entra security group that has access to a Power BI workspace, there is a **delay** before Power BI's internal authorization cache recognizes the new membership. If the user is redirected to the workspace immediately after being added to the group, they get an **access denied** error.

### Current Flow (Broken)

```
User clicks "Analytics"
        │
        ▼
App calls Graph API ──► Adds user to security group
        │
        ▼
App immediately redirects to Power BI (new tab)
        │
        ▼
User does PBI consent
        │
        ▼
❌ ACCESS DENIED (Power BI cache not refreshed yet)
```

### Proposed Flow (Fixed)

```
User clicks "Analytics"
        │
        ▼
App calls Graph API ──► Adds user to security group
        │
        ▼
App shows "Provisioning..." page with countdown (5 min max)
        │
        ▼
[In background]: App polls Power BI Admin API every 15 seconds
        │              GET /admin/groups/{workspaceId}/users
        │
        ├──► User NOT found in list ──► Keep polling...
        │
        └──► User FOUND in list ──► ✅ Cache is ready!
                                          │
                                          ▼
                                   Redirect to Power BI (new tab)
                                          │
                                          ▼
                                   User does PBI consent
                                          │
                                          ▼
                                   ✅ ACCESS GRANTED
```

### Fallback: If polling doesn't confirm within 5 minutes

```
5 minutes elapsed, user still not found
        │
        ▼
Redirect to Power BI anyway
        │
        ▼
User does PBI consent
        │
        ▼
⚠️ Might work, might not — but we tried our best
```

---

## 2. How the Polling Works

### The Key Idea

Power BI has an **Admin API** that returns the list of users who currently have access to a workspace:

```
GET https://api.powerbi.com/v1.0/myorg/admin/groups/{workspaceId}/users
```

This API uses **app-only permissions** (no user context needed). The app itself calls this endpoint with its own token.

### The Logic

1. App adds UserA to the security group (via Graph API)
2. App starts polling the Power BI Admin API every 15 seconds
3. Each poll checks: **Is UserA in the workspace users list?**
4. If YES → Power BI has processed the group membership → safe to redirect
5. If NO after 5 minutes → redirect anyway (best effort)

### Why This Works Without User Context

- The **app** does the polling, not the user
- The app uses its own **app-only token** to call the Power BI Admin API
- The user just sees a "Provisioning..." page with a countdown
- No user token, no user session, no user credentials needed for the check
- The user stays on the MedAdv app page until the check passes

---

## 3. What We Need to Validate

Before implementing this in production, we need to confirm one critical assumption:

> **Does the Power BI Admin API (`/admin/groups/{workspaceId}/users`) update at the same time as (or before) the runtime authorization cache?**

### Two Possible Outcomes

| Admin API shows user | User can access workspace | Meaning |
|---|---|---|
| ✅ Yes | ✅ Yes | Admin API is a **reliable indicator** — use it for polling ✅ |
| ✅ Yes | ❌ No (access denied) | Admin API updates **before** the runtime cache — NOT reliable ❌ |
| ❌ No (after 5 min) | ❌ No | Both are slow — use a fixed countdown instead |

### This Test App Validates This

The test app below will:
1. Add a user to the security group
2. Poll the Admin API and record **when** the user first appears
3. Let you manually test Power BI access at that exact moment
4. Compare the two to see if the Admin API is a reliable signal

---

## 4. Prerequisites

### 4.1 Azure App Registration

Your app (MedAdv) likely already has an Azure App Registration. You need to add the following **Application permissions** (not Delegated) to it.

If you want to test with a **separate** app registration to avoid touching production, create a new one:

#### Create a New App Registration (Optional — for testing only)

1. Go to [Azure Portal](https://portal.azure.com) → **Microsoft Entra ID** → **App registrations**
2. Click **+ New registration**
3. Name: `PBI Cache Test` (or anything)
4. Supported account types: **Accounts in this organizational directory only**
5. Redirect URI: Leave blank (not needed for this test)
6. Click **Register**
7. Note down the **Application (client) ID** and **Directory (tenant) ID**
8. Go to **Certificates & secrets** → **+ New client secret** → Copy the secret value

#### Add API Permissions

Go to **API permissions** → **+ Add a permission**:

**Permission 1: Microsoft Graph**

1. Click **Microsoft Graph** → **Application permissions**
2. Search for and select:
   - `GroupMember.ReadWrite.All` (to add/remove users from groups)
   - `User.Read.All` (to look up users by email)
3. Click **Add permissions**

**Permission 2: Power BI Service**

1. Click **+ Add a permission** → **APIs my organization uses**
2. Search for **Power BI Service** (or `00000009-0000-0000-c000-000000000000`)
3. Click **Application permissions**
4. Search for and select:
   - `Tenant.Read.All` (to read workspace users via Admin API)
5. Click **Add permissions**

**Grant Admin Consent**

1. Back on the **API permissions** page, click **Grant admin consent for [your tenant]**
2. Click **Yes**
3. All permissions should show a green checkmark ✅

#### Enable Power BI Admin API for Service Principals

The Power BI Admin API requires an additional step:

1. Go to [Power BI Admin Portal](https://app.powerbi.com/admin-portal) (you need to be a Power BI admin)
2. Go to **Tenant settings**
3. Find **"Allow service principals to use Power BI APIs"**
4. Enable it
5. Set it to **The entire organization** or add a specific security group containing your app's service principal
6. Click **Apply**

> ⚠️ **Note:** This setting can take up to 15 minutes to take effect after enabling.

### 4.2 Information You'll Need

Before running the test, gather these values:

| Value | Where to find it |
|---|---|
| `CLIENT_ID` | App registration → Overview → Application (client) ID |
| `CLIENT_SECRET` | App registration → Certificates & secrets → Secret value |
| `TENANT_ID` | App registration → Overview → Directory (tenant) ID |
| `WORKSPACE_ID` | Power BI → Workspace → URL contains `/groups/{workspaceId}` |
| `SECURITY_GROUP_ID` | Entra ID → Groups → Your security group → Object ID |
| `TEST_USER_EMAIL` | The email of the user you want to test with |

### 4.3 Node.js

- Node.js 16 or higher installed
- npm (comes with Node.js)

---

## 5. Test App Code

### 5.1 Project Setup

Create a new folder called `pbi-cache-test` and add the following files:

### 5.2 package.json

```json
{
  "name": "pbi-cache-test",
  "version": "1.0.0",
  "description": "Test Power BI Admin API cache timing",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "axios": "^1.6.0",
    "dotenv": "^16.3.0",
    "express": "^4.18.0",
    "@azure/msal-node": "^2.6.0"
  }
}
```

### 5.3 .env

```ini
# Azure AD / Entra App Registration
CLIENT_ID=your-app-client-id
CLIENT_SECRET=your-app-client-secret
TENANT_ID=your-tenant-id

# Power BI Workspace
WORKSPACE_ID=your-workspace-id

# Security Group to add user to
SECURITY_GROUP_ID=your-security-group-id

# Test User (the user you want to test with)
TEST_USER_EMAIL=usera@domain.com

# App Config
PORT=3000
```

### 5.4 index.js

```javascript
require('dotenv').config();
const express = require('express');
const axios = require('axios');
const msal = require('@azure/msal-node');

const app = express();
app.use(express.json());

const {
    CLIENT_ID,
    CLIENT_SECRET,
    TENANT_ID,
    WORKSPACE_ID,
    SECURITY_GROUP_ID,
    TEST_USER_EMAIL,
    PORT = 3000
} = process.env;

// ============================================
// MSAL - App-Only Token (no user context needed)
// ============================================
const msalClient = new msal.ConfidentialClientApplication({
    auth: {
        clientId: CLIENT_ID,
        authority: `https://login.microsoftonline.com/${TENANT_ID}`,
        clientSecret: CLIENT_SECRET
    }
});

async function getGraphToken() {
    const result = await msalClient.acquireTokenByClientCredential({
        scopes: ['https://graph.microsoft.com/.default']
    });
    return result.accessToken;
}

async function getPowerBIAdminToken() {
    const result = await msalClient.acquireTokenByClientCredential({
        scopes: ['https://analysis.windows.net/powerbi/api/.default']
    });
    return result.accessToken;
}

// ============================================
// GRAPH API - Add/Remove user from security group
// ============================================
async function getUserByEmail(email) {
    const token = await getGraphToken();
    const response = await axios.get(
        `https://graph.microsoft.com/v1.0/users/${email}`,
        { headers: { 'Authorization': `Bearer ${token}` } }
    );
    return response.data;
}

async function addUserToGroup(userId, groupId) {
    const token = await getGraphToken();
    try {
        await axios.post(
            `https://graph.microsoft.com/v1.0/groups/${groupId}/members/$ref`,
            { '@odata.id': `https://graph.microsoft.com/v1.0/directoryObjects/${userId}` },
            { headers: { 'Authorization': `Bearer ${token}`, 'Content-Type': 'application/json' } }
        );
        return { success: true, message: 'User added to group' };
    } catch (error) {
        if (error.response?.status === 400 &&
            error.response?.data?.error?.message?.includes('already exist')) {
            return { success: true, message: 'User already in group' };
        }
        throw error;
    }
}

async function removeUserFromGroup(userId, groupId) {
    const token = await getGraphToken();
    try {
        await axios.delete(
            `https://graph.microsoft.com/v1.0/groups/${groupId}/members/${userId}/$ref`,
            { headers: { 'Authorization': `Bearer ${token}` } }
        );
        return { success: true };
    } catch (error) {
        return { success: false, error: error.message };
    }
}

// ============================================
// POWER BI ADMIN API - Check workspace users
// ============================================
async function getWorkspaceUsers() {
    const token = await getPowerBIAdminToken();
    const response = await axios.get(
        `https://api.powerbi.com/v1.0/myorg/admin/groups/${WORKSPACE_ID}/users`,
        { headers: { 'Authorization': `Bearer ${token}` } }
    );
    return response.data.value;
}

// ============================================
// API ROUTES
// ============================================

// Step 1: Remove user from group (to reset the test)
app.post('/api/reset', async (req, res) => {
    try {
        const user = await getUserByEmail(TEST_USER_EMAIL);
        const result = await removeUserFromGroup(user.id, SECURITY_GROUP_ID);
        res.json({ success: true, message: `Removed ${TEST_USER_EMAIL} from group`, userId: user.id });
    } catch (error) {
        res.json({ success: false, error: error.message });
    }
});

// Step 2: Add user to group (simulates clicking "Analytics")
app.post('/api/add-user', async (req, res) => {
    try {
        const user = await getUserByEmail(TEST_USER_EMAIL);
        const result = await addUserToGroup(user.id, SECURITY_GROUP_ID);
        res.json({
            success: true,
            message: result.message,
            userId: user.id,
            userEmail: TEST_USER_EMAIL,
            groupId: SECURITY_GROUP_ID,
            timestamp: new Date().toISOString()
        });
    } catch (error) {
        res.json({ success: false, error: error.message });
    }
});

// Step 3: Check if user appears in workspace users (the cache check)
app.get('/api/check-workspace', async (req, res) => {
    try {
        const users = await getWorkspaceUsers();

        // Check if test user is in the list
        const found = users.find(u =>
            u.emailAddress?.toLowerCase() === TEST_USER_EMAIL.toLowerCase() ||
            u.identifier?.toLowerCase() === TEST_USER_EMAIL.toLowerCase()
        );

        res.json({
            found: !!found,
            userDetails: found || null,
            totalUsersInWorkspace: users.length,
            timestamp: new Date().toISOString()
        });
    } catch (error) {
        res.json({ found: false, error: error.message });
    }
});

// ============================================
// FRONTEND - Test Dashboard
// ============================================
app.get('/', (req, res) => {
    res.send(`
    <!DOCTYPE html>
    <html>
    <head>
        <title>Power BI Cache Timing Test</title>
        <style>
            * { box-sizing: border-box; }
            body {
                font-family: 'Segoe UI', Arial, sans-serif;
                max-width: 900px;
                margin: 0 auto;
                padding: 20px;
                background: #f5f5f5;
            }
            .card {
                background: white;
                border-radius: 8px;
                padding: 25px;
                margin-bottom: 20px;
                box-shadow: 0 2px 10px rgba(0,0,0,0.08);
            }
            h1 { color: #333; margin-top: 0; }
            h2 { color: #444; margin-top: 0; }
            .config {
                background: #f8f9fa;
                padding: 12px;
                border-radius: 6px;
                margin: 10px 0;
                font-family: monospace;
                font-size: 13px;
            }
            .btn {
                padding: 12px 24px;
                border: none;
                border-radius: 6px;
                font-size: 15px;
                font-weight: 600;
                cursor: pointer;
                margin: 5px 5px 5px 0;
            }
            .btn:disabled { opacity: 0.5; cursor: not-allowed; }
            .btn-red { background: #dc3545; color: white; }
            .btn-green { background: #28a745; color: white; }
            .btn-blue { background: #0078d4; color: white; }
            .btn-orange { background: #fd7e14; color: white; }
            .log-box {
                background: #1e1e1e;
                color: #d4d4d4;
                padding: 20px;
                border-radius: 6px;
                font-family: 'Cascadia Code', 'Consolas', monospace;
                font-size: 13px;
                max-height: 500px;
                overflow-y: auto;
                line-height: 1.6;
                white-space: pre-wrap;
            }
            .log-success { color: #4ec9b0; }
            .log-fail { color: #f44747; }
            .log-warn { color: #dcdcaa; }
            .log-info { color: #9cdcfe; }
            .log-time { color: #808080; }
            .status-bar {
                padding: 15px;
                border-radius: 6px;
                margin: 15px 0;
                font-size: 16px;
                font-weight: 600;
            }
            .status-waiting { background: #fff3cd; color: #856404; }
            .status-found { background: #d4edda; color: #155724; }
            .status-timeout { background: #f8d7da; color: #721c24; }
            .status-idle { background: #e2e3e5; color: #383d41; }
            .countdown {
                font-size: 48px;
                font-weight: bold;
                color: #0078d4;
                text-align: center;
                margin: 10px 0;
            }
            .poll-count { text-align: center; color: #666; }
        </style>
    </head>
    <body>
        <div class="card">
            <h1>🧪 Power BI Cache Timing Test</h1>
            <p>This app tests how long it takes for Power BI's authorization cache to
               recognize a user after they're added to a security group.</p>
            <div class="config">
                <strong>Test User:</strong> ${TEST_USER_EMAIL}<br/>
                <strong>Security Group:</strong> ${SECURITY_GROUP_ID}<br/>
                <strong>Workspace:</strong> ${WORKSPACE_ID}
            </div>
        </div>

        <div class="card">
            <h2>Test Controls</h2>
            <button class="btn btn-red" onclick="resetTest()" id="btnReset">
                🗑️ Step 1: Reset (Remove User from Group)
            </button>
            <button class="btn btn-green" onclick="startTest()" id="btnStart" disabled>
                🚀 Step 2: Add User & Start Monitoring
            </button>
            <button class="btn btn-orange" onclick="stopPolling()" id="btnStop" disabled>
                ⏹️ Stop Monitoring
            </button>
            <button class="btn btn-blue" onclick="openPowerBI()" id="btnPBI" disabled>
                🔗 Open Power BI Workspace (Test Access)
            </button>
        </div>

        <div class="card">
            <h2>Status</h2>
            <div class="status-bar status-idle" id="statusBar">
                Ready. Click "Reset" to begin the test.
            </div>
            <div class="countdown" id="countdown"></div>
            <div class="poll-count" id="pollCount"></div>
        </div>

        <div class="card">
            <h2>Live Log</h2>
            <div class="log-box" id="logBox"></div>
        </div>

        <script>
            let pollInterval = null;
            let countdownInterval = null;
            let startTime = null;
            let pollNum = 0;
            let userFoundTime = null;
            const POLL_EVERY_MS = 15000;
            const MAX_DURATION_MS = 5 * 60 * 1000;

            const logBox = document.getElementById('logBox');
            const statusBar = document.getElementById('statusBar');
            const countdown = document.getElementById('countdown');
            const pollCount = document.getElementById('pollCount');

            function log(message, type) {
                const time = new Date().toLocaleTimeString();
                const className = type ? 'class="log-' + type + '"' : '';
                logBox.innerHTML += '<span class="log-time">[' + time + ']</span> <span ' + className + '>' + message + '</span>\\n';
                logBox.scrollTop = logBox.scrollHeight;
            }

            function setStatus(text, type) {
                statusBar.textContent = text;
                statusBar.className = 'status-bar status-' + type;
            }

            function updateCountdown() {
                if (!startTime) return;
                var elapsed = Date.now() - startTime;
                var remaining = Math.max(0, MAX_DURATION_MS - elapsed);
                var mins = Math.floor(remaining / 60000);
                var secs = Math.floor((remaining % 60000) / 1000);
                countdown.textContent = mins + ':' + secs.toString().padStart(2, '0');
            }

            async function resetTest() {
                log('━━━ RESETTING TEST ━━━', 'warn');
                log('Removing user from security group...', 'info');
                try {
                    var res = await fetch('/api/reset', { method: 'POST' });
                    var data = await res.json();
                    if (data.success) {
                        log('✅ User removed from group', 'success');
                        log('⏳ Wait ~30 seconds for removal to propagate, then click Step 2', 'warn');
                        setStatus('Reset done. Wait 30s, then click "Add User & Start Monitoring"', 'idle');
                        document.getElementById('btnStart').disabled = false;
                    } else {
                        log('⚠️  Reset result: ' + data.error, 'fail');
                    }
                } catch (error) {
                    log('❌ Reset failed: ' + error.message, 'fail');
                }
            }

            async function startTest() {
                log('', '');
                log('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━', 'warn');
                log('  STARTING POWER BI CACHE TIMING TEST', 'warn');
                log('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━', 'warn');

                log('Adding user to security group...', 'info');
                try {
                    var res = await fetch('/api/add-user', { method: 'POST' });
                    var data = await res.json();
                    if (data.success) {
                        log('✅ ' + data.message, 'success');
                        log('   User: ' + data.userEmail, 'info');
                        log('   Group: ' + data.groupId, 'info');
                        log('   Time: ' + data.timestamp, 'info');
                    } else {
                        log('❌ Failed to add user: ' + data.error, 'fail');
                        return;
                    }
                } catch (error) {
                    log('❌ Failed: ' + error.message, 'fail');
                    return;
                }

                startTime = Date.now();
                pollNum = 0;
                userFoundTime = null;

                log('', '');
                log('🔄 Starting to poll Power BI Admin API every ' + (POLL_EVERY_MS/1000) + 's...', 'info');
                log('⏱️  Max wait: ' + (MAX_DURATION_MS/60000) + ' minutes', 'info');
                log('', '');

                setStatus('Monitoring... Waiting for user to appear in workspace', 'waiting');
                document.getElementById('btnStart').disabled = true;
                document.getElementById('btnStop').disabled = false;
                document.getElementById('btnReset').disabled = true;

                countdownInterval = setInterval(updateCountdown, 1000);
                await checkWorkspace();

                pollInterval = setInterval(async function() {
                    var elapsed = Date.now() - startTime;
                    if (elapsed >= MAX_DURATION_MS) {
                        stopPolling();
                        log('', '');
                        log('⏰ MAX TIME REACHED (5 minutes)', 'fail');
                        log('User was NOT found in workspace users list within 5 minutes', 'fail');
                        log('', '');
                        log('📋 CONCLUSION:', 'warn');
                        log('   The Power BI Admin API did not reflect the membership in 5 min', 'warn');
                        log('   Recommendation: Use a fixed countdown timer instead of polling', 'warn');
                        setStatus('TIMEOUT: User not found in 5 minutes', 'timeout');
                        document.getElementById('btnPBI').disabled = false;
                        return;
                    }
                    await checkWorkspace();
                }, POLL_EVERY_MS);
            }

            async function checkWorkspace() {
                pollNum++;
                var elapsed = Math.round((Date.now() - startTime) / 1000);
                pollCount.textContent = 'Poll #' + pollNum + ' | Elapsed: ' + elapsed + 's';

                log('🔍 Poll #' + pollNum + ' (elapsed: ' + elapsed + 's) — Checking workspace users...', 'info');

                try {
                    var res = await fetch('/api/check-workspace');
                    var data = await res.json();

                    if (data.found) {
                        userFoundTime = Date.now();
                        var duration = Math.round((userFoundTime - startTime) / 1000);

                        log('', '');
                        log('🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉', 'success');
                        log('✅ USER FOUND IN WORKSPACE!', 'success');
                        log('   Time taken: ' + duration + ' seconds (' + (duration/60).toFixed(1) + ' minutes)', 'success');
                        log('   User details: ' + JSON.stringify(data.userDetails, null, 2), 'success');
                        log('   Total users in workspace: ' + data.totalUsersInWorkspace, 'info');
                        log('🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉🎉', 'success');
                        log('', '');
                        log('👉 NOW: Click "Open Power BI Workspace" to verify actual access', 'warn');
                        log('   If it works → Admin API is a reliable cache indicator ✅', 'warn');
                        log('   If access denied → Admin API updates before runtime cache ❌', 'warn');

                        stopPolling();
                        setStatus('✅ USER FOUND after ' + duration + ' seconds! Now test actual access.', 'found');
                        countdown.textContent = duration + 's';
                        document.getElementById('btnPBI').disabled = false;

                    } else {
                        log('   ❌ Not found yet (workspace has ' + data.totalUsersInWorkspace + ' users)', 'fail');
                    }
                } catch (error) {
                    log('   ⚠️  Error: ' + error.message, 'fail');
                }
            }

            function stopPolling() {
                if (pollInterval) clearInterval(pollInterval);
                if (countdownInterval) clearInterval(countdownInterval);
                pollInterval = null;
                document.getElementById('btnStop').disabled = true;
                document.getElementById('btnReset').disabled = false;
                log('⏹️  Polling stopped', 'info');
            }

            function openPowerBI() {
                var url = 'https://app.powerbi.com/groups/${WORKSPACE_ID}';
                log('🔗 Opening Power BI: ' + url, 'info');
                log('   → If access works: Admin API is reliable for cache check ✅', 'warn');
                log('   → If access denied: Admin API updates BEFORE runtime cache ❌', 'warn');
                window.open(url, '_blank');
            }
        </script>
    </body>
    </html>
    `);
});

// ============================================
// START SERVER
// ============================================
app.listen(PORT, () => {
    console.log('╔════════════════════════════════════════════════╗');
    console.log('║  Power BI Cache Timing Test                   ║');
    console.log('╚════════════════════════════════════════════════╝');
    console.log('Server:     http://localhost:' + PORT);
    console.log('User:       ' + TEST_USER_EMAIL);
    console.log('Group:      ' + SECURITY_GROUP_ID);
    console.log('Workspace:  ' + WORKSPACE_ID);
    console.log('');
    console.log('Open browser and follow the steps.');
});
```

---

## 6. How to Run the Test

### Step-by-step

```bash
# 1. Create the project folder
mkdir pbi-cache-test
cd pbi-cache-test

# 2. Create the files (package.json, .env, index.js) from section 5 above

# 3. Install dependencies
npm install

# 4. Fill in the .env file with your values

# 5. Start the app
node index.js

# 6. Open browser
# Go to http://localhost:3000
```

### In the browser

| Step | Action | What happens |
|------|--------|-------------|
| 1 | Click **"🗑️ Reset"** | Removes the test user from the security group (clean slate) |
| 2 | Wait 30 seconds | Let the removal propagate through Entra |
| 3 | Click **"🚀 Add User & Start Monitoring"** | Adds user to group AND starts polling the Power BI Admin API every 15 seconds |
| 4 | Watch the live log | You'll see each poll attempt and whether the user was found |
| 5 | When user is found (or 5 min timeout) | The **"🔗 Open Power BI"** button becomes active |
| 6 | Click **"🔗 Open Power BI Workspace"** | Opens PBI in a new tab — **this is the real test** |
| 7 | Record the result | Did the user get access? Or access denied? |

---

## 7. Interpreting Results

### Result A: Admin API Found User AND Power BI Access Works ✅

```
Admin API: User found after 45 seconds
Power BI:  ✅ User can access workspace

CONCLUSION: The Admin API is a RELIABLE indicator.
ACTION:     Implement polling in production app.
            When user clicks "Analytics":
            1. Add to group
            2. Show provisioning page
            3. Poll Admin API every 15s
            4. When user found → redirect to PBI
```

### Result B: Admin API Found User BUT Power BI Access Denied ❌

```
Admin API: User found after 45 seconds
Power BI:  ❌ Access denied (403)

CONCLUSION: Admin API updates BEFORE the runtime cache.
            Admin API is NOT a reliable indicator.
ACTION:     Use a fixed countdown timer (5 minutes).
            When user clicks "Analytics":
            1. Add to group
            2. Show "Provisioning..." page with 5-min countdown
            3. After 5 min → redirect to PBI
            (No polling — just wait)
```

### Result C: Admin API Did NOT Find User in 5 Minutes ⏰

```
Admin API: User NOT found after 5 minutes
Power BI:  ❌ Access denied

CONCLUSION: Both Admin API and runtime cache are slow.
ACTION:     Use a fixed countdown timer (5-10 minutes).
            Consider showing a message like:
            "Your workspace is being prepared.
             Please try again in a few minutes."
```

---

## 8. What to Implement in Production (Based on Results)

### If Result A (Polling Works)

Change the "Analytics" click handler in MedAdv:

```
Current:  Click → Add to group → Redirect to PBI immediately
New:      Click → Add to group → Show provisioning page → Poll → Redirect when ready
```

### If Result B or C (Polling Not Reliable)

Change the "Analytics" click handler in MedAdv:

```
Current:  Click → Add to group → Redirect to PBI immediately
New:      Click → Add to group → Show "Preparing workspace..." with 5 min countdown → Redirect after countdown
```

---

## 9. Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `401 Unauthorized` on Admin API | Token issue | Check CLIENT_ID, CLIENT_SECRET, TENANT_ID in .env |
| `403 Forbidden` on Admin API | Permission issue | Ensure "Allow service principals to use Power BI APIs" is enabled in Power BI Admin Portal |
| `404 Not Found` on Admin API | Wrong workspace ID | Double-check WORKSPACE_ID in .env |
| `400 Bad Request` on Graph add member | User already in group | This is fine — the code handles it |
| `AADSTS700016` | Wrong CLIENT_ID | Verify the app registration client ID |
| `AADSTS7000215` | Wrong CLIENT_SECRET | Regenerate the secret in Azure Portal |
