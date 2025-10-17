# Azure Application Gateway Architecture - Public & Private with WAF and Header-Based Filtering

## Overview
This document provides architecture diagrams for Azure Application Gateway configured with both public and private frontend IPs, WAF protection, and header-based routing capabilities.


<img width="536" height="586" alt="image" src="https://github.com/user-attachments/assets/44b9c35c-8e25-46e6-97e3-bb803011a195" />

---

<img width="543" height="244" alt="image" src="https://github.com/user-attachments/assets/8d3e5a3d-cd18-461a-b049-59a89b7bf970" />


---


## Key Differences, between the tiers

| Feature                | Basic      | Standard v2 | WAF v2 |
|------------------------|-----------|-------------|--------|
| **Autoscaling**        | No        | Yes          | Yes     |
| **Zone redundancy**    | No        | Yes          | Yes     |
| **Private Link support**| No       | Yes          | Yes     |
| **Web Application Firewall (WAF)**| No | **No**      | Yes     |
| **Recommended for new deployments?** | No | Yes   | Yes     |


---

## Architecture Diagram 1: High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          INTERNET                                       │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 │ HTTPS/HTTP
                                 ▼
                    ┌────────────────────────┐
                    │   Public IP Address    │
                    │   (Internet-facing)    │
                    └────────────┬───────────┘
                                 │
┌────────────────────────────────┼─────────────────────────────────────────┐
│  Azure Virtual Network         │                                         │
│                                ▼                                         │
│              ┌─────────────────────────────────────┐                     │
│              │  Azure Application Gateway v2       │                     │
│              │  ┌───────────────────────────────┐  │                     │
│              │  │   WAF (Web Application        │  │                     │
│              │  │   Firewall) - OWASP Rules     │  │                     │
│              │  └───────────────────────────────┘  │                     │
│              │                                     │                     │
│              │  ┌───────────────────────────────┐  │                     │
│              │  │   Frontend IP Configuration   │  │                     │
│              │  │   • Public IP: 20.x.x.x       │  │                     │
│              │  │   • Private IP: 10.0.1.10     │  │                     │
│              │  └───────────────────────────────┘  │                     │
│              │                                     │                     │
│              │  ┌───────────────────────────────┐  │                     │
│              │  │   HTTP Listeners              │  │                     │
│              │  │   • Public Listener :443      │  │                     │
│              │  │   • Private Listener :443     │  │                     │
│              │  └───────────────────────────────┘  │                     │
│              │                                     │                     │
│              │  ┌───────────────────────────────┐  │                     │
│              │  │   Routing Rules               │  │                     │
│              │  │   • Path-based routing        │  │                     │
│              │  │   • Header-based routing      │  │                     │
│              │  └───────────────────────────────┘  │                     │
│              │                                     │                     │
│              │  ┌───────────────────────────────┐  │                     │
│              │  │   Rewrite Rules               │  │                     │
│              │  │   • Header manipulation       │  │                     │
│              │  │   • URL rewriting             │  │                     │
│              │  └───────────────────────────────┘  │                     │
│              └──────────┬──────────────┬───────────┘                     │
│                         │              │                                 │
│        ┌────────────────┘              └──────────────┐                  │
│        │                                              │                  │
│        ▼                                              ▼                  │
│  ┌──────────────┐                            ┌──────────────┐            │
│  │ Backend Pool │                            │ Backend Pool │            │
│  │      #1      │                            │      #2      │            │
│  │              │                            │              │            │
│  │ • Web App 1  │                            │ • API App 1  │            │
│  │ • Web App 2  │                            │ • API App 2  │            │
│  └──────────────┘                            └──────────────┘            │
│                                                                          │
│                    ┌────────────────────┐                                │
│                    │  Private Endpoint  │                                │
│                    │  (to other Azure   │                                │
│                    │   services)        │                                │
│                    └────────────────────┘                                │
└──────────────────────────────────────────────────────────────────────────┘
                                 ▲
                                 │
                    ┌────────────┴───────────┐
                    │  Corporate Network     │
                    │  (ExpressRoute/VPN)    │
                    │  via Private IP        │
                    └────────────────────────┘
