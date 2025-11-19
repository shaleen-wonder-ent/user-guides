# How Single Sign-On Works for Contoso360 and Power BI

---

# Proposed Flow: End-Customer Authentication Flow (User vs Behind-the-Scenes)

1. **User navigates to the ContosoApp Databricks/PBI workspace**  
   *(User does it — clicks/opens the Databricks/PBI workspace URL)*  
   ↓

2. **Databricks/PBI redirects the user to Azure Entra ID for authentication**  
   *(Behind the scenes — Databricks/PBI triggers its configured SAML redirect to Entra)*  
   ↓

3. **Azure Entra ID examines the user’s domain (e.g., @yourcompany.com)**  
   *(Behind the scenes — Entra checks domain federation settings)*  
   ↓

4. **Azure Entra ID identifies the domain is SAML-federated and redirects the user to Keycloak**  
   *(Behind the scenes — automatic SAML IdP redirect)*  
   ↓

5. **User enters their Keycloak credentials**  
   *(User does it — user sees the Keycloak login page)*  
   ↓

6. **Keycloak validates the credentials and generates a SAML assertion**  
   *(Behind the scenes)*  
   ↓

7. **Keycloak sends the SAML assertion back to Azure Entra ID**  
   *(Behind the scenes — SAML POST/Redirect with signed assertion)*  
   ↓

8. **Azure Entra ID validates the SAML assertion and issues an Entra token (SAML/OIDC)**  
   *(Behind the scenes — signature + certificate validation)*  
   ↓

9. **Azure Entra returns the user to Databricks/PBI with the Entra-issued token**  
   *(Behind the scenes — automatic redirect)*  
   ↓

10. **Databricks/PBI validates the Entra token and grants access based on SCIM-provisioned roles/groups**  
    *(Behind the scenes — RBAC + SCIM mapping)*  
    *(User sees Databricks/PBI workspace load successfully)*

---


**Goal:** End-customers log in **once** to Contoso360 portal, then click a button to see their Power BI reports **without logging in again**.

**How we achieve this:** We connect three systems together:
1. **Contoso360 Portal** - Your customer-facing application
2. **Keycloak** - Your identity/login system
3. **Microsoft (Azure Entra ID & Power BI)** - Microsoft's services

---

## The User Story: User Sarah at healthcare

### Morning - 9:00 AM: Sarah Logs Into the Portal

