# Microsoft Entra ID – Federation & IAM Considerations for Microsoft Fabric
### Implementation Advisory | May 2026

---

## 📋 Background & Context

This advisory is relevant for organizations implementing **Microsoft Fabric** alongside existing applications that use a **third-party Identity Provider (IdP)** such as **CA SiteMinder**, **AD FS**, or similar SAML/WS-Fed IdPs.

Microsoft Fabric uses **Microsoft Entra ID** as its identity backbone for user authentication, role assignments, and workspace identities. Where Entra ID acts as a relying party in a federated trust model, the security controls available are **directly determined by which Entra ID tier is in use**.

> 🔑 **Important — Scope Clarification:** The federation model described in this document is a **workforce authentication model** using a third-party IdP (SiteMinder) within a **workforce tenant**. It does **not** imply B2B collaboration (guest users from external organisations) or B2C/External ID (customer identities). All users remain **internal Entra ID Member users**.

| Scenario | Applicable Here? |
|---|---|
| Workforce federation (SiteMinder ↔ Entra ID) | ✅ Yes |
| B2B collaboration (guest users from external orgs) | ❌ No |
| B2C / External ID (customer/consumer identities) | ❌ No |

---

## 🏗️ How Federation Works — SiteMinder, Active Directory & Entra ID

Understanding the architecture is essential before evaluating licensing tiers. The following explains how these components relate to each other.

Since, SiteMinder does not store users. It is purely an **authentication broker and policy enforcement engine** — not a user directory, the actual users are stored in a backend identity store. SiteMinder supports integration with multiple directory types including **Active Directory (AD)**, **CA Directory**, and generic **LDAP** directories. However, in most enterprise deployments, **Active Directory is the authoritative user store** behind SiteMinder. SiteMinder sits in front of that store and orchestrates authentication against it.

```
┌──────────────────────────────────────────────┐
│  Active Directory / LDAP / CA Directory      │
│  (Source of Truth — stores users)            │
│  * Active Directory is most common in        │
│    enterprise deployments                    │
└──────────────────────────────────────────────┘
            ▲
            │ SiteMinder queries AD/LDAP to verify credentials
            │ 
            ▼
┌──────────────────────────────────┐
│  CA SiteMinder (Policy Server)   │
│  • Intercepts app requests       │
│  • Authenticates via AD/LDAP     │
│  • Enforces access policies      │
│  • Issues SAML assertions        │
└──────────────────────────────────┘
```

### Are Users Created in Entra ID When They Come via SiteMinder?

**Users are not automatically created in Entra ID at login time.** They must already exist in Entra ID as **Member (internal) users**, pre-synced from the same Active Directory that SiteMinder authenticates against — typically via **Azure AD Connect**.

This Azure AD Connect sync is a **separate process from federation** and must be configured independently. Federation defines *how* users authenticate; Azure AD Connect defines *which* users exist in Entra ID. Both are required and must be in place before federated sign-in to Fabric can work.

```
Step 1 — Azure AD Connect (Sync)
        │
        │  Copy user objects from on-prem AD → Entra ID
        │  Users now exist in Entra as Member users
        │
        ▼
Step 2 — Federation Setup (SiteMinder ↔ Entra)
        │
        │  Establish SAML trust between SiteMinder and Entra
        │  Tell Entra: "for @company.com users, redirect auth to SiteMinder"
        │
        ▼
Step 3 — User Logs In
        │
        │  Entra redirects to SiteMinder → SiteMinder authenticates
        │  → SAML assertion returned → Entra matches to existing user object
        │
        ▼
Step 4 — Access Granted ✅

```

> 📌 **Microsoft's guidance states:** *"Single sign-on relies on identical user accounts being represented in both on-premises AD and in Microsoft Entra ID. Directory synchronization (via Azure AD Connect) is responsible for ensuring the same account exists in Entra ID."* This means the sync is a prerequisite — not an outcome — of federation.

