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
| **SSO (SAML / WS-Fed / OIDC)** | ✅ Unlimited cloud app SSO | ✅ All Free + CA enforcement | ✅ Same as P1 |
| **Federation with External IdPs (e.g. SiteMinder)** | ✅ Basic only — no policy enforcement on federated sessions | ✅ Full federation + Conditional Access | ✅ Same as P1 + risk-based controls |
| **Conditional Access (CA) Policies** | ❌ Security Defaults only (tenant-wide, no granularity) | ✅ Full CA (device, location, app, user/group) | ✅ All P1 + risk-based conditions |
| **Multi-Factor Authentication (MFA)** | ⚠️ All users or none — no exceptions | ✅ Granular MFA via CA (per user/group/app) | ✅ Adaptive/risk-based MFA |
| **B2B Guest User Collaboration** | ✅ Basic — first 50k MAUs free, no CA for guests | ✅ CA applies to guests + dynamic groups | ✅ P1 + Access Reviews + risk evaluation |
| **Dynamic Group Membership** | ❌ Static groups only | ✅ Attribute/role-based dynamic groups | ✅ Included |
| **Identity Protection (Risk Detection)** | ❌ | ❌ | ✅ User & sign-in risk detection |
| **Privileged Identity Management (PIM)** | ❌ | ❌ | ✅ JIT role activation + approval workflows |
| **Access Reviews / Identity Governance** | ❌ | ❌ | ✅ Scheduled reviews + entitlement management |
| **SSPR (Self-Service Password Reset)** | ⚠️ Cloud-only, limited | ✅ Full SSPR with on-prem AD write-back | ✅ Same as P1 |
| **On-Prem App Integration (App Proxy)** | ❌ | ✅ Included | ✅ Included |
| **Workload Identity CA (Service Principals)** | ❌ | ❌ Add-on required | ❌ Add-on required (~$3/SP/month) |
| **Approx. Price** | Free (included with M365) | ~$6/user/month | ~$9/user/month |

> ℹ️ All tiers include core directory services, user management, basic audit logs, and Azure AD Connect for hybrid identity sync. P2 includes all P1 capabilities.

---

## 🔗 Federation with External IdPs (e.g. SiteMinder) — What Each Tier Means in Practice

All Entra ID tiers support SAML 2.0 / WS-Fed federation. The **technical trust** between a third-party IdP and Entra ID can be established regardless of tier. However, the **security posture** changes significantly:

| Tier | What You Get |
|---|---|
| **Free** | Azure accepts the IdP's SAML assertion and grants access — no further conditions applied. All access control responsibility sits with the external IdP. |
| **P1** | Same federation trust, but Entra ID can layer Conditional Access on top of the federated session — enforce MFA, device compliance, and location restrictions regardless of what the external IdP enforces. |
| **P2** | Builds on P1 with risk-based evaluation — session signals (IP, location anomalies) assessed even for federated users. Note: credential-level risk signals are limited for externally managed accounts. |

> ⚠️ **P1 is the minimum required** to enforce any Azure-side security controls on federated users. Without it, the organisation is fully dependent on the external IdP's own enforcement capabilities with no fallback.

---

## 🔐 Conditional Access & MFA — Fabric Scenarios

| Scenario | Free | P1 | P2 |
|---|---|---|---|
| Enforce MFA for all users | ✅ (no exceptions) | ✅ (targeted via CA) | ✅ |
| Enforce MFA for guests/external users only | ❌ | ✅ | ✅ |
| Skip MFA for trusted corporate IPs | ❌ | ✅ Named Locations | ✅ |
| Enforce MFA only when risk is detected | ❌ | ❌ | ✅ Identity Protection |
| Block access from non-compliant devices | ❌ | ✅ Intune integration | ✅ |
| Block access from specific geographies | ❌ | ✅ | ✅ |
| Fabric-specific CA policy (app-scoped) | ❌ | ✅ | ✅ |

---

## 👥 B2B Guest Users — Governance & Security

All tiers allow B2B guest invitations. First **50,000 monthly active guests are free** across all tiers.

| Tier | Guest Capabilities |
|---|---|
| **Free** | Guests can access Fabric if permissions are granted. No CA enforcement. No mechanism to review or expire guest accounts. |
| **P1** | CA policies apply to guests (MFA, device compliance, location). Dynamic groups allow auto-managed guest policies. |
| **P2** | Adds **Access Reviews** — scheduled re-certification of guest access to Fabric workspaces. Critical for environments where guest accounts accumulate over time across multiple engagements or projects. |

---

## ⚙️ Workload Identities & Fabric Service Principals

- Creating and using **service principals is supported in all tiers** — no user license consumed.
- **Conditional Access for service principals** and **risk detection for workload identities** require the **Workload Identities Premium add-on** (~$3/SP/month) — not included in standard P1/P2.
- Fabric's **Workspace Identity** (managed identity) is outside CA/risk detection scope but can be included in **Access Reviews** with P2.

> ⚠️ If Fabric pipelines handle sensitive data via service principals, assess the Workload Identities Premium add-on separately based on the number of automated service accounts in scope.

---

## ✅ Recommendations

| Requirement | Minimum Tier |
|---|---|
| Basic Fabric access with Entra SSO | Free |
| Third-party IdP federation — SSO only, no Azure policy enforcement | Free |
| Enforce MFA / device policies on federated users | **P1 — Required** |
| Conditional Access for Fabric workspaces (internal + guest) | **P1 — Required** |
| Dynamic groups for guest/user policy automation | **P1** |
| Risk-based MFA and adaptive sign-in protection | **P2 — Recommended** |
| Guest Access Reviews / periodic re-certification | **P2 — Recommended** |
| PIM for Fabric / Entra admin roles | **P2 — Recommended** |
| CA and risk detection for service principals | **Workload Identities Premium add-on** |

---

## 🏁 Final Recommendation

### 1️⃣ Entra ID Premium P1 — All Internal Users *(Non-negotiable)*
The baseline for any production Fabric deployment with third-party IdP federation. Without P1, no Azure-side conditional policies can be enforced on federated or guest users.

### 2️⃣ Entra ID Premium P2 — Admins, Privileged Users & Sensitive Workspaces *(Strongly Recommended)*
Provides Identity Protection, PIM, and Access Reviews — essential for compliance and ongoing governance in a multi-IdP, multi-tenant environment.

### 3️⃣ Workload Identities Premium Add-on — Evaluate per Service Principal *(As Needed)*
Required for any service principal carrying privileged access to Fabric data or external systems, particularly where automated pipelines handle sensitive data.

---

> 📎 *All feature details are based on Microsoft's official Entra ID documentation and current licensing terms as of May 2026.*