**What Sarah does:**
1. Opens her web browser
2. Goes to `Contoso360.com`
3. Sees a login page
4. Enters her **username and password**
5. (OR uses her company's corporate single sign-on if they have one)

**What happens behind the scenes:**
- The portal redirects Sarah to Keycloak
- Keycloak checks her credentials: Valid
- Keycloak creates a "session" (thinks: "Sarah is logged in")
- Keycloak stores a small note in Sarah's browser (cookie) saying "Sarah logged in at 9:00 AM"
- Sarah is redirected back to the portal
- Sarah sees her portal homepage with menus and options

**Sarah's experience:** Normal login, like any website.

---

### 15 Minutes Later - 9:15 AM: Sarah Wants to See Reports

**What Sarah does:**
1. Clicks a button that says **"View Reports"** or **"Analytics Dashboard"**
2. Waits a few seconds

**What Sarah sees:**
- Browser shows Power BI website (`app.powerbi.com`)
- Reports appear on screen
- **No login screen!** 

**What Sarah does NOT see:**
- No second username/password prompt
- No "Please login again" message
- No waiting for authentication

**Sarah's experience:** Click button → Reports appear. Seamless!

---

### Behind the Scenes - What Actually Happened (Sarah doesn't see this)

Here's what happened in those "few seconds" when Sarah clicked the button:

#### Step 1: Portal Creates a Link 
- Portal knows Sarah works for Customer A
- Portal generates a special link to Customer A's Power BI workspace
- Browser is redirected to: `https://app.powerbi.com/groups/customer-a-workspace/reports/...`

#### Step 2: Power BI Checks "Who Is This?" 
- Power BI receives the request
- Power BI says: "I don't know who you are. Go login with Microsoft first."
- Browser is redirected to Microsoft login page

#### Step 3: Microsoft Checks Configuration
- Microsoft (Azure Entra ID) sees: "This person has email @customer-a.com"
- Microsoft checks settings: "For @customer-a.com users, I'm supposed to ask **Keycloak** for authentication, not myself!"
- Microsoft sends browser **back** to Keycloak with a request: "Please tell me who this user is"

#### Step 4: Keycloak Remembers Sarah 
- Browser arrives at Keycloak
- Keycloak checks browser's stored note: "This browser has a note from 9:00 AM saying Sarah logged in!"
- Keycloak checks: "Is this note still valid? Yes, it's only 15 minutes old!"
- Keycloak says: "**No need to ask for password again** - I already verified Sarah at 9:00 AM"
- Keycloak creates a secure message for Microsoft: "This is Sarah from Customer A. She's legitimate."

**CRITICAL POINT:** Sarah does **NOT** see a login screen here! Keycloak recognizes her from the earlier login.

#### Step 5: Microsoft Creates Account 
- Microsoft receives Keycloak's message
- Microsoft validates: "Yes, this message is genuine from Keycloak (checks digital signature)"
- Microsoft checks its database: "Do I have a user account for sarah@customer-a.com?"
  - **If No:** Creates new account (no password needed)
  - **If Yes:** Updates account information
- Microsoft generates a "token" (think: digital access key) for Sarah
- Microsoft sends browser back to Power BI with the token

#### Step 6: Power BI Shows Reports
- Power BI receives the token
- Power BI validates the token with Microsoft: "Is this real?"Yes
- Power BI looks at Sarah's identity: `sarah@customer-a.com`
- Power BI applies security rules: "Email has @customer-a.com, so show only Customer A's data"
- Power BI loads and displays the reports


---


## Important Points 

### What This Solution Provides

**1. Seamless User Experience**
- Users log in **once** at the start of their day
- Access all services without repeated logins
- Professional, modern experience

**2. Strong Security**
- Users authenticate through Keycloak (you control security policies)
- Microsoft validates every access request
- Data automatically filtered per customer
- Full audit trail of who accessed what

**3. Scalability**
- Works for 10 customers or 10,000 customers
- Each customer sees only their own data
- Multi-tenant architecture (one Power BI setup, many customers)

**4. Compliance & Audit**
- Complete logs in Keycloak (who logged in, when)
- Complete logs in Microsoft (who accessed Power BI, what reports)
- Can prove user authentication for audits

**5. Cost Efficiency**
- Users already in Keycloak (no duplicate management)
- Single Power BI workspace per customer-product combination
- No need for separate Microsoft accounts with passwords

### Important Limitations & Considerations

**1. Microsoft Account Still Created**
- Even though users login with Keycloak, a user account **is created** in Microsoft
- Format: `sarah@customer-a.com` (Member user)
- **No password stored** in Microsoft (password only in Keycloak)

**2. Session Expiration**
- Sessions expire after set time (default 8 hours)
- After expiration, users must login again
- This is for security - prevents indefinite access

**3. Initial Setup Complexity**
- One-time technical setup required
- Involves configuring Keycloak, Microsoft, and Power BI
- Requires coordination between teams
- Once set up, operates automatically

---

#### Identity Verification
- Every Power BI access is verified:
  1. Who is the user? (from Microsoft token)
  2. Is the token valid? (checked with Microsoft)
  3. What data can they see? (RLS rules applied)

---

### After Go-Live 

**New users:**
1. Login to portal (same as always)
2. Click Power BI link → reports appear automatically
3. **Experience:** Seamless from day one

**Existing users (already have Microsoft accounts):**
1. First time clicking Power BI link after go-live
2. May see one-time prompt: "Connect your accounts"
3. After that: Automatic access

---

## Summary in One Paragraph

**This solution allows your end-customers to login once to the Contoso360 portal, then access Power BI reports with a single click - no second login required. Behind the scenes, we connect Keycloak (your identity system) with Microsoft (Power BI provider) using industry-standard SAML federation. Users authenticate with their familiar Keycloak credentials, and Microsoft trusts Keycloak's verification. The result is a seamless, secure, scalable single sign-on experience that improves user satisfaction and reduces support costs.**

---
