# Keycloak ↔ Microsoft Entra ID Federation for Power BI Access  
_Comprehensive Architecture Explanation & Step‑by‑Step Implementation Guide_

**Last Updated:** 2025-10-13  
**Audience:** Identity / Security Architects, IAM Engineers, Power BI / M365 Platform Owners  
**Goal:** Enable users whose primary identities live in Keycloak to access Microsoft Power BI (and broader Microsoft 365) through Microsoft Entra ID (formerly Azure AD), using either:
- External Identities (Guest) federation (OIDC or SAML)  
- Domain (home realm) federation (SAML / WS‑Fed) for member accounts  

---

## 1. Core Concepts

| Term | Meaning |
|------|---------|
| Microsoft Entra ID | Identity control plane for Microsoft 365 (including Power BI). Only Entra issues tokens accepted by Power BI. |
| Federation | Trust where Entra defers primary authentication to another Identity Provider (IdP) (here: Keycloak). |
| External Identities (B2B) | Mechanism to invite or dynamically create guest (`userType=Guest`) users. |
| Domain (Home Realm) Federation | Tenant-level federation enabling users with a verified domain (e.g. `@company.com`) to authenticate via SAML/WS-Fed at login.microsoftonline.com. |
| Conditional Access (CA) | Entra policy engine to enforce MFA, device, risk, session controls. |
| Guest vs Member | Guests are collaborative/external; Members are first-party workforce identities. Licensing & policy targeting differs. |
| OIDC vs SAML | Both open standards; Entra supports OIDC for External IdPs (guests) but not (as of 2025) for tenant/domain federation (members). |

---

## 2. Architecture Options

### Option 1: External IdP Federation (Recommended Quick Win)
- Entra trusts Keycloak via **OIDC (preferred)** or SAML.
- Users become / use **Guest** accounts (`userType=Guest`).
- Power BI access works after license assignment or Premium capacity sharing.
- Pros: Fast, minimal impact, no domain-level changes.  
- Cons: Guests vs Members licensing & some governance nuances.

### Option 2: Domain (Home Realm) Federation
- Use **SAML or WS-Fed** only (OIDC not supported for this layer).
- Users authenticate at Entra, are redirected transparently to Keycloak.
- Accounts are **Member** (`userType=Member`) with your verified domain (e.g. `alice@company.com`).
- Pros: Native experience, simpler license assignment & CA targeting.  
- Cons: Higher operational risk (IdP outage impacts workforce), more setup complexity.

### Option 3: Provision & Sync Users (SCIM / Scripts) + Local Entra Auth
- Replicate identities into Entra; optionally abandon upstream federation.
- Pros: Full Entra feature parity.  
- Cons: Identity & password (or passwordless credential) lifecycle complexity.

**Decision Heuristic:**
- Need speed + acceptable Guest model → Option 1 (OIDC).
- Need workforce-native status, dynamic groups, device CA, domain-based branding → Option 2 (SAML domain).
- Need advanced conditional / device / risk with no upstream dependency → Option 3.

---

## 3. Decision Matrix

| Requirement | Option 1 (OIDC Guest) | Option 2 (SAML Domain) | Option 3 (Provision) |
|-------------|----------------------|------------------------|----------------------|
| Time-to-Implement | Fast | Medium | Slow–Medium |
| User Type | Guest | Member | Member |
| OIDC Use | Yes | No (not for domain) | Internal (no federation) |
| MFA Centralization | Keycloak OR Entra | Usually Entra | Entra |
| Dynamic Groups by Domain | Limited (guest) | Full | Full |
| Device Compliance (Intune) | Hard (guests) | Feasible | Feasible |
| License Assignment Automation | Script / Group (guests) | Standard (groups) | Standard (groups) |
| Migration Complexity (later) | Higher (guest→member) | N/A | N/A |
| Downtime Blast Radius if Keycloak Fails | Affects guest sign-ins only | Affects workforce sign-ins | None (except local Entra) |

