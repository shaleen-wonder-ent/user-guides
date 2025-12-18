# Managing B2B Guest Users & Groups with Microsoft Graph API in Azure Entra ID

This guide describes **how to set up automation for managing B2B guest users (create, delete, disable) and group membership** in Azure Entra, including all required API permissions, administrator roles, and step-by-step instructions.

---

## 1. Overview

**Typical automation needs:**
- Create (invite) guest/B2B users (e.g., onboard Keycloak users for Power BI)
- Delete users (offboard user accounts)
- Disable (block sign-in) users
- Assign/remove group memberships (for role-based access control)

**Best-practice approach:**  
Create an app registration in Azure Entra ID and use Microsoft Graph API with the minimum required **Application permissions**.

---

## 2. Minimum Permissions Needed

To support all of the above actions, your app requires:

| Functionality        | Microsoft Graph Permission    | Type         |
|--------------------- |----------------------------- |--------------|
| Invite guests        | `User.Invite.All`            | Application  |
| Create/Update/Delete | `User.ReadWrite.All`         | Application  |
| Add/Remove from group| `GroupMember.ReadWrite.All`  | Application  |

> **All permissions should be added as 'Application' type (not 'Delegated').**

---

## 3. Who Can Grant These Permissions?

- Only an **Azure Entra Global Administrator** or **Privileged Role Administrator** can grant admin consent for these "Application" permissions to your app.

---

## 4. Step-By-Step: App Registration and Permission Assignment

### A. Register an App in Azure Entra

1. Go to [Azure Portal](https://portal.azure.com)
2. Navigate to **Microsoft Entra ID** → **App registrations**
3. Click **+ New registration**
4. Name (e.g.): `contoso-provisioner`
5. Supported account types: `Accounts in this organizational directory only`
6. Leave Redirect URI blank (unless your workflow requires it)
7. Click **Register**

---

### B. Add a Client Secret

1. In your app registration, go to **Certificates & secrets**
2. Click **+ New client secret**
3. Provide a description, choose an expiry, and click **Add**
4. **Copy the secret value immediately**—you can't retrieve it later

---

### C. Grant API Permissions

1. In the app, select **API permissions** → **+ Add a permission**
2. Choose **Microsoft Graph** → **Application permissions**
3. Add:
    - `User.Invite.All`
    - `User.ReadWrite.All`
    - `GroupMember.ReadWrite.All`
4. Click **Add permissions**
5. On the API permissions page, click **Grant admin consent for [Org]** and confirm

---

### D. Record Application Information

1. Under **Overview**, copy these for your backend code:
   - **Application (client) ID**
   - **Directory (tenant) ID**
   - (Keep your client secret from step B safe!)

---

## 5. Operations Now Possible With These Permissions

You can now fully automate, via MS Graph API:

- **Invite guest users:** `POST /invitations`
- **Disable/block users:** `PATCH /users/{id}` with `{ "accountEnabled": false }`
- **Delete users:** `DELETE /users/{id}`
- **Add user to group:** `POST /groups/{group-id}/members/$ref`
- **Remove user from group:** `DELETE /groups/{group-id}/members/{directoryObject-id}/$ref`

---

### Microsoft Docs Links (API References):

- [Register an app (MS Learn)](https://learn.microsoft.com/en-us/azure/active-directory/develop/quickstart-register-app)
- [Add/update API permissions](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/configure-permissions-to-call-graph-api)
- [Consent to app permissions](https://learn.microsoft.com/en-us/azure/active-directory/manage-apps/grant-admin-consent)
- [Invite guest user](https://learn.microsoft.com/en-us/graph/api/invitation-post)
- [Update user (disable)](https://learn.microsoft.com/en-us/graph/api/user-update)
- [Delete user](https://learn.microsoft.com/en-us/graph/api/user-delete)
- [Add member to group](https://learn.microsoft.com/en-us/graph/api/group-post-members)
- [Remove member from group](https://learn.microsoft.com/en-us/graph/api/group-delete-members)
- [Microsoft Graph permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference)

---

## 6. Security Best Practices

- **Principle of Least Privilege:** Only grant Directory.ReadWrite.All if absolutely required. The above three permissions cover 99% of use cases for user/group provisioning.
- **Use Application Permissions** for all backend/automation scenarios (never Delegated).
- **Rotate client secrets** regularly. Store secrets in a secure location (e.g., Azure Key Vault).
- **Remove permissions or re-consent** if business needs change.
- **Audit app/service principal activity** through Entra logs.

---

## 7. Summary Table

| Operation                | Required Permission (Application) | Consent Required By             |
|--------------------------|-----------------------------------|---------------------------------|
| Invite guest user        | User.Invite.All                   | Global/Privileged Admin         |
| Update/Delete/Disable    | User.ReadWrite.All                | Global/Privileged Admin         |
| Add/Remove to group      | GroupMember.ReadWrite.All         | Global/Privileged Admin         |

---

## 8. Example Code References

**Invite a Guest:**
```http
POST https://graph.microsoft.com/v1.0/invitations
{
  "invitedUserEmailAddress": "user@example.com",
  "inviteRedirectUrl": "https://your-app-url",
  "sendInvitationMessage": false
}
```

**Block/Disable a User:**
```http
PATCH https://graph.microsoft.com/v1.0/users/{id}
{
  "accountEnabled": false
}
```

**Delete a User:**
```http
DELETE https://graph.microsoft.com/v1.0/users/{id}
```

**Add a User to a Group:**
```http
POST https://graph.microsoft.com/v1.0/groups/{group-id}/members/$ref
Content-Type: application/json
{
  "@odata.id": "https://graph.microsoft.com/v1.0/users/{user-id}"
}
```

**Remove a User from a Group:**
```http
DELETE https://graph.microsoft.com/v1.0/groups/{group-id}/members/{user-id}/$ref
```

---

## 9. FAQ

**Q: Does the app need a Directory role?**  
A: No, it only needs the permissions above with admin consent.

**Q: Can I restrict this app to only certain users or groups?**  
A: You can scope its usage through Conditional Access or App Role Assignments, but with these permissions, the app can manage all user/group objects in the tenant.

**Q: Can I use Delegated permissions instead?**  
A: For backend automation/scenarios with no user login context, use "Application" permissions.

---

## 10. Additional References

- [Graph API Application vs Delegated permissions](https://learn.microsoft.com/en-us/azure/active-directory/develop/v2-permissions-and-consent)
- [Secure application model (best practices)](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-add-app-roles-in-azure-ad-apps)
- [Azure Key Vault for secrets](https://learn.microsoft.com/en-us/azure/key-vault/general/basic-concepts)

---

_This guide ensures your app has exactly what it needs—and nothing more—to securely provision, deprovision, disable, and group-manage Entra ID users and guests for business automation scenarios like “Contoso Application ↔️ Keycloak ↔️ Entra ↔️ Power BI”._
