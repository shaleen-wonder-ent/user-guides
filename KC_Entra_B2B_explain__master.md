# Production Architecture: Keycloak → Entra ID (B2B Direct Federation + Domain Routing) → Power BI  
**Purpose:** Explain *what you need*, *what to configure in Keycloak*, *what to configure in Entra*, and *the resulting runtime flow*.

---

## 1) What you need (high-level)

To enable Keycloak-origin users to access Microsoft Power BI (non-embedded), you need:

1. **Keycloak**  
   - Users authenticate to your application (medadv360) using Keycloak SSO.
   - Keycloak will also act as the **federated IdP** for Entra ID (for those users).

2. **Entra ID tenant** (your tenant)  
   - Power BI authenticates through Entra ID.
   - External users must exist in the tenant as **B2B guest users**.

3. **Federation between Entra and Keycloak**  
   - We use **Entra External Identities → Direct Federation (SAML)** so Entra can redirect users to Keycloak and trust the SAML assertion.

4. **Direct Federation + Domain Routing** (the core pattern)  
   - **Direct Federation** tells Entra *how to trust/authenticate* via Keycloak.
   - **Domain routing** tells Entra *when to redirect* to Keycloak (based on the user’s email domain).

5. **Just-in-time (JIT) guest provisioning**  
   - When a user first clicks the Power BI link, the medadv360 backend uses Microsoft Graph to create a guest invitation and redirects the user through redemption.

---

## 2) Multi-domain note
You may have users coming from multiple email domains, for example:
- `usera@companyA.com`
- `userb@companyB.com`

**The core architecture does not change.**  
Only the operational scale increases because **each domain** requires:
- verification in Entra (DNS TXT record), and
- domain routing to the Keycloak federation.

> If a partner domain cannot be verified (DNS not controlled/cooperative), you cannot reliably apply domain routing for that domain in a single Entra tenant.

---

## 3) What you must prepare in Keycloak (inputs + configuration)

### 3.1 Keycloak must be stable and externally reachable
Keycloak must be reachable over the internet using:
- a stable public hostname (FQDN), e.g., `https://sso.medadv360.com`
- a valid TLS certificate (publicly trusted) with a renewal/rotation plan

> Sometimes Keycloak is “production” but only internal. For this scenario, it must be reachable by browsers and Microsoft sign-in flows over the internet and must remain stable.

### 3.2 Provide IdP metadata over HTTPS
You need:
- **SAML IdP metadata URL** accessible over HTTPS (from the stable FQDN)

This is what Entra imports to learn:
- IdP issuer/entityID
- SSO endpoint
- signing certificates

### 3.3 Ensure the SAML assertion content is Entra-compatible
Keycloak must issue SAML assertions that meet these requirements:

- **Assertions signed** (certificate maintained/rotated)
- **NameID format:** `emailAddress`
- **NameID value:** the user’s email (e.g., `usera@companyA.com`)
- Recommended attributes:
  - `email`
  - `given_name`
  - `family_name`

**Why this matters:** Entra will map the federated assertion to the guest user object created for that email.


**Detailed view of this :**
## 3) Keycloak inputs you need (to configure Entra)

From Keycloak you need the following values (preferably from metadata):

1. **Issuer URI**  
   Typical Keycloak realm issuer format:  
   `https://<keycloak-host>/realms/<realm>`

2. **Passive authentication endpoint** (SAML SSO endpoint)  
   Typical Keycloak SAML endpoint format:  
   `https://<keycloak-host>/realms/<realm>/protocol/saml`

3. **Signing certificate**  
   The public X.509 certificate used by Keycloak to sign SAML assertions/responses.

4. **Metadata URL (recommended)**  
   Typical Keycloak SAML metadata endpoint:  
   `https://<keycloak-host>/realms/<realm>/protocol/saml/descriptor`

Example (lab/ngrok):  
- `https://b61d7161f98c.ngrok-free.app/realms/master/protocol/saml/descriptor`

Example (production):  
- `https://sso.medadv360.com/realms/<realm>/protocol/saml/descriptor`

> In production, do not use ngrok. Use a stable, publicly reachable FQDN + trusted TLS certificate.

**Done**

---

## 4) What you must configure in Entra (one-time + per-domain)

### 4.1 Verify partner domains
For each partner email domain you want to route (e.g., `companyA.com`, `companyB.com`):

