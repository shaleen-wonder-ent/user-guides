# Microsoft Entra ID – Federation & IAM Considerations for Microsoft Fabric
### Implementation Advisory | May 2026

---

## 📋 Background & Context

This advisory is relevant for organizations implementing **Microsoft Fabric** alongside existing applications that use a **third-party Identity Provider (IdP)** such as **CA SiteMinder**, **AD FS**, or similar SAML/WS-Fed IdPs.

Microsoft Fabric uses **Microsoft Entra ID** as its identity backbone for user authentication, role assignments, and workspace identities. Where Entra ID acts as a relying party in a federated trust model, the security controls available are **directly determined by which Entra ID tier is in use**.

---

## 🔢 Core Feature Comparison

| Capability | Free (Standard) | Premium P1 | Premium P2 |
|---|---|---|---|
| **SSO (SAML / WS-Fed / OIDC)** | ✅ Full protocol support — unlimited cloud app SSO | ✅ Same protocol support as Free + CA enforcement capabilities | ✅ Same as P1 |
| **Federation with External IdPs (e.g. SiteMinder)** | ✅ Full SAML 2.0 / WS-Fed integration supported — no policy enforcement on federated sessions | ✅ Same federation support + Conditional Access applicable to federated users | ✅ Same as P1 + risk-based controls on federated sessions |
| **Conditional Access (CA) Policies** | ❌ Not available — Security Defaults only (tenant-wide, no granularity). Must be disabled before enabling CA in P1/P2. | ✅ Full CA (device, location, app, user/group). Security Defaults must be turned off when using CA. | ✅ All P1 CA + risk-based conditions via Identity Protection |
| **Multi-Factor Authentication (MFA)** | ⚠️ Security Defaults only — MFA enforced for all users with no exceptions or targeting | ✅ Granular MFA via CA (per user/group/app, trusted location exclusions) | ✅ Adaptive/risk-based MFA — challenge only on suspicious sign-ins |
| **B2B Guest User Collaboration** | ✅ Basic guest access — first 50,000 MAUs free per month (resets monthly); no CA for guests | ✅ CA policies apply to guests + dynamic groups supported | ✅ P1 + Access Reviews + risk evaluation for guests |
| **Dynamic Group Membership** | ❌ Static groups only | ✅ Attribute/role-based dynamic groups | ✅ Included |
| **Identity Protection (Risk Detection)** | ❌ | ❌ | ✅ User & sign-in risk detection (note: limited for federated/external IdP accounts — see below) |
| **Privileged Identity Management (PIM)** | ❌ | ❌ | ✅ JIT role activation + approval workflows |
| **Access Reviews / Identity Governance** | ❌ | ❌ | ✅ Scheduled reviews + entitlement management |
| **SSPR (Self-Service Password Reset)** | ⚠️ Password *change* only (signed-in users who know current password). Forgotten password reset requires M365 Business or P1. No on-prem write-back. | ✅ Full SSPR (including forgotten password reset) + on-prem AD write-back | ✅ Same as P1 |
| **On-Prem App Integration (App Proxy)** | ❌ | ✅ Included | ✅ Included |
| **Workload Identity CA (Service Principals)** | ❌ | ❌ | ❌ Requires separate **Workload Identities Premium** add-on (pricing varies by agreement — contact Microsoft sales) |
| **Managed Identities (e.g. Fabric Workspace Identity)** | ✅ Free — no license required | ✅ Free — no license required | ✅ Free — no license required. Not subject to CA policies even with Workload ID add-on, but can be included in Access Reviews (P2). |
| **Approx. Price** | Free (included with M365) | ~$6/user/month | ~$9/user/month |

> ℹ️ All tiers include core directory services, user management, basic audit logs, and Azure AD Connect for hybrid identity sync. P2 includes all P1 capabilities. All tiers **fully support standard federation protocols (SAML 2.0, WS-Fed, OIDC)** — Premium tiers add enforcement and governance capabilities on top, not additional protocol support.

---

## 🔗 Federation with External IdPs (e.g. SiteMinder) — What Each Tier Means in Practice

All Entra ID tiers fully support SAML 2.0 / WS-Fed federation — the **technical trust** and SSO integration with a third-party IdP like SiteMinder works identically across tiers. The **security posture layered on top** of that federation is what changes:

| Tier | What You Get |
|---|---|
| **Free** | Entra ID fully accepts the IdP's SAML assertion and grants access. No Conditional Access or policy enforcement is possible on those federated sessions. All access control responsibility sits with the external IdP. |
| **P1** | Same full federation support, with the addition of Conditional Access applied to federated sessions — enforce MFA, device compliance, and location restrictions at the Entra layer, regardless of what the external IdP enforces. |
| **P2** | Builds on P1 with risk-based evaluation. Entra can assess session signals (IP, location anomalies, behaviour patterns) for federated users. **Important caveat:** credential-level risk signals (e.g. leaked password detection) are not available for accounts whose passwords are managed by the external IdP — risk analysis is partial, not a full substitute for on-prem IdP protections. |

> ⚠️ **P1 is the minimum required** to enforce any Entra-side security controls on federated users. Without it, the organisation is fully dependent on the external IdP's own enforcement with no compensating controls available in Azure.

---

## 🔐 Conditional Access & MFA — Fabric Scenarios

> 📌 **Important:** Security Defaults (enabled by default in new tenants) and Conditional Access policies are **mutually exclusive**. Once an organisation moves to P1/P2 and configures custom CA policies, Security Defaults **must be disabled** to avoid conflicts. Microsoft recommends this transition as part of adopting a Zero Trust posture.