```

---

## Architecture Diagram 2: Header-Based Routing Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Traffic Flow with Headers                         │
└─────────────────────────────────────────────────────────────────────────┘

PUBLIC REQUEST                           PRIVATE REQUEST
(Internet User)                          (Corporate User)
      │                                        │
      │ GET /api/users                        │ GET /api/users
      │ Host: api.contoso.com                 │ Host: internal.contoso.local
      │ X-Client-Type: mobile                 │ X-Auth-Token: internal-xyz
      │ X-API-Version: v2                     │ X-Department: Engineering
      ▼                                        ▼
┌─────────────┐                          ┌─────────────┐
│  Public IP  │                          │ Private IP  │
│ 20.x.x.x    │                          │ 10.0.1.10   │
└──────┬──────┘                          └──────┬──────┘
       │                                        │
       └────────────────┬───────────────────────┘
                        ▼
            ┌───────────────────────┐
            │   WAF Inspection      │
            │   • SQL Injection     │
            │   • XSS Protection    │
            │   • Custom Rules      │
            └───────────┬───────────┘
                        ▼
            ┌───────────────────────────────────┐
            │   HTTP Listener Processing        │
            │   • SSL Termination               │
            │   • Host Header Check             │
            └───────────┬───────────────────────┘
                        ▼
            ┌───────────────────────────────────┐
            │   Header-Based Routing Logic      │
            ├───────────────────────────────────┤
            │  IF X-Client-Type = "mobile"      │
            │    → Route to Mobile Backend      │
            │                                   │
            │  IF X-API-Version = "v2"          │
            │    → Route to V2 API Backend      │
            │                                   │
            │  IF X-Auth-Token exists           │
            │    → Route to Internal Backend    │
            │                                   │
            │  IF X-Department = "Engineering"  │
            │    → Add custom headers           │
            │    → Route to Engineering Pool    │
            └───────────┬───────────────────────┘
                        ▼
            ┌───────────────────────┐
            │   Rewrite Rules       │
            │   • Add headers       │
            │   • Remove headers    │
            │   • Modify values     │
            └───────────┬───────────┘
                        ▼
        ┌───────────────┴────────────────┐
        ▼                                ▼
┌───────────────┐                ┌───────────────┐
│ Mobile App    │                │ Internal API  │
│ Backend Pool  │                │ Backend Pool  │
│               │                │               │
│ • Optimized   │                │ • Corporate   │
│   for mobile  │                │   services    │
└───────────────┘                └───────────────┘
```

---

## Architecture Diagram 3: Detailed Component Configuration

