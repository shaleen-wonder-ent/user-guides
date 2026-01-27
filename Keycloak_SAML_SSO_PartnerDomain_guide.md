# Step-by-step: Option B — Use `partners.Fabrikam.com` for partner access to Power BI (SSO preserved via Keycloak)

**Goal:** Partner users (e.g., `shaleen@contoso.com`) authenticate to the Fabrikam app via Keycloak and access **Power BI in Fabrikam tenant** with minimal additional prompts, **without requiring partner DNS verification**.

**Core idea:** Use a Fabrikam-controlled sign-in domain (e.g., `partners.Fabrikam.com`) that Fabrikam can verify in Entra and route to Keycloak. Partner users get a corresponding sign-in ID such as `shaleen@partners.Fabrikam.com`.

---

## 0) Before you start (decisions you must make)

### 0.1 Identity format (avoid collisions)
Pick a deterministic, unique format for the Entra sign-in ID:
- Recommended: `localpart+partner@partners.Fabrikam.com`  
  Example: `shaleen+contoso@partners.Fabrikam.com`
- Alternative: `firstname.lastname+partner@partners.Fabrikam.com`
- Avoid: `localpart@partners.Fabrikam.com` (can collide across partners)

### 0.2 Email deliverability (required for B2B invitation acceptance)
If you plan to use standard Entra B2B **invitation + redemption**, the invited address must be able to receive the invite.
Choose one:
- **A)** Operate mail for `partners.Fabrikam.com` and set forwarding/aliases so invites reach the user’s real mailbox (e.g., forward `shaleen+contoso@partners.Fabrikam.com` → `shaleen@contoso.com`), or
- **B)** Use a controlled onboarding process where you can reliably complete redemption (still typically easier if mail is deliverable).

> If you cannot make `@partners.Fabrikam.com` deliverable, this option becomes difficult to operationalize using standard B2B invitation flows.

---

## 1) Domain + DNS setup (Fabrikam)

### Step 1.1 — Create the domain
- Obtain and control DNS for: `partners.Fabrikam.com` (or `partners.Fabrikam360.com`).

### Step 1.2 — Verify domain in Entra
In **Fabrikam Entra admin center**:
1. Go to **Identity** → **Custom domain names** (or “Domain names”).
2. Click **Add custom domain**.
3. Enter `partners.Fabrikam.com`.
4. Copy the provided **DNS TXT** record.
5. Add the TXT record in Fabrikam DNS.
6. Return to Entra and click **Verify**.

**Outcome:** `partners.Fabrikam.com` shows as **Verified** in Fabrikam Entra.

---

## 2) Keycloak setup (Fabrikam)

### Step 2.1 — Ensure Keycloak is externally reachable and stable
- Use a stable public FQDN, e.g., `https://sso.Fabrikam360.com`
- Ensure a publicly trusted TLS certificate
- Ensure consistent availability (this will be in the sign-in path for Power BI)

### Step 2.2 — Prepare SAML IdP metadata endpoint
Confirm the Keycloak SAML descriptor URL is reachable:
- `https://<keycloak-host>/realms/<realm>/protocol/saml/descriptor`

This is what Entra will use to learn:
- Issuer / EntityID
- SSO endpoint
- Signing certificates

### Step 2.3 — Configure the assertion identity (critical)
You must ensure that when Entra redirects a partner user to Keycloak, Keycloak returns a SAML assertion where:

- **NameID format:** `emailAddress`
- **NameID value:** the Fabrikam-controlled sign-in ID  
  Example: `shaleen+contoso@partners.Fabrikam.com`

Implementation patterns:
- Set Keycloak **username** to the `@partners.Fabrikam.com` value, OR
- Store a user attribute like `entraLogin` and map SAML **NameID** from that attribute

Also keep:
- `realEmail = shaleen@contoso.com` as a separate attribute for communication/auditing.

---

## 3) Entra setup: Direct Federation to Keycloak (one-time)

### Step 3.1 — Create the federated identity provider (SAML/WS-Fed)
In **Fabrikam Entra admin center**:
1. Go to **External Identities** → **All identity providers**
2. Under **Custom**, select **Add new** → **SAML/WS-Fed**
3. Configure with Keycloak values (prefer metadata URL):
   - **Metadata URL:** `https://<keycloak-host>/realms/<realm>/protocol/saml/descriptor`
   - (or manually provide Issuer URI + SSO endpoint + signing cert)

4. In the same configuration, specify the **Domain name(s) of federating IdP**:
   - Add: `partners.Fabrikam.com`

5. Save.

**Outcome:** Entra will route sign-ins for `@partners.Fabrikam.com` to Keycloak and trust the SAML assertions.

---

## 4) App + provisioning setup (Fabrikam) — mapping and JIT onboarding

### Step 4.1 — Store a mapping between real email and Entra login
In your app DB (recommended), store:
- `keycloakSubjectId`
- `realEmail` (e.g., `shaleen@contoso.com`)
- `entraLogin` (e.g., `shaleen+contoso@partners.Fabrikam.com`)
- `entraGuestObjectId` (once created)
- `partnerId` (e.g., contoso)

### Step 4.2 — Create guest users in Fabrikam Entra (first time per user)
On first Power BI launch (or at onboarding), do:
1. Determine `entraLogin` from mapping
2. If Entra guest does not exist, create a B2B invitation for `entraLogin`
3. Ensure the guest is added to the correct Entra group(s) that have Power BI permissions

> Note: Invitation acceptance typically requires email deliverability for `entraLogin` (see Step 0.2).

---

## 5) Power BI permissions (Fabrikam)

### Step 5.1 — Use groups (recommended)
- Create Entra groups such as:
  - `PBI-Partner-Report-Viewers`
- Grant those groups access to:
  - the workspace, or
  - the app/report, as appropriate

### Step 5.2 — Add guest users to groups
- Add the guest object for `entraLogin` into the appropriate groups.

---

## 6) End-to-end runtime flow (what happens when the user clicks the Power BI link)

### First-time user
1. User signs into Fabrikam360 via partner IM → Keycloak.
2. User clicks Power BI link.
3. Fabrikam360 backend computes/looks up `entraLogin` (e.g., `shaleen+contoso@partners.Fabrikam.com`).
4. Backend invites/creates the guest in Fabrikam Entra (if needed) and assigns group-based access.
5. User completes invitation redemption (if first time).
6. User is redirected to the Power BI report URL.
7. Power BI redirects the browser to Fabrikam Entra.
8. Fabrikam Entra sees `@partners.Fabrikam.com` and redirects authentication to Keycloak.
9. Keycloak reuses the existing session and returns a signed SAML assertion with NameID = `entraLogin`.
10. Entra issues token; Power BI renders the report.

### Subsequent access
- Steps 1, 2, 6–10 typically occur; invitation redemption is usually not repeated.

---

## 7) Validation checklist (quick test)

### Validate routing
- In a private browser session, attempt to sign in using a `@partners.Fabrikam.com` identity.
- Confirm Entra redirects to Keycloak automatically.

### Validate assertion mapping
- Confirm Keycloak sends NameID = `user@partners.Fabrikam.com`
- Confirm Entra sign-in logs show federated authentication and success

### Validate authorization
- Confirm the user sees the correct report and cannot access unauthorized workspaces.

---
**End**
