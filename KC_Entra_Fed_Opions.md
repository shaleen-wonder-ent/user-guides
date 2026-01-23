# Partner users accessing Power BI in MedY tenant


---

## Why this problem exists (in plain terms)

Power BI is protected by **Microsoft Entra ID**.  
That means anyone who opens a Power BI report in the MedY tenant must authenticate through **MedY Entra** (because the Power BI content belongs to MedY’s tenant).

The “best” single sign-on experience would be:
- Partner user signs in once (via partner IdP → Keycloak), and
- Power BI automatically recognizes that session without additional prompts.

However, for Entra to automatically redirect a user like `user@XXX.com` to an external IdP (Keycloak), Entra typically requires the partner domain (`XXX.com`) to be **verified** in MedY Entra. Verification is done via a DNS record in the partner domain—something the partner has stated they will not do.

As a result, MedY cannot configure domain-based routing for `@XXX.com`, and must choose one of the two supported alternatives below.

---

## One of the production-supported options:

### Option A (Recommended in most cases): Use Microsoft-managed guest authentication for Power BI (OTP / Microsoft account)

**What it means**
- Partner users are added to MedY Entra as **guest users**.
- When they open a Power BI report, Microsoft Entra will authenticate them using Microsoft-managed methods, typically:
  - **Email one-time passcode (OTP)**, or
  - **Microsoft account** (if their email is already registered with Microsoft).

**User experience (high level)**
- User signs into MedY application normally (via Keycloak).
- User clicks Power BI link.
- First time: user may need to “accept an invitation.”
- Entra may then ask for a **one-time code** sent to their email (or Microsoft account sign-in).
- After successful sign-in, the report opens.

**Pros**
- **No dependency on partner DNS** or partner IT changes.
- Works with any partner IdP (Entra, Okta, Ping, etc.).
- Standard Microsoft approach for partner access.
- Fastest to implement and support operationally.

**Cons / tradeoffs**
- Not fully seamless SSO into Power BI; users may see an extra prompt/code.
- Potential support overhead (OTP emails delayed, code not received, re-auth prompts).
- Sign-in frequency depends on MedY security policies (Conditional Access, session lifetime).

**Best fit when**
- Partner will not verify DNS, and
- leadership is comfortable with an additional verification step for Power BI access.

---

## Another production-supported option:

# Option (SSO-optimized): Keycloak → Entra ID (Direct Federation + Domain Routing) → Power BI

## What it means
MedY configures Microsoft Entra to:
- Trust **MedY Keycloak** as a federated sign-in provider (Direct Federation using SAML), and
- **Route partner email domains** (e.g., `@XXX.com`) to Keycloak during sign-in.

Partner users still exist in MedY Entra as **B2B guest users** so they can be authorized to Power BI workspaces/reports. When a partner user signs in to Power BI, Entra recognizes their email domain and redirects authentication to Keycloak.

**Key dependency:** each partner domain that should be routed (e.g., `XXX.com`) must be **verified** in the MedY Entra tenant (typically via a DNS TXT record published by the partner).

---

## User experience (high level)
**First-time access**
1. User signs into the MedY application via Keycloak (partner IM → Keycloak).
2. User clicks the Power BI link.
3. MedY onboards the user as a guest in MedY Entra (invitation/acceptance) and assigns permissions.
4. User opens Power BI report.
5. Power BI redirects to MedY Entra for sign-in.
6. Entra sees the user’s domain (e.g., `@XXX.com`) and redirects to Keycloak.
7. Keycloak completes sign-in (often silent because the user is already signed into the MedY app).
8. Power BI report loads.

**Subsequent access**
- User clicks Power BI link → sign-in is typically quick/silent (subject to security policies and session lifetime).

---

## Pros
- **Best SSO experience** into Power BI while keeping partner emails (e.g., `user@XXX.com`) as the login identifier.
- Works even if partner’s IdP is not Microsoft (Okta/Ping/etc.), because partner only needs to federate to Keycloak.
- Centralized authentication broker (Keycloak) and centralized authorization/governance (MedY Entra + Power BI).
- Scales across multiple partners once the pattern is established.

---

## Cons / tradeoffs
- **Partner dependency (critical):** each partner domain must be verified in MedY Entra (DNS TXT record). If a partner cannot/will not do DNS verification, this option cannot be implemented for that domain.
- Ongoing operations per domain:
  - domain verification, routing configuration, lifecycle governance.
- Requires Keycloak to be production-grade and internet reachable (stable URL, certificates, availability).
- User prompts may still occur due to Conditional Access (MFA, session limits, device requirements).

---

## Best fit when
- Partner is willing to complete DNS verification for their domain(s), and
- Seamless SSO into Power BI is a priority, and
- MedY wants a scalable approach for multiple partner organizations with consistent governance in MedY Entra.

---



	