```
┌────────────────────────────────────────────────────────────-─────────────┐
│                  Azure Application Gateway Subnet                        │
│                         (10.0.1.0/24)                                    │
│                                                                          │
│  ┌──────────────────────────────────────────────────────-──────────┐     │
│  │             Application Gateway Instance                        │     │
│  │                                                                 │     │
│  │  ┌──────────────────────────────────────────────-───────────┐   │     │
│  │  │  FRONTEND CONFIGURATION                                  │   │     │
│  │  ├───────────────────────────────────────────────-──────────┤   │     │
│  │  │  Frontend IP #1 (Public)                                 │   │     │
│  │  │  • Public IP: 20.x.x.x                                   │   │     │
│  │  │  • Port: 443 (HTTPS), 80 (HTTP)                          │   │     │
│  │  │                                                          │   │     │
│  │  │  Frontend IP #2 (Private)                                │   │     │
│  │  │  • Private IP: 10.0.1.10                                 │   │     │
│  │  │  • Port: 443 (HTTPS), 80 (HTTP)                          │   │     │
│  │  └──────────────────────────────────────────────────────────┘   │     │
│  │                                                                 │     │
│  │  ┌──────────────────────────────────────────────────────────┐   │     │
│  │  │  WAF POLICY                                              │   │     │
│  │  ├──────────────────────────────────────────────────────────┤   │     │
│  │  │  Mode: Prevention                                        │   │     │
│  │  │  Rule Set: OWASP 3.2                                     │   │     │
│  │  │                                                          │   │     │
│  │  │  Custom Rules:                                           │   │     │
│  │  │  ┌─────────────────────────────────────────┐             │   │     │
│  │  │  │ Rule 1: Block if missing X-API-Key      │             │   │     │
│  │  │  │ Priority: 100                           │             │   │     │
│  │  │  │ Condition: RequestHeader X-API-Key      │             │   │     │
│  │  │  │            Does Not Exist               │             │   │     │
│  │  │  │ Action: Block                           │             │   │     │
│  │  │  └─────────────────────────────────────────┘             │   │     │
│  │  │                                                          │   │     │
│  │  │  ┌─────────────────────────────────────────┐             │   │     │
│  │  │  │ Rule 2: Rate limit by Client-ID header  │             │   │     │
│  │  │  │ Priority: 200                           │             │   │     │
│  │  │  │ Condition: RequestHeader X-Client-ID    │             │   │     │
│  │  │  │ Action: Rate Limit (100 req/min)        │             │   │     │
│  │  │  └─────────────────────────────────────────┘             │   │     │
│  │  └──────────────────────────────────────────────────────────┘   │     │
│  │                                                                 │     │
│  │  ┌──────────────────────────────────────────────────────────┐   │     │
│  │  │  LISTENERS                                               │   │     │
│  │  ├──────────────────────────────────────────────────────────┤   │     │
│  │  │  Listener 1: "PublicHTTPSListener"                       │   │     │
│  │  │  • Frontend IP: Public (20.x.x.x)                        │   │     │
│  │  │  • Port: 443                                             │   │     │
│  │  │  • Protocol: HTTPS                                       │   │     │
│  │  │  • SSL Certificate: *.contoso.com                        │   │     │
│  │  │  • Host Name: api.contoso.com                            │   │     │
│  │  │                                                          │   │     │
│  │  │  Listener 2: "PrivateHTTPSListener"                      │   │     │
│  │  │  • Frontend IP: Private (10.0.1.10)                      │   │     │
│  │  │  • Port: 443                                             │   │     │
│  │  │  • Protocol: HTTPS                                       │   │     │
│  │  │  • SSL Certificate: *.internal.contoso.local             │   │     │
│  │  │  • Host Name: internal.contoso.local                     │   │     │
│  │  └──────────────────────────────────────────────────────────┘   │     │
│  │                                                                 │     │
│  │  ┌─────────────────────────────────────────────────────────┐    │     │
│  │  │  ROUTING RULES                                          │    │     │
│  │  ├─────────────────────────────────────────────────────────┤    │     │
│  │  │  Rule 1: "HeaderBasedRouting-Mobile"                    │    │     │
│  │  │  • Listener: PublicHTTPSListener                        │    │     │
│  │  │  • Condition: X-Client-Type = "mobile"                  │    │     │
│  │  │  • Backend Pool: MobileBackendPool                      │    │     │
│  │  │  • HTTP Settings: HTTPSSettings-8443                    │    │     │
│  │  │                                                         │    │     │
│  │  │  Rule 2: "HeaderBasedRouting-Internal"                  │    │     │
│  │  │  • Listener: PrivateHTTPSListener                       │    │     │
│  │  │  • Condition: X-Auth-Token exists                       │    │     │
│  │  │  • Backend Pool: InternalBackendPool                    │    │     │
│  │  │  • HTTP Settings: HTTPSSettings-443                     │    │     │
│  │  │  • Rewrite Set: AddInternalHeaders                      │    │     │
│  │  │                                                         │    │     │
│  │  │  Rule 3: "PathAndHeaderRouting-API"                     │    │     │
│  │  │  • Listener: PublicHTTPSListener                        │    │     │
│  │  │  • Path: /api/v2/*                                      │    │     │
│  │  │  • Condition: X-API-Version = "v2"                      │    │     │
│  │  │  • Backend Pool: APIv2BackendPool                       │    │     │
│  │  │  • HTTP Settings: HTTPSSettings-443                     │    │     │
│  │  └─────────────────────────────────────────────────────────┘    │     │
│  │                                                                 │     │
│  │  ┌─────────────────────────────────────────────────────────┐    │     │
│  │  │  REWRITE RULES                                          │    │     │
│  │  ├─────────────────────────────────────────────────────────┤    │     │
│  │  │  Rewrite Set: "AddInternalHeaders"                      │    │     │
│  │  │  • Add X-Forwarded-For header                           │    │     │
│  │  │  • Add X-Internal-Request: true                         │    │     │
│  │  │  • Remove X-Auth-Token (don't pass to backend)          │    │     │
│  │  │  • Add X-AppGateway-Id: {gateway-id}                    │    │     │
│  │  │                                                         │    │     │
│  │  │  Rewrite Set: "SecurityHeaders"                         │    │     │
│  │  │  • Add Strict-Transport-Security header                 │    │     │
│  │  │  • Add X-Content-Type-Options: nosniff                  │    │     │
│  │  │  • Add X-Frame-Options: DENY                            │    │     │
│  │  └─────────────────────────────────────────────────────────┘    │     │
│  │                                                                 │     │
│  │  ┌─────────────────────────────────────────────────────────┐    │     │
│  │  │  BACKEND POOLS                                          │    │     │
│  │  ├─────────────────────────────────────────────────────────┤    │     │
│  │  │  MobileBackendPool                                      │    │     │
│  │  │  • Target: 10.0.10.10 (App Service)                     │    │     │
│  │  │  • Target: 10.0.10.11 (App Service)                     │    │     │
│  │  │                                                         │    │     │
│  │  │  InternalBackendPool                                    │    │     │
│  │  │  • Target: 10.0.20.10 (Internal API VM)                 │    │     │
│  │  │  • Target: 10.0.20.11 (Internal API VM)                 │    │     │
│  │  │                                                         │    │     │
│  │  │  APIv2BackendPool                                       │    │     │
│  │  │  • Target: privatelink-api.azurewebsites.net            │    │     │
│  │  │    (Private Endpoint)                                   │    │     │
│  │  └─────────────────────────────────────────────────────────┘    │     │
│  │                                                                 │     │
│  │  ┌─────────────────────────────────────────────────────────┐    │     │
│  │  │  HTTP SETTINGS                                          │    │     │
│  │  ├─────────────────────────────────────────────────────────┤    │     │
│  │  │  HTTPSSettings-443                                      │    │     │
│  │  │  • Protocol: HTTPS                                      │    │     │
│  │  │  • Port: 443                                            │    │     │
│  │  │  • Cookie-based affinity: Disabled                      │    │     │
│  │  │  • Request timeout: 30 seconds                          │    │     │
│  │  │  • Override hostname: Yes                               │    │     │
│  │  │                                                         │    │     │
│  │  │  HTTPSSettings-8443                                     │    │     │
│  │  │  • Protocol: HTTPS                                      │    │     │
│  │  │  • Port: 8443                                           │    │     │
│  │  │  • Cookie-based affinity: Enabled                       │    │     │
│  │  │  • Request timeout: 60 seconds                          │    │     │
│  │  └─────────────────────────────────────────────────────────┘    │     │
│  └─────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture Diagram 4: Security and Traffic Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         SECURITY LAYERS                                  │
└──────────────────────────────────────────────────────────────────────────┘

LAYER 1: Network Security
┌─────────────────────────────────────────────────────────────────────────┐
│  • NSG (Network Security Group) on App Gateway Subnet                   │
│    - Allow inbound 443, 80 from Internet (for Public IP)                │
│    - Allow inbound 443, 80 from Corporate Network (for Private IP)      │
│    - Allow inbound 65200-65535 (Gateway Manager ports)                  │
│  • Azure DDoS Protection Standard (optional)                            │
└────────────────────────────────┬────────────────────────────────────────┘
                                 ▼
LAYER 2: WAF (Web Application Firewall)
┌─────────────────────────────────────────────────────────────────────────┐
│  • OWASP ModSecurity Core Rule Set 3.2                                  │
│    - SQL Injection protection                                           │
│    - Cross-Site Scripting (XSS) protection                              │
│    - Local File Inclusion (LFI)                                         │
│    - Remote File Inclusion (RFI)                                        │
│                                                                         │
│  • Custom WAF Rules (Header-Based)                                      │
│    ┌──────────────────────────────────────────┐                         │
│    │ IF RequestHeader "X-API-Key" is missing  │                         │
│    │    → BLOCK REQUEST                       │                         │
│    ├──────────────────────────────────────────┤                         │
│    │ IF RequestHeader "X-Client-Type"         │                         │
│    │    contains "malicious-bot"              │                         │
│    │    → BLOCK REQUEST                       │                         │
│    ├──────────────────────────────────────────┤                         │
│    │ IF RequestHeader "X-Forwarded-For"       │                         │
│    │    from blacklisted IP                   │                         │
│    │    → BLOCK REQUEST                       │                         │
│    └──────────────────────────────────────────┘                         │
└────────────────────────────────┬────────────────────────────────────────┘
                                 ▼
LAYER 3: SSL/TLS Termination
┌─────────────────────────────────────────────────────────────────────────┐
│  • TLS 1.2 / TLS 1.3 only                                               │
│  • Strong cipher suites                                                 │
│  • Certificate validation                                               │
│  • Mutual TLS (mTLS) support (optional)                                 │
└────────────────────────────────┬────────────────────────────────────────┘
                                 ▼
LAYER 4: Header Validation & Routing
┌─────────────────────────────────────────────────────────────────────────┐
│  • Validate required headers                                            │
│  • Route based on header values                                         │
│  • Apply header-based security policies                                 │
│  • Rate limiting per header value (e.g., X-Client-ID)                   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 ▼
LAYER 5: Backend Communication
┌─────────────────────────────────────────────────────────────────────────┐
│  • End-to-end SSL (App Gateway → Backend)                               │
│  • Health probes with custom headers                                    │
│  • Connection draining                                                  │
│  • Session affinity (if required)                                       │
└─────────────────────────────────────────────────────────────────────────┘


TRAFFIC FLOW EXAMPLE - Header-Based Filtering
═══════════════════════════════════════════════

Public User Request:
━━━━━━━━━━━━━━━━━━━
GET /api/data HTTP/1.1
Host: api.contoso.com
X-API-Key: abc123-valid-key
X-Client-Type: web-browser
X-API-Version: v2

         ↓ [NSG: ALLOW]
         ↓ [WAF: Check X-API-Key exists → PASS]
         ↓ [WAF: Check OWASP rules → PASS]
         ↓ [SSL: Validate & Decrypt]
         ↓ [Listener: Match api.contoso.com → PublicHTTPSListener]
         ↓ [Routing: Check X-API-Version=v2 → Route to APIv2BackendPool]
         ↓ [Rewrite: Add X-Forwarded-For, X-AppGateway-Id]
         ↓ [Backend: Forward to Private Endpoint → 10.0.30.5:443]
         ↓
    [RESPONSE RETURNED]


Private User Request (Invalid):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
GET /internal/admin HTTP/1.1
Host: internal.contoso.local
X-Department: Finance

         ↓ [NSG: ALLOW]
         ↓ [WAF: Check X-API-Key exists → MISSING]
         ↓ [WAF CUSTOM RULE: BLOCK]
         ✗
    [403 FORBIDDEN RETURNED]
```

