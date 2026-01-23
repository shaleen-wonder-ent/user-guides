# External users (partners) accessing Power BI in MedAdv tenant without DNS domain verification
 
**Context:** MedAdv application uses Keycloak for SSO (partner identity federates to MedAdv Keycloak). The application needs to launch Power BI reports that live in the **MedAdv Entra tenant**. 
External users have emails like `user@sharp.com`. Sharp (or other partners) will **not** allow DNS TXT domain verification for their domain(s) in MedAdv Entra.

---

## 1) Context

Because **Power BI is protected by Microsoft Entra ID**, every user who accesses Power BI must authenticate through **MedAdv’s Entra tenant** (the tenant that owns the Power BI workspace).

To get “true SSO” from partner identity (e.g., `Sharp IM → MedAdv Keycloak → Power BI`), the most direct method is **Entra Direct Federation + domain routing**. However, **domain routing requires the external 
domain be verified** in MedAdv Entra, typically via DNS TXT record published in the partner domain (e.g., `sharp.com`).

Since Sharp will not provide DNS verification, **domain-based routing for `@sharp.com` cannot be configured** in MedAdv Entra. As a result, Power BI sign-in cannot be deterministically routed to Keycloak using `user@sharp.com`.

Therefore, MedAdv must choose one of the following production-supported alternatives:

1. **Use Microsoft-managed guest authentication** for Power BI (OTP / Microsoft account)  
2. **Use a MedAdv-controlled sign-in domain** for external users (e.g., `user@partners.medadv.com`) to preserve SSO via Keycloak  


---

## 2) What is fixed (non-negotiable constraints)

### 2.1 Power BI always authenticates via Entra ID
Power BI (PowerBI.com) is an Entra-protected resource. Users must obtain tokens from the Entra tenant that hosts the Power BI content (MedAdv tenant in our case).

### 2.2 Why domain verification matters for “true SSO”
To route sign-in for `user@sharp.com` to a custom IdP (Keycloak), Entra needs a **routing rule** based on the user’s domain. In the current Entra design, routing rules typically require that domain to be **verified** in the tenant (DNS proof of control). Without verification, Entra cannot reliably/securely apply domain routing for that domain.

---

## 3) Option A (Recommended when partner DNS is not available): Microsoft-managed B2B guest authentication for Power BI

### 3.1 What it is
External users are created as **B2B guest users** in MedAdv Entra. They authenticate to Power BI using Microsoft-managed guest methods, typically:
- **Email one-time passcode (OTP)**, or
- **Microsoft account** (if the email is already associated with one), or
- other Microsoft-managed flows allowed by tenant policy.

Keycloak SSO continues to be used for the MedAdv application, but Power BI authentication becomes a separate (Microsoft-managed) step.

### 3.2 One-time setup (MedAdv side)
1. **Enable B2B collaboration** (default in most tenants).
2. Ensure guest invitation/redemption is allowed by policy.
3. Establish Power BI permissions model:
   - Use Entra groups for access to workspaces/reports (recommended), or
   - manage direct user permissions.

### 3.3 Runtime end-user flow (first-time)
1. User signs into MedAdv application via Keycloak (Sharp IM → Keycloak).
2. User clicks “Open Power BI report”.
3. MedAdv backend invites the user into MedAdv Entra as a guest (JIT invitation).
4. User completes invitation redemption.
5. User is redirected to Power BI report.
6. Power BI redirects to MedAdv Entra for sign-in.
7. Since `sharp.com` is not federated/routed, Entra prompts user for a Microsoft-managed method:
   - OTP is sent to `user@sharp.com` (or user logs into Microsoft account).
8. After successful authentication, Power BI renders the report (assuming permissions are granted).

### 3.4 Runtime end-user flow (subsequent times)
- User repeats:
  - MedAdv app login via Keycloak
  - Power BI launch
  - **Entra sign-in (may be silent if session still valid, or may require OTP again based on session policy)**

### 3.5 Pros
- **No dependency on partner DNS** or partner IT configuration.
- Works regardless of partner identity provider (Sharp can be Entra, Okta, Ping, custom IM, etc.).
- Standard Microsoft-supported B2B approach.
- Faster to implement and operationalize.

### 3.6 Cons / UX impact
- Not seamless SSO from Keycloak into Power BI.
- Users may have to:
  - receive OTP emails and enter codes,
  - handle “code not received” issues,
  - re-auth periodically depending on session and CA policy.
