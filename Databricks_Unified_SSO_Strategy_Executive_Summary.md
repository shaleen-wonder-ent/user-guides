# Unified Single Sign-On Strategy for ContosoApp on Azure Databricks
## Executive Summary Document

**Document Purpose**: This document outlines the technical strategy for implementing unified identity federation for Contoso's ContosoApp application on Azure Databricks, enabling seamless access for both end-customers and internal teams.

---

## 1. Business Context and Challenge

### Current Situation
Contoso's application serves two distinct user populations:
- **End-Customers**: Users who authenticate using Keycloak Identity Provider
- **Internal Teams**: Contoso employees who authenticate using Azure Entra ID (Microsoft's enterprise identity platform)

### The Challenge
Azure Databricks has **removed support for multiple SSO providers per account** due to security concerns. This creates a critical decision point: we must choose a single authentication pathway while maintaining seamless access for both user populations.

**Critical Architectural Constraint**: Azure Databricks in Azure **only accepts authentication from Azure Entra ID**. It cannot directly authenticate against Keycloak or any other third-party identity provider.

---

## 2. Recommended Solution: SAML Federation via Keycloak

### High-Level Strategy
We will implement **Identity Provider Federation** using Keycloak as the authentication broker, connecting to Azure Entra ID through industry-standard SAML 2.0 protocol. Azure Entra ID will remain the identity provider for Databricks, while Keycloak performs the actual user authentication.

**Think of it as a "trusted authentication chain"**: 
- Databricks **only trusts** Azure Entra ID for authentication tokens
- Azure Entra ID **trusts** Keycloak to verify user credentials
- Keycloak performs the actual username/password verification for end-customers
- Internal teams can authenticate through Azure Entra directly or via Keycloak as a broker

### The Trust Chain
```
Databricks ←trusts only← Azure Entra ID ←trusts← Keycloak
```

### Why SAML Federation?
- **Industry Standard**: SAML 2.0 is the gold standard for enterprise SSO
- **Security**: Mature protocol with extensive audit trails and encryption
- **Databricks Support**: Native, well-tested SAML integration with Azure Entra
- **Flexibility**: Supports complex attribute mapping for user permissions
- **Azure Integration**: Required for domain-level federation in Microsoft Entra

---

## 3. How It Works: User Experience Flow

### For End-Customers (Keycloak Users)

```
1. User navigates to ContosoApp Databricks workspace
   ↓
2. Databricks redirects to Azure Entra ID for authentication
   ↓
3. Azure Entra ID recognizes the domain (@yourcompany.com) is federated
   ↓
4. Azure Entra ID automatically redirects to Keycloak (SAML redirect)
   ↓
5. User enters Keycloak credentials 
   ↓
6. Keycloak validates credentials and generates SAML assertion
   ↓
7. Keycloak sends SAML assertion back to Azure Entra ID
   ↓
8. Azure Entra ID validates the SAML assertion and issues its own token
   ↓
9. User is redirected back to Databricks with Azure Entra token
   ↓
10. Databricks validates the Entra token and grants access based on user's roles/groups
```

**Customer Experience**: Users still use their **Keycloak username and password**. The redirect through Azure Entra ID happens automatically and quickly—they may not even notice it. **No change to their login credentials or process.**

### For Internal Teams (Azure Entra Users)

```
1. Internal employee navigates to Databricks workspace
   ↓
2. Databricks redirects to Azure Entra ID for authentication
   ↓
3. Azure Entra ID recognizes this is an internal employee domain
   ↓
   [Two possible paths - to be decided during implementation]
   
   Path A (Direct Authentication):
   4a. Employee authenticates with Microsoft credentials (seamless SSO if already logged in)
   5a. Azure Entra ID issues token directly
   6a. Employee accesses Databricks with corporate identity
   
   Path B (Via Keycloak Broker):
   4b. Azure Entra federates to Keycloak (identity broker)
   5b. Keycloak recognizes internal employee and federates back to Azure Entra
   6b. Employee authenticates with Microsoft credentials
   7b. Azure Entra sends identity to Keycloak → Keycloak sends SAML to Entra
   8b. Azure Entra issues token
   9b. Employee accesses Databricks
```

**Employee Experience**: Uses familiar Microsoft login, potentially with seamless single sign-on if already logged into Windows/Office 365.

---

## 4. Technical Architecture Flow

### Complete Authentication Chain
```
┌─────────────────┐
│   End-Customer  │
│  or Employee    │
└────────┬────────┘
         │ (1) Access Databricks workspace
         ▼
┌─────────────────────────┐
│  Azure Databricks       │◄────────────────────┐
│  (Service Provider)     │                     │
│  - Only trusts Entra    │                     │
└────────┬────────────────┘                     │
         │ (2) Redirect to Entra for auth       │
         │                                      │ (10) Return with
         ▼                                      │      Entra token
┌─────────────────────────┐                     │
│  Azure Entra ID         │─────────────────────┘
│  (Identity Provider)    │
│  - Issues final token   │
│  - Trusts Keycloak      │
└────────┬────────────────┘
         │ (3) Check if domain is federated
         │     → YES: Redirect to Keycloak
         │ (9) Receive SAML assertion
         │     Validate & issue Entra token
         ▼
┌─────────────────────────┐
│      Keycloak           │
│  (Federated IdP/Broker) │
│  - Authenticates users  │
│  - Issues SAML tokens   │
└────────┬────────────────┘
         │ (4-5) User authentication
         │ (6-8) Generate & send SAML assertion to Entra
         └─────────────────┐
                           │
                    ┌──────▼──────┐
                    │ User Login  │
                    │ Credentials │
                    └─────────────┘
```

---

### Architecture Flow

<img width="2250" height="1518" alt="mermaid-diagram-2025-11-18-131320" src="https://github.com/user-attachments/assets/776150f1-94f6-48e5-86b4-bd5acc55e395" />



### Component Responsibilities

| Component | Role | Responsibility |
|-----------|------|----------------|
| **Keycloak** | Authentication Broker | - Authenticates end-customers with username/password<br>- Can federate with Azure Entra for internal users<br>- Issues SAML assertions **to Azure Entra** (not to Databricks)<br>- Manages user attributes and group mappings |
| **Azure Entra ID** | Primary Identity Provider | - **Only identity provider Databricks trusts**<br>- Receives SAML assertions from Keycloak<br>- Issues final authentication tokens to Databricks<br>- Enforces corporate security policies (MFA, conditional access)<br>- Manages user lifecycle and licensing |
| **Azure Databricks** | Service Provider | - Accepts authentication tokens **only from Azure Entra**<br>- Grants workspace access based on user attributes<br>- Enforces data access controls<br>- Cannot directly authenticate against Keycloak |


---

## 6. What Needs to Be Done

### Configuration Tasks

#### In Azure Entra ID:
1. **Verify domain ownership** for your custom domain (e.g., yourcompany.com)
2. **Create Enterprise Application** for Azure Databricks
3. **Configure domain-level SAML federation** to trust Keycloak as upstream IdP
4. **Upload Keycloak's SAML signing certificate** to Entra
5. **Define SAML endpoints** (from Keycloak) for authentication redirection
6. **Configure user attributes and claims** to pass to Databricks
7. **Set up conditional access policies** (MFA, device compliance, etc.)
8. **Configure SCIM provisioning** (if using Databricks Premium)

#### In Keycloak: (This needs to be consulted and confirmed from Keycloak expert)
1. **Create SAML client** for Azure Entra federation
2. **Configure SAML endpoints** for Entra (Assertion Consumer Service URLs)
3. **Set up protocol mappers** for user attributes:
   - Email address
   - Given name, surname
   - User Principal Name (UPN)
   - Groups for RBAC
4. **Configure identity brokering rules** (optional: for internal users who should authenticate via Entra)
5. **Export realm signing certificate** for Entra trust configuration
6. **Set up authentication flows** and session policies
7. **Configure realm settings** for security compliance

#### In Azure Databricks: (Databricks Experts)
1. **Configure account-level SAML SSO** to use Azure Entra
2. **Upload Azure Entra's SAML signing certificate**
3. **Configure Entity ID and ACS URL** (from Databricks, provide to Entra)
4. **Map SAML attributes** to Databricks user properties (email, name, groups)
5. **Configure workspace-level access controls** and group mappings
6. **Set up SCIM provisioning endpoints** (optional but recommended for Premium)
7. **Configure audit logging and monitoring**

---

**Key Principle**: Azure Databricks **always** consumes tokens from Azure Entra ID. Keycloak performs authentication; Entra issues authorization tokens. This separation of concerns provides security, flexibility, and compliance.

---

## Appendix B: Authentication Flow Diagram (Detailed)

```
┌──────────────┐
│ End-Customer │
│    or        │
│  Employee    │
└──────┬───────┘
       │
       │ (Step 1) Navigate to https://databricks.company.com
       ▼
┌─────────────────────────────────────────┐
│         Azure Databricks                │
│         Workspace URL                   │
└──────┬──────────────────────────────────┘
       │
       │ (Step 2) HTTP 302 Redirect
       │ Location: https://login.microsoftonline.com/...
       ▼
┌─────────────────────────────────────────┐
│         Azure Entra ID                  │
│    (login.microsoftonline.com)          │
└──────┬──────────────────────────────────┘
       │
       │ (Step 3) Check domain federation
       │ Domain: @company.com → Status: FEDERATED
       │
       │ (Step 4) HTTP 302 Redirect  
       │ Location: https://keycloak.company.com/realms/main/protocol/saml
       │ SAMLRequest: <base64 encoded request>
       ▼
┌─────────────────────────────────────────┐
│            Keycloak                     │
│   (keycloak.company.com)                │
└──────┬──────────────────────────────────┘
       │
       │ (Step 5) Present login form
       │ User enters: username + password
       │
       │ (Step 6) Validate credentials
       │ Check against Keycloak database/LDAP
       │
       │ (Step 7) Generate SAML Assertion
       │ - Subject: user@company.com
       │ - Attributes: email, name, groups
       │ - Sign with Keycloak private key
       │
       │ (Step 8) HTTP POST to Entra ACS
       │ Location: https://login.microsoftonline.com/login.srf
       │ SAMLResponse: <signed assertion>
       ▼
┌─────────────────────────────────────────┐
│         Azure Entra ID                  │
│      (Assertion Consumer)               │
└──────┬──────────────────────────────────┘
       │
       │ (Step 9) Validate SAML Assertion
       │ - Verify signature with Keycloak public cert
       │ - Check NotBefore/NotOnOrAfter
       │ - Validate Audience/Issuer
       │
       │ (Step 10) Create/Update user in Entra
       │ - Map SAML attributes to Entra user properties
       │
       │ (Step 11) Issue Azure AD Token
       │ - Access Token
       │ - ID Token  
       │ - Refresh Token
       │
       │ (Step 12) HTTP 302 Redirect to Databricks
       │ Location: https://databricks.company.com/...
       │ Token: <Azure AD access token>
       ▼
┌─────────────────────────────────────────┐
│         Azure Databricks                │
└──────┬──────────────────────────────────┘
       │
       │ (Step 13) Validate Azure AD Token
       │ - Verify signature with Azure AD public key
       │ - Check expiration and audience
       │
       │ (Step 14) Extract user claims
       │ - Email, name, groups from token
       │
       │ (Step 15) Create/Update user in Databricks
       │ - JIT provisioning or match to SCIM-provisioned user
       │
       │ (Step 16) Check authorization
       │ - User's workspace roles
       │ - Group-based permissions
       │
       ▼
┌─────────────────────────────────────────┐
│    User Logged Into Databricks          │
│    Workspace Access Granted             │
└─────────────────────────────────────────┘
```

---

<details>
  <summary> User Provisioning and Lifecycle Management</summary>

   User Provisioning and Lifecycle Management
## Azure Entra ID Federation - Member vs Guest Users, Security Controls, and Offboarding

---

## Table of Contents

1. What Happens in Azure Entra ID During Federation?
2. Option 1: Domain Federation (Recommended) - Member Users
3. Comparison: Member vs Guest Users
4. Recommended Approach: Domain Federation (Member Users)
5. Security Controls and Accessibility
6. What Happens When a User Leaves the Organization?

---

## 1. What Happens in Azure Entra ID During Federation?

When you configure domain-level federation between Azure Entra ID and Keycloak, a critical question arises: **What type of user objects are created in Azure Entra ID?**

The answer depends on **which type of federation** you implement. This decision significantly impacts your security model, licensing, user management, and offboarding procedures.

### The Two Federation Options

<img width="2250" height="1518" alt="mermaid-diagram-2025-11-18-133938" src="https://github.com/user-attachments/assets/d82da7f6-08e4-41fe-98ca-73d6a51ade88" />

---

## 2. Option 1: Domain Federation (Recommended) - Member Users

### What Happens in Entra: Member User Creation

When you configure **domain-level federation** using PowerShell `Set-MsolDomainAuthentication`, users are created as **Member** users, NOT Guest users.

### Authentication Flow and User Creation

<img width="4250" height="2918" alt="mermaid-diagram-2025-11-18-134033" src="https://github.com/user-attachments/assets/c7f69c20-d3a8-49c5-a4da-443831b093f7" />

### User Type Created: **MEMBER** (Not Guest!)

When a federated user logs in for the first time, Azure Entra creates the following user object:

```json
{
  "userPrincipalName": "jdoe@company.com",
  "userType": "Member",
  "accountEnabled": true,
  "mail": "jdoe@company.com",
  "displayName": "John Doe",
  "givenName": "John",
  "surname": "Doe",
  "creationType": "Federated",
  "authenticationSource": "Federated",
  "onPremisesSecurityIdentifier": null,
  "passwordProfile": null
}
```

### Key Characteristics of Member Users

| Attribute | Value | Impact |
|-----------|-------|--------|
| **userType** | `Member` | Full native user, not external |
| **UPN** | `user@company.com` | Your verified domain - clean, professional |
| **Creation Method** | JIT (Just-In-Time) on first login | User doesn't exist until first login attempt |
| **Password Stored in Entra?** |  No | Password is in Keycloak only |
| **Can reset password in Entra?** |  No | Must reset in Keycloak |
| **Appears in Entra Users list?** |  Yes | Visible as regular user |
| **Licensing** | Same as native users | Full license assignment capabilities |
| **Dynamic Groups** |  Full support | Can filter by domain, department, etc. |
| **Conditional Access** |  Full support | Can target like any member |
| **Device Compliance** |  Can enforce | Intune/MDM policies applicable |
| **Azure RBAC** |  Full support | Standard role assignments work |

---

## 3. Comparison: Member vs Guest Users in Your Scenario

Understanding the differences between Member and Guest users is critical for making the right architectural decision.

### Complete Comparison Table

| Feature | Domain Federation (Member) | External Identity (Guest) | Winner for MedAdvantage360 |
|---------|---------------------------|--------------------------|---------------------------|
| **User Type** | `Member` | `Guest` | **Member** - cleaner model |
| **UPN Format** | `jdoe@company.com` | `jdoe_company.com#EXT#@tenant.onmicrosoft.com` | **Member** - professional |
| **Appears as** | Native user | External collaborator | **Member** - better UX |
| **Pre-creation Required?** |  No (JIT on first login) | Optional (can invite or JIT) | **Tie** |
| **Licensing** | Standard assignment | May require special licensing | **Member** - simpler |
| **Dynamic Groups** |  Full support |  Limited (can't filter by domain easily) | **Member** - better control |
| **Conditional Access** |  Full support |  Must target "All Guests" separately | **Member** - easier policies |
| **Device Compliance** |  Can enforce |  Very difficult for external devices | **Member** - better security |
| **Seamless SSO** |  Yes, if on domain |  Depends on configuration | **Member** |
| **Azure RBAC** |  Full support |  Limited in some scenarios | **Member** |
| **Microsoft 365 Integration** |  Full |  Limited guest access | **Member** |
| **Databricks Integration** |  Native experience |  Works but guests have limitations | **Member** |
| **Deprovisioning** | Disable in Keycloak + Entra | Disable in Keycloak + Entra | **Tie** - same complexity |
| **User Experience** | Familiar corporate email | Technical-looking EXT UPN | **Member** - professional |
| **Management Overhead** | Lower | Higher (guest-specific policies) | **Member** - simpler |

---

## 4. Recommended Approach: Domain Federation (Member Users)

### Why This is Better for Your Scenario

For the Contoso use case with end-customers authenticating via Keycloak, we **strongly recommend Domain Federation creating Member users**.

####  **Primary Benefits**

1. **Native User Experience**
   - Users have clean UPNs matching their email: `jdoe@company.com`
   - No confusing `#EXT#` notation
   - Professional appearance in all Microsoft interfaces
   - Better for customer-facing scenarios

2. **Simplified Licensing Model**
   - Treat like any other user for license assignment
   - No special guest licensing considerations
   - Easier to automate license assignment via groups
   - No confusion about guest vs member licensing tiers

3. **Better Conditional Access Targeting**
   - Can create policies specific to the domain
   - Example: "All users with @company.com from untrusted locations"
   - No need for separate "All Guests" policies
   - Easier to implement risk-based authentication

4. **Dynamic Groups Work Seamlessly**
   - Can filter users by domain: `user.mail -contains "@company.com"`
   - Department, location, and other attribute-based groups work normally
   - Automatic group membership based on SAML claims
   - Better for role-based access control (RBAC)

5. **Cleaner User Management**
   - Users appear in standard user lists without guest indicators
   - Same management experience as internal users
   - Less confusion for IT administrators
   - Better for governance and compliance reporting

6. **Better Databricks Integration**
   - Users appear as standard users in Databricks
   - No guest-related permission limitations
   - Simpler SCIM provisioning (if implemented)
   - Better for workspace ownership and sharing

####  **Trade-offs to Consider**

1. **Not Reversible Without Migration**
   - Once you choose Member users, converting to Guest requires migration
   - All existing user objects would need to be recreated
   - Permissions and licenses would need to be reassigned

2. **Less Clear External/Internal Separation**
   - Members look like internal employees in Entra
   - May need additional documentation to identify customer vs employee users
   - Mitigation: Use group membership and attributes to track user type

3. **Same Offboarding Complexity**
   - Must disable in both Keycloak AND Entra
   - No automatic lifecycle synchronization
   - Requires automation or strict procedures
   - (Note: Guest users have the SAME problem!)

### When to Use Guest Users Instead

Guest users (B2B External Identity) would be better ONLY if:

-  You want visual separation between external and internal users
-  You need to comply with policies requiring guests for external users
-  You plan to leverage guest-specific access reviews extensively
-  Users will access multiple Microsoft 365 tenants (guest federation benefits)

---

## 5. Security Controls and Accessibility

### Access Control Decision Points

At each layer, access can be denied based on specific criteria:

| Layer | Control Point | Example Denial Reasons |
|-------|--------------|------------------------|
| **Keycloak** | Initial authentication | - Invalid credentials<br/>- Account disabled<br/>- Not member of required group<br/>- MFA failed |
| **Azure Entra - SAML Validation** | Federation trust | - SAML signature invalid<br/>- Certificate expired<br/>- Timestamp out of range<br/>- Issuer not trusted |
| **Azure Entra - Conditional Access** | Policy enforcement | - Location not allowed<br/>- Device not compliant<br/>- Risk level too high<br/>- MFA not satisfied |
| **Azure Entra - Provisioning** | User existence | - User not provisioned<br/>- Account disabled in Entra<br/>- License not assigned |
| **Databricks - Authentication** | Token validation | - Token expired<br/>- Invalid signature<br/>- Wrong audience<br/>- Token revoked |
| **Databricks - Authorization** | Workspace access | - Not assigned to workspace<br/>- Insufficient role permissions<br/>- License expired |


---

## 6. What Happens When a User Leaves the Organization?

This is a **critical security concern** and represents a significant gap in the federated architecture that you **must** address with proper procedures.

### The Core Problem: No Automatic Synchronization

<img width="2250" height="1518" alt="mermaid-diagram-2025-11-18-134615" src="https://github.com/user-attachments/assets/25e476e9-4b12-4bab-b5c0-9f3a5559c0ee" />


### Why This Happens

**Federation is authentication-time only:**
- Keycloak and Azure Entra only communicate during login attempts
- SAML assertions are only sent when a user tries to authenticate
- There is **NO ongoing synchronization** of user status
- Disabling a user in Keycloak does **NOT** notify Entra
- The user object in Entra is **independent** after creation


---


</details>