---

## Configuration Example: ARM Template Snippet

```json
{
  "type": "Microsoft.Network/applicationGateways",
  "apiVersion": "2023-05-01",
  "name": "appgw-dual-frontend",
  "properties": {
    "sku": {
      "name": "WAF_v2",
      "tier": "WAF_v2"
    },
    "frontendIPConfigurations": [
      {
        "name": "publicFrontendIP",
        "properties": {
          "publicIPAddress": {
            "id": "[resourceId('Microsoft.Network/publicIPAddresses', 'pip-appgw')]"
          }
        }
      },
      {
        "name": "privateFrontendIP",
        "properties": {
          "privateIPAddress": "10.0.1.10",
          "privateIPAllocationMethod": "Static",
          "subnet": {
            "id": "[resourceId('Microsoft.Network/virtualNetworks/subnets', 'vnet-main', 'snet-appgw')]"
          }
        }
      }
    ],
    "httpListeners": [
      {
        "name": "publicListener",
        "properties": {
          "frontendIPConfiguration": {
            "id": "[concat(resourceId('Microsoft.Network/applicationGateways', 'appgw-dual-frontend'), '/frontendIPConfigurations/publicFrontendIP')]"
          },
          "frontendPort": {
            "id": "[concat(resourceId('Microsoft.Network/applicationGateways', 'appgw-dual-frontend'), '/frontendPorts/port-443')]"
          },
          "protocol": "Https",
          "hostName": "api.contoso.com"
        }
      },
      {
        "name": "privateListener",
        "properties": {
          "frontendIPConfiguration": {
            "id": "[concat(resourceId('Microsoft.Network/applicationGateways', 'appgw-dual-frontend'), '/frontendIPConfigurations/privateFrontendIP')]"
          },
          "frontendPort": {
            "id": "[concat(resourceId('Microsoft.Network/applicationGateways', 'appgw-dual-frontend'), '/frontendPorts/port-443')]"
          },
          "protocol": "Https",
          "hostName": "internal.contoso.local"
        }
      }
    ],
    "requestRoutingRules": [
      {
        "name": "headerBasedRule",
        "properties": {
          "ruleType": "PathBasedRouting",
          "httpListener": {
            "id": "[concat(resourceId('Microsoft.Network/applicationGateways', 'appgw-dual-frontend'), '/httpListeners/publicListener')]"
          },
          "urlPathMap": {
            "id": "[concat(resourceId('Microsoft.Network/applicationGateways', 'appgw-dual-frontend'), '/urlPathMaps/pathMap-1')]"
          }
        }
      }
    ],
    "rewriteRuleSets": [
      {
        "name": "headerRewriteSet",
        "properties": {
          "rewriteRules": [
            {
              "ruleSequence": 100,
              "name": "add-internal-header",
              "conditions": [
                {
                  "variable": "http_req_X-Auth-Token",
                  "pattern": ".*",
                  "ignoreCase": true
                }
              ],
              "actionSet": {
                "requestHeaderConfigurations": [
                  {
                    "headerName": "X-Internal-Request",
                    "headerValue": "true"
                  }
                ]
              }
            }
          ]
        }
      }
    ],
    "webApplicationFirewallConfiguration": {
      "enabled": true,
      "firewallMode": "Prevention",
      "ruleSetType": "OWASP",
      "ruleSetVersion": "3.2"
    }
  }
}
```