- Support overhead may increase due to OTP/email delivery issues.
- If partners have strict mail filtering, OTP emails may be delayed/blocked.

### 3.7 When to choose Option A
Choose this when:
- partner will not do DNS verification
- you need the fastest production path
- an additional authentication step for Power BI is acceptable

---

## 4) Option B (Recommended when true SSO is required): MedAdv-controlled sign-in domain for external users (SSO preserved)

### 4.1 What it is
Instead of using `user@sharp.com` as the Entra sign-in identifier, MedAdv issues external users a MedAdv-controlled sign-in identity, for example:
- `user@partners.medadv.com`

The “real email” (e.g., `user@sharp.com`) is stored as an attribute in Keycloak/MedAdv systems for communications and auditing, but **Entra routing is done using the MedAdv-controlled domain** which MedAdv can verify and route to Keycloak.

### 4.2 One-time setup (MedAdv side)
1. Create/own a routable domain, e.g., `partners.medadv.com`.
2. Verify `partners.medadv.com` in MedAdv Entra (DNS TXT record, owned by MedAdv).
3. Configure Entra External Identities:
   - **Direct Federation** to MedAdv Keycloak (SAML)
   - Domain routing (attach `partners.medadv.com` to the Keycloak federation IdP)
4. Configure Keycloak:
   - UserName/NameID emitted as `user@partners.medadv.com`
   - Keep `realEmail=user@sharp.com` as an attribute

### 4.3 Runtime end-user flow
1. User signs into MedAdv application via Keycloak (partner IM → Keycloak).
2. User clicks “Open Power BI report”.
3. MedAdv backend invites `user@partners.medadv.com` (guest in MedAdv Entra).
4. User redeems invitation.
5. User is redirected to Power BI.
6. Power BI redirects to MedAdv Entra.
7. MedAdv Entra sees sign-in domain `partners.medadv.com`, which is routed to Keycloak.
8. Entra redirects to Keycloak; Keycloak has SSO and returns signed assertion.
9. Entra issues token; Power BI renders.

### 4.4 Pros
- Provides the desired seamless sign-in pattern for Power BI:
  - **Power BI → Entra → Keycloak SSO** (silent if Keycloak session exists)
- No dependency on partner DNS verification.
- Works even if partner IdP is not Entra (Okta/anything) because partner only needs to federate to Keycloak for the MedAdv app (which they already do).

### 4.5 Cons / changes required
- Users will have a MedAdv-managed sign-in identifier that differs from their corporate email.
- Onboarding and communications must be clear:
  - “Your Power BI access identity is user@partners.medadv.com”
- If any downstream systems expect `@sharp.com` as the login name, additional mapping is required.
- Requires Keycloak and MedAdv app to treat “sign-in ID” and “real email” as separate fields.

### 4.6 When to choose Option B
Choose this when:
- true SSO into Power BI is mandatory
- partner DNS verification is not possible
- MedAdv can own and operate a dedicated external sign-in domain

---


## 6) Recommendation (based on current constraints)

Given:
- Power BI is in **MedAdv Entra**
- partner will not verify `sharp.com` in MedAdv Entra
- partner identity could be anything (not necessarily Entra)

**The only production-supported choices are:**
- **Option A:** Microsoft-managed B2B guest authentication (OTP/Microsoft account)  
  *Best for fastest delivery; not full SSO*
- **Option B:** MedAdv-controlled sign-in domain (`@partners.medadv.com`) routed to Keycloak  
  *Best for SSO; requires identity model change*

There is no secure, supported way to keep `user@sharp.com` as the sign-in identifier and still have Entra reliably route to Keycloak without partner domain verification.

---

## 7) Customer decision questions (to choose the option)

1. Is an additional Power BI authentication step acceptable (OTP / Microsoft account)?  
   - If **Yes** → Option A is recommended.
   - If **No** → Option B is required (or revisit Power BI Embedded).

2. If Option B: Can MedAdv issue partner-facing sign-in identities under a MedAdv-controlled domain (e.g., `@partners.medadv.com`)?

3. What is the expected support posture?
   - Option A requires operational readiness for OTP delivery/support tickets.
   - Option B requires onboarding/training for the alternate sign-in identity.

---

## 8) What remains the same regardless of option
- Power BI access control must still be enforced in the MedAdv tenant (workspaces, reports).
- External users must exist as B2B guests in MedAdv Entra.
- medadv360 will still manage JIT onboarding via Graph invitations (Option 5A).

---