1. Add domain in Entra tenant (Custom domain names)
2. Entra provides DNS TXT record
3. Domain owner publishes TXT record
4. Entra verifies the domain

**Outcome:** Domain is “Verified” and eligible for routing configuration.

### 4.2 Configure Direct Federation to Keycloak (one-time)
In Entra **External Identities**:
- Add a **Direct Federation** identity provider (SAML) using Keycloak metadata

**Outcome:** Entra trusts Keycloak’s signed assertions for federated authentication.

### 4.3 Configure domain routing to Keycloak (per-domain)
For each verified domain:
- Configure domain routing so that:
  - `@companyA.com` → authenticate using Keycloak Direct Federation
  - `@companyB.com` → authenticate using Keycloak Direct Federation

**Outcome:** When Power BI (or any Entra-protected app) needs authentication for a user in that domain, Entra redirects to Keycloak.

-
**Detailed view of this :**

## 4) Entra setup (federation + domain routing)

### 4.1 Verify partner domains **DOMAIN ROUTING PREREQ**

You must do domain verification in the **same Entra tenant** where you configure federation and where Power BI authenticates.

Steps (per domain, e.g., `companyA.com`):
1. In Entra admin center, go to **Domain names**.
2. Go to **Custom domain names**.
3. Click **Add custom domain**.
4. Enter the domain (e.g., `companyA.com`) and click **Add**.
5. Entra will show a **DNS TXT record** (name/value).
6. Provide this TXT record to the domain owner (partner/customer DNS admin).
7. After the TXT record is published, return to the portal and click **Verify**.

Expected result:
- Domain status changes to **Verified**

---

### 4.2 Configure Direct Federation to Keycloak (SAML IdP)  **FEDERATION SETUP**

This is configured in:
**External Identities → All identity providers → Custom → Add new → SAML/WS-Fed**


Steps:
1. In Entra admin center, go to **External Identities**.
2. Select **All identity providers**.
3. Under **Custom**, click **Add new** and select **SAML/WS-Fed**.
4. Provide:
   - **Display name** (e.g., `Keycloak-Federation`)
   - **Identity provider protocol**: `SAML`
   - **Issuer URI** (from Keycloak)
   - **Passive authentication endpoint** (from Keycloak)
   - **Certificate** (Keycloak signing cert)
   - **Metadata URL** (recommended; Keycloak metadata endpoint)
5. Save the identity provider.

Expected result:
- Identity provider appears in the list with Protocol = SAML.

Notes:
- Prefer using **Metadata URL** so Entra can ingest issuer/endpoints/cert more reliably.
- In production, the metadata URL must be reachable via stable FQDN over HTTPS.

---

### 4.3 Attach domain(s) to the federating IdP (this is domain routing) **DOMAIN ROUTING CONFIG**

In your tenant UI, domain routing is implemented by adding **Domain name of federating IdP** when creating the SAML/WS-Fed IdP (or adding domains later).


Steps:
1. In **External Identities → All identity providers → Custom → Add new → SAML/WS-Fed**, you will see a field:
   - **Domain name of federating IdP**
2. Enter the domain you want Entra to route (example: `companyA.com`).
3. Complete the IdP creation (Section 4.2) and save.

To add additional domains later:
1. Go to **External Identities → All identity providers**.
2. Open your IdP (e.g., `Keycloak-Federation`).
3. Use the portal option to add additional domains (if available) OR create an additional routing association as per portal workflow.

Expected result:
- The IdP list shows a non-zero value in the **Domains** column (e.g., Domains = 1) and you can click to check the Domain name.

Important:
- The domain must be **Verified** (Section 4.1) to be a reliable, supported routing target.

---

### 4.4 Validate federation + routing (recommended)
Test with a user at that domain:
- Use any Entra-protected entry point (Power BI, myapps.microsoft.com, etc.)
- Enter `user@companyA.com`

Expected:
- Entra redirects to **Keycloak** (federated IdP)
- After Keycloak SSO, Entra completes sign-in
- Entra sign-in logs show **Federated** authentication


**Done**
---

## 5) Runtime flow 

### Step 1 — User signs into medadv360 using Keycloak
- User authenticates in Keycloak
- medadv360 creates an application session

### Step 2 — User clicks the Power BI report link (first time)
- medadv360 backend checks if the user exists as a guest in Entra
- If not found, backend creates a **B2B invitation** using Microsoft Graph