---

## Key Considerations

### 1. **Dual Frontend IP Benefits**
- Single gateway instance for both public and private traffic
- Consistent WAF protection across both entry points
- Shared backend pools and routing logic
- Cost-effective compared to separate gateways

### 2. **Header-Based Filtering Capabilities**
- Route traffic based on any HTTP header
- Combine header-based routing with path-based routing
- Use WAF custom rules for header validation
- Implement rate limiting per header value

### 3. **Private Connectivity Options**
- **Private Frontend IP**: VNet-internal access without internet exposure
- **Private Endpoints in Backend**: App Gateway can route to Azure services via Private Endpoint
- **ExpressRoute/VPN**: Corporate users access via private IP
- **Service Endpoints**: Secure access to Azure PaaS services

### 4. **Best Practices**
- Use separate listeners for public and private frontends
- Apply WAF custom rules for header validation before routing
- Implement header-based rate limiting to prevent abuse
- Use rewrite rules to add/remove headers between App Gateway and backends
- Enable diagnostics and monitoring for both frontends
- Use Azure Key Vault for SSL certificate management

### 5. **Limitations & Considerations**
- Application Gateway requires a dedicated subnet (/24 recommended)
- Cannot have same hostname on both public and private listeners
- Header-based routing requires URL path maps with conditions
- Some header-based operations require rewrite rules
- WAF custom rules are evaluated before routing rules

