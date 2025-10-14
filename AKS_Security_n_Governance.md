# Azure Kubernetes Service (AKS) Security Best Practices Guide
## Vulnerability and Compliance Management for Production Deployments


---

## Table of Contents
1. [Summary](#executive-summary)
2. [Quick Reference: Security Best Practices](#quick-reference-security-best-practices)
3. [Container Image Security](#container-image-security)
4. [AKS Cluster Security (Nodes & Host)](#aks-cluster-security-nodes--host)
5. [Runtime Security & Compliance](#runtime-security--compliance)
6. [Secrets & Configuration Management](#secrets--configuration-management)
7. [Compliance & Governance](#compliance--governance)
8. [Microsoft Defender for Cloud Integration](#microsoft-defender-for-cloud-integration)
9. [Pre-Production Implementation Checklist](#pre-production-implementation-checklist)
10. [Reference Architecture](#reference-architecture)
11. [Microsoft Resources](#microsoft-resources)


---


This guide provides comprehensive best practices and recommendations for securing Azure Kubernetes Service (AKS) deployments in production environments. It covers security across all layers:
- **Container Images**: Vulnerability scanning, hardening, and registry security
- **Hosts & Nodes**: OS security, patching, and configuration
- **Runtime**: Threat detection and compliance monitoring
- **Governance**: Policy enforcement and compliance frameworks

---

## Quick Reference: Security Best Practices

### Cluster and Node Security (Host / Worker / System Nodes)

| Area | Best Practices | Tools & Services |
|------|----------------|------------------|
| **Node OS Hardening** | - Use **Azure-managed node pools** (node images maintained by Microsoft)<br>- Keep nodes updated by enabling **automatic node image upgrades**<br>- Enable **Azure Disk Encryption** for node disks<br>- Use VM sizes with **secure boot and vTPM** support | - AKS Node Auto-Upgrade<br>- Azure Disk Encryption<br>- Microsoft Defender for Containers |
| **Cluster Configuration** | - Use **private clusters** to avoid exposing API server publicly<br>- Restrict access with **Azure RBAC for Kubernetes**<br>- Enable **audit logs** and **diagnostic settings**<br>- Disable local accounts (enforce Azure AD only)<br>- Configure **API server authorized IP ranges** | - Azure Private Link<br>- Azure AD Integration<br>- Azure Monitor<br>- Log Analytics Workspace |
| **Network Security** | - Use **Azure CNI + Network Policies (Calico/Azure)** for traffic control<br>- Integrate with **Azure Firewall or NVA** for egress control<br>- Use **Private Link** for AKS API Server<br>- Implement **NSG rules** for node subnets | - Azure CNI<br>- Calico/Azure Network Policies<br>- Azure Firewall<br>- Private Link |

### Container and Image Security

| Area | Best Practices | Tools & Services |
|------|----------------|------------------|
| **Image Scanning** | - Store all images in **Azure Container Registry (ACR)**<br>- Enable **Microsoft Defender for Containers** for continuous image scanning (detect CVEs)<br>- Implement **CI/CD pipeline scanning** before deployment<br>- Enable **image quarantine** workflows | - Azure Container Registry<br>- Microsoft Defender for Containers<br>- ACR Tasks<br>- Azure DevOps/GitHub Actions |
| **Image Source Control** | - Use only **trusted base images** (Microsoft, Red Hat UBI, Alpine, distroless)<br>- Sign images using **Notary v2 / cosign** for provenance<br>- Regularly rebuild images to patch vulnerabilities<br>- Enable **Content Trust** in ACR | - Docker Content Trust<br>- Notary v2<br>- Cosign<br>- ACR Content Trust |
| **Runtime Protection** | - Use **read-only root filesystem**, disable privilege escalation<br>- Drop unnecessary Linux capabilities<br>- Enforce **Pod Security Standards (Baseline / Restricted)**<br>- Run containers as **non-root user** | - Pod Security Standards<br>- Azure Policy for AKS<br>- OPA Gatekeeper<br>- Kyverno |

### Compliance & Governance

| Area | Best Practices | Tools & Services |
|------|----------------|------------------|
| **Compliance Baselines** | - Map your AKS environment to **CIS Benchmark for Kubernetes** and **Azure Security Benchmark**<br>- Use **Defender for Cloud regulatory compliance dashboard**<br>- Implement continuous compliance monitoring | - Microsoft Defender for Cloud<br>- CIS Kubernetes Benchmark<br>- Azure Security Benchmark |
| **Policy Enforcement** | - Apply **Azure Policy for AKS** for continuous compliance<br>- Enforce allowed container registries, disallow privileged containers<br>- Enable **Policy Add-on for AKS**<br>- Implement automated remediation | - Azure Policy<br>- Policy Add-on for AKS<br>- Azure Policy Initiatives |
| **Auditing & Monitoring** | - Enable **Azure Monitor for Containers** for performance and behavior monitoring<br>- Integrate with **Log Analytics Workspace**<br>- Configure **custom alerts** for security events<br>- Enable **Microsoft Sentinel** for SIEM capabilities | - Azure Monitor<br>- Log Analytics Workspace<br>- Container Insights<br>- Microsoft Sentinel |

---

## Container Image Security

### Image Scanning & Vulnerability Management

#### Microsoft Defender for Containers
Enable Microsoft Defender for Containers for comprehensive vulnerability scanning:
- **Automatic CVE scanning** of container images in Azure Container Registry (ACR)
- **Severity ratings** and remediation guidance
- **Continuous scanning** even after deployment
- **Integration** with Microsoft Defender for Cloud for centralized management
- **Real-time alerts** for newly discovered vulnerabilities

**Configuration**
```bash
# Enable Microsoft Defender for Containers
az security pricing create \
  --name Containers \
  --tier standard

# Enable for Container Registry
az security pricing create \
  --name ContainerRegistry \
  --tier standard
```

#### Azure Container Registry (ACR) Best Practices
-  **Use Private Registries**: Deploy ACR instead of public registries
-  **Enable Content Trust**: Implement Docker Content Trust for image signing
-  **Image Quarantine**: Use quarantine patterns to prevent vulnerable images from deployment
-  **ACR Tasks**: Automate image building with integrated security scanning
-  **Geo-Replication**: Enable for high availability and compliance with data residency
-  **Access Control**: Use Azure RBAC and repository-scoped tokens
-  **Vulnerability Assessment**: Enable vulnerability scanning in ACR

**Enable ACR Vulnerability Scanning**
```bash
# Enable Defender for Container Registries
az acr update \
  --name myRegistry \
  --resource-group myResourceGroup \
  --default-action Deny

# Configure network rules
az acr network-rule add \
  --name myRegistry \
  --ip-address <your-ip-address>
```

### Image Hardening

#### Base Image Selection
```dockerfile
# Recommended: Use minimal base images
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine
# Or distroless images
FROM gcr.io/distroless/nodejs:18
# Or scratch for static binaries
FROM scratch
```

#### Security Best Practices
-  Use minimal base images (distroless, Alpine, or scratch)
-  Don't run containers as root user
-  Remove unnecessary packages, tools, and shells
-  Regularly update base images and dependencies
-  Use multi-stage builds to minimize attack surface
-  Scan dependencies for known vulnerabilities
-  Implement image build pipeline with automated security gates

#### Example: Secure Dockerfile
```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
# Create non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
WORKDIR /app
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .
# Switch to non-root user
USER nodejs
EXPOSE 3000
CMD ["node", "server.js"]
```

### CI/CD Pipeline Integration

**Example: Azure DevOps Pipeline with Security Scanning**
```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

steps:
- task: Docker@2
  inputs:
    command: build
    repository: myapp
    dockerfile: '**/Dockerfile'
    tags: |
      $(Build.BuildId)

- task: AzureCLI@2
  displayName: 'Scan Image with Defender'
  inputs:
    azureSubscription: 'MyAzureSubscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      # Push to ACR
      az acr login --name myregistry
      docker push myregistry.azurecr.io/myapp:$(Build.BuildId)
      
      # Wait for scan results
      az security assessment list \
        --resource-id /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ContainerRegistry/registries/myregistry
```

---

## AKS Cluster Security (Nodes & Host)

### Node Security

#### Automatic Security Patching
AKS automatically applies security patches to Linux nodes:
- **Automatic OS Updates**: Enable node image auto-upgrade
- **Update Channels**: Configure appropriate update channel (patch, stable, rapid, node-image)
- **Maintenance Windows**: Schedule maintenance windows for controlled upgrades
- **Kured**: Consider deploying Kured for automatic node reboots after updates

**Configuration Example**
```bash
# Enable automatic node OS upgrades
az aks update --resource-group myResourceGroup \
  --name myAKSCluster \
  --auto-upgrade-channel node-image

# Configure maintenance window
az aks maintenanceconfiguration add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name default \
  --weekday Monday \
  --start-hour 1
```

#### Node Pool Best Practices
-  **Separate System and User Node Pools**: Isolate system workloads
-  **Dedicated Node Pools**: Create pools for different sensitivity levels
-  **Azure Disk Encryption**: Enable encryption for node OS disks
-  **Secure Boot & vTPM**: Use VM sizes that support secure boot and vTPM
-  **Taints and Tolerations**: Control workload placement
-  **Node Isolation**: Use Azure Policy to enforce node isolation requirements

**Example: Create Secure Node Pool**
```bash
# Create system node pool
az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name systempool \
  --node-count 3 \
  --mode System \
  --node-vm-size Standard_D4s_v5 \
  --enable-encryption-at-host \
  --os-sku AzureLinux

# Create user node pool with taints
az aks nodepool add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name userpool \
  --node-count 3 \
  --mode User \
  --node-taints workload=production:NoSchedule \
  --enable-encryption-at-host
```

### Cluster Configuration

#### Core Security Features
```bash
# Create a secure AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-managed-identity \
  --enable-azure-rbac \
  --enable-aad \
  --disable-local-accounts \
  --enable-private-cluster \
  --network-plugin azure \
  --network-policy azure \
  --api-server-authorized-ip-ranges "YOUR_IP_RANGE" \
  --enable-defender \
  --enable-workload-identity \
  --enable-oidc-issuer \
  --node-os-upgrade-channel NodeImage
```

#### Key Security Configurations
1. **Azure Policy for AKS**: Enforce organizational standards
2. **Azure RBAC Integration**: Use Azure AD for cluster access control
3. **Audit Logging**: Enable and send to Log Analytics workspace
4. **Disable Local Accounts**: Force Azure AD authentication only
5. **API Server Authorized IP Ranges**: Restrict access to known IPs
6. **Private Clusters**: Use private endpoints for API server
7. **Managed Identity**: Use managed identities instead of service principals

#### Network Security
-  **Azure CNI**: Use Azure CNI for better network integration
-  **Network Policies**: Implement Calico or Azure Network Policies
-  **Azure Firewall/NVA**: Control egress traffic
-  **Private Link**: Secure connections to Azure services
-  **Service Mesh**: Consider Azure Service Mesh or Istio for advanced traffic management

**Example: Network Policy Configuration**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
```

---

## Runtime Security & Compliance

### Microsoft Defender for Cloud

**Enable Defender for Containers** for comprehensive protection:

**Capabilities**
-  Runtime threat detection
-  Kubernetes audit log analysis
-  Host-level threat detection (on nodes)
-  Behavioral analytics and anomaly detection
-  Compliance dashboard with regulatory standards
-  Security recommendations and remediation steps
-  Attack path analysis

**Configuration**
```bash
# Enable Microsoft Defender for Containers at subscription level
az security pricing create \
  --name Containers \
  --tier standard

# Enable Defender profile on AKS cluster
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-defender
```

### Pod Security

#### Pod Security Standards
Implement pod security using Azure Policy or built-in admission controllers:

**Three Security Levels**
1. **Privileged**: Unrestricted policy (for system workloads only)
2. **Baseline**: Minimally restrictive, prevents known privilege escalations
3. **Restricted**: Heavily restricted, follows pod hardening best practices

**Example: Azure Policy for Pod Security**
```bash
# Assign built-in policy initiative for pod security baseline
az policy assignment create \
  --name "pod-security-baseline" \
  --scope "/subscriptions/{subscription-id}/resourceGroups/{rg-name}" \
  --policy-set-definition "a8640138-9b0a-4a28-b8cb-1666c838647d"
```

**Key Pod Security Controls**
-  Restrict privileged containers
-  Enforce read-only root filesystems
-  Limit container capabilities (drop ALL, add specific only)
-  Control host networking and port access
-  Prevent privilege escalation
-  Enforce running as non-root user
-  Restrict volume types
-  Control host path mounts

**Example: Secure Pod Specification**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myregistry.azurecr.io/myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
        add:
        - NET_BIND_SERVICE
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## Secrets & Configuration Management

### Azure Key Vault Integration

**Use Secrets Store CSI Driver**
```bash
# Enable Secrets Store CSI Driver
az aks enable-addons \
  --addons azure-keyvault-secrets-provider \
  --name myAKSCluster \
  --resource-group myResourceGroup
```

**Best Practices**
-  **Never store secrets in images or code repositories**
-  **Use Azure Key Vault** for all sensitive data
-  **Rotate secrets regularly** (automated rotation)
-  **Use unique secrets per environment**
-  **Audit secret access** via Key Vault logs
-  **Implement least privilege** for secret access

**Example: SecretProviderClass**
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets
  namespace: production
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    tenantId: "YOUR_TENANT_ID"
    keyvaultName: "YOUR_KEYVAULT_NAME"
    objects: |
      array:
        - |
          objectName: database-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: api-key
          objectType: secret
          objectVersion: ""
    tenantId: "YOUR_TENANT_ID"
```

**Example: Pod Using Secrets**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  serviceAccountName: workload-identity-sa
  containers:
  - name: app
    image: myregistry.azurecr.io/myapp:v1
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets-store"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "azure-keyvault-secrets"
```

### Workload Identity

**Use Azure AD Workload Identity** (replaces pod-managed identity)
-  More secure than pod-managed identity
-  Based on industry standards (OIDC)
-  Better integration with Azure services
-  No infrastructure management required

```bash
# Enable workload identity on cluster
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-oidc-issuer \
  --enable-workload-identity

# Create Azure AD application and service account
export USER_ASSIGNED_IDENTITY_NAME="myIdentity"
export FEDERATED_IDENTITY_CREDENTIAL_NAME="myFedIdentity"
export SERVICE_ACCOUNT_NAME="workload-identity-sa"
export SERVICE_ACCOUNT_NAMESPACE="production"

# Create managed identity
az identity create \
  --name "${USER_ASSIGNED_IDENTITY_NAME}" \
  --resource-group myResourceGroup \
  --location eastus

# Get OIDC issuer URL
export AKS_OIDC_ISSUER="$(az aks show -n myAKSCluster -g myResourceGroup --query "oidcIssuerProfile.issuerUrl" -otsv)"

# Create federated identity credential
az identity federated-credential create \
  --name ${FEDERATED_IDENTITY_CREDENTIAL_NAME} \
  --identity-name ${USER_ASSIGNED_IDENTITY_NAME} \
  --resource-group myResourceGroup \
  --issuer ${AKS_OIDC_ISSUER} \
  --subject system:serviceaccount:${SERVICE_ACCOUNT_NAMESPACE}:${SERVICE_ACCOUNT_NAME}
```

**Service Account Configuration**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    azure.workload.identity/client-id: <WORKLOAD_IDENTITY_CLIENT_ID>
  name: workload-identity-sa
  namespace: production
```

---

## Compliance & Governance

### Compliance Frameworks

Microsoft Defender for Cloud provides built-in compliance assessments:

**Supported Frameworks**
-  **CIS Kubernetes Benchmark** v1.6.1
-  **PCI-DSS** v3.2.1 & v4.0
-  **ISO 27001:2013**
-  **NIST SP 800-53** Rev. 4 & Rev. 5
-  **SOC 2** Type 2
-  **HIPAA/HITRUST**
-  **FedRAMP** High
-  **Azure Security Benchmark**

### Key Governance Tools

#### 1. Azure Policy
**Purpose**: Enforce compliance at deployment time
- Define allowed resources and configurations
- Prevent non-compliant resources from being created
- Automatic remediation of non-compliant resources

**Recommended Built-in Initiatives**
- Kubernetes cluster pod security standards
- Container registry should be encrypted
- Azure Kubernetes Service Private Clusters
- Configure AKS clusters to enable Defender profile

**Example: Assign Policy Initiative**
```bash
# Assign AKS security baseline policy initiative
az policy assignment create \
  --name 'aks-security-baseline' \
  --display-name 'AKS Security Baseline' \
  --scope '/subscriptions/{subscription-id}/resourceGroups/{rg-name}' \
  --policy-set-definition '/providers/Microsoft.Authorization/policySetDefinitions/a8640138-9b0a-4a28-b8cb-1666c838647d'
```

#### 2. Microsoft Defender for Cloud
**Purpose**: Continuous compliance monitoring
- Real-time compliance score
- Regulatory compliance dashboard
- Security recommendations with prioritization
- Automated remediation options

**View Compliance Dashboard**
```bash
# Get compliance results
az security regulatory-compliance-standards list
az security regulatory-compliance-assessments list \
  --standard-name "CIS-1.4.0"
```

#### 3. Azure Monitor & Container Insights
**Purpose**: Observability and alerting
```bash
# Enable Container Insights
az aks enable-addons \
  --addons monitoring \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --workspace-resource-id "/subscriptions/{subscription-id}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{workspace}"
```

**Monitor**
- Node and pod performance
- Container logs
- Resource utilization
- Security events
- Custom metrics and alerts

**Example: Create Security Alert**
```bash
# Create alert rule for failed pod security policy violations
az monitor metrics alert create \
  --name 'pod-security-violations' \
  --resource-group myResourceGroup \
  --scopes /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.ContainerService/managedClusters/myAKSCluster \
  --condition "count > 0" \
  --description "Alert on pod security policy violations"
```

#### 4. Azure Advisor
**Purpose**: Security recommendations and optimization
- Cost optimization suggestions
- Performance improvements
- Security best practices
- Reliability enhancements

---

## Microsoft Defender for Cloud Integration

**Microsoft Defender for Containers** (previously Defender for Kubernetes + ACR) provides end-to-end protection:

### Key Capabilities
- **Image scanning** (ACR, CI/CD pipelines)
- **Runtime threat detection** (AKS workloads)
- **Kubernetes audit log analysis**
- **Compliance posture management** (CIS, NIST, ISO)
- **Vulnerability assessment** for running containers
- **Attack path analysis**

### Enable at Subscription Level
```bash
# Enable Defender for Containers
az security pricing create -n Containers --tier standard

# Enable Defender for Container Registry
az security pricing create -n ContainerRegistry --tier standard

# Verify enablement
az security pricing show -n Containers
az security pricing show -n ContainerRegistry
```

### Defender Profile on AKS
```bash
# Enable Defender profile on existing cluster
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-defender

# Verify Defender profile status
az aks show \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --query 'securityProfile.defender'
```

### View Security Alerts
```bash
# List security alerts for AKS
az security alert list \
  --resource-group myResourceGroup \
  --query "[?contains(resourceIdentifiers[0].azureResourceId, 'Microsoft.ContainerService')]"
```

---

## Pre-Production Implementation Checklist

Use this checklist before moving to production:

### Infrastructure Security
- [ ] Enable **Microsoft Defender for Containers**
- [ ] Enable **Microsoft Defender for Cloud** at subscription level
- [ ] Enable **Azure Policy for AKS**
- [ ] Configure **Azure AD integration** for cluster RBAC
- [ ] Disable local accounts (enforce Azure AD only)
- [ ] Configure **API server authorized IP ranges** or enable private cluster
- [ ] Set up **private ACR** with vulnerability scanning enabled
- [ ] Enable **Azure Disk Encryption** for node disks
- [ ] Configure **node image auto-upgrade** channel

### Network Security
- [ ] Implement **network policies** (Calico or Azure)
- [ ] Configure **Azure Firewall** or NVA for egress control
- [ ] Set up **Private Link** connections to Azure services
- [ ] Deploy **Web Application Firewall** (WAF) for ingress
- [ ] Review and restrict **NSG rules**
- [ ] Configure **Azure CNI** networking
- [ ] Implement **service mesh** (if required)

### Monitoring & Logging
- [ ] Enable **audit logging** to Log Analytics workspace
- [ ] Configure **Container Insights**
- [ ] Set up **custom alerts** for security events
- [ ] Enable **diagnostic logs** for ACR, Key Vault, and AKS
- [ ] Configure **log retention** per compliance requirements
- [ ] Set up **Microsoft Sentinel** for SIEM (if required)
- [ ] Configure **Prometheus and Grafana** for metrics

### Node & Workload Security
- [ ] Configure **node auto-upgrade** with appropriate channel
- [ ] Separate **system and user node pools**
- [ ] Implement **workload identity**
- [ ] Set up **Azure Key Vault** integration via CSI driver
- [ ] Apply **pod security standards** (baseline or restricted)
- [ ] Configure **resource quotas** and **limit ranges**
- [ ] Implement **taints and tolerations** for workload isolation
- [ ] Deploy **Kured** for automatic node reboots

### Image Security
- [ ] Enable **vulnerability scanning** in ACR
- [ ] Implement **image signing** with Content Trust
- [ ] Configure **image quarantine** workflow
- [ ] Set up **automated image builds** with ACR Tasks
- [ ] Document **approved base images** list
- [ ] Implement **CI/CD security gates**
- [ ] Configure **container image retention** policies

### Compliance & Governance
- [ ] Assign **Azure Policy initiatives** for compliance frameworks
- [ ] Review **compliance dashboard** in Defender for Cloud
- [ ] Document **security baseline** per Azure Security Benchmark
- [ ] Configure **backup and disaster recovery**
- [ ] Establish **incident response procedures**
- [ ] Define **security contact** and notification settings
- [ ] Conduct **CIS Kubernetes Benchmark** assessment
- [ ] Document **data residency and sovereignty** requirements

### Documentation & Training
- [ ] Document **architecture and security controls**
- [ ] Create **runbooks** for common security scenarios
- [ ] Train team on **security best practices**
- [ ] Document **access control procedures**
- [ ] Create **disaster recovery plan**
- [ ] Document **escalation procedures**
- [ ] Create **security incident playbooks**

---

## Reference Architecture

### High-Level Security Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure Subscription                       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         Microsoft Defender for Cloud                     │   │
│  │  (Vulnerability Management & Compliance Monitoring)      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Azure Policy                          │   │
│  │        (Governance & Compliance Enforcement)             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │            Azure Container Registry (ACR)                │   │
│  │  ┌────────────────────────────────────────────────┐      │   │
│  │  │ • Vulnerability Scanning                       │      │   │
│  │  │ • Content Trust (Image Signing)                │      │   │
│  │  │ • Private Endpoint                             │      │   │
│  │  │ • Geo-Replication                              │      │   │
│  │  └────────────────────────────────────────────────┘      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           Azure Kubernetes Service (AKS)                 │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  Control Plane (Microsoft Managed)                  │ │   │
│  │  │  • Private API Server                               │ │   │
│  │  │  • Azure AD Integration                             │ │   │
│  │  │  • Audit Logging → Log Analytics                    │ │   │
│  │  │  • Authorized IP Ranges                             │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  System Node Pool                                   │ │   │
│  │  │  • Automatic OS updates                             │ │   │
│  │  │  • Defender for Containers agent                    │ │   │
│  │  │  • Azure Monitor agent                              │ │   │
│  │  │  • Disk encryption enabled                          │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  │                                                          │   │
│  │  ┌─────────────────────────────────────────────────────┐ │   │
│  │  │  User Node Pools                                    │ │   │
│  │  │  ┌───────────────────────────────────────────────┐  │ │   │
│  │  │  │  Pods with Security Controls:                 │  │ │   │
│  │  │  │  • Network Policies (Calico/Azure)            │  │ │   │
│  │  │  │  • Pod Security Standards (Restricted)        │  │ │   │
│  │  │  │  • Workload Identity (OIDC)                   │  │ │   │
│  │  │  │  • Secrets Store CSI Driver                   │  │ │   │
│  │  │  │  • Read-only root filesystem                  │  │ │   │
│  │  │  │  • Non-root user execution                    │  │ │   │
│  │  │  └───────────────────────────────────────────────┘  │ │   │
│  │  └─────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Azure Key Vault                             │   │
│  │  • Secrets, Keys, Certificates                           │   │
│  │  • Private Endpoint                                      │   │
│  │  • Audit Logging                                         │   │
│  │  • Automated secret rotation                             │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │         Azure Firewall / Network Virtual Appliance       │   │
│  │  • Egress Traffic Control                                │   │
│  │  • Threat Intelligence                                   │   │
│  │  • Application rules (FQDN filtering)                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Log Analytics Workspace                     │   │
│  │  • Centralized Logging                                   │   │
│  │  • Security Analytics                                    │   │
│  │  • Custom Alerts                                         │   │
│  │  • Retention policies                                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              Microsoft Sentinel (Optional)               │   │
│  │  • SIEM and SOAR capabilities                            │   │
│  │  • Threat hunting                                        │   │
│  │  • Automated response                                    │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### CI/CD Security Flow

```
[ Developers ]
     |
     v
[ Git Repository (GitHub/Azure DevOps) ]
     |
     v
[ CI/CD Pipeline ]
     |
     ├─> Dependency Scanning (Dependabot/Snyk)
     ├─> SAST (Static Analysis)
     ├─> Container Image Build (Multi-stage)
     |
     v
[ Azure Container Registry (ACR) ]
     |
     ├─> Microsoft Defender Scanning
     ├─> Vulnerability Assessment
     ├─> Image Signing (Content Trust)
     |
     v
[ Quarantine/Approval Gate ]
     |
     v
[ AKS Cluster (Private) ]
     |
     ├─> Admission Controller (Azure Policy/OPA)
     ├─> Pod Security Standards
     ├─> Network Policies
     ├─> Runtime Protection (Defender)
     |
     v
[ Production Workload ]
     |
     v
[ Continuous Monitoring ]
   - Azure Monitor
   - Defender for Cloud
   - Log Analytics
   - Microsoft Sentinel
```

---

## Microsoft Resources

### Official Documentation
| Resource | URL |
|----------|-----|
| **AKS Security Best Practices** | https://learn.microsoft.com/azure/aks/security-best-practices |
| **AKS Secure Baseline** | https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks/secure-baseline-aks |
| **Best practices for cluster security** | https://learn.microsoft.com/azure/aks/operator-best-practices-cluster-security |
| **Microsoft Defender for Containers** | https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction |
| **Azure Policy for Kubernetes** | https://learn.microsoft.com/azure/governance/policy/concepts/policy-for-kubernetes |
| **Azure Policy samples for AKS** | https://learn.microsoft.com/azure/governance/policy/samples/aks |
| **AKS Security Baseline** | https://learn.microsoft.com/security/benchmark/azure/baselines/aks-security-baseline |
| **Container Security** | https://learn.microsoft.com/azure/container-registry/container-registry-best-practices |
| **Workload Identity** | https://learn.microsoft.com/azure/aks/workload-identity-overview |
| **CIS Kubernetes Benchmark** | https://www.cisecurity.org/benchmark/kubernetes |
| **Pod Security Standards** | https://learn.microsoft.com/azure/aks/use-pod-security-on-azure-policy |

### Learning Paths (Search at Microsoft documentation as links sometime changes)
- **Secure AKS workloads** (Microsoft Learn)
- **Introduction to Kubernetes security** (Microsoft Learn)
- **AKS Well-Architected Framework** (Security Pillar)
- **Implement Azure Kubernetes Service (AKS)** (Microsoft Learn)

---


## Appendix: Quick Command Reference

### Cluster Creation and Configuration
```bash
# Create secure AKS cluster
az aks create \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-managed-identity \
  --enable-azure-rbac \
  --enable-aad \
  --disable-local-accounts \
  --enable-private-cluster \
  --network-plugin azure \
  --network-policy azure \
  --enable-defender \
  --enable-workload-identity \
  --enable-oidc-issuer \
  --node-os-upgrade-channel NodeImage

# Enable add-ons
az aks enable-addons \
  --addons monitoring,azure-keyvault-secrets-provider \
  --name myAKSCluster \
  --resource-group myResourceGroup \
  --workspace-resource-id <LOG_ANALYTICS_WORKSPACE_ID>
```

### Security Configuration
```bash
# Enable Defender for Cloud
az security pricing create -n Containers --tier standard
az security pricing create -n ContainerRegistry --tier standard

# Assign Azure Policy
az policy assignment create \
  --name 'aks-security-baseline' \
  --scope '/subscriptions/{sub-id}/resourceGroups/{rg}' \
  --policy-set-definition 'a8640138-9b0a-4a28-b8cb-1666c838647d'

# Configure maintenance window
az aks maintenanceconfiguration add \
  --resource-group myResourceGroup \
  --cluster-name myAKSCluster \
  --name default \
  --weekday Monday \
  --start-hour 1
```

### Monitoring and Diagnostics
```bash
# View security alerts
az security alert list

# Get compliance status
az security regulatory-compliance-assessments list \
  --standard-name "CIS-1.4.0"

# View cluster diagnostics
az aks show \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --query '{defender: securityProfile.defender, workloadIdentity: securityProfile.workloadIdentity}'
```


