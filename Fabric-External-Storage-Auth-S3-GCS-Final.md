# Microsoft Fabric ↔ External Object Storage (AWS S3 & Google Cloud Storage)
## Supported Authentication Methods

**Topic:** Connectivity from Microsoft Fabric to AWS S3 and Google Cloud Storage (GCS)

**Scope:** OneLake shortcuts (in-place, read-only access) and Data Factory pipeline Copy activity (read/write data movement)

**Audience:** Data engineering, platform, and cloud architecture teams enabling multi-cloud connectivity in Microsoft Fabric


> **Status note:** The methods below reflect generally available (GA) Microsoft Fabric capabilities at the time of writing. Feature lifecycle status can change; please confirm against the latest [Microsoft Fabric release notes](https://learn.microsoft.com/fabric/release-plan/) and connector documentation before relying on any feature for production.

---

## 1. Overview

Microsoft Fabric connects to external object storage through two paths, each with distinct capabilities and authentication support:

| Connectivity path | Purpose | Access mode |
|---|---|---|
| **OneLake shortcut** | In-place reference/virtualization of external data in a Lakehouse (no data movement). Appears as folders Fabric compute can read directly. Supports some advanced identity integrations (e.g., Microsoft Entra OIDC for S3). | **Read-only** |
| **Data Factory pipeline (Copy activity / Copy job)** | Moves data into or out of Fabric. Relies on explicit credentials (keys/secrets) for external storage. | **Read and write** (source/destination) |

> **Important:** The Amazon S3 and Google Cloud Storage connectors are **not** available in **Dataflow Gen2** as of GA. Use **pipeline Copy** (or **OneLake shortcuts**) for S3/GCS connectivity in Fabric.

---

## 2. AWS S3 → Microsoft Fabric

AWS S3 (native) supports **two** authentication methods, depending on the connectivity path.

### 2.1 AWS Access Key (IAM user credentials) — supported for shortcuts **and** pipelines

- **Usage:** Provide an **AWS Access Key ID** and **Secret Access Key** (permanent credentials of an AWS IAM user). Configured when creating an S3 shortcut in a Lakehouse, or when setting up an S3 connection in Data Factory. Fabric stores these securely and uses them to sign S3 API requests (AWS Signature v4) for list/read/write operations.
- **Required IAM permissions:**
  - Read-only (shortcut): `s3:GetObject`, `s3:ListBucket`, `s3:GetBucketLocation`
  - Write (pipeline sink): add `s3:PutObject` (and `s3:DeleteObject` if deleting)
  - SSE-KMS buckets: also grant KMS encrypt/decrypt on the key.
- **Security:** Treat keys like passwords — store in Fabric's secure connections (or linked Key Vault), use a dedicated least-privilege IAM user, and rotate periodically. This method involves managing long-lived secrets.

### 2.2 Microsoft Entra ID Service Principal → AWS IAM Role (OIDC federated identity) — **OneLake S3 shortcuts only**

> Not available for the Data Factory S3 pipeline connector as of GA. For S3 writes or pipeline scenarios, use the Access Key method (2.1).

- **Usage:** Instead of static IAM user keys, the S3 shortcut uses a **Microsoft Entra service principal** to **assume an AWS IAM role via OpenID Connect (OIDC)**. Fabric obtains a short-lived token from AWS STS using an ID token from Microsoft Entra ID. The acting identity is the Entra application (service principal).
- **Setup (both Entra ID and AWS IAM):**
  1. **Microsoft Entra ID:** Register an app (service principal), create a client secret, and note the **Tenant ID**, **Client (Application) ID**, and **Object ID**.
  2. **AWS IAM:** Create an **OIDC identity provider** — Provider URL `https://sts.windows.net/<tenant-id>/`, Audience `https://analysis.windows.net/powerbi/connector/AmazonS3`. Then create an **IAM role** trusting that provider, with a trust policy scoped to the SPN's **Object ID** (`sts.windows.net/<tenant-id>/:sub`) and audience (`:aud`). Attach an S3 access policy to the role.
  3. **Fabric:** When creating the S3 shortcut, select the **Service Principal / RoleARN** option and provide the **Role ARN**, **Tenant ID**, **Service Principal (Client) ID**, and **Client Secret**. Fabric performs the OIDC flow automatically.
- **Benefits:** No long-lived AWS keys stored in Fabric; access via temporary STS credentials; unified identity management in Entra ID; full auditability via AWS CloudTrail (role-assumption events).
- **Current limitation:** Only the **Service Principal → IAM Role** method is supported. Direct user OAuth and Fabric Workspace (managed) identity to AWS are **not** supported.

### 2.3 AWS S3-compatible (non-AWS) endpoints

- Only **Access Key / Secret** is supported. Microsoft Entra OAuth, Service Principal, and RoleARN are **not** supported for non-AWS S3-compatible endpoints.

### 2.4 Network (AWS S3)

- S3 must be reachable over its HTTPS endpoint (e.g., `https://<bucket>.s3.<region>.amazonaws.com`).
- You do **not** need to make the bucket public or change S3 **Block Public Access** — Fabric authenticates each request with your credentials.
- For private buckets (VPC/firewall), use a Microsoft **On-Premises Data Gateway** (or managed VNet integration) to bridge connectivity.

---

## 3. Google Cloud Storage (GCS) → Microsoft Fabric

GCS uses **HMAC keys** as the only supported authentication method — for **both** shortcuts and pipelines.

### 3.1 HMAC access key (service account credentials) — "Basic" auth

- **Usage:** In Google Cloud, generate **HMAC keys** for a **service account** (preferred) or user, yielding an **Access Key ID** and **Secret**. In Fabric, supply them as a **Basic** pair — **Username = Access ID**, **Password = Secret**. Works for OneLake GCS shortcuts (read-only) and Data Factory pipelines (read/write).
- **GCP prerequisites:**
  1. Enable **Interoperability** on the project (allows HMAC keys).
  2. Use a **service account** with minimal Cloud Storage roles for the target data.
  3. **Generate HMAC credentials** for that service account.
- **Required permissions:** `storage.objects.get`, `storage.objects.list` (read); add `storage.objects.create` (and `storage.objects.delete` if deleting) for pipeline writes. If using the **global endpoint** (`https://storage.googleapis.com`), also `storage.buckets.list`.
- **Not supported:** Google OAuth (user sign-in), service account **JSON key file** authentication, and **Workload Identity Federation / ADC** are **not** available in Fabric as of GA. Only HMAC keys (via the GCS S3-compatible XML API) are supported.
- **Security:** Treat HMAC credentials as sensitive secrets. Use a dedicated least-privilege service account (e.g., **Storage Object Viewer** for read; **Storage Object Admin** or a custom role for write — avoid broad **Storage Admin**). Store in Fabric's secure connection store and rotate periodically.

### 3.2 Network (GCS)

- Fabric accesses GCS via public REST endpoints (`https://storage.googleapis.com` or `https://<bucket>.storage.googleapis.com`). Ensure connectivity is not blocked; for network-restricted buckets, use a secure data gateway / private networking.

---

## 4. Summary of GA Authentication Support

| External storage | OneLake shortcut (read-only) | Data Factory pipeline (read/write) | Notes & limitations |
|---|---|---|---|
| **AWS S3 (native)** | **Access Key (IAM user)** *or* **Entra SPN → IAM Role (OIDC)** | **Access Key (IAM user)** only | SPN/OIDC integration is shortcut-only. Pipelines require Access Key. |
| **AWS S3-compatible (non-AWS)** | **Access Key** only | **Access Key** only | No Entra/OAuth/RoleARN support. |
| **Google Cloud Storage** | **HMAC key (service account)** | **HMAC key (service account)** | Shortcuts read-only; pipelines read/write. HMAC only (no OAuth/JSON key/WIF). |

> Neither S3 nor GCS is available in **Dataflow Gen2** — use pipeline Copy / Copy job or OneLake shortcuts.

---

## 5. Security & Best Practices

- **Least-privilege identities:** Dedicated IAM users / GCP service accounts scoped to only the required buckets and actions; avoid privileged or shared credentials.
- **Prefer federated identity where available:** For native AWS S3 shortcuts, the **Entra SPN → IAM Role** model avoids long-lived keys, centralizes identity lifecycle in Entra ID, and provides CloudTrail auditing.
- **Secure secret management:** Store keys/secrets in Fabric's secure connection store (or linked Key Vault). Never embed credentials in notebooks or source code.
- **Rotation & monitoring:** Rotate AWS access keys and GCS HMAC keys on a schedule. Monitor usage (AWS CloudTrail for key/STS activity; GCP Cloud Monitoring for service-account usage). Remember shortcuts use one **delegated credential** for all workspace users — audit accordingly.

---

## 6. References — Official Microsoft Documentation

**Amazon S3**
- Set up your Amazon S3 connection (pipeline; Access Key): https://learn.microsoft.com/fabric/data-factory/connector-amazon-s3
- Create an Amazon S3 shortcut (OneLake): https://learn.microsoft.com/fabric/onelake/create-s3-shortcut
- Integrate Microsoft Entra with AWS S3 shortcuts using service principal (OIDC/RoleARN): https://learn.microsoft.com/fabric/onelake/amazon-storage-shortcut-entra-integration
- Amazon S3 connector overview (capabilities/limits): https://learn.microsoft.com/fabric/data-factory/connector-amazon-s3-overview

**Google Cloud Storage**
- Set up your Google Cloud Storage connection (pipeline; HMAC/Basic): https://learn.microsoft.com/fabric/data-factory/connector-google-cloud-storage
- Create a Google Cloud Storage (GCS) shortcut (OneLake): https://learn.microsoft.com/fabric/onelake/create-gcs-shortcut
- Google Cloud Storage connector overview (capabilities/limits): https://learn.microsoft.com/fabric/data-factory/connector-google-cloud-storage-overview
