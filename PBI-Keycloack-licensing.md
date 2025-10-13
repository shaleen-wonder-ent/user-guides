# Power BI Licensing & Management Comparison  
_Keycloak Federation via OIDC (Guests) vs SAML Domain Federation (Members)_

**Scope:** Practical differences in planning, assigning, automating, and optimizing Power BI licenses when users authenticate through Keycloak using either OIDC External IdP (Guest accounts) or SAML Domain Federation (Member accounts).  
**Disclaimer:** Always validate with the latest Microsoft licensing documentation before final decisions.

---

## 1. Core Difference Summary

| Aspect | OIDC External IdP (Guest Accounts) | SAML Domain Federation (Member Accounts) |
|--------|------------------------------------|-------------------------------------------|
| Identity Type in Entra | `userType = Guest` | `userType = Member` |
| Typical UPN Pattern | `{guid}#EXT#@tenant.onmicrosoft.com` | `user@company.com` |
| Group-Based Licensing | Supported (static / dynamic; dynamic conditions limited for guests) | Fully supported (rich dynamic rules) |
| Dynamic Group Rules | Basic (`user.userType -eq "Guest"`) | Rich (domain, dept, jobTitle, etc.) |
| Bulk License Assignment Complexity | Higher (event automation needed) | Lower (domain / attribute groups) |
| Premium Capacity Benefit | Free guest viewers for Premium workspaces | Free member viewers for Premium workspaces |
| Pro / PPU Need (Non-Premium Content) | Must assign individually or via group | Straightforward via groups |
| Authoring / Publishing | Requires Pro or PPU per guest | Requires Pro or PPU per member |
| License Reclamation | Custom guest inactivity scripts | Standard offboarding / HR-driven |
| MFA / Conditional Access Targeting | Extra care (guest exclusions, groups) | Simpler (attribute targeting) |
| Cost Predictability | More variable (external spikes) | More stable (workforce size) |
| Admin Reporting Clarity | Need to filter guests vs internal | Native workforce view |
| Future Migration Risk | Guest→Member migration overhead | None (already members) |
| Ease of Initial Rollout | Faster | Moderate (domain federation complexity) |
| Impact of IdP Outage | Only guests blocked | Entire workforce sign-in if federated domain |

---
# Comparison Table

| Feature / Behavior | **SAML Federation (Recommended for Power BI)** | **OIDC Federation (External Identities / B2B)** |
|--------------------|---------------------------------------------|------------------------------------------------|
| **Supported for Power BI (M365)** | ✅ Fully supported — required for domain-level federation. | ❌ Not supported for tenant-level (M365 core) login. Can only be used for external guest users. |
| **Primary Federation Model** | Domain-level federation (Entra trusts Keycloak as IdP for the company domain). | External identity federation (OIDC IdP used for B2B or app-specific login). |
| **User Identity Source** | Internal Entra users (federated domain). | External B2B guest users (created dynamically or invited). |
| **License Assignment (Power BI Pro / PPU / Free)** | Assigned to internal Entra users; standard M365 license assignment process applies. | Guest users cannot hold first-party licenses like Power BI Pro; they rely on the inviter’s tenant capacity. |
| **Power BI Premium Capacity** | Works seamlessly — any licensed user or capacity-based workspace is accessible. | Only accessible if the Power BI workspace is shared to guest users via B2B; limited reporting and dataset refresh capabilities. |
| **License Enforcement** | Managed by Entra via the user’s object ID and UPN domain. | Managed through guest user linkage; limited in functionality and cannot consume tenant-wide licenses. |
| **Conditional Access / MFA** | Fully functional; Entra policies apply to federated users. | Limited; applies based on B2B policies and external IdP configurations. |
| **SCIM / Lifecycle Management** | Possible via SCIM or Entra provisioning API to manage local users. | Not needed (guest users managed by invitation or External Identities). |
| **Best Use Case** | Corporate users who must access Power BI and Microsoft 365 apps under company domain (`@yourcompany.com`). | External contractors or partners who access Power BI shared reports through B2B guest accounts. |
| **Authentication Flow** | Power BI → Entra → Keycloak (SAML) → Entra → Power BI | Power BI → Entra → External IdP (OIDC guest login) → Entra → Power BI |
| **License Management Integration** | Native; supports per-user licenses (Power BI Pro/PPU) and capacity-based (Premium). | Partial; guest users can only view shared content; cannot hold or consume tenant licenses directly. |
| **M365 App Integration (Teams, Excel, Outlook, etc.)** | ✅ Fully supported. | ❌ Not supported for OIDC-based federation to Keycloak. |

---

## Summary

| Scenario | Recommended Protocol | Reason |
|-----------|----------------------|---------|
| Organization wants all employees to log in via Keycloak and use Power BI, Teams, Outlook, etc. | **SAML** | Required for domain-level Entra federation and license enforcement. |
| External consultants or vendors accessing shared Power BI dashboards. | **OIDC (B2B guest)** | Simplifies external access without granting internal licenses. |

---

## Operational Notes

- Power BI licenses are **checked after authentication** inside Entra, so federation only affects identity proofing, not licensing logic.
- For SAML, users appear as internal (`user@yourcompany.com`) and can receive Power BI Pro or Premium Per User (PPU) licenses directly.
- For OIDC, users appear as **guest accounts** (`user_external#EXT#@yourtenant.onmicrosoft.com`) and cannot be assigned first-party licenses.
- Power BI Premium Capacity is **tenant-based**, so content can be shared to OIDC guest users but governed by workspace permissions.

---

**Conclusion:**  
For Power BI (both Per User and Premium Capacity models), **SAML federation** is mandatory to maintain native Microsoft 365 license assignment and seamless Power BI access.  
Use **OIDC federation** only for **external B2B guest access** — not for full tenant integration.