---

## 4. End-to-End Flow Summaries

### 4.1 OIDC Guest Federation Flow
1. User tries to access Power BI URL.
2. Entra login page → user chooses custom IdP (Keycloak) or direct link with `idp` parameter.
3. Browser is redirected to Keycloak (OIDC Authorization Code flow).
4. User authenticates (and possibly MFA) in Keycloak.
5. Keycloak returns authorization code to Entra; Entra redeems it, maps claims → guest user object.
6. Entra issues tokens (ID, Access) for Power BI → Power BI authorizes.

### 4.2 SAML Domain Federation Flow
1. User enters `alice@company.com` at Microsoft login.
2. Home Realm Discovery detects federated domain; redirects to Keycloak SAML endpoint.
3. Keycloak authenticates + returns SAML Assertion (signed).
4. Entra validates assertion → issues tokens as Member identity.
5. Power BI authorizes.

---

## 5. Detailed Implementation: Option 1 (OIDC External IdP, Guest Users)

### 5.1 Prerequisites
- Publicly reachable Keycloak realm (`https://id.example.com/realms/corp-users`).
- Entra Admin (Global Admin or External Identities Admin).
- Power BI licensing strategy (Pro, PPU, or Premium capacity).
- HTTPS & stable Keycloak signing keys (recommend rotation policy).

### 5.2 Keycloak Configuration (OIDC Client)
1. In Keycloak Admin Console → Realm: `corp-users`.
2. Clients → Create:
   - Client Type: OpenID Connect
   - Client ID: `entra-ext`
   - Access Type: `confidential`
   - Standard Flow: ON (Authorization Code)
   - Valid Redirect URIs (placeholder for now): `https://login.microsoftonline.com/*`
3. Save → Credentials tab → copy Client Secret.
4. Realm Settings → OpenID Endpoint Configuration: Note:
   - Issuer
   - Authorization Endpoint
   - Token Endpoint
   - JWKS URI
   - UserInfo Endpoint
5. Ensure User Attributes / Mappers:
   - `preferred_username`, `email`, `given_name`, `family_name`
   - Email should be verified if possible (improves user provisioning clarity).
6. Add final precise Redirect URI once known (see below).

### 5.3 Entra: Add External OIDC Identity Provider
1. Portal: Microsoft Entra ID → External Identities → Identity providers → Add OIDC.
2. Fields:
   - Name: `Keycloak`
   - Client ID: `entra-ext`
   - Client Secret: (from Keycloak)
   - Issuer URL: `https://id.example.com/realms/corp-users`
3. Entra discovers endpoints automatically (if not, input manually).
4. Copy the generated Redirect URI → paste back into Keycloak client’s Valid Redirect URIs (exact match).
5. Save.

### 5.4 Claims & Mapping (Defaults Usually OK)
- `sub` → stable unique identifier (Keycloak `sub`).
- `email` claim present.
- `name` or `given_name`/`family_name` for display.
- Optional: Add custom claim mappers in Keycloak if you need group hints (Entra may not automatically process groups; could use downstream logic).

### 5.5 Initial Testing
1. Construct test URL (use any Microsoft 1P app):
   ```
   https://myapps.microsoft.com/{tenantId}?tenantId={tenantId}&idp=Keycloak
   ```
2. Sign in → complete Keycloak login → verify guest user appears:
   - Entra → Users → filter by Recent Users.
   - Check user principal name pattern `user_guid#EXT#@tenant.onmicrosoft.com`.
3. Confirm email populated.

### 5.6 Licensing for Power BI
- If using Premium capacity: Guests may need only free license (depending on features).
- If using Pro / PPU:
  1. Create security group `Keycloak-Guests-Licensed`.
  2. Assign Power BI Pro or PPU license to group.
  3. Automation (Graph / Logic App) adds new guest user to group upon creation.