| Scenario | Free | P1 | P2 |
|---|---|---|---|
| Enforce MFA for all users | ✅ (via Security Defaults — no exceptions) | ✅ (targeted via CA) | ✅ |
| Enforce MFA for guests/external users only | ❌ | ✅ | ✅ |
| Skip MFA for trusted corporate IPs | ❌ | ✅ Named Locations | ✅ |
| Enforce MFA only when risk is detected | ❌ | ❌ | ✅ Identity Protection |
| Block access from non-compliant devices | ❌ | ✅ Intune integration | ✅ |
| Block access from specific geographies | ❌ | ✅ | ✅ |
| Fabric-specific CA policy (app-scoped) | ❌ | ✅ | ✅ |

---

## 👥 B2B Guest Users — Governance & Security

All tiers allow B2B guest invitations using the same MAU-based billing model. The first **50,000 monthly active guest users are free per month** (this allowance resets monthly; usage beyond 50,000 MAUs in a given month incurs additional charges per Microsoft's External ID pricing). This free allowance applies equally across Free, P1, and P2 tiers.

| Tier | Guest Capabilities |
|---|---|
| **Free** | Guests can access Fabric if permissions are granted. No CA enforcement on guest sign-ins. No built-in mechanism to review, expire, or govern guest accounts over time. |
| **P1** | CA policies fully apply to guest users (MFA, device compliance, location). Dynamic groups (e.g. all users where `userType = Guest`) allow automated, policy-driven guest management without manual intervention. |
| **P2** | Adds **Access Reviews** — scheduled re-certification of guest access to Fabric workspaces and other resources. Essential where guest accounts accumulate across multiple engagements or projects and need periodic validation. Risk-based evaluation of guest sign-ins is also available, though limited for guests authenticated via external IdPs (see Identity Protection note). |

---

## ⚙️ Workload Identities, Service Principals & Managed Identities

### Service Principals
- Creating and using service principals is **supported in all tiers** at no additional user license cost.
- **Conditional Access for service principals** and **risk detection for workload identities** are **not** covered by standard P1/P2 user licenses.
- These capabilities require the separate **Workload Identities Premium add-on**. Pricing varies by enterprise agreement — contact Microsoft sales for current rates (commonly referenced at approximately $3/SP/month, but not published as a fixed list price).
- Without this add-on, CA policies cannot be created or modified to target service principals, and risk-based blocking for SP sign-ins is unavailable.

### Managed Identities (e.g. Fabric Workspace Identity)
- Managed identities are **free to use and require no license** across all tiers.
- They are **not subject to Conditional Access policies** — this limitation applies even with the Workload Identities Premium add-on, which covers app registration-based service principals only.
- Managed identities **can** be included in **Access Reviews** with P2, enabling governance over their permissions and usage patterns.

> ⚠️ If Fabric pipelines handle sensitive data via service principals with privileged access, the Workload Identities Premium add-on should be evaluated based on the number of automated service accounts in scope.

---

## 🔍 Identity Protection — Nuances for Federated & Guest Accounts

Entra ID P2's Identity Protection can evaluate sign-in signals for **any account** in the directory — including guests and federated users — based on observable session data such as IP address, location, and behavioural patterns.

However, the following limitations apply:

- **Credential-level risk signals** (e.g. leaked password detection) are **only effective for cloud-managed accounts** whose credentials are held by Entra ID. Accounts whose passwords are managed by an external IdP (e.g. SiteMinder) are outside the scope of this detection.
- **Risk-based CA for federated users** can still mitigate some risk (e.g. blocking sign-ins from anomalous IPs or locations), but it is **not a full substitute** for risk controls in the on-prem IdP itself.
- Stakeholders should understand P2's risk-based CA as a **complementary layer**, not a replacement for security enforcement within the external IdP.

---

## ✅ Recommendations

| Requirement | Minimum Tier |
|---|---|
| Basic Fabric access with Entra SSO | Free |
| Third-party IdP federation — SSO only, no Entra policy enforcement | Free |
| Enforce MFA / device policies on federated users | **P1 — Required** |
| Conditional Access for Fabric workspaces (internal + guest) | **P1 — Required** |
| Disable Security Defaults and adopt custom CA policies | **P1** |
| Dynamic groups for guest/user policy automation | **P1** |
| Risk-based MFA and adaptive sign-in protection | **P2 — Recommended** |
| Guest Access Reviews / periodic re-certification | **P2 — Recommended** |
| PIM for Fabric / Entra admin roles | **P2 — Recommended** |
| CA and risk detection for service principals | **Workload Identities Premium add-on** |
| Managed Identity governance | **P2** (Access Reviews) |

---

## 🏁 Final Recommendation

### 1️⃣ Entra ID Premium P1 — All Internal Users *(Non-negotiable)*
The baseline for any production Fabric deployment involving third-party IdP federation. Without P1, no Entra-side conditional policies can be enforced on federated or guest users. Upon adopting P1, **Security Defaults should be disabled** in favour of custom Conditional Access policies.

### 2️⃣ Entra ID Premium P2 — Admins, Privileged Users & Sensitive Workspaces *(Strongly Recommended)*
Provides Identity Protection, PIM, and Access Reviews — essential for compliance and ongoing governance in a multi-IdP, multi-tenant environment. Note that risk-based capabilities for federated accounts are partial and should be treated as a complementary control layer.

### 3️⃣ Workload Identities Premium Add-on — Evaluate per Service Principal *(As Needed)*
Required for any service principal carrying privileged access to Fabric data or external systems. Not applicable to managed identities. Pricing is agreement-based — engage Microsoft sales for accurate figures.

---

> 📎 *All feature details are based on Microsoft's official Entra ID documentation and current licensing terms as of May 2026. Guest usage pricing is subject to Microsoft's External ID MAU billing model — refer to the [Microsoft External ID pricing page](https://learn.microsoft.com/en-us/azure/active-directory-b2c/) for current rates.*
