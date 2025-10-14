# Azure Kubernetes Service (AKS) Security Best Practices Guide
## Vulnerability and Compliance Management for Production Deployments

**Prepared For:** Production AKS Deployment Security Review

---

## 1. Container Image Security

### 1.1 Image Scanning & Vulnerability Management

#### Microsoft Defender for Containers
Enable Microsoft Defender for Containers for comprehensive vulnerability scanning:
- **Automatic CVE scanning** of container images in Azure Container Registry (ACR)
- **Severity ratings** and remediation guidance
- **Continuous scanning** even after deployment
- **Integration** with Microsoft Defender for Cloud for centralized management
- **Real-time alerts** for newly discovered vulnerabilities

#### Azure Container Registry (ACR) Best Practices
- ✅ **Use Private Registries**: Deploy ACR instead of public registries
- ✅ **Enable Content Trust**: Implement Docker Content Trust for image signing
- ✅ **Image Quarantine**: Use quarantine patterns to prevent vulnerable images from deployment
- ✅ **ACR Tasks**: Automate image building with integrated security scanning
- ✅ **Geo-Replication**: Enable for high availability and compliance with data residency
- ✅ **Access Control**: Use Azure RBAC and repository-scoped tokens
- ✅ **Vulnerability Assessment**: Enable vulnerability scanning in ACR

### 1.2 Image Hardening

**Base Image Selection**
```dockerfile
# Recommended: Use minimal base images
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine
# Or distroless images
FROM gcr.io/distroless/nodejs:18
```

**Security Best Practices**
- ✅ Use minimal base images (distroless, Alpine, or scratch)
- ✅ Don't run containers as root user
- ✅ Remove unnecessary packages, tools, and shells
- ✅ Regularly update base images and dependencies
- ✅ Use multi-stage builds to minimize attack surface
- ✅ Scan dependencies for known vulnerabilities
- ✅ Implement image build pipeline with automated security gates

**Example: Secure Dockerfile**
```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
WORKDIR /app
COPY --from=builder --chown=nodejs:nodejs /app/node_modules ./node_modules
COPY --chown=nodejs:nodejs . .
USER nodejs
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## 2. AKS Cluster Security (Nodes & Host)

### 2.1 Node Security

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
```

#### Node Pool Best Practices
- ✅ **Separate System and User Node Pools**: Isolate system workloads
- ✅ **Dedicated Node Pools**: Create pools for different sensitivity levels
- ✅ **Azure Disk Encryption**: Enable encryption for node OS disks
- ✅ **Secure Boot & vTPM**: Use VM sizes that support secure boot and vTPM
- ✅ **Taints and Tolerations**: Control workload placement
- ✅ **Node Isolation**: Use Azure Policy to enforce node isolation requirements

### 2.2 Cluster Configuration

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
  --enable-defender
```

**Key Security Configurations**
1. **Azure Policy for AKS**: Enforce organizational standards
2. **Azure RBAC Integration**: Use Azure AD for cluster access control
3. **Audit Logging**: Enable and send to Log Analytics workspace
4. **Disable Local Accounts**: Force Azure AD authentication only
5. **API Server Authorized IP Ranges**: Restrict access to known IPs
6. **Private Clusters**: Use private endpoints for API server
7. **Managed Identity**: Use managed identities instead of service principals

#### Network Security
- ✅ **Azure CNI**: Use Azure CNI for better network integration
- ✅ **Network Policies**: Implement Calico or Azure Network Policies
- ✅ **Azure Firewall/NVA**: Control egress traffic
- ✅ **Private Link**: Secure connections to Azure services
- ✅ **Service Mesh**: Consider Azure Service Mesh or Istio for advanced traffic management

---

## 3. Runtime Security & Compliance

### 3.1 Microsoft Defender for Cloud

**Enable Defender for Containers** for comprehensive protection:

**Capabilities**
- ✅ Runtime threat detection
- ✅ Kubernetes audit log analysis
- ✅ Host-level threat detection (on nodes)
- ✅ Behavioral analytics and anomaly detection
- ✅ Compliance dashboard with regulatory standards
- ✅ Security recommendations and remediation steps
- ✅ Attack path analysis

**Configuration**
```bash
# Enable Microsoft Defender for Containers
az security pricing create \
  --name Containers \
  --tier standard
```

### 3.2 Pod Security

#### Pod Security Standards
Implement pod security using Azure Policy or built-in admission controllers:

**Three Security Levels**
1. **Privileged**: Unrestricted policy (for system workloads only)
2. **Baseline**: Minimally restrictive, prevents known privilege escalations
3. **Restricted**: Heavily restricted, follows pod hardening best practices

**Example: Azure Policy for Pod Security**
```bash
# Assign built-in policy initiative
az policy assignment create \
  --name "pod-security-baseline" \
  --scope "/subscriptions/{subscription-id}/resourceGroups/{rg-name}" \
  --policy-set-definition "a8640138-9b0a-4a28-b8cb-1666c838647d"