### 5.7 Sharing Power BI Content
1. Publish workspace; if Premium, assign capacity.
2. Add guest user (or group) as Viewer/Contributor.
3. Provide direct app/report link or let user access via My Apps with `idp=Keycloak` link for smoother IdP selection.

### 5.8 Conditional Access (CA) Strategy
Decide MFA authority:
- **Option A:** Keycloak performs MFA → Exclude group `Keycloak-Guests-MFAUpstream` from Entra “Require MFA” CA policy.
- **Option B:** Entra performs MFA → Create CA policy: Users = All guests, App = Power BI, Grant = Require MFA.

Avoid double MFA by explicit CA exclusions or by using federated IdP MFA trust (if feature & claim mapping available).

### 5.9 Automation (Recommended)
- Webhook / Event: Keycloak Admin Event (user created) triggers workflow (serverless function) to invite or note potential user. (Often first login auto-creates guest; you then license.)
- Graph Script:
  - Monitor for new guests (filter by `createdDateTime > lastRun`).
  - Add to licensing group; assign roles; send welcome email.

### 5.10 Operational & Security Checklist
| Control | Action |
|---------|--------|
| Key Rotation | Monitor Keycloak realm keys; if using static JWKS published, Entra auto-fetches. |
| Logging | Export Entra Sign-in logs to Log Analytics; correlate with Keycloak audit events. |
| Least Privilege | Workspace roles minimal (Viewer). Avoid broad `Build` permissions unless needed. |
| Revocation | Disable user in Keycloak or remove groups; periodically prune stale guests (no sign-in 90d). |
| Threat Detection | If relying on Entra risk policies, ensure Entra still evaluates risk even with external IdP (it does post-token issuance). |

---

## 6. Detailed Implementation: Option 2 (SAML Domain Federation, Member Users)

### 6.1 Prerequisites
- Verified custom domain in Entra (DNS TXT record).
- Control over DNS to set up or revert federation quickly.
- High availability Keycloak (consider multi-node cluster + health monitoring).
- SAML signing certificate management (expiry calendar).

### 6.2 Keycloak Configuration (SAML Client)
1. Clients → Create:
   - Client ID: `urn:federation:entra` (arbitrary but stable)
   - Client Protocol: `saml`
   - NameID Format: `persistent` or `email` (must match plan for ImmutableID/UPN)
   - Assertion Consumer Service URL (ACS): `https://login.microsoftonline.com/te/{tenantId}/saml2`
   - Sign Assertions: ON
   - Sign Documents: ON
   - Force POST Binding for SSO: ON
2. Mappers:
   - `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` → `${email}`
   - `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` → `${preferred_username}`
   - Optionally `ImmutableID` (if using a stable attribute like internal GUID).
3. Obtain signing certificate (export X.509 without private key).

### 6.3 Entra Domain Federation (PowerShell / Graph)
Older MSOnline (still illustrative):
```powershell
Connect-MsolService
$cert = Get-Content .\keycloak-signing-cert.cer | Out-String
Set-MsolDomainAuthentication `
  -DomainName "company.com" `
  -FederationBrandName "Company" `
  -Authentication Federated `
  -PassiveLogOnUri "https://id.company.com/realms/corp-users/protocol/saml" `
  -ActiveLogOnUri "https://id.company.com/realms/corp-users/protocol/saml" `
  -IssuerUri "https://id.company.com/realms/corp-users" `
  -SigningCertificate $cert `
  -LogOffUri "https://id.company.com/realms/corp-users/protocol/saml" `
  -PreferredAuthenticationProtocol SAMLP
```

Modern Graph (conceptual):
- Use `domainFederation` endpoint: `PATCH /domains/{id}/federationConfiguration/{federationId}` with JSON containing `issuerUri`, `signingCertificate`, etc.

### 6.4 Test
1. Browse to `https://portal.office.com` or Power BI sign-in.
2. Enter `alice@company.com` → expect redirect to Keycloak.
3. Authenticate → verify Member account sign-in logs.