> If a user does not already exist as a synced object in Entra ID, the federated login will **fail** — Entra cannot match the SAML assertion to any user object and will not issue a token.

### Are SiteMinder-Federated Users Guests or Internal Members?

**They are internal Member users — not guests.** Guest (B2B) users are a completely separate concept and only apply when users from **outside the organisation** are invited in. SiteMinder-federated users are the organisation's own employees, authenticated via the organisation's own infrastructure.

### The Full Architecture

```
Active Directory (Source of Truth)
        │
        ├─── Azure AD Connect ──────────────► Entra ID
        │                                    (Member users, pre-synced)
        │                                             │
        └─── SiteMinder                               │
                │                                     │
                │ Authenticates user                  │
                │ against AD                          │
                │                                     │
                ▼                                     │
        SAML Assertion issued ───────────────► Entra matches assertion
        "This is John, authenticated"          to existing Member user
                                                       │
                                                       ▼
                                              CA policies applied (P1/P2)
                                                       │
                                                       ▼
                                              Fabric access granted ✅
```

### Key Takeaways

| Question | Answer |
|---|---|
| Where are users actually stored? | Active Directory (most common), CA Directory, or LDAP |
| Does SiteMinder authenticate users? | ✅ Yes — by delegating credential verification to AD/LDAP |
| Are federated users Guests in Entra? | ❌ No — they are **Member (internal)** users |
| Are new users created in Entra at login? | ❌ No — users must already exist, synced via Azure AD Connect |
| What if the user doesn't exist in Entra? | ❌ Login fails — no matching object = no token issued |

---

## 🔢 Core Feature Comparison

| Capability | Free (Standard) | Premium P1 | Premium P2 |
|---|---|---|---|
| **SSO (SAML / WS-Fed / OIDC)** | ✅ Full protocol support — unlimited cloud app SSO | ✅ Same protocol support as Free + CA enforcement capabilities | ✅ Same as P1 |
| **Federation with External IdPs (e.g. SiteMinder)** | ✅ Federated authentication using a third-party IdP (SiteMinder) fully supported — no policy enforcement on federated sessions | ✅ Same federated domain / workforce tenant support + Conditional Access applicable to federated users | ✅ Same as P1 + risk-based controls on federated sessions |
| **Conditional Access (CA) Policies** | ❌ Not available — Security Defaults only (tenant-wide, no granularity). | ✅ Full CA (device, location, app, user/group). Security Defaults must be turned off when using CA. | ✅ All P1 CA + risk-based conditions via Identity Protection |
| **Multi-Factor Authentication (MFA)** | ⚠️ Security Defaults only — MFA enforced for all users with no exceptions or targeting | ✅ Granular MFA via CA (per user/group/app, trusted location exclusions) | ✅ Adaptive/risk-based MFA — challenge only on suspicious sign-ins |
| **B2B Guest User Collaboration** | ✅ Basic guest access — no CA for guests | ✅ CA policies apply to guests + dynamic groups supported | ✅ P1 + Access Reviews + risk evaluation for guests |
| **Dynamic Group Membership** | ❌ Static groups only | ✅ Attribute-based (user or device) dynamic groups | ✅ Included |
| **Identity Protection (Risk Detection)** | ❌ | ❌ | ✅ User & sign-in risk detection (note: limited for federated/external IdP accounts — see below) |
| **Privileged Identity Management (PIM)** | ❌ | ❌ | ✅ JIT role activation + approval workflows |
| **Access Reviews / Identity Governance** | ❌ | ❌ | ✅ Scheduled reviews + entitlement management |
| **SSPR (Self-Service Password Reset)** | ⚠️ Password *change* only (signed-in users who know current password). Forgotten password reset requires M365 Business or P1. No on-prem write-back. | ✅ Full SSPR (including forgotten password reset) + on-prem AD write-back | ✅ Same as P1 |
| **On-Prem App Integration (App Proxy)** | ❌ | ✅ Included | ✅ Included |
| **Workload Identity CA (Service Principals)** | ❌ | ❌ | ❌ Requires separate **Workload Identities Premium** add-on (per service principal — contact Microsoft sales for pricing) |
| **Managed Identities (e.g. Fabric Workspace Identity)** | ✅ Free — no license required | ✅ Free — no license required | ✅ Free — no license required. Not subject to CA policies even with Workload ID add-on, but can be included in Access Reviews (P2). |
| **Approx. Price** | Free (included with M365) | ~$6/user/month | ~$9/user/month |

