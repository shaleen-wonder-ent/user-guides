# High-Level Technical Implementation Guide
Contoso360 + Keycloak + Azure Service Principal + Power BI/Fabric

This document provides a high-level, non-code implementation plan for enabling external customer authentication via **Keycloak** and delivering multi-tenant analytics through **Contoso360** using an **Azure Service Principal** to securely access **Power BI/Fabric**.

---

## 1. Establish Identity Model

### 1.1 Define Authentication Boundaries
- **Keycloak** handles all external user authentication.
- **Contoso360** handles authorization, RBAC, tenant mapping, and session control.
- **Azure Entra ID** stores and manages the **Service Principal**, not customer identities.
- **Power BI/Fabric** is accessed only via the Service Principal.

### 1.2 Confirm Identity Separation
- External users do *not* exist in Azure Entra ID.
- External users never sign into Power BI or Fabric.
- No SAML federation between Keycloak and Entra.
- No creation of guest users.

---

## 2. Configure Keycloak for Customer Authentication

### 2.1 Create Contoso360 Client in Keycloak
- Add Contoso360 as an application (OIDC or SAML client).
- Set redirect URIs to Contoso360’s login endpoints.

### 2.2 Enable SSO and Direct Login
- Configure required SSO methods (SAML/OIDC).
- Allow fallback to username/password login.

### 2.3 Map Identity Attributes
Ensure Keycloak sends:
- User ID
- Email
- First/Last Name
- Tenant or Organization Identifier
- Role or profile attributes (optional)

Contoso360 uses these attributes for authorization.

---

## 3. Implement Authentication Handling in Contoso360

### 3.1 Accept Keycloak Tokens
- Integrate with Keycloak using OIDC/SAML middleware.
- Validate signatures, issuer, and expiry.

### 3.2 Establish User Session
- Create session after successful Keycloak authentication.
- Extract tenant and role information.

### 3.3 Apply Authorization Logic
- Match Keycloak user → Contoso360 tenant.
- Enforce RBAC at Contoso360 level.

---

## 4. Configure Azure Entra Service Principal

### 4.1 Create Service Principal
- Register a new application in Azure Entra ID.
- Name it "Contoso360 Fabric Access" or similar.

### 4.2 Generate Credentials
- Create Client Secret or Certificate.
- Store securely in Azure Key Vault.

### 4.3 Enable Power BI Service Principal Access
- In Power BI Admin Portal:
  - Enable **Allow service principals to use Power BI APIs**.
  - Add the SP to the relevant workspaces.

### 4.4 Assign Workspace Roles
- Viewer or Contributor depending on need.
- Ensure SP can read datasets, reports, and generate embed tokens.

---

## 5. Configure Fabric / Power BI Workspaces

### 5.1 Decide Tenant Isolation Model
Choose one:
- **One workspace per customer tenant** (strong isolation), or
- **Shared workspace with RLS** (scalable).

### 5.2 Upload Reports and Datasets
- Deploy the relevant datasets, semantic models, and reports.

### 5.3 Apply RLS (If Using Shared Workspace)
- Configure row-level security rules based on tenant ID.
- Ensure SP can enforce RLS.

---

## 6. Implement Analytics Retrieval in Contoso360

### 6.1 Detect Request for Analytics
- When user clicks “Analytics”, backend determines:
  - Tenant
  - Workspace ID
  - Dataset ID
  - Report ID
  - RLS filters

### 6.2 Authenticate Backend to Power BI
- Use SP credentials to:
  - Request embed token
  - Retrieve report metadata
  - Enforce RLS filters

### 6.3 Return Embed Configuration to UI
- Provide UI with:
  - Embed token
  - Report ID
  - Workspace ID
  - Visual configuration

UI renders report seamlessly inside Contoso360.

---

## 7. Test End-to-End Flow

### 7.1 Test External Authentication
- Validate Keycloak login.
- Confirm Contoso360 session creation.

### 7.2 Test Workspace Routing
- Ensure user sees correct tenant workspace or filtered reports.

### 7.3 Test Service Principal Access
- Validate token generation and report retrieval.

### 7.4 Validate RLS and Permissions
- Confirm data visibility matches tenant rules.

---

## 8. Operational Management

### 8.1 Certificate/Secret Rotation
- Rotate SP credentials periodically.
- Keep Key Vault updated.

### 8.2 Tenant Onboarding Automation
- Automate:
  - Workspace creation
  - SP assignment
  - Report publishing
  - Tenant mapping inside Contoso360

### 8.3 Logging and Monitoring
- Log Keycloak authentication events.
- Track SP API usage.
- Enable audit logs for data access.

### 8.4 Security Controls
- Least-privilege SP permissions.
- No external identities in Azure.
- RLS and workspace boundaries enforce multi-tenancy.

---

## 9. Summary

External customers:
- Authenticate via Keycloak.
- Access all analytics through Contoso360.
- Never interface with Azure Entra or Power BI directly.

Contoso360:
- Controls all access, RBAC, and tenant separation.
- Uses one Service Principal to securely retrieve Power BI content.

Fabric:
- Acts as the analytics engine.
- Enforces workspace-level or RLS-based tenant isolation.

This architecture provides a secure, scalable, multi-tenant analytics delivery model without requiring federation or guest identities.

