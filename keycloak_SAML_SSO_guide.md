# How Single Sign-On Works for Contoso360 and Power BI

---

# Proposed Flow: End-Customer Authentication Flow (User vs Behind-the-Scenes)

1. **User logs into the Contoso360 portal via SSO (SAML) or credentials**  
   *(User action — navigates to Contoso360.com and authenticates via Keycloak. A Keycloak session is created.)*  
   ↓

2. **User clicks a Power BI link within the portal to view analytics/reports**  
   *(User action — selects an analytics/report feature/navigational element in the portal. Session is still active from portal login.)*  
   ↓

3. **Portal generates a deep link to the user's Power BI workspace/report**  
   *(Behind the scenes — Portal backend determines customer context, product/workspace, and generates correct Power BI URL, e.g. `https://app.powerbi.com/groups/customer-a-workspace/reports/...`)*  
   ↓

4. **Browser is redirected to Power BI site**  
   *(Browser navigates to the Power BI URL. User leaves the portal site, now at app.powerbi.com.)*  
   ↓

5. **Power BI checks for Microsoft authentication**  
   *(Behind the scenes — Power BI sees no active Azure AD session for the user, and so redirects to Azure Entra ID for login)*  
   ↓

6. **Azure Entra ID examines the email/domain**  
   *(Behind the scenes — During login or home realm discovery, Entra ID detects the user's domain, e.g., @customer-a.com)*  
   ↓

7. **Azure Entra ID identifies the domain is SAML-federated and redirects to Keycloak**  
   *(Behind the scenes — Entra ID auto-redirects to Keycloak as the federated IdP for @customer-a.com users)*  
   ↓

8. **Keycloak checks for an existing browser session**  
   - *If session from step 1 is still valid (e.g. <8 hours):*  
     **NO login required — user is recognized; SSO proceeds seamlessly.**  
   - *If session expired or browser cookies cleared:*  
     **User is prompted by Keycloak to enter credentials (login again).**  
   ↓

9. **Keycloak generates a SAML assertion and returns to Azure Entra ID**  
   *(Behind the scenes — Keycloak validates the session, then signs and sends a SAML assertion back to Entra.)*  
   ↓

10. **Azure Entra ID validates the SAML assertion and issues a token**  
    *(Behind the scenes — Entra checks signature, timestamps, certificate, and creates/updates the user if needed.)*  
    ↓

11. **User is redirected back to Power BI with the Azure Entra-issued token**  
    *(Behind the scenes — SSO/OIDC protocol, browser is pointed back to the intended Power BI report/workspace)*  
    ↓

12. **Power BI validates the Entra token and grants access based on Azure roles/groups/RLS**  
    *(Behind the scenes — Power BI recognizes user as `user@customer-a.com`, applies licensing, group, RBAC, and row-level security rules to show correct data. User sees reports.)*

---

**Goal:** End-customers log in **once** to Contoso360 portal, then click a button to see their Power BI reports **without logging in again** if their Keycloak session is still active.

**How we achieve this:**  
We connect three systems together using SAML/OIDC federation and user session management:  
1. **Contoso360 Portal**: Where users begin, authenticating via Keycloak as the IAM
2. **Keycloak**: Identity provider that authenticates the user, manages session/cookies, and federates to Microsoft as needed
3. **Microsoft (Azure Entra ID & Power BI)**: Consumes SAML assertions, creates tokens/account, grants data access via workspace and security groups

---

## The User Story: User Sarah at healthcare

### Morning - 9:00 AM: Sarah Logs Into the Portal

- Opens browser, goes to `Contoso360.com`
- Sees login page, authenticates via corporate SSO (SAML) **or** enters Keycloak credentials
- Keycloak validates user, creates session, sets session cookie
- User is redirected back to portal and sees the dashboard

---

### 15 Minutes Later - 9:15 AM: Sarah Wants to See Reports

- Sarah clicks "View Reports" or "Analytics Dashboard"
- Portal prepares a link for Customer A's Power BI workspace
- Sarah's browser is redirected to Power BI (`app.powerbi.com`)
- Power BI, seeing no Microsoft session, redirects to Azure Entra ID
- Azure Entra checks Sarah's domain, sees it is federated, and redirects browser **back to Keycloak**
- Keycloak checks if session is still valid:
  - If **yes**: No password needed, SSO is seamless
  - If **no**: User must re-authenticate with Keycloak
- Keycloak generates and returns the required SAML assertion to Entra
- Azure Entra validates the assertion, updates or creates user as needed, and issues an Entra access token for Sarah
- Sarah is redirected back to her Power BI workspace/report
- Power BI validates the token and applies all RLS and workspace security — Sarah sees Customer A’s data with **no additional login** if her session was valid

---

### CRITICAL POINT

If Sarah’s Keycloak session is *still* active from her portal login, she will **not** see any additional login prompt when accessing Power BI—a seamless SSO experience.

---

#### Important

- **If Sarah’s Keycloak session has expired** (8+ hours, or browser cookies cleared), she will need to re-authenticate at Keycloak to regain access to Power BI.
- **User account in Microsoft** will be created upon first SSO (JIT-provisioned Member user), but password is NOT stored in Microsoft.

---

## Visual timeline

```
1. Sarah logs in at Contoso360.com (Keycloak session started)
2. Sarah uses portal as normal
3. [15 minutes later] Sarah clicks "View Reports"
4. Browser→Power BI→Microsoft→Keycloak→(session valid=no login needed)
5. Power BI shows reports instantly — NO login screen!
6. [8 hours later, session expired] Keycloak asks for login again
```

---

## What does the user see? (short)

- One login at the start of the day (via portal & Keycloak)
- Click View Reports later: immediate access to Power BI
- No additional logins needed unless Keycloak session expires
- Clean handoff of identity across all platforms (with strong security and governance)

---

## Behind the Scenes: SSO Magic

- The session cookie from initial portal login allows Keycloak to recognize the user mid-day, skipping any further password prompt.
- SAML assertion is used to securely hand identity to Microsoft, which then grants the user access to Power BI based on group, role, and row-level security.
- The whole process is invisible to the user when SSO works as designed.

---

## Security Mechanisms in Play

- All communication over HTTPS
- All SAML assertions signed/certificates managed
- Short session/token lifetimes for reduced risk
- All identities and access are logged for audit
- Only the correct user is able to see the correct data in Power BI/Databricks, always

---

## Limitation

- If a user’s Keycloak session is expired or forcibly logged out (by admin or browser clearing cookies), the user will see a login page on the next Microsoft/Keycloak authentication attempt.

---