> ℹ️ All tiers include core directory services, user management, basic audit logs, and Azure AD Connect for hybrid identity sync. P2 includes all P1 capabilities. All tiers **fully support standard federation protocols (SAML 2.0, WS-Fed, OIDC)** — Premium tiers add enforcement and governance capabilities on top, not additional protocol support.

---

## 🔗 Federated Authentication Using a Third-Party IdP (SiteMinder) — What Each Tier Means in Practice

All Entra ID tiers fully support **federated authentication using a third-party IdP (SiteMinder)** within a **federated domain in a workforce tenant**. The technical trust and SSO integration works identically across tiers. The **security posture layered on top** of that federation is what changes:

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

> ℹ️ **Note:** B2B guest access is a **separate and distinct scenario** from workforce federation. It applies only when users from **outside the organisation** (partners, vendors, clients) need access to internal Fabric workspaces. It is **not applicable** to the SiteMinder federation scenario where all users are internal Member users.

All tiers allow B2B guest invitations. The free MAU allowance and per-MAU billing described below apply **only to B2B guest users (UserType = Guest)** — they do **not** apply to internal Member users under workforce federation.

> ⚠️ **Scope note:** The MAU free allowance and External ID pricing referenced here is specific to **B2B guest (external) identities only**. Internal workforce users federated via SiteMinder are unaffected by External ID pricing and are licensed under standard Entra ID user tiers.

| Tier | Guest Capabilities |
|---|---|
| **Free** | Guests can access Fabric if permissions are granted. No CA enforcement on guest sign-ins. No built-in mechanism to review, expire, or govern guest accounts over time. |
| **P1** | CA policies fully apply to guest users (MFA, device compliance, location). Attribute-based dynamic groups (e.g. all users where `userType = Guest`) allow automated, policy-driven guest management without manual intervention. |
| **P2** | Adds **Access Reviews** — scheduled re-certification of guest access to Fabric workspaces and other resources. Essential where guest accounts accumulate across multiple engagements or projects and need periodic validation. |

---

## ⚙️ Workload Identities, Service Principals & Managed Identities

### Service Principals
- Creating and using service principals is **supported in all tiers** at no additional user license cost.
- **Conditional Access for service principals** and **risk detection for workload identities** are **not** covered by standard P1/P2 user licenses.
- These capabilities require the separate **Workload Identities Premium add-on**, licensed per service principal. This add-on is required to create CA policies targeting service principals or to leverage risk-based blocking for SP sign-ins. 
- Without this add-on, CA policies cannot be created or modified to target service principals, and risk-based blocking for SP sign-ins is unavailable.

### Managed Identities (e.g. Fabric Workspace Identity)
- Managed identities are **free to use and require no license** across all tiers.
- They are **not subject to Conditional Access policies** — this limitation applies even with the Workload Identities Premium add-on, which covers app registration-based service principals only.
- Managed identities **can** be included in **Access Reviews** with P2, enabling governance over their permissions and usage patterns.

> ⚠️ If Fabric pipelines handle sensitive data via service principals with privileged access, the Workload Identities Premium add-on should be evaluated based on the number of automated service accounts in scope.

---

## 🔍 Identity Protection — Nuances for Federated & Guest Accounts

Entra ID P2's Identity Protection provides risk assessments **primarily for identities whose credentials and sign-ins are directly managed by Entra ID**. For federated accounts, initial authentication occurs outside Entra ID (e.g. via SiteMinder), which means risk signals available to Identity Protection are inherently partial.

