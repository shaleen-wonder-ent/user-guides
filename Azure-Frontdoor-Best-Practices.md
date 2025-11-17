# Azure Front Door – Best Practices, Configuration, Caching, and Logging 

This document summarizes recommended practices, configuration steps, sample scripts, and links to official Microsoft documentation for deploying Azure Front Door.

---

## 1. Overview

**Azure Front Door (AFD)** is a scalable and secure entry point for fast delivery of your global web applications. It combines load balancing, intelligent routing, WAF, and application acceleration via caching.

*References:*
- [Azure Front Door official overview](https://learn.microsoft.com/en-us/azure/frontdoor/front-door-overview)

---

## 2. General Best Practices

### 2.1 Choose the Right Global Load Balancer

- Use **Azure Front Door** for TLS termination, advanced routing, application-layer security, caching/CDN, and acceleration.  
- Use **Azure Traffic Manager** for DNS-based load balancing without HTTP(s) features.
- If combining, Front Door should sit in front of Traffic Manager for high-availability setups.
- Restrict direct origin access (block requests to app service/public IP except those routed via AFD).

*More details:*  
[Microsoft Best Practices](https://learn.microsoft.com/en-us/azure/frontdoor/best-practices)  
[Well-Architected AFD Guide](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-front-door)

---

## 3. Security & TLS

- **Enable end-to-end TLS**: Require HTTPS both between client→AFD and AFD→origin. Set TLS version to 1.2 or higher.
- Use **custom domains** and certificates managed in Azure Key Vault for automation.
- **HTTP to HTTPS redirect**: Enforce redirection using route rules.
- **Enable Web Application Firewall (WAF)**: Protect against OWASP top threats and DDoS.

*Example portal steps:*
1. Go to Front Door profile → “Frontend/domains” → Add custom domain.
2. Under “Routing rules”, enable “Redirect HTTP to HTTPS”.
3. Under “Security”, set minimum TLS version.
4. Attach or create WAF policy.

*Reference:*  
[Microsoft AFD Security Best Practices](https://learn.microsoft.com/en-us/azure/frontdoor/best-practices#security)

---

## 4. Caching Best Practices

- Set **Cache-Control headers** in app responses for precise expiration (do not rely on AFD’s 1-3 day default).
- Use **short or disabled caching** (Cache-Control: no-store) in non-prod; use longer TTLs for static assets in production.
- **Purge cache** via portal/API/CLI after content changes.

*Sample: Azure CLI for purge*
```bash
az network front-door purge-endpoint \
  --resource-group <RG> \
  --front-door-name <FDName> \
  --content-paths "/images/*"
```

*Reference:*  
[How to Avoid Caching Woes with Microsoft Azure Front Door](https://clearmeasure.com/how-to-avoid-caching-woes-with-microsoft-azure-front-door/)

---

## 5. Logging and Monitoring

- **Enable diagnostic logs**: Go to Azure Portal → Front Door → Diagnostic settings.
  - Routes logs to Log Analytics, Storage, Event Hub.
- **Log types**: WAF logs, Front Door access logs (request/response, cache status, attacker IPs, geo, latency).
- **Monitor metrics**: Request/response count, cache hit ratio, health probe status, error codes.

*Sample: Enable Diagnostic Logging via Azure CLI*
```bash
az monitor diagnostic-settings create \
  --resource <FrontDoorResourceId> \
  --workspace <LogAnalyticsWorkspaceId> \
  --logs '[{"category":"FrontdoorAccessLog","enabled":true},{"category":"FrontdoorWebApplicationFirewallLog","enabled":true}]'
```

---

## 6. Standard Practices/Architecture

- Prefer **active-active** multi-region deployments for high-availability.
- For scaling, design with stateless web services (minimize need for sticky sessions).
- Deploy and update via **ARM/Bicep templates** for repeatable, versioned infrastructure.

---

## 7. Example Step-by-Step: Minimal Setup

### a. Create Front Door (Portal Steps)
1. **Create resource** → Azure Front Door (Standard/Premium)
2. **Add frontend/domains**: Provide custom domain, enable HTTPS.
3. **Add backend pools**: Add web apps, VMs, or APIs as backends.
   - Set “backend host type” accordingly.
   - Secure origins; allow only AFD via Private Link or IP header rule.
4. **Create routing rules**: Define matching paths and origin pools.
   - Enable caching if needed.
5. **Enable diagnostic logging/WAF.**

### b. Sample ARM Snippet for Caching Rule
```json
{
  "properties": {
    "cachingEnabled": true,
    "cacheDuration": "P1D",
    "queryParameters": "*"
  }
}
```
*(Add to routingRules properties in ARM template)*

---

## 8. Additional References

- [Azure Front Door Best Practices (Microsoft)](https://learn.microsoft.com/en-us/azure/frontdoor/best-practices)
- [Azure Well-Architected AFD Guide](https://learn.microsoft.com/en-us/azure/well-architected/service-guides/azure-front-door)
- [GitHub: AFD Best Practices Doc](https://github.com/MicrosoftDocs/azure-docs/blob/main/articles/frontdoor/best-practices.md)
- [Azure Front Door Configuration: Step by Step](https://avicrowntechsolutions.com/2023/06/16/azure-front-door-configuration-step-by-step/)

---

## 9. Summary

1. Use AFD in front for acceleration, security, and global scaling.
2. Always secure origins; block direct traffic.
3. Set explicit caching headers.
4. Enable logs/diagnostics for visibility and tuning.
5. Use ARM/Bicep for infra-as-code deployments.
6. Reference Microsoft best practices regularly as features evolve.

---