```

**Key Pod Security Controls**
- ✅ Restrict privileged containers
- ✅ Enforce read-only root filesystems
- ✅ Limit container capabilities (drop ALL, add specific only)
- ✅ Control host networking and port access
- ✅ Prevent privilege escalation
- ✅ Enforce running as non-root user
- ✅ Restrict volume types
- ✅ Control host path mounts

### 3.3 Network Security

**Implement Defense in Depth**
```yaml
# Example: Network Policy to restrict pod communication
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
```

---

## 4. Secrets & Configuration Management

### 4.1 Azure Key Vault Integration

**Use Secrets Store CSI Driver**
```bash
# Enable Secrets Store CSI Driver
az aks enable-addons \
  --addons azure-keyvault-secrets-provider \
  --name myAKSCluster \
  --resource-group myResourceGroup
```

**Best Practices**
- ✅ **Never store secrets in images or code repositories**
- ✅ **Use Azure Key Vault** for all sensitive data
- ✅ **Rotate secrets regularly** (automated rotation)
- ✅ **Use unique secrets per environment**
- ✅ **Audit secret access** via Key Vault logs
- ✅ **Implement least privilege** for secret access

**Example: SecretProviderClass**
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault-secrets
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
```

### 4.2 Workload Identity

**Use Azure AD Workload Identity** (replaces pod-managed identity)
- ✅ More secure than pod-managed identity
- ✅ Based on industry standards (OIDC)
- ✅ Better integration with Azure services
- ✅ No infrastructure management required

```bash
# Enable workload identity
az aks update \
  --resource-group myResourceGroup \
  --name myAKSCluster \
  --enable-oidc-issuer \
  --enable-workload-identity
```

---

## 5. Compliance & Governance

### 5.1 Compliance Frameworks

Microsoft Defender for Cloud provides built-in compliance assessments:

**Supported Frameworks**
- ✅ **CIS Kubernetes Benchmark** v1.6.1
- ✅ **PCI-DSS** v3.2.1 & v4.0
- ✅ **ISO 27001:2013**
- ✅ **NIST SP 800-53** Rev. 4 & Rev. 5
- ✅ **SOC 2** Type 2
- ✅ **HIPAA/HITRUST**
- ✅ **FedRAMP** High
- ✅ **Azure Security Benchmark**

### 5.2 Key Governance Tools

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

#### 2. Microsoft Defender for Cloud
**Purpose**: Continuous compliance monitoring
- Real-time compliance score
- Regulatory compliance dashboard
- Security recommendations with prioritization
- Automated remediation options

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

#### 4. Azure Advisor
**Purpose**: Security recommendations and optimization
- Cost optimization suggestions
- Performance improvements
- Security best practices
- Reliability enhancements

---

## 6. Pre-Production Implementation Checklist

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

### Network Security
- [ ] Implement **network policies** (Calico or Azure)
- [ ] Configure **Azure Firewall** or NVA for egress control
- [ ] Set up **Private Link** connections to Azure services
- [ ] Deploy **Web Application Firewall** (WAF) for ingress
- [ ] Review and restrict **NSG rules**

### Monitoring & Logging
- [ ] Enable **audit logging** to Log Analytics workspace
- [ ] Configure **Container Insights**
- [ ] Set up **custom alerts** for security events
- [ ] Enable **diagnostic logs** for ACR, Key Vault, and AKS
- [ ] Configure **log retention** per compliance requirements

### Node & Workload Security
- [ ] Configure **node auto-upgrade** with appropriate channel
- [ ] Separate **system and user node pools**
- [ ] Implement **workload identity**
- [ ] Set up **Azure Key Vault** integration via CSI driver
- [ ] Apply **pod security standards** (baseline or restricted)
- [ ] Configure **resource quotas** and **limit ranges**

### Image Security
- [ ] Enable **vulnerability scanning** in ACR
- [ ] Implement **image signing** with Content Trust
- [ ] Configure **image quarantine** workflow
- [ ] Set up **automated image builds** with ACR Tasks
- [ ] Document **approved base images** list

### Compliance & Governance
- [ ] Assign **Azure Policy initiatives** for compliance frameworks
- [ ] Review **compliance dashboard** in Defender for Cloud
- [ ] Document **security baseline** per Azure Security Benchmark
- [ ] Configure **backup and disaster recovery**
- [ ] Establish **incident response procedures**
- [ ] Define **security contact** and notification settings

### Documentation & Training
- [ ] Document **architecture and security controls**
- [ ] Create **runbooks** for common security scenarios
- [ ] Train team on **security best practices**
- [ ] Document **access control procedures**
- [ ] Create **disaster recovery plan**

---

## 7. Microsoft Resources