Specifically:

- **Credential-level risk signals** (e.g. leaked password detection) are **only effective for cloud-managed accounts** whose credentials are held by Entra ID. Accounts authenticated via a federated domain in a workforce tenant (e.g. via SiteMinder) are outside the scope of this detection.
- **Behavioural session signals** (IP address, location anomalies, atypical travel) **can still be evaluated** by Identity Protection for federated users after the SAML assertion is accepted by Entra ID.
- **Risk-based CA for federated users** can therefore still mitigate some risk (e.g. blocking sign-ins from anomalous IPs or locations), but it is **not a full substitute** for risk controls enforced within the external IdP itself.
- Stakeholders should understand P2's risk-based CA as a **complementary layer**, not a replacement for security enforcement within SiteMinder or other external IdPs.

---

## ✅ Recommendations

| Requirement | Minimum Tier |
|---|---|
| Basic Fabric access with Entra SSO | Free |
| Workforce federation with third-party IdP (SiteMinder) — SSO only, no Entra policy enforcement | Free |
| Enforce MFA / device policies on federated users | **P1 — Required** |
| Conditional Access for Fabric workspaces (internal + guest) | **P1 — Required** |
| Disable Security Defaults and adopt custom CA policies | **P1** |
| Attribute-based dynamic groups for user/guest policy automation | **P1** |
| Risk-based MFA and adaptive sign-in protection | **P2 — Recommended** |
| Guest Access Reviews / periodic re-certification | **P2 — Recommended** |
| PIM for Fabric / Entra admin roles | **P2 — Recommended** |
| CA and risk detection for service principals | **Workload Identities Premium add-on** |
| Managed Identity governance | **P2** (Access Reviews) |

---

## 🏁 Final Recommendation

### 1️⃣ Entra ID Premium P1 — All Internal Users *(Non-negotiable)*
The baseline for any production Fabric deployment involving federated authentication using a third-party IdP (SiteMinder) within a workforce tenant. Without P1, no Entra-side conditional policies can be enforced on federated users. Upon adopting P1, **Security Defaults should be disabled** in favour of custom Conditional Access policies.

### 2️⃣ Entra ID Premium P2 — Admins, Privileged Users & Sensitive Workspaces *(Strongly Recommended)*
Provides Identity Protection, PIM, and Access Reviews — essential for compliance and ongoing governance in a multi-IdP, workforce tenant environment. Note that Identity Protection's risk-based capabilities for federated accounts are partial (behavioural signals only) and should be treated as a complementary control layer alongside SiteMinder's own protections.

### 3️⃣ Workload Identities Premium Add-on — Evaluate per Service Principal *(As Needed)*
Required for any service principal carrying privileged access to Fabric data or external systems. Licensed per service principal — not applicable to managed identities. 

---

## 📚 Reference Notes

> 1. **Federated SSO & User Sync Requirement:** Microsoft states: *"Single sign-on relies on identical user accounts being represented in both on-premises AD and in Microsoft Entra ID. Directory synchronization is responsible for ensuring the same account exists in Entra ID."* — [Microsoft Entra Hybrid Identity Documentation](https://learn.microsoft.com/en-us/entra/identity/hybrid/)
>
> 2. **Workload Identities Premium:** Required for Conditional Access and risk detection targeting service principals. — [Microsoft Entra Workload Identities](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview)
>
> 3. **Security Defaults vs Conditional Access:** These are mutually exclusive. Organisations adopting CA policies must disable Security Defaults. — [Microsoft Security Defaults Documentation](https://learn.microsoft.com/en-us/entra/fundamentals/security-defaults)

---

> 📎 *All feature details are based on Microsoft's official Entra ID documentation and current licensing terms as of May 2026. This document covers workforce federation scenarios only. B2B guest and B2C/External ID scenarios are out of scope.*
