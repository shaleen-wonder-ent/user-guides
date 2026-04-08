# Microsoft Entra Workload ID — Complete Guide

> **Purpose:** This document defines the scope, applicability, and licensing model of Microsoft Entra Workload ID to help propose an accurate license count for Contoso's procurement.
>


---

## Table of Contents

1. [What is Entra Workload ID?](#1-what-is-entra-workload-id)
2. [How Does It Work?](#2-how-does-it-work)
3. [Benefits of Entra Workload ID Premium](#3-benefits-of-entra-workload-id-premium)
4. [How to Onboard Entra Workload ID at Your Organization](#4-how-to-onboard-entra-workload-id-at-your-organization)
5. [How to Calculate License Count](#5-how-to-calculate-license-count)

---

## 1. What is Entra Workload ID?

### The Problem It Solves

In every organization, it's not just **people** who access data and resources — **applications, automation scripts, CI/CD pipelines, APIs, microservices, and containers** do too. These are called **"workloads."**

Each workload needs an identity to authenticate and access resources (databases, APIs, secret vaults, etc.). Traditionally, this was done by:

- Creating an **App Registration** in Azure AD (now Entra ID) → which generates a **Service Principal**
- Generating a **Client Secret** (essentially a password) or certificate
- Storing that secret in config files, environment variables, or key vaults

#### Risks of the Traditional Approach

| Risk | Impact |
|------|--------|
| **Secret sprawl** | Secrets copied across config files, CI/CD pipelines, and environment variables |
| **Credential leaks** | A leaked secret gives an attacker full access to impersonate the app from anywhere |
| **No expiry discipline** | Secrets are often long-lived, rarely rotated, and forgotten |
| **No visibility** | No centralized view of how many service principals exist, what they access, or if they're still needed |
| **No conditional access** | Unlike users, workloads traditionally had no risk-based or location-based access policies |

> **Bottom line:** Organizations protect *human* identities with MFA, Conditional Access, and risk detection — but *non-human* identities (which outnumber humans 10:1 in most enterprises) have been a blind spot.

### The Solution

**Microsoft Entra Workload ID** is Microsoft's solution to **secure, manage, and govern non-human identities** — the identities used by apps, services, and automation rather than people.

> Think of it as: *"Everything you already do for user identity (Entra ID P1/P2) — but now for applications and services."*

### What Counts as a "Workload Identity"?

| Workload Type | Examples |
|--------------|---------|
| **Service Principals** | App registrations used for authentication |
| **Enterprise Applications** | Custom LOB apps, third-party SaaS, non-gallery apps |
| **Managed Identities** | System-assigned or user-assigned identities for Azure resources |
| **DevOps Pipelines** | GitHub Actions, Azure DevOps service connections |
| **Automation Scripts** | PowerShell/Python scripts running scheduled jobs |
| **Multi-cloud Workloads** | Apps on AWS/GCP that access Azure resources |
| **Third-party SaaS Integrations** | ServiceNow, Salesforce connectors authenticating to your tenant |

---

## 2. How Does It Work?

### Core Concept: Eliminate Secrets, Use Federation

Entra Workload ID replaces static client secrets with **Workload Identity Federation** — a trust relationship where the compute platform (Kubernetes, GitHub, AWS, GCP) provides a short-lived OIDC token that Entra validates directly. **No secret is stored or transmitted.**

### Authentication Flow Comparison

#### ❌ Old Way (Client Secret)

```
1. Developer registers an app in Entra → gets a service principal
2. Generates a client secret (password) → stores it somewhere
3. App reads the secret at runtime → sends it to Entra to authenticate
4. Secret sits there for months/years → nobody rotates it
5. Attacker steals the secret → uses it from ANYWHERE in the world
```

#### ✅ New Way (Workload Identity Federation)

```
1. App registration exists with a Federated Credential (trust to a platform)
2. At runtime, the platform (K8s, GitHub, etc.) gives the app a short-lived OIDC token
3. App sends the OIDC token to Entra → Entra validates it against the trust config
4. Token expires in minutes → no secret ever exists
5. Attacker compromises the app → gets a temporary token that's already expired
```

### What Changes in the App Registration?

| Component | Before Migration | After Migration |
|---|---|---|
| **Client ID** | ✅ Exists | ✅ Still exists (public identifier — not sensitive) |
| **Client Secret** | ✅ Exists (**this is the risk**) | ❌ **DELETED. Gone. Doesn't exist.** |
| **Federated Credential** | ❌ | ✅ Trust relationship configured |

> **The client ID alone is harmless** — it's like knowing someone's email address. The dangerous part was always the **client secret**, and that gets removed entirely.

### Code Change — Before and After

#### ❌ BEFORE (Client Secret):

```python
import requests

tenant_id = "<TENANT_ID>"
client_id = "<CLIENT_ID>"
client_secret = "xK8Q~super-secret-value-stored-somewhere-risky"  # ← THE PROBLEM
scope = "https://graph.microsoft.com/.default"

token_url = f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"

data = {
    "grant_type": "client_credentials",
    "client_id": client_id,
    "client_secret": client_secret,         # ← Static secret, lives for months/years
    "scope": scope
}

response = requests.post(token_url, data=data)
access_token = response.json()["access_token"]
```

#### ✅ AFTER (Workload Identity Federation):

```python
import requests

tenant_id = "<TENANT_ID>"
client_id = "<CLIENT_ID>"                    # ← Same app registration, same client_id
oidc_token = get_oidc_token_from_platform()  # ← Platform provides this automatically
scope = "https://graph.microsoft.com/.default"

token_url = f"https://login.microsoftonline.com/{tenant_id}/oauth2/v2.0/token"

data = {
    "grant_type": "client_credentials",
    "client_id": client_id,
    "scope": scope,
    "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
    "client_assertion": oidc_token            # ← Short-lived token, no secret stored
}

response = requests.post(token_url, data=data)
access_token = response.json()["access_token"]
```

#### For Azure-Hosted Workloads (Even Simpler — Managed Identity):

```python
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()  # ← That's it. No secrets. Azure handles everything.
```

### Why a Stolen Token Is Useless

| Attack Vector | Client Secret | OIDC Federated Token |
|---|---|---|
| Stolen from config/pipeline | Works forever, from anywhere | Expires in **minutes** |
| Used from attacker's machine | ✅ Works — no location binding | ❌ Fails — bound to specific platform |
| Replayed after capture | ✅ Works — doesn't expire for months | ❌ Fails — already expired |
| Generated without the platform | ✅ Works — it's just a string | ❌ Fails — can't generate without running on trusted platform |

---

## 3. Benefits of Entra Workload ID Premium

### Edition Comparison

| Capability | Free | Premium ($3/identity/month) |
|------------|:----:|:---------------------------:|
| Managed identities & federation | ✅ | ✅ |
| Basic lifecycle management | ✅ | ✅ |
| Audit & sign-in logs | ✅ | ✅ |
| **Conditional Access for workloads** | ❌ | ✅ |
| **Identity Protection & risk detection** | ❌ | ✅ |
| **Access reviews for workload identities** | ❌ | ✅ |
| **Health recommendations** | ❌ | ✅ |

### 3.1 Conditional Access for Workloads

Conditional Access evaluates **every authentication attempt in real-time** and can block it based on rules.

```
Without Conditional Access:
  App authenticates → ✅ Access granted (every time, no questions asked)

With Conditional Access:
  App authenticates → 🛂 CHECKPOINT → evaluates conditions → ✅ Allow or ❌ Block
```

#### Available Conditions

| Condition | What It Evaluates | Example Policy |
|---|---|---|
| **Location / IP range** | Where is this authentication coming from? | "Only allow from our datacenter IPs" |
| **Sign-in risk** | Has Microsoft's AI detected something suspicious? | "Block if risk is medium or high" |
| **Target resource** | Which app/API is being accessed? | "Stricter rules for Finance API" |

#### Example: Protecting a Payroll App

**Scenario:** A Payroll App (service principal) runs on AKS in East US and accesses the HR Database.

**Policy Configuration:**

```
Policy Name:    "Block Payroll App Outside Trusted Network"
Applies to:     Workload identity → "Payroll-Processing-App"
Target:         Microsoft Graph API
Condition:      Location → Include: Any location
                           Exclude: "East US Datacenter" (trusted IPs)
Action:         Block access
```

**What happens:**
- App authenticates from East US datacenter → ✅ Allowed
- Attacker tries to use a token from a foreign IP → ❌ **Blocked**

#### Example: Block Risky Sign-Ins Automatically

```
Policy Name:    "Block High-Risk Workload Sign-Ins"
Applies to:     All workload identities
Condition:      Sign-in risk = Medium or High
Action:         Block access
```

**What it catches:**
- Authentication from IPs associated with known botnets → **Blocked**
- Impossible travel (East US 2 minutes ago, now Southeast Asia) → **Blocked**
- Token replay patterns → **Blocked**

### 3.2 Identity Protection & Risk Detection

Microsoft's AI **continuously monitors** workload identity behavior and flags anomalies:

- Unusual authentication patterns
- Access from suspicious IP addresses
- Impossible travel scenarios
- Leaked credential detection
- Privilege escalation attempts

### 3.3 Access Reviews

Periodic, auditable review of what each workload identity can access:

- Reviewers certify or revoke access on a schedule
- Prevents permission creep over time
- Provides compliance evidence for auditors
- Automatically removes access for unreviewed identities

### 3.4 Health Recommendations

Proactive recommendations to improve your workload identity posture:

- Identify **inactive** service principals (haven't authenticated in 90+ days)
- Flag **over-permissioned** identities (more access than needed)
- Detect **stale credentials** that should be removed
- Recommend least-privilege adjustments

### Defense-in-Depth Summary

```
┌──────────────────────────────────────────────────────────────┐
│  LAYER 1 (Free): Federated Credentials — No Secrets          │
│  → There's no password to steal                              │
├──────────────────────────────────────────────────────────────┤
│  LAYER 2 (Free): Platform-Bound, Short-Lived Tokens          │
│  → Even if intercepted, token can't be reused                │
├──────────────────────────────────────────────────────────────┤
│  LAYER 3 (Premium): Conditional Access Policies              │
│  → Even valid authentication gets evaluated & can be blocked │
├──────────────────────────────────────────────────────────────┤
│  LAYER 4 (Premium): Identity Protection + Risk Detection     │
│  → AI continuously monitors workload behavior                │
├──────────────────────────────────────────────────────────────┤
│  LAYER 5 (Premium): Access Reviews                           │
│  → Periodic human review prevents permission creep           │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. How to Onboard Entra Workload ID at Your Organization

### Phase 1: Discover (No Code Changes, No Cost)

1. **Inventory all app registrations & service principals** in the tenant
   - Entra Admin Center → Applications → App registrations
   - Entra Admin Center → Applications → Enterprise applications
2. **Identify authentication methods** for each:
   - Client secrets? Certificates? Managed identities? Federated credentials?
3. **Flag stale/unused apps** for cleanup
4. **Categorize by criticality:**
   - High (accesses financial data, PII, production resources)
   - Medium (internal tools, monitoring)
   - Low (dev/test environments)

### Phase 2: Upgrade Authentication (Code Changes, Free Tier)

**For Azure-hosted workloads:**
- Switch to **Managed Identity** (system-assigned or user-assigned)
- Code change: Replace `ClientSecretCredential` with `DefaultAzureCredential`

**For external platforms (GitHub Actions, Kubernetes, AWS, GCP):**
1. Go to App Registration → Certificates & secrets → **Federated credentials** tab
2. Click **+ Add credential**
3. Configure the trust:
   - Select the identity provider (GitHub, Kubernetes, AWS, GCP, etc.)
   - Specify the issuer URL and subject identifier
4. Update application code to use OIDC token exchange instead of client secret
5. **Delete the client secret** from the app registration

**For CI/CD pipelines (GitHub Actions example):**
- Configure OIDC federation in the app registration
- Update workflow YAML to use `azure/login` action with federated credentials
- Remove stored secrets from GitHub repository settings

### Phase 3: Apply Premium Protections ($3/identity/month)

1. **Purchase Workload ID Premium licenses** for identities you want to protect
2. **Create Conditional Access policies:**
   - Entra Admin Center → Protection → Conditional Access → + New policy
   - Select "Workload identities" under assignments
   - Configure location, risk, and resource conditions
   - Set grant/block controls
3. **Enable Identity Protection:**
   - Entra Admin Center → Protection → Identity Protection
   - Review workload identity risk reports
   - Configure automated responses to risk detections
4. **Set up Access Reviews:**
   - Entra Admin Center → Identity Governance → Access reviews
   - Create reviews for workload identity role assignments
   - Assign reviewers and schedule cadence

### Phase 4: Ongoing Governance

- Monitor workload identity risk dashboard regularly
- Review and recertify access on schedule
- Decommission workload identities when apps are retired
- Continuously reduce permissions based on health recommendations

---

## 5. How to Calculate License Count

### Licensing Model

| Detail | Value |
|--------|-------|
| **SKU** | Microsoft Entra Workload ID Premium |
| **Cost** | $3 per workload identity per month |
| **Billing** | Standalone add-on — **NOT included in Microsoft 365 E3/E5** |
| **Scope** | Per workload identity under Premium policy management |

### What Requires a Premium License?

| Identity Type | When License Is Needed |
|---|---|
| **Service Principals / Enterprise Apps** | When you apply Conditional Access or Identity Protection to them |
| **Managed Identities** | When you perform Access Reviews on them |
| **Microsoft First-Party Apps** (Outlook, Teams, etc.) | ❌ **No license required** — even with Conditional Access |

### What Does NOT Require a License?

- Workload identities using **only Free tier features** (federation, managed identity, basic logs)
- Microsoft first-party application service principals
- Identities you choose **not** to place under Premium policies

### License Counting Formula

```
Required Licenses = Number of workload identities placed under Premium policies
                    (Conditional Access + Identity Protection + Access Reviews)
                  - Microsoft first-party app service principals (excluded)
```

### Step-by-Step License Calculation

#### Step 1: Get Total Workload Identity Count

```
Entra Admin Center → Applications → Enterprise applications
  → Filter by Application type
  → Count: Service Principals + Managed Identities
```

#### Step 2: Subtract Exclusions

| Subtract | Reason |
|----------|--------|
| Microsoft first-party apps | Not licensable |
| Dev/test identities (if not protecting) | Only license what you protect |
| Inactive/stale identities (clean up first) | Delete before counting |

#### Step 3: Determine Scope of Premium Protection

Decide which identities will be placed under Premium policies:

| Tier | Identities | Premium? |
|------|-----------|----------|
| **Critical** | Payroll, Finance, HR apps, production APIs | ✅ Yes — license these |
| **Important** | Internal tools, monitoring, integration connectors | ✅ Recommended |
| **Low Risk** | Dev/test apps, sandbox environments | ❌ Optional |

#### Step 4: Calculate Final Count

**Example Calculation:**

```
Total service principals in tenant:                    500
- Microsoft first-party apps:                         -150
- Stale/unused (to be cleaned up):                     -80
= Active non-Microsoft workload identities:            270

Decision: Protect Critical + Important tiers:
  - Critical (production, finance, HR):                120
  - Important (internal tools, integrations):           90
  - Low Risk (dev/test — skip for now):                 60

═══════════════════════════════════════════════════
  LICENSES NEEDED:  210 × $3/month = $630/month
═══════════════════════════════════════════════════
```

### Summary Table for Contoso License Proposal

| Line Item | Count | Unit Cost | Monthly Cost |
|-----------|-------|-----------|-------------|
| Workload ID Premium — Critical tier | _[count]_ | $3/identity | _[total]_ |
| Workload ID Premium — Important tier | _[count]_ | $3/identity | _[total]_ |
| Workload ID Premium — Low Risk (optional) | _[count]_ | $3/identity | _[total]_ |
| **Total** | **_[sum]_** | **$3/identity** | **_[grand total]_** |

> **Note:** Licenses are not assigned individually to each identity. Purchase enough licenses to cover the total count of identities under Premium policy management in the tenant.

---

## References

- [Microsoft Entra Workload ID — Product Page](https://www.microsoft.com/en-us/security/business/identity-access/microsoft-entra-workload-id)
- [Workload ID FAQ — Microsoft Learn](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-faqs)
- [Conditional Access for Workload Identities — Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/conditional-access/workload-identity)
- [Microsoft Entra Licensing — Microsoft Learn](https://learn.microsoft.com/en-us/entra/fundamentals/licensing)
- [Demystifying Microsoft Entra Workload ID Licensing](https://secureazcloud.com/microsoft-security/f/demystifying-microsoft-entra-workload-id-licensing)