### Step 3 — Backend redirects user to the invitation redemption URL
- Graph returns `inviteRedeemUrl`
- Backend immediately redirects user to `inviteRedeemUrl` (Option 5A)
- User completes redemption/acceptance UX (tenant policy dependent)

> If the redemption UX cannot be configured to redirect back automatically, use a fallback “Continue” page/button in medadv360 and resume the flow from a known endpoint.

### Step 4 — User is sent to Power BI
- After redemption, medadv360 redirects user to the Power BI report URL

### Step 5 — Power BI authenticates via Entra
- Power BI redirects to Entra ID for authentication

### Step 6 — Entra applies domain routing and redirects to Keycloak
- Entra sees the user’s domain (e.g., `companyA.com`)
- Because the domain is verified + routed, Entra redirects to Keycloak

### Step 7 — Keycloak sends signed SAML assertion back to Entra
- Keycloak issues signed assertion with NameID=email
- Entra maps it to the guest user object and issues tokens

### Step 8 — User reaches the report
- Power BI receives tokens from Entra and shows the report (authorization handled by Power BI permissions setup)

---

## 6) Key operational notes 

### 6.1 Direct Federation vs “enterprise app SAML”
This is **B2B Direct Federation + domain routing**, not classic “enterprise application SAML SSO”.
- You should not expect Entra to present a classic “SP metadata/ACS URL” page like an enterprise app.
- Success is validated via **routing behavior** and **Entra sign-in logs**.

### 6.2 Identity mapping record (app-side)
The medadv360 backend should store a mapping:
- `keycloakSubjectId + partnerId + email → entraUserObjectId`

This is not a SAML responsibility; it is required so the app can manage JIT onboarding reliably (avoid duplicate invites, track lifecycle, speed Graph operations).

---

## 7) What we need to proceed
1. List of partner domains (companyA.com, companyB.com, …)
2. Confirmation each domain owner will publish DNS TXT records so domains can be **verified** in Entra
3. Keycloak stable public FQDN + metadata URL (HTTPS)
4. Keycloak SAML signing certificate rotation plan
5. Tenant policy expectations for invitation redemption UX (to finalize return-path design)

---
**End state:** Users authenticate in Keycloak, access Power BI through Entra, and Entra consistently routes those users back to Keycloak for federated authentication using verified domains + domain routing.

---

## Appendix A) Federation + Domain Routing: Required Steps (Checklist)

### A) Federation prerequisites (Keycloak side) — REQUIRED for federation to work
1. Publish Keycloak with a **stable public FQDN** over HTTPS (internet reachable).
2. Ensure a valid **TLS certificate** is installed and maintained/rotated.
3. Ensure **SAML IdP metadata** is accessible over HTTPS (stable URL).
4. Configure Keycloak to send **signed SAML assertions** (signing cert maintained/rotated).
5. Configure SAML **NameID format = emailAddress** and **NameID value = user email**.
6. (Recommended) Configure attribute mappers: `email`, `given_name`, `family_name`.

### B) Federation configuration (Entra side) — REQUIRED for federation to work
7. In Entra **External Identities → Direct Federation**, add Keycloak as the SAML IdP using the Keycloak metadata.

### C)  Domain routing steps (Entra side) — THESE ARE THE DOMAIN ROUTING REQUIREMENTS
8.  **Verify each external domain** you want to route (e.g., `companyA.com`, `companyB.com`) in the Entra tenant  
   - Add domain in Entra  
   - Publish DNS TXT record  
   - Verify domain shows as **Verified**

9.  **Configure domain routing** for each verified domain to use the Keycloak Direct Federation IdP  
   - `@companyA.com` → Keycloak federation  
   - `@companyB.com` → Keycloak federation  

10.  **Validate routing behavior** (functional test)  
   - Start an Entra auth flow with `user@companyA.com`  
   - Confirm Entra redirects to **Keycloak** (not Microsoft login/OTP)  
   - Confirm Entra sign-in logs show **Federated** authentication

---


<img style="max-width: 800px; cursor: pointer; border: 1px solid #ddd; padding: 4px;" 
     alt="Architecture" 
     src="https://github.com/user-attachments/assets/499989f2-d38a-4906-8586-d708ba23e29d"
     onclick="window.open(this.src, 'Image', 'width='+this.naturalWidth+',height='+this.naturalHeight); return false;" />
<br>
<em>Click to view full size</em>


