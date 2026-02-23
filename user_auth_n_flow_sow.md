# Keycloak Integration with Entra ID - Authentication Flow


---

## Table of Contents

1. [Overview](#overview)
2. [User Categories](#user-categories)
3. [Detailed Authentication Flow](#detailed-authentication-flow)
   - [Phase 1: Initial Authentication & Guest Invitation](#phase-1-initial-authentication--guest-invitation)
   - [Phase 2: Consent & Provisioning](#phase-2-consent--provisioning)
   - [Phase 3: Power BI Access & Authentication Routing](#phase-3-power-bi-access--authentication-routing)
4. [Visual Flow Diagram](#visual-flow-diagram)
5. [Implementation Guide](#implementation-guide-for-developers)

---

## Overview

This document describes the authentication flow for end users accessing Power BI/Fabric workspaces through the MedAdv application using Keycloak SSO, with Microsoft Entra ID B2B guest access integration.

The flow supports multiple identity provider scenarios and ensures seamless authentication while maintaining security and compliance requirements.

### Key Features

- **Multi-IdP Support**: Handles users from various identity providers
- **Automated Guest Provisioning**: Uses Microsoft Graph API for B2B invitation
- **Role-Based License Assignment**: Automatically assigns appropriate Power BI licenses
- **Flexible Authentication**: Supports Entra, MSA, federated IdP, and Email OTP
- **Security Compliance**: Requires explicit user consent at multiple stages

---

## User Categories

The system supports four distinct user types, each with different authentication methods:

| User Type | Description | Authentication Method | Example |
|-----------|-------------|----------------------|---------|
| **Entra-to-Entra Guest** | User from another Entra organization | Home tenant (Entra IdP) | user@partner.com (Entra tenant) |
| **Non-Entra with MSA Account** | User with personal Microsoft Account | Home tenant (Microsoft Account) | user@gmail.com (with MSA) |
| **Non-Entra Federated with Entra** | User from IdP federated with Entra | Home tenant (Federated IdP) | user@company.com (via Okta/ADFS) |
| **Non-Entra, No MSA, No Federation** | User with no existing Microsoft identity | Email One-Time Passcode (OTP) | user@startup.com (OTP only) |

---

## Detailed Authentication Flow

### Phase 1: Initial Authentication & Guest Invitation

#### Step 1: User Login to MedAdv
- End user navigates to MedAdv application
- User authenticates using Keycloak SSO
- User's corporate Identity Provider (IdP) is pre-federated with Keycloak
- Keycloak issues SAML/OIDC token to MedAdv application

**Technical Details:**
- Protocol: SAML 2.0 or OpenID Connect
- User identity: kc@contoso.com
- Session established in MedAdv application

#### Step 2: Power BI Access Request
- Authenticated user clicks on Power BI/Fabric workspace link
- This is the user's first-time access attempt
- MedAdv backend intercepts the request to check guest status

**Business Logic:**
- Trigger: User interaction with Power BI link
- Action: Initiate guest user validation flow

#### Step 3: Guest User Check
- MedAdv backend calls Microsoft Graph API to verify user existence
- Searches for user in the resource Entra tenant (hosting Power BI workspace)

**API Call:**
```http
GET https://graph.microsoft.com/v1.0/users?$filter=mail eq 'kc@contoso.com'
```

**Possible Outcomes:**
- User found (200): Proceed to license check
- User not found (404): Continue to Step 4
- Error (5xx): Implement retry logic

#### Step 4: Guest Invitation Creation
- If user doesn't exist, backend creates B2B guest invitation
- Uses Microsoft Graph API `/invitations` endpoint

**API Call:**
```http
POST https://graph.microsoft.com/v1.0/invitations
Content-Type: application/json

{
  "invitedUserEmailAddress": "kc@contoso.com",
  "inviteRedirectUrl": "https://app.powerbi.com/groups/{workspaceId}",
  "sendInvitationMessage": false,
  "invitedUserDisplayName": "KC User",
  "invitedUserType": "Guest"
}
```

**Parameters:**
- `invitedUserEmailAddress`: User's email from Keycloak token
- `inviteRedirectUrl`: Target Power BI workspace URL
- `sendInvitationMessage`: Set to `false` for seamless flow
- `invitedUserType`: Always "Guest" for external users

#### Step 5: Redemption URL Retrieval
- Graph API returns invitation object
- Backend extracts `inviteRedeemUrl` from response

**Response Example:**
```json
{
  "id": "7b92124c-9fa9-406f-8b8e-225270c3730f",
  "inviteRedeemUrl": "https://login.microsoftonline.com/redeem?...",
  "invitedUserDisplayName": "KC User",
  "invitedUserEmailAddress": "kc@contoso.com",
  "status": "PendingAcceptance",
  "invitedUser": {
    "id": "a87e3c80-7b72-4f92-a982-e895eea5730f"
  }
}
```

**Key Fields:**
- `inviteRedeemUrl`: URL for guest invitation redemption
- `invitedUser.id`: Guest user object ID (needed for license assignment)
- `status`: Should be "PendingAcceptance"

#### Step 6: User Redirection to Consent
- MedAdv backend immediately redirects user to `inviteRedeemUrl`
- This initiates the B2B guest invitation redemption process
- User's browser navigates to Microsoft login page

**HTTP Response:**
```http
HTTP/1.1 302 Found
Location: https://login.microsoftonline.com/redeem?rd=...
```

---

### Phase 2: Consent & Provisioning

#### Step 7: Microsoft Consent Screen
- User is presented with Microsoft consent screen
- Screen displays requesting organization details and required permissions

**Consent Screen Elements:**
- **Organization Name**: Resource tenant name (e.g., "Contoso Corp")
- **Permissions Requested**: 
  - Sign you in and read your profile
  - Maintain access to data you have given it access to
- **User Actions**: 
  - **[Cancel]** - Abort the process
  - **[Accept]** - Grant permissions and continue

**Critical Requirement:**
> ⚠️ **User must explicitly click Accept to proceed.** Canceling will abort the guest invitation flow.

**After Acceptance:**
- Guest user account created in resource tenant
- User Principal Name (UPN): `kc_contoso.com#EXT#@resourcetenant.onmicrosoft.com`
- User Type: Guest
- Source: Invited User

#### Step 8: Group Membership Assignment
- Upon successful consent, MedAdv backend receives callback
- Backend adds user to designated security group(s)
- Group membership determines workspace access permissions

**API Call:**
```http
POST https://graph.microsoft.com/v1.0/groups/{groupId}/members/$ref
Content-Type: application/json

{
  "@odata.id": "https://graph.microsoft.com/v1.0/users/{userId}"
}
```

**Group Strategy:**
- **Power BI Viewers Group**: Read-only access
- **Power BI Contributors Group**: Edit capabilities
- **Power BI Admins Group**: Full administrative access

**Implementation Note:**
- Group assignment may be role-based from MedAdv user profile
- Multiple group memberships possible based on user role

#### Step 9: License Assignment Preparation
- MedAdv application determines required Power BI license
- Decision based on user role and workspace requirements

**License Types:**
- **Power BI Free**: Basic features (limited for guest users)
- **Power BI Pro**: Full collaboration features
- **Power BI Premium Per User (PPU)**: Advanced features
- **Fabric Capacity License**: Access to Fabric workspaces

**Role-to-License Mapping Example:**
```javascript
const licenseMappings = {
  'viewer': 'POWER_BI_STANDARD', // Free
  'analyst': 'POWER_BI_PRO',
  'developer': 'POWER_BI_PREMIUM_PER_USER',
  'admin': 'POWER_BI_PREMIUM_PER_USER'
};
```

#### Step 10: License Assignment via Graph API
- Backend calls Graph API to assign the determined license
- Uses the user ID from Step 5

**API Call:**
```http
POST https://graph.microsoft.com/v1.0/users/{userId}/assignLicense
Content-Type: application/json

{
  "addLicenses": [
    {
      "skuId": "f8a1db68-be16-40ed-86d5-cb42ce701560",
      "disabledPlans": []
    }
  ],
  "removeLicenses": []
}
```

**SKU IDs (Examples):**
- Power BI Pro: `f8a1db68-be16-40ed-86d5-cb42ce701560`
- Power BI Premium Per User: `edd24c03-0b36-4a3a-9607-3dc49e4c4069`

**Prerequisites:**
- Available licenses in tenant subscription
- Sufficient permissions (User.ReadWrite.All)
- User account in valid state

#### Step 11: License Confirmation
- User may see additional consent screen for license-specific permissions
- User must accept to complete license assignment

**Verification:**
Backend should verify license assignment status:
```http
GET https://graph.microsoft.com/v1.0/users/{userId}/licenseDetails
```

**Expected Response:**
```json
{
  "value": [
    {
      "id": "...",
      "skuId": "f8a1db68-be16-40ed-86d5-cb42ce701560",
      "skuPartNumber": "POWER_BI_PRO",
      "servicePlans": [...]
    }
  ]
}
```

---

### Phase 3: Power BI Access & Authentication Routing

#### Step 12: Power BI Redirection
- User is redirected to Power BI/Fabric workspace URL
- Browser navigates to workspace

**URL Format:**
```
https://app.powerbi.com/groups/{workspaceId}/list?experience=power-bi
```

**Alternative Formats:**
- Report: `https://app.powerbi.com/groups/{workspaceId}/reports/{reportId}`
- Dashboard: `https://app.powerbi.com/groups/{workspaceId}/dashboards/{dashboardId}`
- Dataset: `https://app.powerbi.com/groups/{workspaceId}/datasets/{datasetId}`

#### Step 13: Authentication Request to Resource Entra
- Power BI service detects unauthenticated user
- Redirects to resource Entra tenant for authentication
- Entra ID detects user is a guest (external user)

**Authentication Flow Initiation:**
```http
HTTP/1.1 302 Found
Location: https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/authorize?
  client_id={powerbi-client-id}&
  response_type=code&
  redirect_uri=https://app.powerbi.com/&
  scope=https://analysis.windows.net/powerbi/api/.default&
  state={state}
```

#### Step 14: Home Tenant Authentication Routing

Entra ID determines the appropriate authentication method based on the user's identity configuration.

##### Step 14.1: Entra-to-Entra Guest User

**Scenario:** User's home organization uses Microsoft Entra ID

**Flow:**
1. Resource Entra redirects to user's home Entra tenant
2. User sees their corporate login page (e.g., login.microsoftonline.com/contoso.com)
3. User authenticates with corporate credentials (username/password + MFA)
4. Home Entra tenant issues SAML/OIDC assertion
5. Assertion sent back to resource Entra tenant

**Authentication Details:**
- Protocol: SAML 2.0 or OpenID Connect
- Identity Source: User's home Entra directory
- Claims: UPN, email, name, groups (if configured)

**Example User Journey:**
```
Resource Entra → Home Entra (contoso.com) → User Login → MFA → 
Token Issued → Resource Entra
```

##### Step 14.2: Non-Entra with Microsoft Account (MSA)

**Scenario:** User has personal Microsoft Account (e.g., @gmail.com, @yahoo.com)

**Flow:**
1. Entra redirects to Microsoft Account login (login.live.com)
2. User sees Microsoft Account login page
3. User authenticates with MSA credentials
4. MSA service may require additional verification (SMS, email code)
5. MSA service issues token
6. Token sent to resource Entra tenant

**Authentication Details:**
- Protocol: OpenID Connect
- Identity Source: Microsoft Account service
- Account Types: Outlook.com, Hotmail.com, Live.com, Gmail.com (with MSA), etc.

**Example User Journey:**
```
Resource Entra → login.live.com → User Login → SMS Verification → 
Token Issued → Resource Entra
```

##### Step 14.3: Non-Entra Federated with Entra

**Scenario:** User's organization uses third-party IdP federated with Entra (e.g., Okta, Keycloak, ADFS, Ping Identity)

**Flow:**
1. Resource Entra redirects to user's federated IdP
2. User sees their corporate IdP login page
3. User authenticates with corporate credentials
4. Federated IdP performs authentication (may include SSO if already logged in)
5. IdP issues SAML/OIDC assertion
6. Assertion sent to resource Entra via federation trust

**Authentication Details:**
- Protocol: SAML 2.0 or OpenID Connect (based on federation config)
- Identity Source: Corporate IdP (Keycloak, Okta, ADFS, etc.)
- Claims Mapping: Configured in federation settings

**Example User Journey:**
```
Resource Entra → Keycloak (contoso.com) → [SSO Session Exists] → 
SAML Assertion → Resource Entra
```

**Benefits:**
- Single Sign-On (SSO) if user already authenticated to corporate IdP
- Corporate password policies apply
- Centralized authentication management

##### Step 14.4: Non-Entra, No MSA, No Federation

**Scenario:** User has no existing Microsoft identity and organization not federated

**Flow:**
1. Entra initiates Email One-Time Passcode (OTP) flow
2. System sends 6-8 digit code to user's email (kc@contoso.com)
3. User receives OTP email from Microsoft
4. User enters OTP code in authentication page
5. Entra validates OTP code
6. Upon successful validation, session token issued

**OTP Email Details:**
- **From:** Microsoft Invitations (invites@microsoft.com)
- **Subject:** "Your verification code"
- **Code Validity:** 30 minutes
- **Retry Limit:** 3 attempts

**Authentication Page:**
```
┌────────────────────────────────────┐
│  Enter the code sent to            │
│  kc@contoso.com                    │
│                                    │
│  Code: [_][_][_][_][_][_][_][_]    │
│                                    │
│  [Verify]    [Resend Code]         │
└────────────────────────────────────┘
```

**Example User Journey:**
```
Resource Entra → OTP Email Sent → User Checks Email → 
Enters Code → Validation → Session Token → Resource Entra
```

**Considerations:**
- Email delivery time (typically 1-2 minutes)
- Spam/junk folder filtering
- Corporate email security policies
- User experience (additional friction)

#### Step 15: Token Reception at Resource Tenant

- Resource Entra tenant receives authentication token/assertion from home tenant
- Entra performs token validation and claim mapping

**Token Validation Steps:**
1. **Signature Verification**: Validates token signature using public key
2. **Issuer Verification**: Confirms token issued by trusted authority
3. **Audience Verification**: Ensures token intended for this resource
4. **Expiration Check**: Validates token not expired
5. **Claims Extraction**: Extracts user identity claims

**Identity Mapping:**
```
Home Tenant Identity: kc@contoso.com
                 ↓
Resource Tenant Guest UPN: kc_contoso.com#EXT#@resourcetenant.onmicrosoft.com
                 ↓
Email Attribute: kc@contoso.com
```

**Claims Mapping Example:**
```json
{
  "aud": "https://analysis.windows.net/powerbi/api",
  "iss": "https://sts.windows.net/{resource-tenant-id}/",
  "iat": 1708704000,
  "exp": 1708707600,
  "name": "KC User",
  "email": "kc@contoso.com",
  "oid": "{guest-user-object-id}",
  "tid": "{resource-tenant-id}",
  "unique_name": "kc_contoso.com#EXT#@resourcetenant.onmicrosoft.com",
  "upn": "kc_contoso.com#EXT#@resourcetenant.onmicrosoft.com"
}
```

#### Step 16: Power BI Workspace Access Granted

- Entra issues access token for Power BI service
- Power BI validates user authorization

**Authorization Checks:**
1. **License Validation**: 
   - User has active Power BI license
   - License type supports requested operations
   
2. **Group Membership Validation**:
   - User is member of authorized security group
   - Group has permissions to workspace
   
3. **Workspace Role Validation**:
   - User assigned appropriate role (Viewer, Contributor, Member, Admin)
   - Role permissions match requested operations

4. **Conditional Access Policies**:
   - Device compliance (if required)
   - Location/IP restrictions (if configured)
   - MFA requirements met

**Access Token Structure:**
```json
{
  "aud": "https://analysis.windows.net/powerbi/api",
  "appid": "{power-bi-client-id}",
  "roles": ["Report.Read.All", "Dataset.Read.All"],
  "scp": "Dataset.Read.All Report.Read.All",
  "wids": ["{workspace-role-id}"]
}
```

**Success Response:**
- User successfully accesses Power BI/Fabric workspace
- Workspace content loads based on user permissions
- User can interact with reports, dashboards, datasets per role

---

## Visual Flow Diagram

### Sequence Diagram

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Flow diagram" 
     src="https://github.com/user-attachments/assets/9a911904-8916-4e38-bd24-7159dd00fe04"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>

### Simplified Flow Diagram

<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Flow diagram" 
     src="https://github.com/user-attachments/assets/beaaa58e-899a-4ecc-98ee-84ada1d3c0ba"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>

---

## Implementation Guide for Developers

### Prerequisites

#### 1. Azure/Entra ID Configuration

**App Registration:**
- Create app registration in resource Entra tenant
- Configure API permissions (see below)
- Generate client secret for backend application
- Configure redirect URIs

**Required API Permissions:**

| Permission | Type | Purpose |
|------------|------|---------|
| `User.Invite.All` | Application | Create B2B guest invitations |
| `User.ReadWrite.All` | Application | Read and update user properties |
| `GroupMember.ReadWrite.All` | Application | Manage group memberships |
| `Organization.Read.All` | Application | Read organization license SKUs |
| `Directory.Read.All` | Application | Read directory data |

**Admin Consent Required:** Yes (all application permissions)

**Grant Admin Consent:**
```bash
# Using Azure CLI
az ad app permission admin-consent \
  --id {application-id}
```

#### 2. Power BI Configuration

**Workspace Setup:**
- Create Power BI workspace
- Enable guest user access in workspace settings
- Configure workspace roles (Viewer, Contributor, Member, Admin)
- Link workspace to security groups

**Tenant Settings:**
- Enable "Allow Azure AD guest users to access Power BI"
- Configure "Export and sharing settings" for guests
- Set "External user permissions" appropriately

#### 3. Keycloak Configuration

**Identity Provider Setup:**
- Configure Keycloak as SAML/OIDC provider for MedAdv
- Map user attributes (email, name, roles)
- Configure session management

**User Attribute Mapping:**
```json
{
  "email": "user.email",
  "firstName": "user.firstName",
  "lastName": "user.lastName",
  "roles": "user.roles"
}
```

---

### Microsoft Graph API Implementation

#### Authentication Setup

**Using Client Credentials Flow:**

```javascript
const msal = require('@azure/msal-node');

const msalConfig = {
  auth: {
    clientId: process.env.AZURE_CLIENT_ID,
    authority: `https://login.microsoftonline.com/${process.env.AZURE_TENANT_ID}`,
    clientSecret: process.env.AZURE_CLIENT_SECRET,
  }
};

const cca = new msal.ConfidentialClientApplication(msalConfig);

async function getAccessToken() {
  const tokenRequest = {
    scopes: ['https://graph.microsoft.com/.default'],
  };
  
  try {
    const response = await cca.acquireTokenByClientCredential(tokenRequest);
    return response.accessToken;
  } catch (error) {
    console.error('Error acquiring token:', error);
    throw error;
  }
}
```

#### Core Functions

##### 1. Check if User Exists

```javascript
const axios = require('axios');

async function checkUserExists(email) {
  const accessToken = await getAccessToken();
  
  try {
    const response = await axios.get(
      `https://graph.microsoft.com/v1.0/users?$filter=mail eq '${email}'`,
      {
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    return response.data.value.length > 0 ? response.data.value[0] : null;
  } catch (error) {
    if (error.response && error.response.status === 404) {
      return null;
    }
    throw error;
  }
}
```

##### 2. Create Guest Invitation

```javascript
async function createGuestInvitation(email, displayName, redirectUrl) {
  const accessToken = await getAccessToken();
  
  const invitationData = {
    invitedUserEmailAddress: email,
    invitedUserDisplayName: displayName,
    inviteRedirectUrl: redirectUrl,
    sendInvitationMessage: false,
    invitedUserType: 'Guest'
  };
  
  try {
    const response = await axios.post(
      'https://graph.microsoft.com/v1.0/invitations',
      invitationData,
      {
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    return {
      userId: response.data.invitedUser.id,
      redeemUrl: response.data.inviteRedeemUrl,
      status: response.data.status
    };
  } catch (error) {
    console.error('Error creating invitation:', error.response?.data);
    throw error;
  }
}
```

##### 3. Add User to Group

```javascript
async function addUserToGroup(userId, groupId) {
  const accessToken = await getAccessToken();
  
  const memberData = {
    "@odata.id": `https://graph.microsoft.com/v1.0/users/${userId}`
  };
  
  try {
    await axios.post(
      `https://graph.microsoft.com/v1.0/groups/${groupId}/members/$ref`,
      memberData,
      {
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    return { success: true, message: 'User added to group successfully' };
  } catch (error) {
    if (error.response && error.response.status === 400) {
      // User might already be a member
      console.warn('User might already be a member of the group');
      return { success: true, message: 'User already a member' };
    }
    throw error;
  }
}
```

##### 4. Assign License

```javascript
async function assignLicense(userId, skuId) {
  const accessToken = await getAccessToken();
  
  const licenseData = {
    addLicenses: [
      {
        skuId: skuId,
        disabledPlans: []
      }
    ],
    removeLicenses: []
  };
  
  try {
    await axios.post(
      `https://graph.microsoft.com/v1.0/users/${userId}/assignLicense`,
      licenseData,
      {
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    return { success: true, message: 'License assigned successfully' };
  } catch (error) {
    console.error('Error assigning license:', error.response?.data);
    throw error;
  }
}
```

##### 5. Get Available Licenses

```javascript
async function getAvailableLicenses() {
  const accessToken = await getAccessToken();
  
  try {
    const response = await axios.get(
      'https://graph.microsoft.com/v1.0/subscribedSkus',
      {
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    // Filter for Power BI licenses
    const powerBILicenses = response.data.value.filter(sku => 
      sku.skuPartNumber.includes('POWER_BI')
    );
    
    return powerBILicenses.map(sku => ({
      skuId: sku.skuId,
      skuPartNumber: sku.skuPartNumber,
      available: sku.prepaidUnits.enabled - sku.consumedUnits,
      total: sku.prepaidUnits.enabled
    }));
  } catch (error) {
    console.error('Error fetching licenses:', error.response?.data);
    throw error;
  }
}
```

---

### Complete Onboarding Flow Implementation

```javascript
async function onboardGuestUser(userEmail, displayName, userRole, workspaceId) {
  try {
    console.log(`Starting onboarding for ${userEmail}...`);
    
    // Step 1: Check if user exists
    let user = await checkUserExists(userEmail);
    
    if (!user) {
      console.log('User not found, creating guest invitation...');
      
      // Step 2: Create guest invitation
      const redirectUrl = `https://app.powerbi.com/groups/${workspaceId}`;
      const invitation = await createGuestInvitation(
        userEmail,
        displayName,
        redirectUrl
      );
      
      console.log(`Invitation created. Redeem URL: ${invitation.redeemUrl}`);
      
      // Return redemption URL to redirect user
      return {
        status: 'invitation_created',
        redeemUrl: invitation.redeemUrl,
        userId: invitation.userId
      };
    }
    
    // User already exists, proceed with license and group assignment
    console.log('User exists, assigning permissions...');
    
    // Step 3: Determine group based on role
    const groupId = getRoleBasedGroupId(userRole);
    
    // Step 4: Add to group
    await addUserToGroup(user.id, groupId);
    console.log('User added to security group');
    
    // Step 5: Determine license based on role
    const licenseSkuId = getRoleBasedLicenseSkuId(userRole);
    
    // Step 6: Check license availability
    const availableLicenses = await getAvailableLicenses();
    const targetLicense = availableLicenses.find(l => l.skuId === licenseSkuId);
    
    if (!targetLicense || targetLicense.available <= 0) {
      throw new Error('No available licenses for assignment');
    }
    
    // Step 7: Assign license
    await assignLicense(user.id, licenseSkuId);
    console.log('License assigned successfully');
    
    // Step 8: Return success with Power BI URL
    return {
      status: 'onboarding_complete',
      powerBiUrl: `https://app.powerbi.com/groups/${workspaceId}`,
      userId: user.id
    };
    
  } catch (error) {
    console.error('Onboarding error:', error);
    throw {
      status: 'error',
      message: error.message,
      details: error.response?.data
    };
  }
}

// Helper function: Get group ID based on user role
function getRoleBasedGroupId(userRole) {
  const groupMappings = {
    'viewer': process.env.PBI_VIEWERS_GROUP_ID,
    'analyst': process.env.PBI_CONTRIBUTORS_GROUP_ID,
    'developer': process.env.PBI_MEMBERS_GROUP_ID,
    'admin': process.env.PBI_ADMINS_GROUP_ID
  };
  
  return groupMappings[userRole.toLowerCase()] || groupMappings['viewer'];
}

// Helper function: Get license SKU ID based on user role
function getRoleBasedLicenseSkuId(userRole) {
  const licenseMappings = {
    'viewer': process.env.PBI_PRO_SKU_ID,
    'analyst': process.env.PBI_PRO_SKU_ID,
    'developer': process.env.PBI_PPU_SKU_ID,
    'admin': process.env.PBI_PPU_SKU_ID
  };
  
  return licenseMappings[userRole.toLowerCase()] || licenseMappings['viewer'];
}
```

---




---

**End of Document**