### 6.5 MFA & CA
- Usually centralize in Entra CA (Require MFA for all users).
- Optionally configure Keycloak MFA but avoid user confusion; rely on one layer.

### 6.6 Risks & Mitigations
| Risk | Mitigation |
|------|------------|
| IdP outage blocks workforce | Break-glass cloud-only accounts (at least 2), document rollback to Managed domain. |
| Cert expiration | 60/30/15 day monitoring alerts. |
| Token replay / assertion misuse | Ensure strict audience & recipient, short assertion validity (default). |
| Incorrect NameID mapping | Test pilot users before bulk rollout. |

### 6.7 Rolling Back (Emergency)
- Switch domain to `Managed` via PowerShell:  
  `Set-MsolDomainAuthentication -DomainName company.com -Authentication Managed`
- Users then authenticate with Entra-managed credentials (have break-glass accounts ready).

---

## 7. Conditional Access (CA) Deep Dive

### 7.1 Components
- Conditions: Users, Apps, Platform, Location, Device, Risk, Client App, Authentication Context, External user type.
- Controls: Grant (MFA, compliant device, password change), Block, Session (sign-in frequency, CAE, app control, persistence).

### 7.2 Guest vs Member Impact
| Feature | Guest | Member |
|---------|-------|--------|
| Device Compliance Enforcement | Limited (non-managed) | Common |
| Dynamic Groups by Domain | Partial | Full |
| Licensing Assignment Simplicity | Slightly more manual | Straightforward |
| MFA Consistency | Might rely on upstream | Centralized |

### 7.3 Avoiding Double MFA
Strategy if upstream (Keycloak) handles MFA:
1. Group: `Keycloak-Upstream-MFA`.
2. CA Policy “Require MFA”:
   - Include All Users.
   - Exclude `Keycloak-Upstream-MFA`.
3. Upstream MFA logging remains in Keycloak; Entra sees single-factor but allowed via exclusion.

### 7.4 Example Policies (Conceptual)
| Policy | Purpose | Key Settings |
|--------|---------|--------------|
| Block Legacy Auth | Security baseline | Users=All; Client Apps=Legacy protocols → Block |
| Require MFA for Power BI | Secure external access | Users=All (exclude upstream-MFA group); Apps=Power BI; Grant=Require MFA |
| Guest Session Hardening | Control persistence | Users=All guests; Session=Sign-in frequency 12h; Persistent OFF |
| Risk Mitigation | Identity Protection | Users=All; Sign-in Risk=Medium/High; Grant=Require MFA / Block High |

---

## 8. MFA Ownership Models

| Model | Description | Pros | Cons |
|-------|-------------|------|------|
| Keycloak Only | MFA at Keycloak login | Single UX; IdP-central | Entra risk-based step-up not leveraged |
| Entra Only | Keycloak password → Entra MFA | Unified reporting; adaptive risk | Extra redirect step (if domain federation) |
| Federated MFA Trust | Keycloak asserts MFA (via `acr`/`amr`) recognized by Entra | Seamless + Entra CA satisfaction | Requires support & correct claims mapping (may be limited) |

Sample `amr` claim from Keycloak custom protocol mapper:
```json
"amr": ["pwd","otp"]
```
Entra can potentially treat presence of `otp` + `pwd` as fulfilling “Require MFA” (validate with current Microsoft documentation—feature availability evolves).

---

## 9. Licensing Strategy for Power BI

| Scenario | Approach |
|----------|----------|
| Small number of external users | Direct manual license assignment or group membership |
| Medium/Large, Guest model | Automation: Azure Function / Logic App + Graph API to add new guest to license group |
| Premium capacity | Minimize Pro licenses; still license authors with Pro |

Graph API (pseudocode):
```http
POST https://graph.microsoft.com/v1.0/groups/{groupId}/members/$ref
{
  "@odata.id":"https://graph.microsoft.com/v1.0/directoryObjects/{guestObjectId}"
}
```