---

## Monitoring and Diagnostics

```
Recommended Monitoring Setup:
├── Azure Monitor Metrics
│   ├── Throughput (by frontend IP)
│   ├── Failed Requests
│   ├── Backend Response Time
│   └── WAF Blocked Requests
│
├── Diagnostic Logs
│   ├── ApplicationGatewayAccessLog
│   ├── ApplicationGatewayPerformanceLog
│   ├── ApplicationGatewayFirewallLog
│   └── Custom header tracking in logs
│
└── Alerts
    ├── High WAF block rate
    ├── Backend health probe failures
    ├── SSL certificate expiration
    └── Abnormal traffic patterns by header
```

---

## Additional Resources

- [Azure Application Gateway documentation](https://learn.microsoft.com/azure/application-gateway/)
- [WAF Custom Rules](https://learn.microsoft.com/azure/web-application-firewall/ag/custom-waf-rules-overview)
- [Rewrite HTTP headers and URL](https://learn.microsoft.com/azure/application-gateway/rewrite-http-headers-url)
- [Multi-site hosting](https://learn.microsoft.com/azure/application-gateway/multiple-site-overview)
- [Configure Azure Application Gateway Private Link](https://learn.microsoft.com/en-us/azure/application-gateway/private-link-configure)
- [Quickstart: Direct web traffic with Azure Application Gateway - Azure portal](https://learn.microsoft.com/en-us/azure/application-gateway/quick-create-portal)
- [Application Gateway frontend IP address configuration](https://learn.microsoft.com/en-us/azure/application-gateway/configuration-frontend-ip)

---


