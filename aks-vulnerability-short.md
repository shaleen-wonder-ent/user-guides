# AKS Vulnerability & Compliance Guidance – Customer Insights

## 1️⃣ Best Practices on Vulnerability Management for AKS

| Component | Best Practices |
|-----------|----------------|
| **System / Worker Nodes** | - Use **Microsoft-managed node pools** for automatic OS patching. <br> - Regularly perform **node image upgrades** (`az aks nodepool upgrade --node-image-only`). <br> - Harden nodes using **Azure Linux (Mariner) or Ubuntu CIS-hardened images**. <br> - Restrict network access and enforce RBAC. |
| **Containers** | - Enforce **Pod Security Standards (Baseline / Restricted)**. <br> - Run containers with **read-only root filesystem**, drop unnecessary capabilities. <br> - Avoid privileged containers and limit host access. |
| **Images** | - Use **trusted base images** (Microsoft, Red Hat UBI, Alpine). <br> - Enable **vulnerability scanning** in CI/CD pipelines (ACR Tasks, GitHub Actions, Azure DevOps). <br> - Sign images using **Notary v2 / Cosign**. <br> - Regularly rebuild images to include patched base images. |

## 2️⃣ Customer Approaches for Adopting AKS in Production

- Many customers **start with non-mission-critical applications** to test AKS in production.  
- Some adopt **mission-critical apps** directly but in a phased rollout approach with pilot workloads.  
- **Fallback / DR mechanisms:**
  - Some maintain a **parallel VM-based environment** for quick rollback.  
  - Others rely on **multi-cluster deployments** (production + staging) and Kubernetes-native **blue-green / canary deployments** for failover.  
- Key focus: start small, validate security, scale progressively.

## 3️⃣ Handling Vulnerability or Compliance Issues in Production

- **Verify if node images are up-to-date**. Upgrade system and user node pools as needed.  
- **Check CVEs against Microsoft Security Response Center (MSRC)**.  
- **Report unpatched vulnerabilities to Microsoft** if using managed node pools.  
- **For critical workloads:** apply **network isolation, Pod Security Standards, and runtime monitoring** to mitigate risks while awaiting patches.  
- **Document and track** all findings in vulnerability management systems.  

## 4️⃣ Acceptable Vulnerability Categories

- **Industry practice varies:**
  - Some organizations aim for **zero vulnerabilities**, especially for regulated workloads (finance, healthcare).  
  - Others accept **low/medium vulnerabilities** if they are **non-exploitable, backported, or mitigated**, particularly in managed node pools.  
- Key observation: **system node pools are typically treated as “vendor-managed”**, and only actionable CVEs are escalated.  
- Customers often define a **risk-based acceptance threshold** in their vulnerability management policy.

## 5️⃣ Addressing Vulnerability / Compliance Issues in System Nodes

- **Managed system node pools:** manual patching is **not possible**.  
  - Upgrade to latest node image regularly.  
  - Validate reported vulnerabilities against **MSRC** and Microsoft guidance.  
  - Document as **“Managed by Microsoft – no customer action required”** in Prisma or other scanners.  
- **Custom/self-managed node pools:** rebuild and patch the node images, enforce hardened OS configuration.  
- Use **Azure Policy** and **Defender for Cloud** to enforce compliance and reduce risk.

## 6️⃣ Advantages of Using Microsoft Defender for AKS

| Feature | Advantage |
|---------|-----------|
| **Azure-native integration** | Automatic awareness of AKS clusters, node pools, and workloads. No extra agent needed for managed nodes. |
| **Managed node image awareness** | Reduces false positives on system nodes; understands Microsoft patching cycle. |
| **Continuous image & runtime protection** | Integrated with ACR, CI/CD pipelines, and runtime threat detection for AKS workloads. |
| **Compliance & benchmark alignment** | Maps to **CIS, Azure Security Benchmark, ISO, NIST** automatically. Audit-ready reporting. |
| **Operational simplicity** | Single pane of glass via Defender for Cloud; integrates with Azure Monitor and Sentinel for alerting and SIEM. |
| **Cost/efficiency** | Part of Defender for Cloud subscription; reduces agent overhead and operational effort. |

**Observation:** For Azure-only workloads, Microsoft Defender often provides **more accurate vulnerability reporting, lower operational noise, and better compliance alignment** compared to third-party scanners.

## References

- [AKS Security Baseline](https://learn.microsoft.com/azure/architecture/reference-architectures/containers/aks/secure-baseline-aks)  
- [Best practices for cluster security and upgrades](https://learn.microsoft.com/azure/aks/operator-best-practices-cluster-security)  
- [Microsoft Defender for Containers overview](https://learn.microsoft.com/azure/defender-for-cloud/defender-for-containers-introduction)  
- [Azure Policy samples for AKS](https://learn.microsoft.com/azure/governance/policy/samples/aks)  
- [AKS Node Image Upgrade](https://learn.microsoft.com/en-us/azure/aks/node-image-upgrade)