---

## 10. Troubleshooting Guide

| Symptom | Likely Cause | Resolution |
|---------|-------------|-----------|
| `AADSTS700016: application with identifier not found` | Wrong Client ID or not configured | Re-check IdP config in Entra |
| User loops between Keycloak and Entra | Missing code exchange / invalid redirect | Validate Redirect URI matches exactly |
| Guest created without email | Keycloak not sending `email` claim | Add email mapper & ensure attribute present |
| Double MFA prompts | Both Keycloak & Entra enforcing MFA | Exclude group in CA or disable one side |
| Access denied in Power BI | Missing license or workspace role | Assign license / add to workspace |
| SAML federation fails after cert rotation | Old cert still configured | Update domain federation signingCertificate |
| Very long redirect times | Network latency / DNS / misconfigured load balancer | Verify TLS termination & upstream health |

---

## 11. Security Hardening Checklist

| Area | Action |
|------|--------|
| TLS | Enforce modern ciphers; monitor cert expiration |
| Keycloak Updates | Patch schedule (CVE review) |
| Realm Keys | Rotate; minimal unused keys; monitor JWKS |
| Rate Limiting | Protect Keycloak login endpoints (WAF) |
| Monitoring | SIEM integration (Entra sign-in + Keycloak audit) |
| Admin Access | RBAC principle of least privilege, just-in-time elevation |
| Break-Glass Accounts | At least 2 cloud-only Global Admins w/ robust MFA |
| Conditional Access Baselines | Block legacy auth, enforce MFA / risk policies |
| Guest Lifecycle | Quarterly cleanup for stale accounts |
| Data Exfil Controls | Power BI: Sensitivity labels, DLP, export restrictions via tenant settings |

---

## 12. Sample Configurations & Snippets

### 12.1 Keycloak OIDC Client (Excerpt JSON via Admin REST)
```json
{
  "clientId": "entra-ext",
  "protocol": "openid-connect",
  "publicClient": false,
  "redirectUris": [
    "https://login.microsoftonline.com/te/<tenantId>/oauth2/authresp"
  ],
  "standardFlowEnabled": true,
  "directAccessGrantsEnabled": false,
  "attributes": {
    "pkce.code.challenge.method": "S256"
  }
}
```

### 12.2 Keycloak SAML Client (Important Settings)
```xml
<EntityDescriptor entityID="https://id.company.com/realms/corp-users">
  <!-- Standard realm metadata; ensure signing key valid -->
</EntityDescriptor>
```
Ensure Keycloak admin UI → Client Settings:
- Force NameID Format = email
- Signature Algorithm = RSA-SHA256
- Assertion Signature = ON

### 12.3 Forcing IdP Parameter (Guest OIDC)
```
https://login.microsoftonline.com/{tenantId}/oauth2/v2.0/authorize
 ?client_id={some_ms_app_id}
 &response_type=code
 &redirect_uri={encodedRedirectUri}
 &scope=openid%20profile%20email
 &idp=Keycloak
```

### 12.4 PowerShell (Legacy) to View Domain Auth
```powershell
Get-MsolDomainFederationSettings -DomainName company.com
```

### 12.5 Break-Glass Guidance Banner (Internal SOP)
> Maintain 2+ cloud-only Global Admin accounts with strong phishing-resistant MFA (FIDO2). Exclude from risky CA experiments. Store recovery credentials in a sealed procedure-controlled vault.

---

## 13. Migration & Future Considerations
| Future Change | Impact | Preparation |
|---------------|--------|------------|
| Switching Guest → Member | Re-permissioning & license changes | Pilot a small batch; map existing permissions |
| OIDC support for domain federation (if added) | Could simplify config | Track Microsoft roadmap |
| SCIM Provisioning from Keycloak | Automate entity lifecycle | Evaluate Keycloak SCIM extensions |
| Passkeys / WebAuthn-first strategy | MFA simplification | Ensure Keycloak + Entra both support FIDO flows |
| Attribute Enrichment | Add department, costCenter claims | Extend Keycloak mappers; use Graph to patch guest attributes |