### Official Documentation
| Resource | URL |
|----------|-----|
| **AKS Security Best Practices** | https://learn.microsoft.com/azure/aks/security-best-practices |
| **Microsoft Defender for Containers** | https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction |
| **Azure Policy for Kubernetes** | https://learn.microsoft.com/azure/governance/policy/concepts/policy-for-kubernetes |
| **AKS Security Baseline** | https://learn.microsoft.com/security/benchmark/azure/baselines/aks-security-baseline |
| **Container Security** | https://learn.microsoft.com/azure/container-registry/container-registry-best-practices |
| **Workload Identity** | https://learn.microsoft.com/azure/aks/workload-identity-overview |
| **CIS Kubernetes Benchmark** | https://www.cisecurity.org/benchmark/kubernetes |

### Learning Paths
- **Secure AKS workloads** (Microsoft Learn)
- **Introduction to Kubernetes security** (Microsoft Learn)
- **AKS Well-Architected Framework** (Security Pillar)

---

## 8. Engagement Options

### Direct Microsoft Support Channels

#### 1. Azure Architecture Review
- **Description**: Structured review with Microsoft architects
- **Focus**: Security, reliability, performance, cost optimization
- **Eligibility**: Available to all Azure customers
- **How to Request**: Contact your Microsoft Account Team

#### 2. Microsoft FastTrack
- **Description**: Free engineering assistance for Azure deployments
- **Focus**: Best practices, architecture guidance, deployment support
- **Eligibility**: Customers with qualifying Azure commitments
- **Website**: https://azure.microsoft.com/programs/azure-fasttrack/

#### 3. Well-Architected Framework Review
- **Description**: Comprehensive assessment across 5 pillars (Security, Reliability, Performance, Cost, Operational Excellence)
- **Focus**: Security pillar for AKS workloads
- **Deliverables**: Assessment report with prioritized recommendations
- **How to Request**: Through Microsoft Account Team or Azure portal

#### 4. Azure Security Workshop
- **Description**: Interactive workshop on Azure security capabilities
- **Topics**: Defender for Cloud, security best practices, compliance
- **Format**: Virtual or on-site
- **How to Request**: Through Microsoft Account Team

#### 5. Microsoft Technical Support
- **Professional Direct**: Unlimited 24/7 technical support
- **Unified Support**: Comprehensive support for hybrid environments
- **Premier/Unified**: Includes Technical Account Manager (TAM)

### Community Resources
- **Microsoft Tech Community** (AKS forums)
- **GitHub Issues** (AKS repository)
- **Stack Overflow** (azure-kubernetes tag)
- **Azure Security Community** (Discord/Slack)

---

## Appendix: Security Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Azure Subscription                        │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Microsoft Defender for Cloud                      │  │
│  │  (Vulnerability Management & Compliance Monitoring)       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Azure Policy                           │  │
│  │        (Governance & Compliance Enforcement)              │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │            Azure Container Registry (ACR)                 │  │
│  │  ┌────────────────────────────────────────────────┐      │  │
│  │  │ • Vulnerability Scanning                       │      │  │
│  │  │ • Content Trust (Image Signing)                │      │  │
│  │  │ • Private Endpoint                              │      │  │
│  │  └────────────────────────────────────────────────┘      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           Azure Kubernetes Service (AKS)                  │  │
│  │                                                            │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │  Control Plane (Microsoft Managed)                  │ │  │
│  │  │  • Private API Server                               │ │  │
│  │  │  • Azure AD Integration                             │ │  │
│  │  │  • Audit Logging → Log Analytics                    │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  │                                                            │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │  System Node Pool                                   │ │  │
│  │  │  • Automatic OS updates                             │ │  │
│  │  │  • Defender for Containers agent                    │ │  │
│  │  │  • Azure Monitor agent                              │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  │                                                            │  │
│  │  ┌─────────────────────────────────────────────────────┐ │  │
│  │  │  User Node Pools                                    │ │  │
│  │  │  ┌───────────────────────────────────────────────┐ │ │  │
│  │  │  │  Pods with Security Controls:                 │ │ │  │
│  │  │  │  • Network Policies                           │ │ │  │
│  │  │  │  • Pod Security Standards                     │ │ │  │
│  │  │  │  • Workload Identity                          │ │ │  │
│  │  │  │  • Secrets Store CSI Driver                   │ │ │  │
│  │  │  └───────────────────────────────────────────────┘ │ │  │
│  │  └─────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Azure Key Vault                              │  │
│  │  • Secrets, Keys, Certificates                            │  │
│  │  • Private Endpoint                                       │  │
│  │  • Audit Logging                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Azure Firewall / Network Virtual Appliance        │  │
│  │  • Egress Traffic Control                                 │  │
│  │  • Threat Intelligence                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Log Analytics Workspace                      │  │
│  │  • Centralized Logging                                    │  │
│  │  • Security Analytics                                     │  │
│  │  • Custom Alerts                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