---

## 14. Frequently Asked Questions (FAQ)

**Q:** Can Power BI directly trust Keycloak?  
**A:** No. Power BI only trusts tokens issued by Entra. Keycloak is upstream for authentication; Entra remains STS for authorization.

**Q:** Is OIDC allowed for workforce domain federation?  
**A:** Not as of 2025; only SAML / WS-Fed for domain-level (home realm) federation.

**Q:** Do guests need Power BI Pro?  
**A:** Only if accessing content not backed by Premium capacity or using Pro-required features. Premium capacity can reduce Pro license needs for viewers.

**Q:** How to avoid double MFA?  
**A:** Decide one MFA authority; configure CA exclusions or trusted MFA signals.

**Q:** Can we map Keycloak groups to Entra groups automatically?  
**A:** Not natively. You’d need a middleware process reading a Keycloak claim and updating Entra via Graph.

---

## 15. Recommended Minimal Path (Fastest Working Pilot)

1. Create OIDC client in Keycloak (`entra-ext`) with necessary claims.  
2. Add OIDC External IdP in Entra (name: Keycloak).  
3. Add Redirect URI to Keycloak client.  
4. Test sign-in (`myapps` URL with `idp=Keycloak`).  
5. Assign Power BI license (direct or group) or use Premium workspace.  
6. Share a sample report with pilot user.  
7. Create baseline CA policy: Block legacy auth, Require MFA (decide upstream vs Entra).  
8. Document support & rollback steps.  
9. Expand to broader population after validating logs & UX.

---

## 16. Appendix: Graph API Licensing Example (Pseudo-PowerShell)

```powershell
$tenantId = "<tenantId>"
$appId = "<appId>"
$clientSecret = "<secret>"
$licenseSkuId = "<powerBiProSkuGuid>"
$groupId = "<licenseGroupId>"

# Acquire token (simplified)
$token = (Invoke-RestMethod -Method Post -Uri "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/token" -Body @{
  client_id = $appId
  client_secret = $clientSecret
  scope = "https://graph.microsoft.com/.default"
  grant_type = "client_credentials"
}).access_token

# Add new guest to group for license assignment
Invoke-RestMethod -Method Post -Uri "https://graph.microsoft.com/v1.0/groups/$groupId/members/\$ref" -Headers @{Authorization="Bearer $token"} -Body (@{
  "@odata.id"="https://graph.microsoft.com/v1.0/directoryObjects/$guestObjectId"
} | ConvertTo-Json)
```

---

## 17. Summary

| If You Need | Choose |
|-------------|--------|
| Fast enablement & minimal changes | OIDC External IdP (Guests) |
| Native workforce identity & domain branding | SAML Domain Federation |
| Deep Entra device/risk governance with minimal external dependency | Provision + (optionally) Domain Federation |

Key principle: Power BI always consumes Entra-issued tokens; your federation selection only alters how Entra authenticates users, not how Power BI performs authorization.

---

## 18. Next Steps (Action Checklist)

| Step | Owner | Done? |
|------|-------|-------|
| Confirm federation model (Guest OIDC vs Domain SAML) | IAM Architect |  |
| Collect Keycloak endpoints & configure client | Keycloak Admin |  |
| Configure Entra IdP or Domain Federation | Entra Admin |  |
| Pilot user sign-in test | Test Engineer |  |
| License assignment automation | Platform Engineer |  |
| Configure Conditional Access baseline | Security Engineer |  |
| Logging & monitoring integration | SecOps |  |
| Draft end-user comms (login instructions) | Comms |  |
| Production rollout & validation | Project Lead |  |
| Stale guest lifecycle script | IAM Ops |  |

---

**Need more?** Request add-ons: SCIM provisioning design, advanced CA examples (JSON), or a rollback runbook.

_End of Document_
